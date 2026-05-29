---
name: optimize-prompt-portable
description: A portable, environment-agnostic loop for iterating an LLM prompt, classifier, filter, qualification, or segmentation step toward higher accuracy against an eval set (a "gabarito": inputs with known-correct labels). Every change is measured, isolated, logged, and checked for regression and noise; iteration without that is guessing. The loop stays constant across environments (local code, Clay, OpenAI, automation tools, or a custom tool the skill has never seen); only the artifacts adapt. Reach for this whenever someone wants to make a prompt more accurate, tune a classifier/filter/segmentation, cut false positives or negatives in an AI step, or set up a disciplined A/B prompt-eval workflow.
when_to_use: Trigger on "/optimize-prompt-portable", "optimize this prompt", "iterate the prompt", "improve the classifier", "tune the filter", "improve my AI column", "reduce false positives", "run another version"; and in Spanish "iterar el prompt", "mejorar el clasificador", "optimizar el filtro", "afinar el prompt", "otra version". Also trigger when a user has an eval set (or can build one) and wants a measured way to raise a prompt's accuracy, in Claude Code, Clay, the OpenAI surfaces, n8n/Zapier/Make, or any custom LLM tool. Do NOT use for writing a brand-new prompt with nothing to measure yet (no eval set exists or can be built), or a one-off tweak the user is not iterating on.
allowed-tools: Read, Write, Edit
---

# optimize-prompt-portable

A disciplined loop for iterating an LLM prompt, classifier, filter, qualification, or segmentation step toward higher accuracy, without flying blind.

The core idea is small and it is the whole point: **every change is a hypothesis, and the eval set is how you try to falsify it.** One change at a time, measured against known-correct labels, logged, and checked for regression and noise. A change you cannot measure is not an improvement; it is a guess wearing a confident face.

This skill is environment-agnostic. The loop below is identical whether the prompt lives in a Claude Code file, a Clay AI column, the OpenAI Playground, an n8n node, or a tool this skill has never heard of. What changes per environment is only the mechanics: where the prompt physically lives, how you run it over the eval set, and where the run-log goes. Those mechanics are in `references/environments/`.

## The governing rule (read this first)

**Do not assume you know how the user's environment works. Ask, then adapt.**

If the environment is one of the ones documented in `references/environments/` (Clay, OpenAI, automation tools), read that file for the concrete mechanics. If it is anything else, use `references/environments/generic.md`, which gives you the minimal contract and the exact clarifying questions to ask. When a mechanic is unclear (can the user copy the prompt out? does a run cost money? can they run one input at a time or only a batch?), ask a short, concrete question rather than guessing. A wrong assumption about the environment wastes the user's money (some environments meter every run) or silently breaks the loop.

Whenever you ask the user to choose something, **offer 2-3 labeled options (A / B / C) and mark the one that works anywhere as the standard default**, so a user who does not want to think can pick the safe path and move on. (When you are an agent driving with a user present, offer the options and wait for a yes. When a person is running this loop solo from the doc, they just pick the standard default and proceed.)

**Know who holds the filesystem.** This loop leans on local files (a snapshot, an eval CSV, a run-log). If you (the assistant) can write files, offer to create them. If you are running inside a chat UI or a tool with no file access (ChatGPT, Clay, a console), you cannot write to the user's disk: emit the exact CSV or markdown text in your reply and have the user save it in their own editor. The artifacts are local files the user owns either way; only who types them changes. Never promise to "create a file" in an environment where you cannot.

## Phase -1: Environment questionnaire (always, first thing)

Before any baseline, establish the environment. Read `references/01-environment-questionnaire.md` and ask the five questions there:

1. Where does the prompt live?
2. Where does the eval set (gabarito) live?
3. Which model and temperature?
4. Who labeled the eval set (human = truth, AI = provisional)?
5. Where should the run-log go?

Echo back your understanding of the answers and confirm before proceeding. Then **teach the shortcut**: tell the user they can skip these questions next time by declaring the setup in the invocation, e.g.

```
/optimize-prompt-portable env=clay model=gpt-4o-mini eval=sheet:<url> log=local-csv labeler=human
```

If they declared some fields in the invocation, only ask for the ones still missing.

## Phase 0: Offer the walkthrough, then confirm the method (always)

People misuse this loop in predictable ways: they change three things at once, they tune to labels a human never reviewed, they read noise as signal. A 60-second shared understanding prevents all three.

So, every time: **offer to explain how the loop works and how to use it.** Point the user to `references/00-how-it-works.md`, or summarize it inline in five bullets:

1. **One change per eval. Always.** Two changes at once and you cannot attribute the delta to either.
2. **Every run is logged** (version, model, accuracy by cut, notes). No log means the change did not happen.
3. **Regression scan after every change:** which inputs that were correct just became wrong?
4. **Noise floor:** run the same prompt twice; count the outputs that differ. A change smaller than that floor is not signal. Re-measure when the model or temperature changes.
5. **Never tune against unreviewed labels.** Human-labeled rows are truth; AI-labeled rows are provisional. Tuning a prompt to agree with a model is circular.

Then ask: "Does this make sense, or do you want to adjust anything before we start?" Wait for an explicit yes. Without a shared method, the rest of the loop is theater.

## Phase 1: The gabarito (eval set) ⭐

This is the foundation. A prompt loop is only as trustworthy as the eval set it is measured against. Read `references/02-the-gabarito.md` in full; it documents every column (what it is for, how to fill it, worked examples) and how to scaffold a set from scratch.

The eval set needs four columns at minimum: `input`, `expected_label`, `labeler`, `categoria`. The `categoria` column decides what counts as a miss:

- `clean`: the input alone is enough to decide. **You score accuracy on these rows only.**
- `needs_context`: the input is genuinely insufficient (a missing field, an ambiguous reference). This is the input-only ceiling, not a classifier miss.
- `out_of_scope`: the row should not be in the set (wrong format, duplicate). Remove it; it is not a miss.

Any number you show a stakeholder needs at least 20 clean, human-labeled rows. Smaller samples get an explicit caveat or no number at all (a three-point move on 20 rows is inside the noise). If every label is AI-produced and unreviewed, you may use the set to triage which cases a human should review first, but report no accuracy and claim no improvement until a human confirms at least 20 clean rows.

If the user has no eval set yet, **offer to scaffold one as the standard path**: a local CSV with the four columns plus a couple of sample rows. The templates in `examples/` are ready to copy. Before Phase 3, print a short gabarito audit (total rows, counts by categoria, how many are clean-and-human-labeled, whether you are above the n=20 threshold). If you are below it, pause and say so.

## Phase 2: Versioning structure (once per project)

You need three durable artifacts. **By default these are plain local files** (they depend on nothing in the environment and survive losing access to the tool). Offer to create them if you can write files; otherwise emit their contents for the user to save (see "Know who holds the filesystem" above):

1. **A snapshot per version.** A frozen copy of the exact prompt text, one file per version, named `<name>_v<major>.<minor>.md`. The highest number is the working version. Never edit a released version in place; copy it to the next number and edit that.
2. **A run-log CSV.** One row per eval run. Canonical columns: `date, version, model, temperature, dataset, n_clean, accuracy_clean, noise_floor, change_made, regressions, notes`. Append, never overwrite. Each column is documented (what it is, an example, how to fill) in `references/05-the-run-log.md`; the filled template is `examples/run-log-template.csv`. Score `accuracy_clean` over clean, human-labeled rows only; track the count of `needs_context` rows in the gabarito audit, not as an accuracy.
3. **A changelog** (can live as a header section in the snapshot or the README): the table of versions with the measured lift per version.

How you snapshot depends on the environment, and you must use the right mechanic (see `references/environments/`):

- **Claude Code / your own code:** the prompt is already a file; version it however you version code (git, or the dated `.md` snapshot if you do not use version control).
- **Clay:** a classic "Use AI" column has no built-in versioning, so the default is to **copy the prompt text out of the Configure tab into a local .md or .csv** (also copy the model name and the output-field definitions; they are part of the version). A Claygent in the Claygent builder does have native versioning; if so, rely on it for the prompt body and keep only the run-log locally. See `references/environments/clay.md`.
- **OpenAI dashboard Prompts:** native versioning exists (Update / Publish / History); pin the version explicitly. Plain API strings have none; version them yourself. See `references/environments/openai.md`.
- **n8n / Zapier / Make:** these version the whole workflow, not the prompt, and retention expires. Copy the prompt text into a local .md regardless. See `references/environments/automation-tools.md`.
- **Anything else:** `references/environments/generic.md`. Snapshot to a local .md the moment you can read the prompt text.

## Phase 3: Baseline

Run the current version over the eval set. Log it. This is the number to beat. If running costs money or credits (Clay always does; OpenAI and metered tools do), say so and confirm the user wants to spend before you run.

## Phase 4: One change at a time

- Apply exactly ONE change: one rule, one example, one wording fix. Never two. The delta against the previous run is that change's effect, and only if nothing else moved.
- **With-example sub-test:** if the change targets a subtle distinction, try the rule wording alone first. If it does not land, add an example-anchor (the positive case plus the contrast negative case) and re-run. The two runs show whether the example was needed. (See `examples/classifier-prompt_v1.1.md` for a worked instance.)
- **Causal attribution:** a change should only move inputs it logically targets. A flip with no causal link to the change is a candidate for noise, but verify it (Phase 5); diffuse prompt effects are real, not always noise.
- **Prefer deterministic over LLM where you can.** If a step can be a regex, a formula, a string match, or a lookup instead of a model judgment, propose that; it is cheaper, faster, and has no noise floor. (In Clay, score with a free Formula column, not another AI column. In OpenAI Evals, use a `string_check` grader, not an LLM judge.)

## Phase 5: Regression and noise (after every change)

- **Regression scan:** list every input that was correct and is now wrong. An aggregate accuracy can hold steady while specific rows silently flip, so scan per-input, not just the top-line number.
- **Noise floor:** once per (model + temperature + eval set), run the same prompt twice over the eval set and count the differing outputs. That is your noise floor for that combination. A delta at or below it is not signal. Re-measure whenever the model or temperature changes. Note: temperature 0 does NOT guarantee identical outputs in 2026; measure the floor, do not assume it is zero. Clearing the floor is necessary but not sufficient: a small gain can beat the floor and still sit inside the sampling interval at low row counts (see `references/03-inference-falseability-risk.md`).
- **Regression protocol:** if a change regresses, OR the gain is not evident above the noise floor, make at least two more attempts to fix that change. If it still does not resolve, write an *anamnese* (a short diagnosis: what was tried, why it failed, your hypotheses) and report it to the user with options for them to decide. Do not silently move on. See `references/04-anamnese-protocol.md`.

## Phase 6: Close

- Freeze the new version file. Update the changelog with the lift per version.
- **Report honestly.** A number measured on an eval set that includes the change's own target rows is fitted, not blind generalization. Always caveat which subset, fitted versus held-out, and the noise floor. A true generalization number needs fresh inputs the change never saw. See `references/03-inference-falseability-risk.md` for exactly what an accuracy number does and does not let you infer.

## Inference, falseability, risk, opportunity

This loop is an instrument for *not fooling yourself*. Four things to keep explicit at all times (full treatment in `references/03-inference-falseability-risk.md`):

- **Inference:** state what a result lets you conclude and what it does not. "v3 scores 94% (30/32) on clean human-labeled rows, fitted, above the 1-row noise floor but inside the wide CI at n=32, so suggestive not proven" is honest. "v3 is 94% accurate" is not. Clear the noise floor AND the sampling interval before calling a gain real (see `references/03-inference-falseability-risk.md`).
- **Falseability:** a change you cannot test against an eval set is not an improvement you can claim. If there is no way to falsify it, name that as the gap and either build the test or label the claim as a hypothesis.
- **Risks:** overfitting to the eval set; tuning to labels no human reviewed (circular); reading noise as signal; the model or hidden context drifting under you; runaway cost in metered environments (Clay can fan a re-run across linked tables; isolate a standalone copy).
- **Opportunities:** every loop is a chance to replace a fragile LLM judgment with a deterministic check, to split a fused prompt into a frozen part and an iterated part, and to turn a recurring failure into a reusable example-anchor.

## Hard rules

1. One change per eval run. Always.
2. Every run logged (version, model, accuracy, notes). No log, the change did not happen.
3. Never report accuracy without its caveats: which subset, fitted versus held-out, the noise floor.
4. A regression is not "probably noise" until a re-run confirms it. Verify.
5. Do not tune against labels a human has not reviewed. It is circular.
6. Do not assume the environment; ask and adapt.
7. Running a cheap A/B eval is fine to suggest and run. Spending money or credits, or pushing a changed prompt to anything live, needs the user's explicit yes.

## Map of this skill

- `references/00-how-it-works.md` , the plain-language walkthrough to offer the user.
- `references/01-environment-questionnaire.md` , the five questions, with options and the standard default.
- `references/02-the-gabarito.md` , the eval set, column by column, with how to build one.
- `references/03-inference-falseability-risk.md` , what a result means, and how not to fool yourself.
- `references/04-anamnese-protocol.md` , the regression protocol and the anamnese template.
- `references/05-the-run-log.md` , the run-log, column by column, with how to fill each.
- `references/environments/` , per-environment mechanics: `clay.md`, `openai.md`, `automation-tools.md`, `generic.md`.
- `examples/` , a complete worked example: two prompt versions, an eval set, and a run-log showing one change and its lift.

## Output style

Tight and declarative. Report at checkpoints (after the structure, after a batch of eval runs, on an anamnese), not after every single run. Explain the why behind each step; this loop works best when the user understands it, not just follows it.
