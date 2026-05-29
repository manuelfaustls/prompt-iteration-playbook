# Phase -1: Environment questionnaire

Run this before anything else. The loop is identical everywhere, but the mechanics and the **cost** of a single run depend entirely on where the prompt lives. Getting this wrong wastes the user's money (some environments meter every run) or silently breaks the measurement.

**Rule for every question: offer 2-3 labeled options and mark the standard one that works anywhere.** A user who does not want to think should be able to pick the standard and move on. A user with a specific setup can say so.

**Rule when you do not recognize an answer: ask, do not assume.** If the user names a tool you have no environment file for, drop to [`environments/generic.md`](environments/generic.md) and ask its clarifying questions. Never invent a mechanic.

---

## The five questions

### 1. Where does the prompt live?

- **A. In a file in my own code / repo** (Claude Code, a Python script, a notebook). Versioning is git; running is a script. Cheapest to iterate.
- **B. In a SaaS tool's AI step** (Clay AI column, OpenAI dashboard/Playground, n8n/Zapier/Make node). Mechanics vary a lot; read the matching `environments/` file.
- **C. Somewhere else / a custom or internal tool.** Use [`environments/generic.md`](environments/generic.md) and ask whether the prompt text can be copied out as plain text.
- **Standard if unsure:** treat the prompt text as the thing to snapshot, wherever it is, into a local `.md` file. That works no matter the answer.

### 2. Where does the eval set (gabarito) live, or do we need to build one?

- **A. I have one** (a CSV, a Google Sheet, an Airtable, rows in a Clay table). Confirm it has the four columns from [`02-the-gabarito.md`](02-the-gabarito.md); add the missing ones.
- **B. I have labeled examples but no structure.** Help shape them into the four-column format.
- **C. I have nothing yet.** Offer to scaffold a local CSV from the template in `examples/eval-set-template.csv`.
- **Standard if unsure:** a local CSV with `input, expected_label, labeler, categoria`. It works everywhere and you own it.

### 3. Which model and temperature?

- **A. I know exactly** (e.g. `gpt-4o-mini`, temperature 0). Pin it; record it in every run-log row.
- **B. I am not sure / the tool hides it.** Find it together; if it cannot be pinned, log `model=env default as-of <date>` and re-baseline if it changes.
- **C. I want to compare a few models.** Fine, but treat the model as a variable too: change the model OR the prompt between runs, never both at once.
- **Standard if unsure:** pin one model at temperature 0 and hold it fixed while you iterate the prompt. (Temperature 0 reduces, but does not eliminate, run-to-run noise; you will measure the real floor in Phase 5.)

### 4. Who labeled the eval set?

- **A. A human** (you or a teammate). This is ground truth; you can tune against it.
- **B. A model or an agent**, not yet human-reviewed. Provisional only. **Do not tune the prompt to these labels**; it is circular. Get a human to review them first.
- **C. A mix.** Add a `labeler` column so each row is marked, and score only against the human-reviewed rows.
- **Standard if unsure:** add the `labeler` column now and treat any unreviewed label as provisional until a human confirms it.

### 5. Where should the run-log go?

- **A. A local CSV** next to the prompt snapshots. Append one row per run. **This is the standard, and the playbook offers to create it for you.**
- **B. A new tab in the same spreadsheet** as the eval set, if the gabarito lives in Sheets/Excel and the user prefers one place.
- **C. Somewhere else** (Notion, a pinned message). Fine, as long as it is append-only and one row per run.
- **Standard:** option A. A local CSV depends on nothing and diffs cleanly. Offer to create `run-log.csv` from `examples/run-log-template.csv`.

---

## Teach the shortcut

After the user answers, tell them they can skip these questions next time by declaring the setup in the invocation. For example:

```
/optimize-prompt-portable env=clay model=gpt-4o-mini eval=clay-table log=local-csv labeler=human
```

```
/optimize-prompt-portable env=openai-api model=gpt-4o-mini@2026-xx-xx eval=csv:evals/set.csv log=local-csv
```

The shortcut grammar (accepted keys and values):

- `env=` `local` | `clay` | `openai-dashboard` | `openai-api` | `n8n` | `zapier` | `make` | `other`
- `model=` any model id; append `@<date>` to pin a dated snapshot (e.g. `gpt-4o-mini@2026-05-29`)
- `eval=` a path or a URL, prefixed by kind: `csv:<path>`, `sheet:<url>`, `clay-table`, or `airtable:<url>`
- `log=` `local-csv` | `sheet-tab` | `<path>`
- `labeler=` `human` | `ai` | `mixed`

If they declare some fields, only ask for the ones still missing. Always echo back your interpretation of the declared setup and confirm before running.

---

## Echo and confirm

Before moving on, state your understanding in one block and ask for a yes:

```
Here is what I have:
- Prompt lives in: <answer>
- Eval set: <answer> (<n> rows, <labeler mix>)
- Model + temperature: <answer>
- Run-log: <answer>
Read environments/<env>.md for the mechanics. Anything to correct before we baseline?
```

If the environment is one you have no file for, say so plainly and switch to the generic contract rather than guessing how the tool behaves.
