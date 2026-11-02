# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

**Read HANDOFF.md first for current state, history, latest numbers, job IDs and next steps.** This file holds only the
durable rules.

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

Goal: an emotionally responsive companion that hears words and tone, reacts within a second, answers with expressive,
physically reachable motion, keeps a mood, and learns from the user's ratings. The user has **no robot yet**: everything
is checked in the 3D viewer and the MuJoCo simulator (`rmr/robot.py` is simulator-tested only).

README.md has the architecture, training story and status table; docs/ROADMAP.md the per-step plan and commands;
docs/results/*.md the measured results (including negative results; record new ones there).

## Commands

```bash
pip install -e ".[ik,dev]"               # core + Stewart-platform IK + pytest
pytest                                   # all tests (testpaths = tests)
pytest tests/test_recipe.py -v           # one file
```

Extras: `generator` (torch), `hf`, `planner`, `voice`, `serve`, `render` (MuJoCo; also `pip install --no-deps
reachy-mini`, headless `MUJOCO_GL=osmesa` + `libosmesa6`). Tests `importorskip` missing extras and skip JS-parity tests
without `node`, so check the skip count. `.claude/hooks/session-start.sh` sets this project up on Claude Code on the web.

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
- The static viewer Space `mszarski/reachy-mini-motions` (old gallery) is updated by re-uploading `visualizer/`.
- Hub repos, all private under `mszarski` (`HF_TOKEN` required): `reachy-voice` (students, GGUF, `audio/student/*`,
  benches, job outputs), `reachy-motion-generator`, `reachy-mini-planner-4b`, `reachy-planner-sft`, `reachy-listening`
  (non-commercial, opt-in), `reachy-feedback` (live thumbs and puppet clips under `live/`).
- The live Space serves the audio path (`--audio-model ... --vllm`); `live_space.py` without `--audio-model` still
  defaults to the text path (llama.cpp GGUF). The exact redeploy command is in HANDOFF.md.

Infra gotchas:
- HF L4/L40S/A10G capacity is often short: launch a job on two flavors and cancel the loser; a Space may loop on
  restarts until capacity frees. A Space BUILD_ERROR exit 128 with no logs is transient: push a new commit.
- `vllm/vllm-openai` has no `python` binary: use `python3`. llama.cpp server image needs `LD_LIBRARY_PATH=/app`;
  `start.sh` exits if llama-server dies so the Space restarts instead of hanging.
- torch wheels: use the cu128 index (the default targets CUDA 13, which job drivers lack).
- transformers 5: no `warmup_ratio` (use `warmup_steps`); `use_cpu`/bf16 must depend on GPU presence. Smoke-test
  training code on tiny `trl-internal-testing` models before paying for a GPU.
- Voxtral: `transformers==5.18.0`, `mistral-common==1.12.0`, no system message with audio (`voxtral_inputs`).
  Chatterbox: own venv, `pyarrow<17`, `setuptools<81` (else the perth watermarker is `None`).
- Sandbox may block `*.hf.space` (test from an HF job until the user allows it) and jsdelivr (Playwright: route
  three/urdf-loader/mediapipe to local npm copies; proxy doesn't bypass loopback). Monitor expires after ~5 min: use
  `run_in_background` loops, and re-create them after container restarts.

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
- Servo limits are enforced in `rmr/reach.py` (`MOTOR_LIMITS` from the URDF), not just IK solvability: IK alone
  accepted out-of-range poses ("neck off the platform"). The viewer mirrors it (`StewartIK.feasible()`).
- Tests that count listener nods must monkeypatch `listen.STYLE = listen.CLASSIC` (x06 turns gestures off).
  `Mood.update()` must return `state(self.t)`, not state decayed to wall-clock time; keep that test.
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
- Hold-to-talk: finger can lift before the mic opens (`holding`/`opening` flags); `pointercancel` not `pointerleave`;
  block `contextmenu`/`-webkit-touch-callout`; wrap `setPointerCapture` in try/catch.
- Inputs need `font-size: 16px` (iOS zooms otherwise); `dvh` with `vh` fallback and `env(safe-area-inset-bottom)`.
- Lower pixel ratio and plain PCF shadows on touch devices (Safari drops heavy WebGL contexts; `webglcontextlost`
  shows a message); a `ResizeObserver` handles toolbar slide and rotation.

## Decisions (keep unless new evidence)

- **Ratings decide, judges don't.** LLM text and VLM video judges agreed with the user at chance (even at 4+ fps).
  Success is the user's own ratings (duels, thumbs).
- **No synthetic post-processing.** A tremor effect was removed at the user's request: "We should want the model to be
  better, not add synthetic changes after". Fix the model or data instead.
- Audio responder: Qwen2-Audio-7B fine-tune with transcript + tone line in the input, served by vLLM FP8 on an L4
  (bf16 didn't fit comfortably; FP8 kept accuracy). Voxtral-Mini-3B fine-tune got 38% vs 91% on tone; the
  transcript-free variant (v3, `--audio-no-transcript`) lost on cloned lines (p=0.035), the user said keep v2.
- Hearing: Whisper-large-v3-turbo (better WER and faster than small); Voxtral-Realtime was neither more accurate nor
  faster. Text student: Q8_0 GGUF, not Q4 (physical fit 78% vs 70%). Text student labels come from Kimi-K3, then DPO
  on the user's preference pairs (v8-dpo, user-preferred).
- `END_OF_TURN = 0.7` s, not 0.6 (speech has natural 0.6 s gaps). Listener style x06 won the user's duel votes.
- Training audio: written lines voiced by Chatterbox zero-shot cloning of the paired CREMA-D clip (training actors
  only), drop lines with Whisper WER > 0.35; held-out actors are `actor % 5 == 0`. Plain TTS (e.g. Zonos) makes every
  voice sound sad. MELD is out for licence reasons (`scripts/listen_meld.py` is evaluation/history only).
- Other dead ends: planner grounding on real clips (`docs/results/planner_finetune.md`). Full table in HANDOFF.md.

## Working with the user

- Plain answers to direct questions ("does the server take the microphone", "what branch are we on?"): answer
  first, then offer next steps.
- Asked "what's next": a short numbered list of options with a recommendation. They often reply "do all of them" or
  pick numbers.
- Happy for long autonomous runs (they've been away for days): keep going, commit as you go, report when asked.
- They value real model improvements over hacks, open licences (Apache/MIT models, licence-clean data), and being
  runnable locally on open weights.
- Their ratings beat any automatic score. Rating tools must never repeat clips (they noticed the duel page doing it).
- Often on an **iPhone**: keep the viewer Safari-friendly, send rendered MP4s as files, and give direct links
  ("Link to space?"): Space https://huggingface.co/spaces/mszarski/reachy-mini-motions-live, app
  https://mszarski-reachy-mini-motions-live.hf.space (puppet page at `/puppet.html`).
- Money: HF spend is fine within stated caps (e.g. "cap at $15" for training). Always pass `--max-usd` on jobs. Ask
  before switching a Space to bigger paid hardware (the auto-mode classifier blocks it otherwise).
