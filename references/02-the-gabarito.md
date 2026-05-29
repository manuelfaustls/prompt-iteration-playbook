# The gabarito (eval set), in depth

The gabarito is a set of inputs whose correct answers you already know. It is the single most important thing in this playbook, because every accuracy number you will ever report is measured against it. A loop built on a sloppy eval set produces confident, wrong conclusions. Build this carefully and the rest of the loop is reliable.

"Gabarito" is the Portuguese word for an answer key. That is exactly what this is: the answer key you grade the prompt against.

> The golden rule: **a prompt loop is only as trustworthy as the eval set it is measured against.** Spend your care here.

---

## The four required columns

Every gabarito needs these four columns at minimum. Below is what each is for, how to fill it, and worked examples.

### `input`

**What it is for:** the exact input the prompt will see at runtime, one per row. This is what you feed the prompt during an eval run.

**How to fill it:** copy a real input verbatim. Do not paraphrase or clean it up; if the prompt has to handle messy real-world inputs, the eval set should contain messy real-world inputs. If the prompt normally receives several fields (a title plus a company note, say), put the same combination here, formatted the way the prompt actually receives it.

**Examples (a contact-role classifier):**

| input |
|---|
| `Founder & CEO` |
| `Senior Growth Analyst` |
| `Marketing Intern at a 12-person Shopify store` |

**Scenario:** if your prompt classifies support tickets, `input` is the ticket text. If it segments companies, `input` is the company row (name, industry, size, whatever the prompt reads). If it qualifies leads from a scraped profile, `input` is the scraped text. Match reality.

### `expected_label`

**What it is for:** the known-correct answer for that input. This is the truth you grade against.

**How to fill it:** decide the correct output as a human who understands the task. Use the exact label set the prompt is supposed to produce (same spelling, same casing), so scoring can be a simple match. For a `needs_context` row (see `categoria` below), leave this **blank** on purpose; the point of that row is that there is no correct answer from the input alone.

**Examples:**

| input | expected_label |
|---|---|
| `Founder & CEO` | `decision_maker` |
| `Senior Growth Analyst` | `influencer` |
| `Marketing Intern at a 12-person Shopify store` | `not_relevant` |
| `Manager` | *(blank: which function? cannot tell)* |

**How to fill it well:** keep the label vocabulary small and unambiguous. If two labels overlap so much that you, the human, hesitate, the prompt will too; tighten the label definitions before blaming the prompt.

### `labeler`

**What it is for:** records who decided the correct answer, so you know which rows are trustworthy enough to tune against.

**How to fill it:** write `human:<name>` for a row a person reviewed (e.g. `human:manuel`), or `ai:<model>` for a row a model or agent labeled and no human has checked (e.g. `ai:gpt-4o-mini`).

**Why it matters (this is load-bearing):** if you tune a prompt to match labels that a model produced, you are not making the prompt more *correct*, you are making it agree with another model. The accuracy number will climb and mean nothing. So:

- Score and tune against `human:` rows only.
- Treat `ai:` rows as provisional. They are useful for triage and for finding candidate cases, but a human must confirm a label before it counts.

**Examples:**

| input | expected_label | labeler |
|---|---|---|
| `Founder & CEO` | `decision_maker` | `human:manuel` |
| `Sales Director` | `decision_maker` | `ai:gpt-4o-mini` |

The second row does not count toward your score until a human reviews it and flips the labeler to `human:`.

### `categoria`

**What it is for:** classifies the *row itself*, so you score on the rows that are actually decidable and do not punish the prompt for the others. This is the column people most often skip, and skipping it is why eval numbers lie.

**The three values:**

- **`clean`** , the input alone is enough for a competent human to decide the correct answer. **You score accuracy on `clean` rows only.** This is the real number.
- **`needs_context`** , the input is genuinely insufficient to decide (a missing field, an ambiguous reference, a name with no role). No prompt could reliably get this right from the input alone. These rows are not misses; together they mark the *ceiling* of what any prompt could achieve on your data. If the prompt guesses one right, that is luck, not skill, so do not count it either way. (When you find yourself wanting to add a rule to "fix" a `needs_context` row, stop: the fix would be the prompt guessing, and guessing does not generalize.)
- **`out_of_scope`** , the row should not be in the eval set at all: a malformed entry, a duplicate, an input from a different task, a company name where a person title was expected. Remove these. They are neither a hit nor a miss; they are noise in the set.

**How to fill it:** for each row ask, "could I, a competent human, decide the correct answer from this input alone?" Yes and the input belongs to this task -> `clean`. Genuinely not enough information -> `needs_context`. Does not belong here -> `out_of_scope`.

**Examples:**

| input | expected_label | categoria | why |
|---|---|---|---|
| `Head of Growth` | `decision_maker` | `clean` | role and authority are clear |
| `Manager` | *(blank)* | `needs_context` | manager of what? cannot decide |
| `Acme Retail Co` | *(blank)* | `out_of_scope` | a company, not a person title |

---

## Optional columns that earn their place

- **`actual`** , where you paste (or your script writes) the prompt's output for a given run, so you can compare it to `expected_label`. In a multi-run setup, name it per version (`actual_v1`, `actual_v2`) or keep run outputs in separate files.
- **`id`** , a stable key per row. Useful when results come back unordered (some batch environments) so you can align outputs to inputs. Recommended for any set above ~30 rows.
- **`notes`** , a free-text note on why a row is labeled the way it is, or what makes it a tricky case. Future-you will thank present-you.

---

## How big does the set need to be?

- **At least 20 `clean`, `human`-labeled rows** before you report any accuracy number to anyone. Below that, the number is too noisy to trust: a single row is five percentage points on a 20-row set, and a three-point "improvement" on such a set is almost certainly noise. Note that n=20 is a floor for reporting at all, not a precision guarantee: even at n=20 the 95% confidence interval on a ~90% result is roughly plus or minus 13 points. Treat the interval, not the point estimate, as the claim; larger sets tighten it.
- More is better, up to a point. 30 to 80 clean rows is a comfortable working range for iteration. You do not need thousands.
- Cover the **hard cases on purpose.** A set of only obvious inputs will show 100% and teach you nothing. Seed it with the near-misses, the ambiguous-but-decidable cases, and the categories the prompt currently confuses. Those rows are where the iteration actually happens.

---

## Building a gabarito from scratch

If the user has nothing, offer to scaffold one (the standard path is a local CSV; the template is `examples/eval-set-template.csv`):

1. **Pull real inputs.** Take 30 to 80 actual inputs from the real source. If the prompt already runs in production, sample its real inputs (random, not cherry-picked) so the set reflects reality.
2. **Have a human label them.** The person who understands the task fills `expected_label`. Mark each `labeler: human:<name>`.
3. **Tag `categoria`.** Walk each row through the clean / needs_context / out_of_scope question above.
4. **Remove `out_of_scope` rows.** They do not belong.
5. **Check the count.** Are there at least 20 `clean` human-labeled rows? If not, label more before measuring.

If the user only has AI-generated labels, you can use them to *draft* the set, but mark every row `ai:<model>` and tell the user plainly: no accuracy number is trustworthy until a human reviews these labels. Offer to set up a quick review pass.

---

## Where the gabarito lives, per environment

The four-column structure is the same everywhere; only the container changes.

- **Local CSV** (the standard): columns as above. Works for every environment, you own it, it diffs in git.
- **Google Sheet / Excel:** same columns, one sheet. Convenient when the team already works there; n8n's evaluation feature can read a Google Sheet directly.
- **Clay table:** each eval case is a row; add a hand-typed `expected_label` column (never AI-run, so it costs no credits) and score with a free Formula column comparing the AI output to `expected_label`. See [`environments/clay.md`](environments/clay.md).
- **OpenAI Evals:** the set becomes a JSONL file, one object per line, with your fields under `item`. See [`environments/openai.md`](environments/openai.md).

---

## The gabarito audit (print this before Phase 3)

Before the baseline run, summarize the set so the user can see exactly what is being measured and whether it is enough:

```
GABARITO AUDIT
total rows: 47
  clean:          32   (28 human-labeled, 4 ai-labeled)
  needs_context:  12   (all human-labeled)
  out_of_scope:    3   (remove these)

scorable now (clean AND human-labeled): 28 rows
status: above the n=20 threshold, good to measure
warning: 4 clean rows are ai-labeled and will NOT count until a human reviews them
```

If the scorable count is below 20, **pause and say so.** Do not produce a headline accuracy number from a set too small to support one; either label more rows or report only with an explicit small-sample caveat.
