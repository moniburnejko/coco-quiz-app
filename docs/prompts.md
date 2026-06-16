# Prompts - Snowflake CoCo in Snowsight

Two short prompts. They stay short **on purpose**: all the procedure lives in the skills, all the project context in `AGENTS.md`. The prompt just attaches the context and invokes the skill — it never re-describes the steps.

How to invoke in Snowsight CoCo:
- **`@AGENTS.md`** attaches the context file (kept in view for the whole session).
- **`/setup-exam`** (also `/cortex`, `/sis`, `/quiz`, `/adapt-questions`) invokes a skill — type `/` to pick it. The skill carries every step, stop, and check.

---

## Setup prompt - paste to build the app

Attach the context, then invoke the skill (type `/` and pick `setup-exam`):

```
@AGENTS.md

/setup-exam  —  I'm setting up: SnowPro Core (COF-C03).
```

Add anything optional on that line — features ("add exam simulation and flashcards"), advanced mode ("use the quality model profile"), or look ("let me pick the colors"). That's the whole prompt; `/setup-exam` drives the rest and stops at each checkpoint.

### When to paste
- First use of the workspace - before anything is created.
- Adding another exam - same prompt, different exam name. A fresh `QUIZ_<NEW_CODE>` schema is created; the previous one is untouched.

### What the skill will ask you during the run (in order)
1. Exam name + code (e.g. "SnowPro Core", "COF-C03").
2. PDF filename (required) — the study guide you'll upload.
3. CSV/JSON question-bank filename — optional; without one the bank starts empty (runtime AI questions; seed it later from Admin / worksheet / Automation).
4. Optional features + advanced mode (quality model / self-verify / Automations).
5. Default look, or a short custom-theming dialog.
6. Deploy prerequisites (compute pool + PyPI EAI) — with the exact DDL if missing, or the warehouse fallback.
7. "uploaded" after the PDF stage upload → Approve/Re-extract/Abort after domain extraction → deploy → final report.

---

## Fix prompt - paste when something breaks

```
@AGENTS.md

Something broke: [paste the error text / describe the wrong behaviour].
Triage with the relevant skill (/cortex, /sis, or /quiz) and propose a fix before changing anything. Never redeploy on a failing pre-deploy scan.
```

The parent skills route by intent: Cortex/AI errors → `/cortex`; app crash, deploy, or pre-deploy scan → `/sis`; page or generation behaviour → `/quiz`.

### When to paste
- `AI_COMPLETE` / `AI_PARSE_DOCUMENT` errors (NULL, "model not found", "file not accessible", cross-region).
- Deploy fails (e.g. the PyPI/EAI package error) or the app shows a Python traceback.
- Explanations/questions look wrong, or the dashboard shows wrong numbers.

---

## Why the prompts are short

- **`AGENTS.md` is always in context** (`@AGENTS.md`) — project constraints, env table, data model, skill index. Don't repeat them in the prompt.
- **Skills carry the procedure** — `/setup-exam` has its own Step 0–10 with mandatory stops (placeholder guard, stage verify, deploy preflight, domain approval, pre-deploy scan, deploy). You don't re-specify them.
- **Invoking `/setup-exam` loads the skill** — far more reliable than describing the steps in prose. If the agent drifts, nudge it: "what did the `/sis` scan return?" or "show me the `EXAM_DOMAINS` rows".

Invoke a skill directly when you want just one piece: `/sis` (the pre-deploy scan), `/cortex` (a prompt audit), or a `/quiz` sub-skill like `/quiz/design`.
