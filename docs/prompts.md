# Prompts - Snowflake CoCo in Snowsight

Two short prompts: the procedure lives in the skills, the project context in `AGENTS.md`. The prompt attaches the context and invokes the skill.

How to invoke in Snowsight CoCo:
- **`@AGENTS.md`** attaches the context file (kept in view for the whole session).
- **`/setup-exam`** (also `/cortex`, `/sis`, `/quiz`, `/adapt-questions`) invokes a skill - type `/` to pick it. The skill carries every step, stop, and check.

---

## Setup prompt - paste to build the app

Attach the context, then invoke the skill (type `/` and pick `setup-exam`):

```
@AGENTS.md

/setup-exam  -  I'm setting up: SnowPro Core (COF-C03).
```

Add anything optional on that line - advanced mode ("use the quality model profile") or look ("let me pick the colors"). That's the whole prompt; `/setup-exam` drives the rest and stops at each checkpoint.

### When to paste
- First use of the workspace - before anything is created.
- Adding another exam - same prompt, different exam name. A fresh `QUIZ_<NEW_CODE>` schema is created; the previous one is untouched.

### What the skill will ask you during the run (in order)
1. Exam name + code (e.g. "SnowPro Core", "COF-C03").
2. PDF filename (required) - the study guide you drop into the workspace file tree (CoCo stages it for you via `COPY FILES`; no manual stage upload).
3. CSV/JSON question-bank filename - optional; without one the bank starts empty (runtime AI questions; seed it later from Admin / worksheet / Automation).
4. Advanced mode (quality model / self-verify / Automations).
5. Default look, or a short custom-theming dialog.
6. Doc grounding mode - set once and stored in `QUIZ_CONFIG` (no runtime toggle): `cke` (default, Snowflake-docs CKE; a hard gate that stops the run if the CKE listing isn't installed), `custom` (your own Cortex Search service), or `none` (ungrounded - non-Snowflake exams only).
7. Confirm the PDF is in the workspace (CoCo stages it via `COPY FILES`) → Approve/Re-extract/Abort after domain extraction → deploy → final report.

---

## Fix prompt - paste when something breaks

```
@AGENTS.md

Something broke: [paste the error text / describe the wrong behaviour, on which screen, and what you expected].
Triage with the relevant skill (/cortex, /sis, or /quiz) and propose a fix before changing anything.
Before redeploying, RE-READ the affected app files from disk (not from memory) and re-run the $sis pre-deploy scan and the $quiz/screens UX gate; only redeploy if both are clean.
```

The parent skills route by intent: Cortex/AI errors → `/cortex`; app crash, deploy, or pre-deploy scan → `/sis`; page or generation behaviour → `/quiz`.

### When to paste
- `AI_COMPLETE` / `AI_PARSE_DOCUMENT` errors (NULL, "model not found", "file not accessible", cross-region).
- Deploy fails or the app shows a Python traceback.
- Explanations/questions look wrong, or the dashboard shows wrong numbers.

---

## Iterate / improve prompt - paste after using the app

The first deploy is the start, not the end - using the app and feeding back what's off is a normal, core part of the work. Paste this when you've tried the app and want changes (not just crashes):

```
@AGENTS.md

I tried the app. Things to improve:
- [screen/page]: [what you saw] → [what you expected]. (attach a screenshot if you can)
- ...
Apply these via the relevant /quiz sub-skill or /sis. After fixing, RE-READ the app files
from disk (not from memory) and re-run the $sis scan + the $quiz/screens UX gate; redeploy
only if both are clean.
```

**After a chat reload / "session stopped" / page refresh,** add a first line: *"Re-read every app file from disk and re-run the scan + gate - do NOT use your memory - before changing anything."*

---

## Tips

- **`AGENTS.md` is always in context** (`@AGENTS.md`) - project constraints, env table, data model, skill index.
- **Skills carry the procedure** - `/setup-exam` has its own Step 0-10 with mandatory stops (placeholder guard, stage verify, deploy preflight, domain approval, pre-deploy scan, deploy).
- If the agent drifts, nudge it: "what did the `/sis` scan return?" or "show me the `EXAM_DOMAINS` rows".
- **Iteration is normal** - the first deploy is the start, not the end. Expect a few rounds of *use it → report what's off → fix → redeploy*, and **always ask for the `$sis` scan + `$quiz/screens` UX gate (re-read from disk, not memory) before each redeploy**.

Invoke a skill directly when you want just one piece: `/sis` (the pre-deploy scan), `/cortex` (a prompt audit), or a `/quiz` sub-skill like `/quiz/design`.
