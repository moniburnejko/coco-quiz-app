# Prompts - Snowflake CoCo in Snowsight

A **single** end-to-end prompt drives the whole pipeline via `$setup-exam`. No phase splitting - Snowsight's CoCo auto-routes through skill descriptions, and the skill itself builds in stopping points for manual uploads and approvals.

A second prompt covers the fix-session path when something breaks post-deploy.

---

## Setup prompt - paste to build the app

```
verify the session context matches AGENTS.md (role, warehouse, database). then run $setup-exam end-to-end for the exam i am about to name.

stop at the manual upload (pdf study guide to STAGE_QUIZ_DATA) and wait for me to confirm with "uploaded".

stop at the domain-extraction checkpoint - show me the domain list, weights sum, and key_facts lengths; ask for approve / re-extract / abort.

before deploy, run the full pre-deploy scan from $sis/pre-deploy across all app files. do NOT deploy on any FAIL - fix and re-scan until clean.

at deploy time default to the workspaces flow: i will run app/main.py for the dev preview and click Deploy myself, then confirm with "deployed". offer the stage path only as fallback.

after deploy, run SHOW STREAMLITS to confirm and report: exam name, exam code, schema, domain count, question count, app URL.

i am setting up: [exam name + any optional features or advanced options you want, e.g. "SnowPro Core COF-C03, add a timed exam simulation mode, use the quality model profile"].
```

### When to paste

- First use of the workspace - before anything is created.
- Adding a second exam - same prompt, different exam name at the bottom. The agent spins up a fresh `QUIZ_<NEW_CODE>` schema, does not touch the previous one.

### What the agent will ask you during the run

1. Exam name + exam code (e.g. "SnowPro Core", "COF-C03").
2. PDF filename (required) - the study guide you will upload.
3. CSV/JSON question-bank filename - optional; without one the bank stays empty (runtime AI questions only; seed the bank later from Admin / a worksheet recipe / an Automation).
4. Any additional customisations (optional features, advanced mode — quality model profile / self-verify / Automations, alternative scoring, etc.).
5. Default look or custom — custom runs a short theming dialog (Streamlit theme keys only, no CSS).
5. "uploaded" confirmation after the input-file stage upload.
6. Approve / Re-extract / Abort after domain extraction.
7. Deploy path: Path A (workspace **Run + Deploy**, default), Path B (scripted stage + `CREATE STREAMLIT`, container runtime), or Path C (warehouse fallback, no compute pool).
8. "Done / Open app / Review" at the end.

---

## Fix prompt - paste when something breaks

```
read AGENTS.md, then triage the following:

[describe the symptom - error message from Snowsight, wrong behaviour on a screen, parse error in logs, etc.]

routing:
- if the error mentions AI_COMPLETE, AI_PARSE_DOCUMENT, "model not found", "file not accessible", or is about cross-region → run the 5-step diagnostic from $cortex/patterns first, report pass/fail, then propose a fix.
- if the app crashes in the browser or shows a python traceback → re-run $sis/pre-deploy on the current app files, fix all fails, then ask me to re-deploy (workspace Deploy, or re-upload to STAGE_SIS_APP on the scripted path).
- if AI explanations or questions look wrong or come back incomplete → run the 7-item audit from $cortex/prompt-audit on the offending prompt.
- if screen transitions misbehave (stuck on a button, duplicate renders, stale widget values) → re-read $quiz/screens and propose a patch.

never redeploy on a failing pre-deploy scan. always stop and ask me to re-deploy after a fix.
```

### When to paste

- `AI_COMPLETE` returns `NULL` or an endpoint error.
- PDF upload looks fine but `AI_PARSE_DOCUMENT` says "file not accessible".
- Streamlit app fails to load / shows a Python traceback.
- Explanations in the Quiz screen are empty or truncated.
- Dashboard shows wrong numbers (weights, counts, percentages).

---

## Tips for keeping prompts short

The reason a single prompt works in Snowsight is that:

- **AGENTS.md is always in context** - you don't need to repeat project constraints.
- **Skills auto-route** - mentioning `$setup-exam`, `$cortex`, `$sis` triggers the corresponding skill description and the agent knows what to do.
- **Checkpoints live in the skill**, not in the prompt - `$setup-exam` has its own stopping points (step 1b, 4, 5d, 8 scan, 9 deploy, 10 report). You don't re-specify them.

If the agent skips a checkpoint or goes off-track, just say "wait - what did `$sis/pre-deploy` return?" or "before you proceed, show me the `EXAM_DOMAINS` rows". It will back up.

---

## What we intentionally did NOT do

- No "phase 1 / phase 2 / phase 3" breakdown - in Snowsight there is no shell session to lose, and the agent can keep a 10-step skill coherent in one go.
- No separate prompt for "load PDF" vs "parse PDF" vs "extract domains" - that is all inside `$setup-exam` step 4–5.
- No separate deploy prompt - step 9 of the skill handles it.

If you want that granularity, invoke sub-skills directly: `$cortex/patterns`, `$sis/pre-deploy`, `$quiz/questions`. They work standalone.