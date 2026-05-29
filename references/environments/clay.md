# Environment: Clay

Clay (clay.com) runs the prompt inside an AI column in a workspace table. There are no local files and **every run costs credits**, so the iteration loop here is not free the way a local script is. Two things make Clay different from every other environment, and getting them right is the whole game:

1. **There are two AI surfaces with opposite versioning behavior** (classic "Use AI" column vs the Claygent builder). Never say "Clay has no versioning" flatly; ask which one.
2. **Re-running an eval spends data credits every single pass.** The "run twice to measure the noise floor" step is real money. Make the user opt in.

*Mechanics verified against Clay docs, May 2026. Confirm anything cost-related in the user's own workspace before relying on it; pricing changes.*

---

## Where the prompt lives

**Surface A , classic "Use AI" column.** Add column -> Use AI. Two tabs:
- *Generate*: describe the goal in plain English; Clay auto-builds the prompt, model, and output fields (a "Help me" metaprompter button exists).
- *Configure*: the advanced tab holding the **actual editable Prompt field**, the **model selector**, and **"Add and define outputs"** (output field names plus a data type or JSON schema). The prompt text you iterate sits in the Configure tab's Prompt box. You reference other columns inside the prompt by typing `/` to insert a column token.

**Surface B , Claygent builder.** A separate, newer hub where you build, test, and deploy a "Claygent" conversationally. Here the prompt is the source of truth in the builder, and the deployed table column mirrors it.

Both store the prompt in Clay's cloud, not on disk. The official Clay MCP exposes research/enrichment tools only; it **cannot read, author, or version a column's prompt**. So pulling the prompt text out is copy-paste by hand.

---

## Versioning and snapshot (this forks by surface)

**Surface A (classic "Use AI" column): no built-in versioning at all.** No history, no diff, no undo for the prompt or the column config. So **snapshot manually, and this is your point-6 default:**

1. Open the AI column -> Configure tab -> click into the Prompt box -> select all -> copy.
2. Paste the prompt text into a local `.md` (or `.csv`) file next to your notes.
3. **Also copy two things that are part of the version:** the **model name** from the selector, and the **output field definitions** (names plus data types / JSON schema). Changing the model or the output schema changes behavior just as much as changing the words, so a prompt-only snapshot is an incomplete version and will confound your A/B.
4. Name the file so versions sort, e.g. `clay_<table>_<column>_v3_2026-05-29.md`. One file per version; never overwrite.

**Surface B (Claygent builder): native versioning.** Per Clay docs (May 2026): "Every change you make is saved automatically. You can view all previous versions, see what changed between versions, and roll back to any previous version." If the prompt is (or can be) a Claygent, you get version history and rollback for free and do **not** need the manual `.md` snapshot for the prompt body. You still keep the run-log locally, because Clay versions the prompt, not your accuracy and noise measurements.

**Recommendation to surface to the user:** if this is a net-new classifier you expect to iterate a lot, consider building it as a Claygent to inherit native versioning and free test cases. If it is an existing in-table "Use AI" column they do not want to rebuild, use the manual `.md` snapshot.

---

## Running an eval

### Path A , in-table eval (classic "Use AI" column), credit-metered

1. **Build the eval set as rows in a Clay table**, one row per case. Add a hand-typed `expected_label` column for the known-correct answer (the gabarito). Because you type it, it is never AI-run and costs no credits.
2. **Split scrape from analysis (the two-agent split).** Keep any scrape/extract step in its own upstream column (frozen) and the analysis/label prompt you are tuning in a separate downstream AI column. Then a re-run only re-spends credits on the analysis you are actually changing, not the scrape. If the two are fused in one column, every re-run also re-pays for the scrape, multiplying your cost.
3. **Turn auto-update OFF** on the analysis column (Edit column -> Run settings; it is off by default). This stops silent credit burn and, critically, stops a re-run from cascading into downstream auto-run tables.
4. **Run a controlled slice.** Right-click the column -> "Run Column" -> "Choose number of rows to run" -> set the count (e.g. 20) and the start row. Or select rows with checkboxes and use the row-actions menu, which shows a cost estimate first.
5. **Score for free with a Formula column.** Add column -> Formula (JavaScript / FormulaJS; `/` to reference columns). Write an `IF` that compares the AI-output column to `expected_label` and returns `MATCH` or `MISS`. Formula columns do not consume data credits, so scoring is free. Count MATCH vs MISS for accuracy. (This is the deterministic-over-LLM principle: score with a formula, never a second AI column.)
6. **To re-run after a prompt edit,** re-trigger "Run Column -> Choose number of rows" on the same slice. **This spends credits again every time.**

### Path B , Claygent builder eval (free for small sets)

The builder lets you import rows as test inputs (up to 10 test cases at a time, free; test runs do not cost credits) and switch models to compare side by side. The catch: the builder has no native expected-output or grading field, so you compare outputs to your gabarito by eye, or export and diff locally. Best when your eval set is 10 cases or fewer and you want zero-cost iteration.

---

## Cost model (the defining constraint)

Every AI prompt is one Action and consumes **data credits per row**; re-running a column re-charges those rows every pass. One eval pass over N rows costs N x per-row credits; running it twice for a noise check doubles that.

- **Fixed-price models** (about 80% of Clay's models, including Clay's own) show a flat cost like "3/row"; the charge is exact.
- **Variable-price models** (the advanced reasoning models) show an estimate like "~2/row", withhold credits up front, then reconcile after each row. The same prompt on the same rows can cost slightly different amounts run to run, so treat **accuracy** as the signal and **cost** as a budget line, not a deterministic number.
- You see the exact per-row cost only **after** a run. Clay recommends running on 10 or 50 rows first to learn the true per-row charge before going wider. Do that.
- **Two free levers:** Formula-column scoring (no data credits) and Claygent builder test runs (free, up to 10 cases). Lean on both.
- **Scale hazard:** Clay "auto-run" tables fan a source change out across all downstream tables, and dedupe is only within-table. A careless re-run across linked tables can fan out into a large surprise charge (hundreds of dollars). Iterate on a **standalone** eval table with auto-run OFF, never one wired into a downstream chain.

---

## How to run the loop here, end to end

1. Snapshot the current prompt (+ model + output schema) to a local `.md`. This is the baseline version.
2. Confirm the eval table is standalone with auto-update OFF; confirm the `expected_label` column is filled and human-labeled.
3. Tell the user the per-row credit cost and the cost of one pass over N rows. Get an explicit yes to spend.
4. Run the slice. Score with the Formula column. Log the run (date, version, model, n, accuracy, est credits, the one change) in a local CSV run-log.
5. Make ONE change in the Configure tab. Snapshot the new version to a new `.md`. Re-run the same slice. Log it.
6. Regression scan: which MATCH rows became MISS? Noise check (opt-in, costs credits): run the same version twice on a small slice and count flips.
7. Keep or revert per the result; follow the [anamnese protocol](../04-anamnese-protocol.md) on a regression or a flat result.

---

## Clarifying questions to ask before iterating in Clay

- Which surface is the prompt in: a classic in-table "Use AI" column (no native versioning -> use the `.md` snapshot) or a Claygent in the builder (native versioning + 10 free test cases)? (Standard if unsure: assume a classic "Use AI" column and snapshot the prompt to a local `.md`.)
- Is the eval table standalone, or wired into downstream auto-run tables? If linked, isolate a copy first to avoid a fan-out credit blowout.
- What is the per-eval credit budget and the acceptable row count N per pass?
- Which model does the column use, and is it fixed-price or variable-price?
- Is `expected_label` already filled and human-labeled, or do we build and label the gabarito first?
- Is the scrape already split from the analysis into separate columns? If fused, split it before iterating so re-runs only re-pay for the analysis.
