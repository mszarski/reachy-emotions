# How it was trained

Each model learns one thing, from data that suits it. The architecture these models fit into is in
[architecture.md](architecture.md); every number here comes from [results/](results/), which also records the
negative results.

| model | learns | data | method | cost |
|---|---|---|---|---|
| generator v2 | how Reachy Mini moves | Pollen's 85 emotion clips + 19 dances (~9 min), 12 emotions held out | flow matching on the clips' own plans, simplified like recipes | 25-run sweep, $1.68 (A10G) |
| planner (optional) | text → recipe | 17,364 prompt → recipe rows written by frontier LLMs (the reference's teacher data) | LoRA on Qwen3.5-4B, 2 epochs | $5.79 (A100) |
| responder v8 | heard → feeling, response, recipe | 4,450 examples: LLM-written lines + real CREMA-D tone readings, answered by Kimi-K3 | LoRA SFT on Qwen3-4B-Instruct-2507 | ~$1 (one GPU, 1 h) |
| audio responder v2 (live) | heard audio → feeling, recipe, response | the same 4,238 examples, with the written lines voiced by cloning real CREMA-D actors | LoRA SFT on Qwen2-Audio-7B-Instruct | ~$3 (one A100, 46 min) |
| text responder v8-dpo | your taste | 160 preference pairs from your ratings | DPO, LoRA r16, 3 epochs | under $1 |
| listener style | your taste | 25 side-by-side votes over 24 styles | Bradley-Terry fit | free |

**1. The generator: motion without text.** There are only about 9 minutes of real expressive motion for this robot,
far too little to learn language from, but enough to learn how it moves. Every clip is turned into its own plan
automatically (`rmr.plan.extract`: posture below 1 Hz plus the energy of the faster detail), mirrored and
time-stretched (552 training clips), and the generator learns to rebuild the full 25 Hz motion from the plan.
Held out, it rebuilds a clip that is identified among the 12 unseen emotions 91.7% of the time (the reference: 89%).
Version 2 fixed a deployment gap: real plans still carry traces of the fast motion, recipe plans are smooth, so the
first generator came out too calm on recipes. Training on recipe-like simplifications of each clip brought the
generated liveliness to real levels (0.96× vs 0.63×). [Results](results/generator.md).

**2. The planner: language without motion.** Knowing what *heartbroken* looks like is world knowledge, so it comes
from LLMs. Frontier models wrote prompt → idea + recipe examples (the reference's teacher data), and a 4B model was
LoRA-tuned on them; held-out prompts and probe prompts are filtered out by embedding similarity so they stay a real
test. The fine-tuned 4B matches more real clips (28% top-1 vs 20%); a strong open model used zero-shot passes all
16 probe prompts and needs no hosting, so the live system uses it for typed prompts.
Grounding the planner in transcriptions of Pollen's real clips did not help (documented as a negative result).
[Zero-shot](results/planner_zeroshot.md), [fine-tune](results/planner_finetune.md).

**3. The responder: reading people, licence-clean.** The responder is *distilled*: a large teacher LLM shows a
small model what to answer, thousands of times, until the small model can do it alone in one call. Five steps:

*What it sees and writes.* Each example is one chat turn. The input is what hearing produced, as text:

```
Earlier in the conversation:
- "I finally finished the report."
They said: "And now they want it redone by Monday."
Their voice sounds angry (confidence 61%).
Arousal 0.68, valence 0.31.
```

The target is one JSON object, in this order so the motion can start before the answer is finished:
`{"feeling": "angry", "response": "Calm attention. You hold steady ... one slow nod", "recipe": "go .8 p=4 e=30 E=1 | hold 1.5 | ...", "reading": "They sound fed up ..."}`.
A short system prompt explains the robot, the recipe language and the house rules (trust the words when the voice
reading disagrees; gentle with sadness, calm with anger, reassuring with worry).

*Step 1, the words* (`scripts/synth_lines.py`). The teacher LLM wrote about 4,200 things a person might say to a home
robot, in six feelings (neutral, happy, sad, angry, anxious, surprised). Each batch draws a random topic (work, a pet,
the commute, ...), a random speaker (a teenager, a nurse after a shift, ...) and a style that fits the feeling
(openly, subtly, in a rush; sarcasm only for anger), and some lines come with the line before them, so the model
learns to use context. Early neutral lines came out weary or sarcastic, so styles are now set per feeling.

*Step 2, the tone* (`scripts/tone_bank.py`, `scripts/synth_heard.py`). Written lines have no voice, and synthetic
voices turned out to sound sad whatever they were asked to do. So the tone comes from real people: the tone models
were run on all 7,442 clips of CREMA-D (91 actors reading set sentences in six emotions, ODbL), and each line gets
the reading of a real clip of the same feeling, never the audio itself. 20% of emotional lines get a reading from a
neutral clip instead, because people often say emotional things flatly, and the model should then rely on the words.
Then 800 *tone-only* examples use CREMA-D's own flat sentences ("It's eleven o'clock.") as the words, so the feeling
can come only from the voice. This is where the responder learns what the tone models' readings mean on real
voices (for example, that they rarely say "happy" for a happy speaker). 18 of the 91 actors are kept out of every
training example, for testing.

*Step 3, the teacher's answers* (`scripts/label_respond.py`). Kimi-K3, a large open model chosen in a bake-off of
four, answers every example in two calls: first the feeling, response and reading, then the recipe for that response
(as the planner would write it). Three filters keep only good answers:

- The feeling must match the feeling the line was written with. When it doesn't, the teacher is asked once more, with
  the true feeling as a hint, to find the cues that show it and answer accordingly; the hint never appears in the
  training example.
- The recipe must pass a physical check: for sad or angry people the head moves slowly with little energy, for happy
  ones the ears come up and the motion is lively, otherwise it stays moderate.
- A cap per feeling, preferring answers the teacher got right without a hint, keeps the hardest feelings from
  crowding out the rest.

That left 4,450 training examples and 100 for validation.

*Step 4, supervised fine-tuning* (`rmr/planner/finetune.py train`, on a Hugging Face Jobs GPU). The base is
Qwen3-4B-Instruct-2507 (Apache-2.0). Only small LoRA adapters are trained (rank 32 on every attention and MLP
projection), and the loss counts only the answer, not the prompt. Two epochs, learning rate 1e-4 with a cosine
schedule, 32 examples per step (8 × 4 accumulation), 8-bit AdamW; the checkpoint with the lowest validation loss is
kept (0.59, at the end of epoch 2). About an hour on one GPU, around $1. The adapters are then merged into the model.

*Step 5, preference tuning* (`scripts/dpo_from_ratings.py`). On the rating page you picked the best and the worst of
four answers to the same line. Each such choice gives five ordered pairs (best over each of the others, each of the
others over worst), 160 pairs in all, from training lines only. DPO then nudges the model to make the chosen answers
more likely than the rejected ones, relative to where it started (beta 0.1, LoRA rank 16, 3 epochs, learning rate
1e-5). That is v8-dpo, the default: its automatic scores are unchanged, while 158 of 255 recipes on the test lines
changed, in the direction you preferred.

*How the versions got here* (all scores on held-out actors and lines):

| version | what changed | tone test (chance 20%) | responses that fit physically |
|---|---|---|---|
| v3 | trained on MELD (TV dialogue) | 32% | |
| v4 | written lines only, no MELD | 28%: it ignored tone | |
| v5 | + the 800 tone-only examples | 47% | 76% |
| v8 | the same data answered by Kimi-K3 instead of Qwen3-Next | 43% | 90% |
| v8-dpo | + your 160 preference pairs | 42% | 90% |

A stronger teacher did not read people better (the tone models' readings are the limit there), but it wrote much
better responses and recipes. [Full results](results/voice.md).

**4. Your preferences.** Automatic scores turned out not to predict what you prefer: an LLM judge and two video
judges (vision models watching the rendered motion) all sat at chance against your ratings. So your ratings decide.
On the rating page you picked the best and worst of four responses; those choices became 160 preference pairs, and
DPO tuned the responder on them (v8-dpo, the default). The automatic scores held while the motions shifted toward what
you preferred. The listener style was picked the same way: an adaptive page showed two styles side by side over
real meeting speech (AMI), and a Bradley-Terry fit of your votes chose fewer, bigger nods.

**5. Serving.** The responder is exported to GGUF and served by llama.cpp: the 8-bit build scores the same as full
precision and is about twice as fast; the 4-bit build was faster still but its motions fit less often on the tone
test, so it is not used.

**6. The audio responder: hearing the voice itself.** The text responder only ever sees the tone models' one-line
reading, and on the tone test that caps it at 42%. An open audio LLM can hear the voice directly. Qwen2-Audio-7B-
Instruct (Apache-2.0) won a zero-shot bake-off (78% on the tone test). It is trained on the same examples and teacher
answers as the text responder, so the only new part is audio for the written lines:

- *Voicing the lines* (`scripts/clone_speech.py voice`). Each written line was already paired with a real CREMA-D clip
  of its feeling (training actors only). Chatterbox (MIT) clones that clip's voice and delivery zero-shot onto the
  new words, with `exaggeration` set per feeling. Plain TTS was tried first and makes every voice sound sad.
- *Checking them* (`scripts/clone_speech.py check`). Whisper and the tone models run on every voicing. Lines whose
  Whisper word error rate is above 0.35 are dropped as garbled, leaving 4,238 examples. The 676 tone-only examples
  keep their real CREMA-D audio.
- *Training* (`scripts/audio_llm.py build` then `train`, on an HF Jobs A100, 46 min, about $3). The input is the
  audio, the Whisper transcript, the tone line and up to two earlier lines; the target is the teacher's JSON answer.
  LoRA rank 32 on the language model (80M trainable of 8.5B), audio encoder frozen, 2 epochs.
- *v2*, the live version, writes the recipe right after the feeling, so the motion starts sooner, and 20% of its
  training examples get one or two unrelated earlier lines with the answer unchanged, so it stops inventing links to
  things said before.

| on the same held-out audio | tone test (real actors, flat sentences) | cloned test lines | + unrelated earlier lines |
|---|---|---|---|
| text pipeline (Whisper + tone line → v8-dpo) | 42.3% | 57.8% | |
| audio responder v1 | 90.7% | 67.9% | 64.1% |
| **audio responder v2 (live)** | **91.0%** | **68.3%** | **67.9%** |

Tried and rejected: a Voxtral-Mini-3B fine-tune (38% on the tone test), and v3, trained without the transcript so it
could start before Whisper finishes (62.8% vs 68.3% on cloned lines, p=0.035). Details in
[results/voice.md](results/voice.md#audio-native-responder-2026-10-13).
