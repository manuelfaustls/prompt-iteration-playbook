# Environment: automation tools (n8n, Zapier, Make)

These are workflow builders where the LLM is one node or module inside a larger flow, not a standalone prompt console. The key divergence: **n8n has a real built-in eval framework; Zapier and Make do not.** And none of the three version a prompt by itself.

*Verified against current docs and community posts, May 2026. Plan-gated features and retention windows change; confirm the user's plan.*

---

## Where the prompt lives

- **n8n:** inside the **Basic LLM Chain** node (or AI Agent node), in the "Prompt (User Message)" field (Source = "Define below"). The model and key live in a separate Chat Model sub-node wired to it.
- **Zapier:** inside the **AI by Zapier** app, action "Analyze and Return Data", in the Prompt field (with separate Input context fields and Output fields). Runs on OpenAI behind the scenes; no separate AI account.
- **Make:** inside the **OpenAI module** (the messages/prompt fields) or the newer "Simple Text Prompt" module.

In all three the prompt is a config field on one node, addressable only by opening that node in the editor.

---

## Versioning and snapshot

**None of them version a single node's prompt.** They snapshot the whole workflow/scenario, plan-gated, with retention that expires:

- **n8n:** Workflow history (three-dots menu -> Workflow history). Retention: 24h for all users, ~5 days on Cloud Pro, full on Enterprise. Cleaner: **export the node JSON** (Download workflow, or select the node and Ctrl+C copies its JSON).
- **Zapier:** each publish makes a Zap version; version history and rollback exist only on **paid plans** (Professional and up), with plan-gated retention.
- **Make:** scenario editor -> "previous versions" icon; retention roughly 30 to 90 days by plan.

**Default regardless of tool: copy the prompt text into a local `.md` per version.** This is the only portable, diffable record and the one move that works identically across all three, because (a) in-app retention expires, (b) they version the whole flow not the prompt, and (c) you cannot diff just the prompt text. In n8n, also save the exported node JSON next to the `.md`. In Zapier and Make there is no clean per-node export, so the copy-paste `.md` is your snapshot.

> If you commit n8n node JSON to a public repo, scrub credential names first; the export embeds them.

---

## Running an eval

**Regime A , n8n (the real way):** n8n has a built-in Evaluations feature. Put the eval set in a Google Sheet or n8n Data Table (one row per case: input columns + expected output). Add an **Evaluation Trigger** node that feeds rows through the workflow one at a time, then an **Evaluation** node that writes the output back and computes a metric. For a classifier use the **Categorization** metric (exact match = 1, else 0); that averaged column is your accuracy. (Prefer it over the AI-graded Correctness/Helpfulness metrics, which are extra LLM calls with their own cost and noise.) Re-run the whole evaluation after each prompt edit.

**Regime B , Zapier and Make (no eval feature, manual):** there is no dataset/eval node. You iterate one case at a time in the editor:
- *Zapier:* open the AI step -> Test step -> it reads one trigger sample and returns the output; change the sample to test another input. Testing in the editor does **not** count as a task, so single-case iteration is effectively free of task cost (Zapier still calls OpenAI for you). A live Zap charges 1 task per AI step, so running a real N-row eval through a published Zap burns N+ tasks.
- *Make:* "Run once" with one input, read the output bubble; change input, "Run once" again. **"Run once" is metered** , every test consumes operations and OpenAI tokens.

To run a whole eval set in Zapier or Make, you fire the workflow once per input row (e.g. loop a Sheet/Tables trigger over N rows), which is clumsy and, on Make, metered per run. Record each output and a pass/fail you judge into your local CSV run-log.

---

## Cost model

- **n8n self-hosted (Community Edition):** executions are free and unlimited; you pay only the server and the LLM provider's tokens. Cheapest place to iterate. n8n Cloud meters by executions (an N-row eval = N executions) plus tokens.
- **Zapier:** billed by tasks. Editor testing is free of task cost; a live N-row eval is N+ tasks.
- **Make:** billed by operations. Every "Run once" consumes operations + tokens, so every single test run is metered.

Non-determinism is unmanaged in all three GUIs: no exposed seed, and temperature is buried (n8n model sub-node, Make advanced settings) or not exposed (AI by Zapier). Set temperature to 0 where you can; otherwise expect noise and re-run the eval 2 to 3 times before trusting a small delta. Ignore the cosmetic "prompt strength" bars; they do not measure accuracy against your eval set.

---

## Clarifying questions

- Which of the three tools, and which plan? The eval story and the version retention both depend on it.
- For n8n: self-hosted (free, unlimited executions) or Cloud (metered)? And is the Evaluations feature available on your tier?
- Where does the eval set live? n8n's Evaluation Trigger reads a Google Sheet or Data Table directly; Zapier/Make have no native dataset, so the set's location decides how painful the manual loop is.
- Is the task a classifier/extractor (deterministic exact-match scoring works) or open-ended (needs a judge or human grading)?
