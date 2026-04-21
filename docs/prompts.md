# Prompts - Cortex Code in Snowsight

A **single** end-to-end prompt drives the whole pipeline via `$setup-exam`. No phase splitting - Snowsight's Cortex Code auto-routes through skill descriptions, and the skill itself builds in stopping points for manual uploads and approvals.

A second prompt covers the fix-session path when something breaks post-deploy.

---

## setup prompt - paste to build the app

```
verify the session context matches AGENTS.md (role, warehouse, database). then run $setup-exam end-to-end for the exam i am about to name.

stop at each manual upload (pdf study guide to STAGE_QUIZ_DATA, then quiz.py + environment.yml to STAGE_SIS_APP) and wait for me to confirm with "uploaded".

stop at the domain-extraction checkpoint - show me the domain list, weights sum, and key_facts lengths; ask for approve / re-extract / abort.

before deploy, run the full 22-item pre-deploy scan from $sis/pre-deploy. do NOT deploy on any FAIL - fix and re-scan until clean.

after CREATE STREAMLIT, run SHOW STREAMLITS to confirm and report: exam name, exam code, schema, domain count, question count, app URL.

i am setting up: [exam name + any optional features you want, e.g. "SnowPro Core COF-C03, add a timed exam simulation mode"].
```

### when to paste

- first use of the workspace - before anything is created.
- adding a second exam - same prompt, different exam name at the bottom. The agent spins up a fresh `QUIZ_<NEW_CODE>` schema, does not touch the previous one.

### what the agent will ask you during the run

1. exam name + exam code (e.g. "SnowPro Core", "COF-C03").
2. PDF filename (required) - the study guide you will upload.
3. CSV/JSON question-bank filename - optional; if you don't have one, it generates questions via AI.
4. any additional customisations (optional features, alternative scoring, etc.).
5. "uploaded" confirmation after each stage upload.
6. Approve / Re-extract / Abort after domain extraction.
7. Path A (stage upload) vs Path B (workspace right-click deploy) for the final deploy.
8. "Done / Open app / Review" at the end.

---

## fix prompt - paste when something breaks

```
read AGENTS.md, then triage the following:

[describe the symptom - error message from Snowsight, wrong behaviour on a screen, parse error in logs, etc.]

routing:
- if the error mentions AI_COMPLETE, AI_PARSE_DOCUMENT, "model not found", "file not accessible", or is about cross-region → run the 5-step diagnostic from $cortex/patterns first, report pass/fail, then propose a fix.
- if the app crashes in the browser or shows a python traceback → re-run $sis/pre-deploy on the current quiz.py, fix all fails, then ask me to re-upload to STAGE_SIS_APP and redeploy.
- if AI explanations or questions look wrong or come back with parse errors → run the 8-item audit from $cortex/prompt-audit on the offending prompt.
- if screen transitions misbehave (stuck on a button, duplicate renders, stale widget values) → re-read $quiz/screens and propose a patch.

never redeploy on a failing pre-deploy scan. always stop and ask me to re-upload after a fix.
```

### when to paste

- `AI_COMPLETE` returns `NULL` or an endpoint error.
- PDF upload looks fine but `AI_PARSE_DOCUMENT` says "file not accessible".
- Streamlit app fails to load / shows a python traceback.
- Explanations in the Quiz screen are empty or truncated.
- Dashboard shows wrong numbers (weights, counts, percentages).

---

## tips for keeping prompts short

The reason a single prompt works in Snowsight is that:

- **AGENTS.md is always in context** - you don't need to repeat project constraints.
- **Skills auto-route** - mentioning `$setup-exam`, `$cortex`, `$sis` triggers the corresponding skill description and the agent knows what to do.
- **Checkpoints live in the skill**, not in the prompt - `$setup-exam` has its own stopping points (step 1b, 4, 5d, 8 scan, 9 deploy, 10 report). You don't re-specify them.

If the agent skips a checkpoint or goes off-track, just say "wait - what did `$sis/pre-deploy` return?" or "before you proceed, show me the `EXAM_DOMAINS` rows". It will back up.

---

## what we intentionally did NOT do

- no "phase 1 / phase 2 / phase 3" breakdown - in Snowsight there is no shell session to lose, and the agent can keep a 10-step skill coherent in one go.
- no separate prompt for "load PDF" vs "parse PDF" vs "extract domains" - that is all inside `$setup-exam` step 4–5.
- no separate deploy prompt - step 9 of the skill handles it.

If you want that granularity, invoke sub-skills directly: `$cortex/patterns`, `$sis/pre-deploy`, `$quiz/questions`. They work standalone.