# Models and data on the Hub

All private, under the `mszarski` account on Hugging Face (`HF_TOKEN` required). Model paths in this repo are written
`hf://repo/file` or `repo:sub/dir` (`rmr.planner.finetune.resolve`).

| repo | what is in it |
|---|---|
| `reachy-voice` (model) | responders and their evals. `audio/student/v2/merged`: the live audio responder. `student/v8-dpo/merged` and `gguf/v8-dpo/reachy-voice-4b-Q8_0.gguf`: the text responder. `student/v1…v8/adapter` and `student/v3…v5-1.7b/adapter`: LoRA adapters of earlier text students. `audio/voiced/`: the cloned training and test audio. `synth/`: lines, tone readings, tone bank. Also benches and job logs. |
| `reachy-motion-generator` | `generator.pt` (v1) and `generator_v2.pt` (live) |
| `reachy-mini-planner-4b` | the fine-tuned typed-prompt planner (optional; the live system uses a zero-shot planner) |
| `reachy-planner-sft` (dataset) | the planner's SFT data, and `code/`, which `deploy/hf_job.py` uploads for each job |
| `reachy-listening` | the learned listening head (`listener/v1/listener.json`; trained on CC-BY-NC data, opt-in only) |
| `reachy-feedback` (dataset) | live thumbs, their audio and puppeteered clips, under `live/` |
| `reachy-mini-motions-live` (Space) | the live app |
| `reachy-mini-motions` (Space) | the older static gallery |

## Removed in the storage cleanup (2026-11)

The account went over its 1 TB storage limit, so superseded weights were deleted. Their results stay in
[results/](results/); the runs can be repeated from the scripts.

- The merged models of text students v1–v8, v5-dpo and the 1.7B versions. Their adapters are kept and can be
  re-merged, except v5-dpo, which had none.
- All intermediate `checkpoint-*` folders.
- Every GGUF except `gguf/v8-dpo/reachy-voice-4b-Q8_0.gguf`.
- The audio responders `audio/student/v1`, `v3` and `voxtral-v1`.
- The repos of the negative planner-grounding experiment (`reachy-mini-planner-4b-ground-*`, `reachy-planner-sft-ground*`).

So paths such as `audio/student/v1/merged` or `gguf/v8/...` in older result pages no longer resolve.

## Storage rules

Hub storage is billed on everything a repo has ever held, history included, not just its current files.

- **Push LoRA adapters, not merged models.** An adapter is 0.1–0.3 GB; a merged 4B model is 8 GB, a 7B audio model
  17 GB. Merge at load time or at Space start-up. Only the served model may also be kept merged.
- **No intermediate checkpoints** unless a run needs resuming; delete them once the run is done.
- **Never overwrite a big file in place.** Each push keeps the old copy in history until the repo is squashed. Write
  a new version to a new path, and delete the old one when it is superseded.
- **Keep results on the job's volume too**, so a failed upload does not lose the run.
- **Check and squash.** Billed size: `HfApi().repo_info(repo, expand=["usedStorage"]).used_storage`. Current files:
  the sum of `siblings` sizes with `files_metadata=True`. When billed is much larger, `HfApi().super_squash_history(repo)`
  drops the history and keeps the current files. The billed number can take a while to update after deletions.
