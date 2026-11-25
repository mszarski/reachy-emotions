# Running it

Everything runs on open-weight models. The trained weights are in private Hugging Face repos under `mszarski`
([hub.md](hub.md)), so commands that load them need `HF_TOKEN` in the environment.

## Install

```bash
pip install -e ".[ik,dev]"   # core + Stewart-platform IK + pytest
pytest                       # tests skip what a missing extra or `node` would need; check the skip count
```

| extra | for |
|---|---|
| `ik` | the SDK's Stewart-platform IK (`rmr.reach`) |
| `generator` | torch, to train or sample the motion generator |
| `hf` | Hugging Face Hub and datasets downloads |
| `voice` | Whisper, the tone models and the text responder in-process |
| `planner` | fine-tuning (transformers, PEFT, TRL) |
| `serve` | the live server (FastAPI, uvicorn) |
| `render` | MuJoCo videos; also `pip install --no-deps reachy-mini` for the robot's model files and, headless on a CPU, `apt install libosmesa6` with `MUJOCO_GL=osmesa` |

## Text to motion, offline

```bash
python -m rmr.pipeline --prompt "sneezing. You build up and then sneeze." --out runs/one \
    --ckpt hf://mszarski/reachy-motion-generator/generator_v2.pt --render      # + videos/ and grid.mp4
python -m rmr.renderer sheet runs/one/motions/*.json --out runs/one/sheet.png  # contact sheet
```

```python
from rmr.recipe import variants
plans = variants("go 1 e=150 p=22 z=-16 E=5 | osc 3 z 4 .9 E=6 | hold 1 E=4", n=4, seed=0)   # sobbing
```

## The live server

`rmr.server` serves the 3D viewer and the API on port 7860: hold the mic button and talk, or type a prompt.

```bash
# text responder in-process (CPU is fine; a GPU is faster)
python -m rmr.server --voice-model mszarski/reachy-voice:student/v8-dpo \
    --ckpt hf://mszarski/reachy-motion-generator/generator_v2.pt

# text responder behind llama.cpp or Ollama (any OpenAI-compatible URL)
python -m rmr.server --voice-model http://localhost:8080/v1 --ckpt hf://mszarski/reachy-motion-generator/generator_v2.pt

# audio responder (needs a GPU with ~20 GB in bf16, or a vLLM URL)
python -m rmr.server --audio-model mszarski/reachy-voice:audio/student/v2 \
    --ckpt hf://mszarski/reachy-motion-generator/generator_v2.pt
```

The server also reads env vars (`GENERATOR_CKPT`, `VOICE_MODEL`, `AUDIO_MODEL`, `PLANNER_MODEL`, `RESPONDER_MODEL`,
`CALIBRATE_VOICE`, `FEEDBACK_DIR`, ...; see the bottom of `rmr/server.py`).

| endpoint | what |
|---|---|
| `POST /api/respond` | WAV body; `?stream=1` streams NDJSON stages (`heard`, `react`, `motion`, `done`) |
| `POST /api/generate` | typed prompt → move |
| `GET /api/idle` | a small idle move, picked by mood |
| `POST /api/feedback`, `POST /api/clips` | thumbs and puppeteered clips |
| `GET /api/mood`, `GET /api/listener`, `GET /api/health` | session mood, listener style, status |

Pages: `/` is the viewer and talk UI, `/puppet.html` the puppeteering page. The static viewer alone (gallery, drop
any move JSON) runs with `python -m http.server -d visualizer 8000`.

## On a Reachy Mini

```bash
python -m rmr.robot --server http://localhost:7860
```

The robot's microphone feeds the listener on board; hearing, the responder and the generator stay on the server.
Tested on the SDK's MuJoCo simulator only, not yet on hardware.

## The live Space

`deploy/live_space.py` uploads `rmr/`, `pyproject.toml`, `visualizer/`, the Dockerfile and `deploy/start.sh` to the
private Docker Space [`mszarski/reachy-mini-motions-live`](https://huggingface.co/spaces/mszarski/reachy-mini-motions-live)
(app: https://mszarski-reachy-mini-motions-live.hf.space). `start.sh` starts vLLM (FP8, audio responder) or
llama.cpp (8-bit GGUF, text responder), waits for it, then runs the server. As deployed now:

```bash
python deploy/live_space.py --repo mszarski/reachy-mini-motions-live \
    --ckpt hf://mszarski/reachy-motion-generator/generator_v2.pt \
    --audio-model mszarski/reachy-voice:audio/student/v2 --vllm --hardware l4x1 --sleep 1800
```

Without `--audio-model` it deploys the text path (`deploy/Dockerfile`, llama.cpp). The Space sleeps after 30 minutes
without visitors; the first request after waking takes a few seconds longer.

## GPU jobs

`deploy/hf_job.py` runs any command from this repo on a Hugging Face Jobs GPU with a hard cost cap (the timeout is
`--max-usd` divided by the flavor's price) and uploads `/work/out` when it ends:

```bash
python deploy/hf_job.py --max-usd 2 --flavor l4x1 --out-repo mszarski/reachy-voice --out-path audio/<name> \
    --name <name> --packages "transformers>=5.5 soundfile" -- python scripts/audio_llm.py <subcommand> ...
```

Before pushing results to the Hub, read the storage rules in [hub.md](hub.md).
