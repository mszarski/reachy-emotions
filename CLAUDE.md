# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A recreation of the Reachy Mini text-to-motion system from
[pham-tuan-binh/reachy-motion-generator](https://github.com/pham-tuan-binh/reachy-motion-generator)
(the blog post *The best expressive harness for robots*), extended into a voice loop: you talk to Reachy Mini and it
listens, reads your words and tone, and answers with a generated move. Python package `rmr` (dist name
`reachy-motion-recreation`). Split out of the `claude_playground` monorepo.

Motion pipeline: text → **planner** (LLM) → **recipe** → `rmr.recipe.variants` → **plan** (8 ch) →
**generator** (flow matching) → 25 Hz 9-DoF trajectory → `rmr.reach.Reach.project` → reachable **move** JSON.

Voice loop (one name per stage, used in code, docs and README):
- **listener** `rmr/listen.py` (`STYLE`): rules, no model, moves the head while you speak; ported to `visualizer/src/Listen.js`.
- **hearing** `rmr/voice.py`: Whisper (large-v3-turbo on GPU, small on CPU) + WavLM **tone models**, per-session `Calibration`.
- **responder** `rmr/respond.py` (`Student`; "student" versions v3 … v8-dpo, default v8-dpo): one streamed call →
  `{"feeling", "response", "recipe", "reading"}` in that order, so motion starts before the reading is written.
  `rmr/audio_responder.py` is the optional audio-native variant (Qwen2-Audio, in-process or a vLLM URL).
  `rmr/react.py` sends a quick reaction as soon as the feeling is read ('react' stage).
- Typed prompts skip hearing: the zero-shot planner (`rmr/planner/`, HF Inference Providers) writes the recipe.

README.md has the architecture, training story and status table; docs/ROADMAP.md the per-step plan and commands;
docs/results/*.md the measured results (including negative results; record new ones there).

## Commands

```bash
pip install -e ".[ik,dev]"               # core + Stewart-platform IK + pytest
pytest                                   # all tests (testpaths = tests)
pytest tests/test_recipe.py -v           # one file
```

Extras: `generator` (torch), `hf`, `planner`, `voice`, `serve` (fastapi/uvicorn), `render` (MuJoCo; also
`pip install --no-deps reachy-mini`, and headless `MUJOCO_GL=osmesa` + `libosmesa6`). Tests `importorskip` the extras
they need (torch, fastapi, mujoco, soundfile, `reachy_mini_rust_kinematics`) and skip the JS-parity tests without
`node`, so a bare install silently skips a lot: check the skip count. `.claude/hooks/session-start.sh` installs the
full set on Claude Code on the web.

```bash
python -m rmr.pipeline --prompt "sneezing" --out runs/one --ckpt hf://mszarski/reachy-motion-generator/generator_v2.pt --render
HF_TOKEN=... python -m rmr.server --ckpt hf://mszarski/reachy-motion-generator/generator_v2.pt   # viewer + API on :7860
python -m rmr.robot --server http://localhost:7860                                             # robot/simulator client
python -m http.server -d visualizer 8000                                                       # static viewer only
python -m rmr.generator train|evaluate|sample ...                                              # see docs/ROADMAP.md
```

`rmr.server` is configured by env vars as well as flags (`GENERATOR_CKPT`, `VOICE_MODEL`, `AUDIO_MODEL`,
`PLANNER_MODEL`, `RESPONDER_MODEL`, `CALIBRATE_VOICE`, `FEEDBACK_DIR`, ...; see the bottom of `rmr/server.py`).
API: `/api/respond` (WAV body, `?stream=1` → NDJSON stages), `/api/generate` (typed prompt), `/api/idle`, `/api/health`.
Model paths accept `hf://repo/file` and `repo:sub/dir` (`rmr.planner.finetune.resolve`).

## Deploy and GPU work

- `deploy/live_space.py` uploads `rmr/`, `pyproject.toml`, `visualizer/`, `deploy/Dockerfile` (or `Dockerfile.vllm`)
  and `deploy/start.sh` to the private Docker Space `mszarski/reachy-mini-motions-live` (L4, sleeps when idle).
  `start.sh` starts llama.cpp on the responder's 8-bit GGUF (`LLAMA_GGUF`) and/or vLLM FP8 for the audio responder
  (`AUDIO_VLLM`), waits for health, then runs the server. Q8_0 is the served build; Q4_K_M loses physical fit.
- `deploy/hf_job.py` runs any repo command on a Hugging Face Jobs GPU with a hard `--max-usd` cap and uploads
  `/work/out`. Use it for training and evals; record cost and scores in docs/results.
- The static viewer Space `mszarski/reachy-mini-motions` is updated by re-uploading `visualizer/`.
- All Hub repos (`mszarski/reachy-motion-generator`, `mszarski/reachy-voice`) are private: `HF_TOKEN` is required.

## Conventions

- `rmr/` modules are ports of the reference. Each docstring names the file it follows. Keep the semantics
  identical (units, channel order, limits, randomisation ranges), because the teacher data and published models
  depend on them; `tests/test_recipe.py::test_every_teacher_recipe_is_valid` guards the recipe language.
- Units: trajectories are SI (m, rad); plans and recipes are deg and mm. Plan channel order is `rmr.plan.CH`.
  The right antenna droops with negative angles, the left with positive.
- 25 Hz everywhere (`rmr.motion.FPS`). Library clips are ~100 Hz and must be resampled on their `time` array.
- `data/teacher/val.jsonl` and the held-out emotions are evaluation only: never train on them. The same goes for the
  18 held-out CREMA-D actors and the test lines used for responder scores.
- Hugging Face and GPU-dependent work: put downloads behind the `hf` extra and keep the core importable without torch.
- Licence-clean by design: the default responder uses no MELD or other non-commercial data; Seamless Interaction
  (CC-BY-NC) is optional only. Keep NOTICE up to date for anything taken from the reference or Pollen.
- Checkpoints (`*.pt`, `checkpoints/`, `runs/`) are gitignored: publish to the Hub, never commit them.
- Commit messages state the measured result (e.g. "38% vs 91%, negative result; Space stays on v2").

## Python ↔ JS parity

The browser viewer duplicates Python logic and tests run both through `node` to check they agree:
`Listen.js` ↔ `rmr/listen.py` (constants like `END_OF_TURN = 0.7`, `MIN_TURN` must change in both, and in
`rmr/robot.py`'s docstring), `ListenModel.js` ↔ `rmr/listen_model.py`, `StewartIK.js`/`Kinematics.js` ↔ the SDK's
Rust IK (to 1e-9 rad, `tests/test_viewer.py`). Change one side, change the other, run the node tests.

## Viewer (`visualizer/`)

Plain ES modules, no build step; three.js, urdf-loader and Draco load from cdn.jsdelivr.net. `index.html`/`app.js`
is the gallery + live talk UI, `puppet.html`/`puppet_app.js` the puppeteering page (face landmarker). iPhone Safari
gotchas already handled in `app.js`/`SceneManager.js`/`style.css`; keep them when editing:
- Create the `AudioContext` synchronously inside the tap, before any `await` (otherwise iOS records silence).
- Hold-to-talk: the finger can lift before the mic opens (`holding`/`opening` flags); use `pointercancel`,
  not `pointerleave`; block `contextmenu` and `-webkit-touch-callout` on the talk button.
- Inputs need `font-size: 16px` on phones (iOS zooms otherwise); use `dvh` with a `vh` fallback and
  `env(safe-area-inset-bottom)`.
- Lower pixel ratio and plain PCF shadows on touch devices (Safari drops heavy WebGL contexts; `webglcontextlost`
  shows a message); a `ResizeObserver` handles toolbar slide and rotation.

## Issue Tracking

Use the `bd` command for issue tracking instead of markdown TODOs (the session-start hook installs it):
```bash
bd create "Task description" -p 1 --json
bd ready --json
bd update <id> --status in_progress --json
bd show <id> --json
```
