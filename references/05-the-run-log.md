# The run-log, column by column

The run-log is the memory of the loop. One row per eval run, appended and never overwritten, it is how you prove (to yourself, later) that a version was actually better and what you changed to get there. Without it, you will misremember which version scored what, and the discipline collapses into vibes.

The canonical schema, used identically in every environment:

```
date, version, model, temperature, dataset, n_clean, accuracy_clean, noise_floor, change_made, regressions, notes
```

The filled template is [`../examples/run-log-template.csv`](../examples/run-log-template.csv). Here is what each column is for, an example value, and how to fill it.

| Column | What it is for | Example | How to fill it |
|---|---|---|---|
| `date` | When the run happened | `2026-05-29` | The run date, ISO format. |
| `version` | Which prompt version was run | `v1.1` | Match the snapshot filename (`<name>_v1.1.md`). This ties the score to an exact, frozen prompt. |
| `model` | The model that produced the outputs | `gpt-4o-mini` | The exact model. If you can pin a dated snapshot id, use it; the model is part of the experiment, and a model swap invalidates a baseline. |
| `temperature` | The sampling temperature | `0` | The setting used. If the environment hides it, write `unknown`. |
| `dataset` | Which eval set was scored | `eval-set-template.csv` | The eval-set file or table name. If you change the set, that is a new dataset; do not compare scores across different sets as if they were the same. |
| `n_clean` | How many rows the score is over | `22` | The count of `clean`, human-labeled rows scored. This is the denominator; a score without it is meaningless. |
| `accuracy_clean` | The headline accuracy | `0.955` | (correct clean rows) / `n_clean`. Score on clean, human-labeled rows only. Provisional `ai`-labeled rows and `needs_context` rows do not count. |
| `noise_floor` | The run-to-run jitter for this setup | `1 row` | From the noise check: run the same prompt twice over the eval set and count differing outputs. Leave blank on runs where you did not measure it; re-measure when model or temperature changes. A gain at or below the floor is not signal. |
| `change_made` | The ONE thing changed vs the previous version | `added authority-not-seniority example-anchor to rule 1` | Describe the single change. `baseline` for the first run; `noise re-run (no change)` for a noise check. If you cannot name one single change, you changed too much. |
| `regressions` | What broke | `none` | List any previously-correct rows that became wrong (e.g. `2 rows: founder titles`). `n/a` on a baseline. This column forces the per-row regression scan; a flat top-line can hide churn underneath. |
| `notes` | Everything else worth remembering | see template | The honest caveats: fitted vs held-out, sample-size/CI warnings, why a change was kept or reverted, anything surprising. This is where you keep yourself honest. |

## A worked sequence (from the template)

| version | n_clean | accuracy_clean | noise_floor | change_made | regressions |
|---|---|---|---|---|---|
| v1.0 | 22 | 0.818 | (not measured) | baseline | n/a |
| v1.0 | 22 | 0.818 | 1 row | noise re-run (no change) | n/a |
| v1.1 | 22 | 0.955 | 1 row | added authority-not-seniority example-anchor | none |

Read across: the baseline scored 0.818 on 22 clean rows; a re-run of the same prompt established a noise floor of about 1 row; v1.1's one change lifted accuracy by 3 rows (above the floor) with no regressions. The template's `notes` then add the honest caveat: a 3-row gain on 22 rows is promising but not proven, because the change was fitted to those rows and the confidence interval at n=22 is wide. That caveat is the difference between a measurement and a boast (see [`03-inference-falseability-risk.md`](03-inference-falseability-risk.md)).

## Where the run-log lives

A local CSV is the standard (it depends on nothing and diffs cleanly). A tab in the same spreadsheet as the eval set works too. Wherever it lives, the rule is the same: append one row per run, never overwrite, and keep the schema identical so the rows stay comparable.

In an environment where you (the assistant) cannot write files (a chat UI, Clay, a console), you cannot create this CSV on the user's disk. Emit the header and each new row as text in your reply and have the user paste it into their own file. The run-log is a file the user owns either way; only who types it changes.
