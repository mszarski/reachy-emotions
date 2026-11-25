# Status by component

What each part of `rmr/` does and how far it got. Numbers link to [results/](results/); the plan behind each step
is in [ROADMAP.md](ROADMAP.md).


| step | module | status |
|---|---|---|
| Move format ↔ 9-DoF arrays, mirror, time-stretch | `rmr/motion.py` | ✅ done, tested |
| Plans: extract from any clip, interpolate to frames | `rmr/plan.py` | ✅ done, tested |
| Recipe language: parse, validate, expand, randomised variants | `rmr/recipe.py` | ✅ done, tested on all teacher recipes |
| Reachability projection (Stewart-platform IK) | `rmr/reach.py` | ✅ done, tested |
| Generator: 21.8M flow-matching transformer | `rmr/generator/` | ✅ 91.7% top-1 on the 12 held-out emotions (reference 89%); v2 trained on recipe-like plans matches real motion's liveliness in deployment. See [results](results/generator.md) |
| Planner, zero-shot stand-in: open-weight LLM via HF Inference Providers, probe suite | `rmr/planner/` | ✅ Kimi-K3 passes 16/16 probes; real clips 20% top-1 (reference zero-shot 22%). See [results](results/planner_zeroshot.md) |
| Planner: LoRA fine-tune of Qwen3.5-4B on the teacher data (HF Jobs, $5.79) | `rmr/planner/` | ✅ probes 0.88 / 0.81, real clips 28% top-1 (reference 4B: 0.91 / 0.875, 32%). See [results](results/planner_finetune.md) |
| Offline pipeline (text → reachable moves) | `rmr/pipeline.py` | ✅ done |
| Live server: type a prompt, watch it in the viewer (zero-shot planner + CPU generator) | `rmr/server.py`, `deploy/` | ✅ private Docker Space `mszarski/reachy-mini-motions-live`: L4 GPU, audio responder v2 on vLLM FP8 (reaction ~0.5-0.9 s, full motion ~3-4 s after you stop), sleeps when idle |
| Voice: hold to talk; Reachy responds to what you say and how you sound (open models, runs locally) | `rmr/voice.py`, `rmr/respond.py`, `rmr/converse.py` | ✅ reads people right 48% (4 classes, real dialogue), responds appropriately 66%; a distilled local 4B does it in one call (50%, 73%); a 1.7B starts moving in under 6 s on a CPU (streaming, GGUF for llama.cpp / Ollama); hands-free turn-taking. Default student v8-dpo is licence-clean (no MELD: written lines + real CREMA-D voice readings, a frontier open teacher, then preference-tuned on human ratings): uses tone (42% on a tone-only test, chance 25%), physically fitting responses 90% of the time, 72% appropriate on MELD with per-session calibration (on by default). Automatic judges, LLM or video, don't agree with human raters, so ratings decide. See [results](results/voice.md) |
| Listening: while you talk, Reachy leans in, nods at your pauses and perks its antennas when your voice lifts (live in the viewer, no model) | `rmr/listen.py`, `visualizer/src/Listen.js` | ✅ done; the JS port matches Python exactly (tested). `python -m rmr.listen talk.wav --video out.mp4` |
| Learned listening head: a GRU trained on 25 h of real listeners (Seamless Interaction, CC-BY-NC; optional) | `rmr/listen_model.py`, `scripts/train_listener.py` | ✅ moves like real listeners (RMS, speed, nod rate); real listeners' nods are not pause-timed, the rules' are. See [results](results/listening.md) |
| Listener style tuned by your own ratings: 10 styles on AMI meeting speech (CC-BY 4.0), side by side; Bradley-Terry fit picks the default | `rmr/listen.py` (`STYLE`), `scripts/listen_rating.py` | ✅ 25 votes over 24 styles: fewer, bigger nods (x06) won and is the default; the learned head came 15th. See [results](results/listening.md) |
| Robot: listen, nod along and answer on a Reachy Mini; models stay on the server | `rmr/robot.py` | ✅ tested on the SDK's MuJoCo simulator; not yet on hardware |
| Human ratings: blind A/B page for responses; ratings on training clips become DPO pairs | `scripts/rating_set.py`, `deploy/rating_page.html`, `scripts/dpo_from_ratings.py` | ✅ two rounds rated; 160 pairs DPO-tuned the default responder (v8-dpo). Automatic judges sat at chance against the ratings. See [results](results/voice.md) |
| MuJoCo renderer: videos, contact sheets, grids (official Reachy Mini model, SDK IK, 500 Hz physics) | `rmr/renderer/` | ✅ done, tested |
| Browser viewer (three.js): gallery of results, drop any move JSON | `visualizer/`, `rmr/viewer.py` | ✅ done; browser IK checked against the SDK's to 1e-9 rad |

| Audio responder: hears the voice itself (Qwen2-Audio-7B fine-tune, vLLM FP8) | `rmr/audio_responder.py`, `scripts/audio_llm.py` | ✅ live (v2): 91.0% on the tone test vs 42.3% for the text pipeline, 68.3% vs 57.8% on cloned test lines. See [results](results/voice.md#audio-native-responder-2026-10-13) |
| Instant reaction, mood, idle moves | `rmr/react.py`, `rmr/mood.py`, `rmr/idle.py` | ✅ live: Reachy starts moving ~1.3-1.6 s after you stop talking |
| Feedback: thumbs on every answer, puppeteered clips (face → head and antennas) | `rmr/feedback.py`, `visualizer/puppet.html` | ✅ live; pushed to `mszarski/reachy-feedback`; ratings not yet collected |
| Serving (FP8, MTP speculative decoding, CUDA graphs) | | FP8 done (audio responder on vLLM); the rest optional |
