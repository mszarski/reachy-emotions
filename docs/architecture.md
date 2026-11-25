# Architecture

How Reachy Mini goes from your voice (or a typed prompt) to a move it can physically play. Training is in
[training.md](training.md), measured results in [results/](results/).

Four stages, each with one name used throughout:

| name | what it is | in code |
|---|---|---|
| **listener** | rules (no model) that move the head while you are speaking; its **style** is a set of numbers (nod size, timing, lean, sway) | `rmr/listen.py`, `STYLE` |
| **hearing** | Whisper for the words, plus two **tone models** that read the feeling in your voice | `rmr/voice.py` |
| **responder** | the fine-tuned LLM that decides how Reachy answers and writes the recipe. It is *distilled* from a large teacher LLM, so the code and results call it the *student*. Two kinds: the **audio responder** (Qwen2-Audio-7B fine-tune, hears the voice itself; v2 is live) and the **text responder** (Qwen3-4B fine-tune that reads the tone models' line; v3 … v8-dpo) | `rmr/audio_responder.py`, `rmr/respond.py` |
| **generator** | the motion model that turns a recipe's plan into 25 Hz motion | `rmr/generator/` |

```
 you speak ─────────────────────────────────────────────────────────────────────────────┐
   │ microphone (browser viewer or the robot), 25 Hz loudness                           │
   ▼                                                                                    │
 LISTENER (rules, no model, in its chosen style) ── leans in, nods at your pauses, perks antennas ──► head pose while you talk
   │ when you stop: the recording goes to the server
   ▼
 HEARING     Whisper ─► words                tone models (WavLM) ─► tone: feeling + arousal/valence/dominance
   │                                            └─ per-session calibration (mic and room levels)
   ▼
 RESPONDER   audio responder v2 (Qwen2-Audio-7B fine-tune, vLLM FP8): audio + transcript + tone line + the
   │           last two lines of the conversation, one call, streamed: {"feeling", "recipe", "response", "reading"}
   │           (or the text responder: Qwen3-4B student v8-dpo, 8-bit GGUF on llama.cpp, no audio)
   │           (typed prompts skip hearing: a zero-shot LLM planner writes the recipe)
   │ ── the moment "feeling" is decoded: REACT (rmr/react.py) sends a short reaction move (~0.5-0.9 s)
   ▼
 RECIPE      go .3 e=-15 p=-8 z=12 E=6 | osc 1.2 z 5 .5 E=7 | hold 1.2 E=1.5 | ...
   │ expand + randomise (amplitude, timing)
   ▼
 PLAN        8 channels at 4 Hz: ears L/R, pitch, roll, yaw, height, body yaw, energy
   │
   ▼
 GENERATOR   21.8M flow-matching transformer: plan ─► 9-DoF motion at 25 Hz (8 Euler steps, ~50 ms)
   │
   ▼
 REACH       Stewart-platform IK on every frame; a pose with no solution, or needing a motor past
   │           its servo range, is pulled back toward the last good pose
   ▼
 MOVE        Reachy Mini recorded-move JSON ─► the 3D viewer, or the robot (rmr.robot)
```

**The listener** (`rmr/listen.py`, ported line for line to `visualizer/src/Listen.js`) is a small controller on the
loudness of your voice, so it reacts within a frame with no network. It tracks the room's noise floor, nods after a
pause that follows enough speech, leans in while you talk, perks the antennas when your voice lifts, and adds a slow
sway and an occasional glance. Its style (nod size, timing, lean, sway) is the one you picked in the duel ratings
(see [training.md](training.md)).

**Hearing** (`rmr/voice.py`). Whisper turns speech into words (large-v3-turbo on a GPU, small on a CPU). Two tone models (WavLM, trained on MSP-Podcast) read
the tone: a feeling (neutral, happy, sad, angry, surprised, ...) and three attributes, arousal, valence and dominance.
The same voice reads differently on different microphones, so each session keeps running means and shifts its
readings toward the levels the responder was trained on (`Calibration`). The text responder never hears audio: it
gets the words and one line of text with the tone models' output ("Their voice sounds angry (confidence 61%). Arousal 0.68, valence
0.31."). Each tone model is WavLM (a self-supervised speech encoder that captures pitch, energy, rate and voice quality,
not words), attentive pooling over the utterance, and a small head: probabilities for 8 emotions, or scores for the
three attributes. How much to trust that line is what the responder learns in training.

**The responder.** One model call turns what was heard into an answer: the feeling it reads, a one-sentence
response ("Calm attention. You hold steady, ears relaxed, one slow nod"), the motion recipe, and a short reading of
the person. The server streams the answer: as soon as the feeling is out, a short reaction move is sent
(`rmr/react.py`); as soon as the recipe is complete, the full motion is generated and sent; the reading arrives after.

- **Audio responder** (`rmr/audio_responder.py`, live). A fine-tune of Qwen2-Audio-7B-Instruct (Apache-2.0) that hears
  the recording itself, alongside the transcript and the tone line. It reads tone far better than the text path (91%
  vs 42% on held-out actors). Served by vLLM in FP8 on the Space's L4 (`deploy/Dockerfile.vllm`), or in-process with
  transformers. Whisper and the tone models still run for the transcript, the conversation memory and the mood.
- **Text responder** (`rmr/respond.py`, `Student`). A fine-tune of Qwen3-4B-Instruct (Apache-2.0) that only sees
  text. Small enough for a laptop: an 8-bit GGUF on llama.cpp or Ollama, CPU is fine. The default of
  `deploy/live_space.py` without `--audio-model`.

For typed prompts, a zero-shot open LLM (`rmr/planner/`) writes the recipe instead.

**Recipes and plans** (`rmr/recipe.py`, `rmr/plan.py`). A recipe is a tiny language an LLM can write reliably:
`go` (ease to a pose), `hold` and `osc` (a sinusoid on one channel), over ears, pitch, roll, yaw, height, body yaw
and an energy level. It expands into a plan: keyframes every 0.25 s of the posture (below 1 Hz) and the energy,
meaning how much fast organic detail to add. Each expansion is randomised a little, so the same recipe never plays
twice the same way.

**The generator** (`rmr/generator/`). A transformer with one token per 25 Hz frame (d=384, 8 layers, 6 heads). Each
token carries the noisy motion and the plan at that frame; the flow time enters every block through AdaLN. It is
trained with flow matching and samples in 8 steps with classifier-free guidance 1.5. It never sees text: it only
knows how Reachy Mini moves, which is why it can learn from every clip.

**Reach** (`rmr/reach.py`). The head sits on a 6-motor Stewart platform, so its reachable poses are coupled: a tilt
that works at one height fails at another. Every frame goes through the SDK's analytical IK, and any frame with no
solution, or that would need a motor beyond its servo range, is line-searched back toward the last good frame. The
browser viewer has the same IK (checked against the SDK's to 1e-9 rad) and holds the last good frame as a safety net.

**Mood** (`rmr/mood.py`). Each session keeps a valence/arousal mood that moves toward each feeling read and decays
with a 90 s half-life. It biases the listener's style and picks the idle moves (`rmr/idle.py`).

**Feedback** (`rmr/feedback.py`). Thumbs up/down on any answer, and clips recorded on the puppeteering page
(`visualizer/puppet.html`: your face drives the head and antennas through MediaPipe), are pushed to the private
dataset `mszarski/reachy-feedback` for the next round of training.

**Around it.** `rmr/server.py` serves the viewer and the API (`/api/respond` for speech, `/api/generate` for
typed prompts, `/api/idle` for small idle moves when the room is quiet, `/api/feedback`, `/api/clips`, `/api/mood`). `rmr/robot.py` runs the same loop on a
Reachy Mini: the robot's microphone, the listener on board, the models on the server. `deploy/live_space.py`
deploys everything to a private Hugging Face Space on an L4 GPU that sleeps when idle.

