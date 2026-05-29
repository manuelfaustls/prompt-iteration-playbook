# Environment: generic / unknown

Use this when the prompt lives somewhere the playbook has no specific file for: a custom internal tool, a brand-new product, a proprietary LLM console, a chat UI with no API, a no-code builder you have not seen. This is the contract that lets the loop run **without pretending to know mechanics you have not verified.**

> The governing rule: **do not assume any mechanic. Ask, then degrade gracefully** to a filesystem-only loop the user drives by hand if needed.

---

## The minimal contract

The loop needs five capabilities from any environment. Establish which exist by asking (questions below); do not infer them.

1. **See and edit the prompt text.** The floor is being able to copy the prompt out as plain text. If you can only see it on a screen, the user transcribes or screenshots it. If you cannot get the text out at all (no copy, no transcribe, no screenshot), **the loop cannot run; say so plainly and stop**, rather than inventing a mechanic.
2. **Run the prompt over inputs.** One input at a time is ideal; batch-only is workable; find out which.
3. **Observe the output per input.** You need to read what the prompt produced for each input to score it.
4. **Keep an eval set with expected labels.** A local CSV always works (see [`02-the-gabarito.md`](../02-the-gabarito.md)).
5. **Record a run-log.** A local CSV, one row per run.

---

## The safe defaults (offer to create these)

All three are local files, depend on nothing in the environment, and survive losing access to the tool. **Proactively offer them as the standard path**, and only deviate if the user's environment clearly does it better and they confirm. If you (the assistant) can write files, offer to create them. If you have no file access (a chat UI, a console), emit each file's contents as text in your reply and have the user save it in their own editor; the files are theirs to own either way.

1. **Snapshot:** a local file `<name>_v<major>.<minor>.md` with the verbatim prompt text, one frozen file per version, highest number = working version. The moment you can read the prompt text, paste it here. If the env has native versioning, still mirror to the local file so the run-log can cite a stable filename the env cannot mutate. To make a new version, copy the file to the next number and edit that; never edit a released version in place.
2. **Eval set:** a local CSV with `input, expected_label, labeler, categoria` (and an `actual` column to fill per run). At least 20 clean, human-labeled rows for any reported number.
3. **Run-log:** a local CSV, one row per eval run, using the canonical schema: `date, version, model, temperature, dataset, n_clean, accuracy_clean, noise_floor, change_made, regressions, notes`. Each column is documented in [`../05-the-run-log.md`](../05-the-run-log.md).

The loop then becomes: snapshot -> run over the eval CSV by whatever means the env allows -> write outputs into `actual` -> score against `expected_label` -> append one run-log row -> change exactly ONE thing -> repeat.

---

## Degrade to the simplest run path that works

Pick by what the env supports (ask first):

1. **Scriptable** (API/CLI/SDK reachable, key in hand): write a tiny runner that loops the eval rows, sends prompt + input, captures output to the `actual` column. The only fully automatable path.
2. **Semi-manual** (a UI that runs one input at a time, no API): paste the frozen prompt once, feed each eval input through the UI, copy each output back into the CSV row by row. You guide and record.
3. **Batch-only** (you can only run the whole set): run the batch, then align outputs to expected labels by a stable `id` key. Watch for per-input regressions that an aggregate number hides.
4. **Pure manual floor** (a chat box, copy-paste only): paste prompt + input, read the answer, type the result into the CSV. Slow, but always available.

Whatever the path: run the same prompt over the same eval set, score on the `clean` subset only, keep n at 20 or more for any reported number, change one thing between two runs, and measure the noise floor (run the same prompt twice, count differing outputs; a delta at or below it is not signal).

---

## Traps specific to unknown environments

- **Do not assume determinism.** Temperature 0 is not a guarantee in 2026. Always run the same prompt twice and measure the noise floor before believing a small delta.
- **Hidden state you cannot see.** The env may inject a system prompt, prior conversation turns, retrieved documents, tools, or a different model version that are not in the prompt text you snapshotted. Two "identical" runs can differ because of context you did not control. Ask what wraps the prompt; start each eval input from a fresh or cleared session if the env has conversation memory.
- **Silent model/default drift.** The env may swap the model or change a default without telling you, invalidating a baseline. Log the model and parameters every run; if hidden, log `model=env default as-of <date>` and re-baseline when in doubt.
- **Edit-without-snapshot loses versions.** UIs autosave and overwrite; their "undo" is not a version store you have verified. Snapshot to the local `.md` before you edit, every time.
- **Copy-paste mangling.** Rich-text fields inject smart quotes, em-dashes, non-breaking spaces, or strip newlines on paste. Diff what you pasted against the source; a behavior change may just be a mangled paste.
- **Batch-only hides per-input regressions.** Align outputs to inputs by a stable key and scan per row, not just the top-line number.
- **Cost is unknown until you ask.** If you cannot determine it, assume metered (the conservative default), estimate a batch's cost, and surface the number before running.
- **No-export traps your results on screen.** If you cannot export, capture each output into the CSV by hand as you go; a 50-row on-screen-only batch is unrecoverable if you do not capture it live.

---

## The clarifying questions to ask (verbatim, do not skip)

1. Can you copy the prompt text out as plain text (select-and-copy, or open the file it lives in)? If not, can you screenshot or transcribe it? (If neither, the loop cannot run and I will say so.)
2. Where does the prompt physically live: a field in the tool, a config or code file, a doc, or only on a screen?
3. Can you run the prompt on one input at a time, or only the whole batch at once?
4. Does each run cost money or credits, or is it free? If metered or capped, roughly what is the per-call cost or the quota ceiling? (I will estimate a batch before running it.)
5. Can you export results (download CSV, copy a table), or only read them on screen?
6. Is anything wrapping the prompt that I cannot see: a hidden system prompt, prior turns, retrieved documents, tools, a fixed example set? Does the tool remember previous turns?
7. Which model and settings (temperature, max tokens) does this env use, and can you see or pin them?
8. Do you already have an eval set, and who labeled it: a human (truth) or an AI/agent (provisional)?
9. Do you already keep a snapshot and a run-log, or should I draft the standard files for you to save? (Standard if unsure: yes, draft them; I will create them if I can write files, or hand you the text to save if I cannot.)
