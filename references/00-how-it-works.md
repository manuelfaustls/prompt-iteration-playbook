# How it works (the walkthrough to offer the user)

Offer this explanation at the start of every session, before running a baseline. The loop works far better when the user understands *why* each step exists than when they just follow steps. Keep it to a couple of minutes; adapt the depth to the person.

## The problem this solves

You have a prompt (a classifier, a filter, a qualifier, an extractor) and you want it to be more accurate. The tempting way is to read a few wrong outputs, rewrite the prompt until they look right, and ship it. The trouble is you have no idea whether you actually improved anything or just moved the errors somewhere you did not look. You might even have made it worse.

This playbook replaces "rewrite until it looks right" with "change one thing, measure, keep it only if the number moved."

## The mental model: a change is a hypothesis

Think like a scientist, not an editor.

- A prompt edit is a **hypothesis**: "adding this rule will fix the titles it gets wrong."
- The **eval set** (the gabarito) is your experiment: a fixed set of inputs whose correct answers you already know.
- Running the prompt over the eval set is the **test**: it either supports the hypothesis (accuracy went up, nothing else broke) or refutes it (no change, or something else broke).
- A hypothesis you **cannot test** is not knowledge. If there is no eval set, there is nothing to falsify, and "it looks better" is just a feeling.

## The five rules that keep you honest

1. **One change at a time.** If you change two things and the score goes up, which one helped? You do not know. Maybe one helped and one hurt. Isolate every change so the result means something.

2. **Log every run.** Version, model, temperature, the accuracy, and the one thing you changed. A run you did not log is a result you will misremember. The log is also how you prove (to yourself, later) that a version was actually better.

3. **Scan for regressions.** A new rule that fixes three inputs might break two others. The top-line accuracy can even stay flat while the specific rows churn underneath. After every change, check which previously-correct inputs just became wrong.

4. **Measure the noise floor.** Most LLMs are not perfectly repeatable; even at temperature 0 in 2026, the same input can yield different outputs run to run. So run the same prompt twice over the eval set and count how many outputs differ. That number is your noise floor. If a "gain" is smaller than the floor, it is not a gain; it is noise. Re-measure the floor whenever you change the model or temperature.

5. **Never tune to unreviewed labels.** Your eval set's "correct" answers have to be correct. If a model (or an agent) produced those labels and no human checked them, then tuning the prompt to match them just teaches the prompt to agree with another model. You will get a high number that means nothing. Human-reviewed labels are truth; model labels are provisional until a human confirms them.

## What you walk away with

- A prompt that is measurably better, with the numbers to show it.
- A frozen file for every version, so you can always go back.
- A run-log that tells the story: what you tried, what it scored, what you kept.
- Honest claims: not "this is 95% accurate" but "this scored 95% on 40 clean human-labeled rows, with the change fitted to two of them, noise floor one row."

## How to use it

- In **Claude Code**: invoke the skill and answer the five environment questions; the agent runs the loop with you.
- In **Clay, ChatGPT, a notebook, or any tool**: read `SKILL.md` and run the phases yourself. The playbook is written to be followed by a person.

When you are ready, the agent will ask the five environment questions (`references/01-environment-questionnaire.md`) and then help you build or check the eval set (`references/02-the-gabarito.md`).
