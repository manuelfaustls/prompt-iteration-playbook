# Inference, falseability, risk, opportunity

This loop is an instrument for not fooling yourself. The numbers it produces are easy to over-read, and an over-read number is worse than no number, because it carries false confidence into a decision. This file makes the four things you must keep explicit, explicit.

---

## 1. Inference: say exactly what a result lets you conclude

An accuracy number is a measurement under specific conditions. State the conditions, every time. The difference between an honest claim and a misleading one is entirely in the caveats.

**Misleading:** "v3 is 94% accurate."

**Honest:** "v3 scored 94% (30/32) on 32 clean, human-labeled rows, versus v2's 88% (28/32). The change that produced v3 was fitted to 2 of those rows (it was written to fix them), so this is a fitted number, not blind generalization. The re-run noise floor is 1 row, so the 2-row move is above pure run-to-run jitter; but on 32 rows the 95% confidence interval is roughly plus or minus 9 points and overlaps v2, so this is suggestive, not significant. Confirm it by holding out fresh rows or enlarging the set."

Note the two separate gates in that claim, because conflating them is the most common error here:

- **Gate 1, the re-run noise floor.** Run the *same* prompt twice and count differing outputs. This measures one version's run-to-run stochasticity. A between-version delta must clear this floor to be more than jitter. It is necessary, not sufficient.
- **Gate 2, the sampling error.** Even with zero run-to-run noise, your accuracy is estimated from a finite sample, so it carries a confidence interval. On 32 rows the 95% interval around 90% is roughly plus or minus 9 points; on 20 rows it is wider still (about plus or minus 13). A small between-version gain can clear the noise floor and still sit inside the sampling interval, which means you cannot yet call it real. The fix is more rows, or a held-out confirmation.

Never present a fitted, small-sample gain as flatly "real." It is, at most, "promising, confirm it."

What every reported number should carry:

- **Which subset.** Accuracy on `clean` rows only. Report the `needs_context` ceiling separately if it matters; never blend them into one number.
- **Fitted vs held-out.** If the change was written to fix rows that are *in* the eval set, the score on those rows is fitted: it tells you the change did what you intended, not that it generalizes. A true generalization number needs fresh inputs the change never saw. When you can, hold out a slice of the set that you never look at while iterating, and report on it at the end.
- **Sample size.** "94% on 32 rows" not "94%". On 20 rows, one row is 5 points; small deltas are meaningless.
- **The noise floor.** State it next to the delta so the reader can see whether the change cleared it.

## 2. Falseability: a claim you cannot test is not a result

The whole method rests on being able to *refute* a change. If there is no way the eval set could show a change to be bad, then a "good" result tells you nothing.

- **No eval set, no claim.** If you cannot build a gabarito for a task, you can still write the prompt, but you cannot claim it improved. Say "hypothesis, untested" and stop pretending otherwise.
- **A change must predict a specific, checkable effect.** "This rule should flip the senior-IC titles from decision_maker to influencer" is falseable: you look at exactly those rows after the run. "This makes the prompt better" is not; it predicts nothing you can check.
- **Circular tests are not tests.** If the eval labels came from the same model family you are tuning, the test cannot refute anything; the prompt and the answer key share a bias. This is why `labeler` matters (`references/02-the-gabarito.md`).
- **With only provisional (AI) labels, triage is allowed; scoring is not.** When every label was produced by a model and no human has reviewed them, and a review pass is not feasible this week, you may still use the set to surface candidate failure cases and to decide what to review first. You may NOT report an accuracy number or claim an improvement from it. The smallest honest unlock is 20 clean rows a human has confirmed; until then, say "directional only, not measured."
- **If a result cannot be falsified, name the gap.** Tell the user "we cannot measure this with the current set" and either build the missing test or label the change as an unverified hypothesis. Do not let an unfalseable change ride along as if it were proven.

## 3. Risks: the ways this loop quietly goes wrong

- **Overfitting to the eval set.** Iterate long enough and you will fit the prompt to the quirks of these specific rows, not the underlying task. Defenses: keep a held-out slice you never tune against; expand or refresh the set periodically; be suspicious when accuracy on the fitted set is far above accuracy on fresh inputs.
- **Tuning to unreviewed truth.** Covered above and worth repeating: AI-labeled rows are provisional. Tuning to them is the most common way to manufacture a meaningless high score.
- **Noise read as signal.** A delta below the measured noise floor is not an improvement. Always have the floor in hand before you celebrate a small gain. (Temperature 0 is not a determinism guarantee in 2026; measure the floor empirically.)
- **Silent drift.** The model can be repointed by the vendor, a hidden system prompt or prior conversation turns can change, defaults like temperature can move. Any of these invalidates a baseline without telling you. Pin and log the model and parameters; start eval runs from a clean state; re-baseline when something underneath changes.
- **Runaway cost.** Some environments meter every run. Clay charges credits per row on every re-run and can fan a re-run across linked tables, which has produced surprise charges in the hundreds of dollars. Estimate the cost of a batch before running it, isolate a standalone eval table, and make the user opt into spending.
- **Repeated testing against one fixed set.** Trying many one-change variants against the same small eval set and keeping whichever clears the noise floor is multiple comparisons: with enough attempts, a chance gain will look real. Defenses: confirm a kept gain on a held-out slice the variants never touched; be more skeptical of the Nth marginal win than the first; re-measure the noise floor periodically.
- **Copy-paste corruption.** Moving a prompt between a tool and a file can inject smart quotes, em-dashes, or strip newlines, changing behavior in ways that look like "the change did something." Diff what you pasted against the source.

## 4. Opportunities: what a good loop unlocks

The loop is not only defense. Each iteration is a chance to make the whole system better, not just the prompt:

- **Replace an LLM judgment with a deterministic check.** If a step can be a regex, a string match, a formula, or a lookup, it should be: deterministic checks are cheaper, faster, and have no noise floor. Score classifiers with a string match or a formula, not a second LLM call. Push exact, rule-based exclusions (a banned-category list, a format check) into code or a formula column, not into the model's prompt. (In Clay, that is a free Formula column; in OpenAI Evals, a `string_check` grader.)
- **Split a fused prompt into a frozen part and an iterated part.** If one prompt both extracts data and judges it, separate them: freeze the extraction, iterate only the judgment. Now a re-run only re-pays for (and only risks changing) the part you are actually tuning. This is the "two-agent split."
- **Turn a recurring failure into a reusable example-anchor.** When the prompt keeps missing a specific kind of case, a single well-chosen example pair (the positive case plus the contrasting negative case) often fixes it where more words do not. Keep the good anchors; they are durable.
- **Let the eval set teach you about the task.** The rows the prompt keeps getting wrong are often telling you the *labels* are fuzzy or the task is genuinely ambiguous, not that the prompt is dumb. Fixing the label definitions sometimes beats any prompt change.

---

## A one-line honesty test

Before you report a result, ask: *"If someone competent and skeptical read this claim, is there anything true I am leaving out that would change how much they trust it?"* If yes, add it. That sentence is the difference between a measurement and a sales pitch.
