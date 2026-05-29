# Environment: OpenAI

There are four realistic surfaces where someone iterates a prompt on OpenAI, and the snapshot mechanic differs on each. Ask which one the user is in before assuming.

*Mechanics verified against OpenAI docs and developer-community posts current as of May 2026. Pricing and the free-lane limits change; confirm in the user's account.*

---

## The four surfaces and where the prompt lives

1. **Dashboard "Prompts"** , a reusable prompt object stored in the user's OpenAI project, with an ID like `pmpt_...`. Project-scoped (anyone in the project sees it). Variables written as `{{name}}`. Created and edited only in the dashboard; you cannot create one via the API as of 2026.
2. **Playground** , the GUI editor for those same prompt objects. Ad-hoc system/developer text typed here is **not saved** unless you explicitly Save it as a prompt.
3. **Plain API** , the prompt is a literal string (the `system`/`developer` message, or `instructions`) inside the user's own code. OpenAI stores nothing; it lives wherever their code lives.
4. **OpenAI Evals** , the prompt under test lives inside an eval run's template, and the eval, every run, and its report are stored server-side.

---

## Versioning and snapshot, per surface

- **Dashboard Prompts (native versioning):** edit and click **Update** to make a new version; **Publish** to lock it. Roll back via **History** -> pick a version -> **Restore**. The bare Prompt ID points to the **latest published** version and will silently change what runs the moment you publish a new draft, so for a frozen eval **pin the version explicitly** in the API `prompt` param (`{"id":"pmpt_...","version":"<n>","variables":{...}}`), never rely on "current".
- **Playground:** same Save/Update/Publish/History mechanism. Ad-hoc text is not a snapshot until you Save it.
- **Plain API:** no built-in versioning. Snapshot by committing the prompt string to git or a dated `.md`. Also **pin the model to a dated snapshot id** (not a floating alias), or the model can drift under you between runs.
- **Evals:** each run is itself an immutable snapshot of (prompt text in the run template + model + dataset). Give every run a descriptive `name` (e.g. `v3_temp0`). This is the most durable snapshot because prompt, model, data, and score are bound together in one stored run.

Default for a user not working in their own repo: keep the prompt in a dated local `.md` per version regardless of surface, so you have one durable record the dashboard cannot mutate. If they use dashboard Prompts, mirror the published `version` number into the `.md` so the two stay in lockstep.

---

## Running an eval

**Manual (any surface):** keep the prompt fixed, send it once per eval row through the Playground or a tiny script, record each output and a pass/fail against `expected_label`. To measure noise, loop the same row 3 to 5 times and look at agreement. Fully under your control; you tally accuracy yourself.

**OpenAI Evals (the product, best for repeatability):**
1. Build the dataset as JSONL, one object per line, fields under `item` (e.g. `{"item":{"input_text":"...","correct_label":"..."}}`).
2. Create the eval once with `client.evals.create(...)`, exposing `{{sample.output_text}}` (the model output) and `{{item.<field>}}` (your fields).
3. Prefer a **deterministic grader** (`string_check` with `eq` -> binary 0/1) over an LLM-judge grader (`label_model`/`score_model`). The LLM judge is itself a prompt that costs tokens per row and adds its own noise; if the judge is flaky, your accuracy delta is noise. Use it only for genuinely open-ended outputs.
4. Run a variant with `client.evals.runs.create(...)`, giving it a `name` (the prompt version). It runs over the whole dataset; read `report_url` for pass/fail counts.
5. Change ONE thing, make a NEW run with a new name and the same dataset and grader, compare reports.
6. There is **no native "run X times" or stddev.** To measure run-to-run noise, loop `runs.create(...)` N times yourself and compute the spread from the N report numbers (or from your CSV).

---

## Cost model

Metered on tokens, per run; the Evals tooling layer itself is free.

- Plain API / Playground / dashboard-prompt runs bill the model's input+output token rates on every call. One eval pass = rows x tokens; running it N times multiplies by N. Reasoning models also bill thinking tokens.
- In Evals you pay model tokens to generate each sample, plus tokens for any model grader (an LLM judge roughly doubles per-row cost). Deterministic graders are free.
- There is a free lane (shared eval runs, a small number per week) but it requires **opting to share your eval data with OpenAI** , not acceptable for confidential prompts or data. If the content is sensitive, do not opt in; pay full token cost.
- Net: re-running is cheap in effort (one call) but metered in money. Keep the eval set small (20 to 50 rows) while iterating; expand only for a final confirmation run.

---

## Clarifying questions

- Which of the four surfaces are you in? The snapshot mechanic differs completely.
- Is the eval set's `correct_label` human-verified or model-labeled? If model-labeled, the number is circular; build a hand-checked truth set first.
- Is the prompt/data confidential? If yes, the free shared-eval lane is off the table.
- Do you need run-to-run noise measured (loop the same config K times), or just one pass per version?
- Which model and temperature, and can you pin a dated model snapshot id? Floating aliases drift and break week-over-week comparisons.
