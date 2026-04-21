# Troubleshooting

Snowsight-specific pitfalls, ordered by how frequently they come up during first-time setup.

If none of these match: paste the **fix prompt** from [prompts.md](prompts.md) with your symptom. The agent will run a diagnostic.

---

## `AI_COMPLETE` fails with "not allowed to access this endpoint"

**Cause:** cross-region inference is disabled. `claude-sonnet-4-6` lives in `AWS_US`; EU accounts need explicit permission to reach it.

**Fix (as `ACCOUNTADMIN`, once per account):**

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'AWS_US';
```

Verify:

```sql
SHOW PARAMETERS LIKE 'CORTEX_ENABLED_CROSS_REGION' IN ACCOUNT;
-- expected value: AWS_US
```

If you cannot get `ACCOUNTADMIN`, change the model in `AGENTS.md` > `cortex llm` to one available in your region.

---

## `AI_PARSE_DOCUMENT` fails with "file not accessible"

**Cause:** `STAGE_QUIZ_DATA` was created without `ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')` or without `DIRECTORY = (ENABLE = TRUE)`. Both are required.

`$setup-exam` always uses the right DDL - this only bites when you recycle a stage created by hand.

---

## upload UI rejects a file with "too large"

**Cause:** single-file upload via Snowsight UI is capped at **250 MB**.

**Fix options:**
- compress the PDF (most study guides are under 10 MB; if yours is 300 MB it is scanned images - run it through a PDF optimiser first).
- if you must upload >250 MB, use `snow stage copy` or `snowsql PUT` from a machine with the Snowflake CLI and keep the chat in Snowsight. The agent will verify via `LIST @stage` after your upload - it does not care how the file arrived.

---

## agent proceeds past a checkpoint without waiting for my "uploaded"

**Cause:** Cortex Code sometimes optimistically assumes upload completion when the chat is active.

**Fix:** cut it off with "stop - did you verify `LIST @STAGE_QUIZ_DATA`?" The agent backs up, runs `LIST`, and reports actual contents. If the file is missing it asks you to retry.

---

## `EXAM_DOMAINS.weight_pct` sums to 97 or 103

**Cause:** `AI_COMPLETE` occasionally rounds or drops a domain when the PDF has ambiguous formatting (sidebars, footers interrupting the domain blueprint).

**Fix:** at the domain-extraction checkpoint, choose **Re-extract**. The agent wipes `EXAM_DOMAINS` and retries with a reinforced prompt. If it still fails after 2 retries:

```sql
-- check what the PDF actually parsed to
SELECT SUBSTR(AI_PARSE_DOCUMENT(
    TO_FILE('@<your_database>.QUIZ_<CODE>.STAGE_QUIZ_DATA', 'SnowProCoreStudyGuide_c03.pdf'),
    {'mode': 'LAYOUT'}
):content::VARCHAR, 1, 10000);
```

and inspect manually. If the PDF is scanned without OCR, switch mode:

```sql
AI_PARSE_DOCUMENT(..., {'mode': 'OCR'});
```

---

## AI-generated questions come back with parse errors

**Cause:** `AI_COMPLETE` hit the output token limit on a batch of 10 questions, producing truncated JSON. The agent's `parse_cortex_json` swallows the error and logs it to `session_state.last_cortex_error`.

**Fix:** reduce batch size (step 6b of `$setup-exam`) from 10 to 5 and rerun generation for that domain only:

```
rerun question generation for domain_id = 3 with batch_size = 5.
```

The agent already has retry logic for exactly this case - usually one rerun is enough.

---

## `quiz.py` deploys but the app shows a blank screen / python traceback

**Cause:** a regression that slipped past the pre-deploy scan. Usually one of:
- `st.rerun()` count drifted (must be exactly 6);
- `CORTEX_MODEL` constant not defined;
- a session-state key read before being initialised;
- `parse_cortex_json` called on a `None`.

**Fix:** paste the **fix prompt** with the error text. The agent re-runs `$sis/pre-deploy`, which catches these categories. Do not redeploy on a failing scan.

---

## pre-deploy scan passes but the app logs a `KeyError` at runtime

**Cause:** an AI prompt produces JSON with an unexpected key name - `parse_cortex_json` returns a dict that doesn't contain what `render_quiz` expects.

**Fix:** run `$cortex/prompt-audit` on the offending prompt:

```
run $cortex/prompt-audit on the question generation prompt. focus on key name mismatches and JSON completeness.
```

Usually one field was renamed in the prompt but not updated in the parser (or vice versa).

---

## dashboard is empty after my first completed round

**Cause:** round ended in an unclean way (tab close, browser refresh, error on the Summary screen) and the write-back to `QUIZ_SESSION_LOG` didn't fire.

**Fix:** check:

```sql
SELECT COUNT(*) FROM <your_database>.QUIZ_<CODE>.QUIZ_SESSION_LOG;
```

If 0, start and finish one more round cleanly by clicking **Finish** on the quiz screen. If it stays at 0 after a clean finish, the write-back logic is broken - run:

```
run $quiz/screens and $sis/pre-deploy on the current quiz.py. focus on the finish_round handler and QUIZ_SESSION_LOG INSERT.
```

---

## everything looks fine but the Streamlit app is using an old cached version

**Cause:** SiS caches app bundles by stage URL. After uploading a new `quiz.py` to `STAGE_SIS_APP`, the running app does not auto-refresh.

**Fix:** either:

```sql
CREATE OR REPLACE STREAMLIT ... FROM '@...STAGE_SIS_APP' MAIN_FILE = '/quiz.py' ...;
```

(this is what the agent runs - `OR REPLACE` invalidates the cached bundle), or in the Streamlit UI: click the three-dot menu > **Restart app**.

---

## skill slash command (`/setup-exam`) does not show up in the chat

**Cause:** skill upload didn't complete, or the folder structure got flattened.

**Fix:** check that the workspace has `/setup-exam/SKILL.md` at the expected path. In Cortex Code chat:

```
list the skills available in this workspace.
```

If `setup-exam` is missing, re-upload: **Cortex Code > + > Upload Folder(s) » `.snowflake/cortex/skills/`** (not just one sub-folder).

Common flattening mistake: uploading `skills/` content directly into the workspace root (without the `.snowflake/cortex/` prefix). Snowsight expects the exact path `.snowflake/cortex/skills/<skill-name>/SKILL.md`.

---

## how to reset everything and start over

Assuming you want to keep the database but scrap one exam's schema:

```sql
-- destructive - makes sure
DROP STREAMLIT <your_database>.QUIZ_<CODE>.SNOWPRO_QUIZ;
DROP SCHEMA   <your_database>.QUIZ_<CODE>;
```

Then re-run the **setup prompt** with the same exam. `$setup-exam` will recreate the schema, stages, tables, and ask for the PDF upload again.
