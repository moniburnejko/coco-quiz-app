# Troubleshooting

Snowsight-specific pitfalls, ordered by how frequently they come up during first-time setup.

If none of these match: paste the **fix prompt** from [prompts.md](prompts.md) with your symptom. The agent will run a diagnostic.

---

## App deploy fails: "Failed to retrieve package… Have you enabled External Access Integration (EAI)?"

**Cause:** The container runtime (the default deploy target) installs `pandas`/`altair` from PyPI, which needs an External Access Integration. The base image ships only Python, Streamlit, and Snowpark — so without an EAI the package fetch fails (DNS/connect error to `pypi.org`).

**Fix (as `ACCOUNTADMIN`, once per account):**

```sql
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION pypi_access_integration
  ALLOWED_NETWORK_RULES = (snowflake.external_access.pypi_rule)   -- Snowflake's managed rule; no custom rule needed
  ENABLED = TRUE;
GRANT USAGE ON INTEGRATION pypi_access_integration TO ROLE <your_role>;
```

Then attach it and redeploy — Deploy dialog → **Network → External Access Integrations** → `pypi_access_integration`, or for an existing app:

```sql
ALTER STREAMLIT <your_database>.QUIZ_<CODE>.SNOWPRO_QUIZ
  SET EXTERNAL_ACCESS_INTEGRATIONS = (pypi_access_integration);
```

**No EAI / no ACCOUNTADMIN?** Switch to the warehouse fallback (`$setup-exam` Step 9 Path C): `environment.yml` from the Snowflake Anaconda channel — no EAI, no compute pool, Streamlit 1.52.2.

---

## `AI_COMPLETE` fails with "not allowed to access this endpoint"

**Cause:** Cross-region inference is disabled. `claude-sonnet-4-6` is hosted in US regions; accounts that cannot reach it in-region need explicit permission. (Accounts created after 2026-03-09 default to `ANY_REGION` and are not affected.)

**Fix (as `ACCOUNTADMIN`, once per account):**

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

(`'AWS_GLOBAL'` is a narrower alternative; the legacy `'AWS_US'` still works but is narrowest.) Verify:

```sql
SHOW PARAMETERS LIKE 'CORTEX_ENABLED_CROSS_REGION' IN ACCOUNT;
-- expected: ANY_REGION (or AWS_GLOBAL / AWS_US)
```

If you cannot get `ACCOUNTADMIN`, change the model in `AGENTS.md` > `cortex llm` to one available in your region.

---

## Deploy dialog shows no compute pool / "compute pool not found"

**Cause:** The container runtime (default) needs a compute pool your role can use. The account has none, or your role lacks `USAGE` on it.

**Fix:** Check what exists:

```sql
SHOW COMPUTE POOLS;
```

If the list is empty or unusable, ask an admin to create/grant one - or tell the agent to use the **warehouse fallback** (`$setup-exam` Step 9 Path C): it generates `environment.yml` and deploys without `RUNTIME_NAME`/`COMPUTE_POOL` (Streamlit capped at 1.52.2).

---

## `AI_PARSE_DOCUMENT` fails with "file not accessible"

**Cause:** `STAGE_QUIZ_DATA` was created without `ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')` or without `DIRECTORY = (ENABLE = TRUE)`. Both are required.

`$setup-exam` always uses the right DDL - this only bites when you recycle a stage created by hand.

---

## Upload UI rejects a file with "too large"

**Cause:** Single-file upload via Snowsight UI is capped at **250 MB**.

**Fix options:**
- Compress the PDF (most study guides are under 10 MB; if yours is 300 MB it is scanned images - run it through a PDF optimiser first).
- If you must upload >250 MB, use `snow stage copy` or `snowsql PUT` from a machine with the Snowflake CLI and keep the chat in Snowsight. The agent will verify via `LIST @stage` after your upload - it does not care how the file arrived.

---

## Agent proceeds past a checkpoint without waiting for my "uploaded"

**Cause:** Snowflake CoCo sometimes optimistically assumes upload completion when the chat is active.

**Fix:** Cut it off with "stop - did you verify `LIST @STAGE_QUIZ_DATA`?" The agent backs up, runs `LIST`, and reports actual contents. If the file is missing it asks you to retry.

---

## `EXAM_DOMAINS.weight_pct` sums to 97 or 103

**Cause:** `AI_COMPLETE` occasionally rounds or drops a domain when the PDF has ambiguous formatting (sidebars, footers interrupting the domain blueprint).

**Fix:** At the domain-extraction checkpoint, choose **Re-extract**. The agent wipes `EXAM_DOMAINS` and retries with a reinforced prompt. If it still fails after 2 retries:

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

## Question-bank seeding fails / returns NULL

**Cause:** `AI_COMPLETE` hit the output token limit on a 10-question batch (Admin "Generate batch", the worksheet recipe, or an Automation run). With structured outputs that surfaces as a failed/NULL call (not malformed JSON).

**Fix:** Reduce the batch to 5 questions and rerun for that domain only — in the worksheet recipe change "Generate 10" to "Generate 5"; from chat:

```
seed the question bank for domain_id = 3 with batch_size = 5.
```

Usually one rerun is enough.

---

## The app deploys but shows a blank screen / Python traceback

**Cause:** A regression that slipped past the pre-deploy scan. Usually one of:
- `CORTEX_MODEL` constant not defined (or an import between app modules broken);
- A session-state key read before being initialised;
- A Cortex JSON helper called on a `None`.

**Fix:** Paste the **fix prompt** with the error text. The agent re-runs `$sis/pre-deploy`, which catches these categories. Do not redeploy on a failing scan.

---

## Pre-deploy scan passes but the app logs a `KeyError` at runtime

**Cause:** The `response_format` schema does not cover a key the rendering code reads - the call returns a schema-conformant dict, but the code expects a field the schema never asked for.

**Fix:** Run `$cortex/prompt-audit` on the offending prompt (item 1 checks schema-vs-code key coverage):

```
run $cortex/prompt-audit on the question generation prompt. focus on key name mismatches and JSON completeness.
```

Usually one field was renamed in the prompt but not updated in the parser (or vice versa).

---

## Dashboard is empty after my first completed round

**Cause:** Round ended in an unclean way (tab close, browser refresh, error on the Summary screen) and the write-back to `QUIZ_SESSION_LOG` didn't fire.

**Fix:** Check:

```sql
SELECT COUNT(*) FROM <your_database>.QUIZ_<CODE>.QUIZ_SESSION_LOG;
```

If 0, start and finish one more round cleanly by clicking **Finish** on the quiz screen. If it stays at 0 after a clean finish, the write-back logic is broken - run:

```
run $quiz/screens and $sis/pre-deploy on the current app files. focus on the finish-round handler and the QUIZ_SESSION_LOG INSERT.
```

---

## Everything looks fine but the Streamlit app is using an old version

**Cause (Path A - Workspaces):** **Run** updates only your private *dev app*; the published app changes only on **Deploy**. If others see stale behaviour, you previewed but never re-deployed.

**Fix (Path A):** Click **Deploy** again in the project toolbar.

**Cause (Path B - stage):** SiS caches app bundles by stage URL. After uploading new `app/` files to `STAGE_SIS_APP`, the running app does not auto-refresh.

**Fix (Path B):** Either:

```sql
CREATE OR REPLACE STREAMLIT ... FROM '@...STAGE_SIS_APP' MAIN_FILE = 'main.py' ...;
```

(this is what the agent runs - `OR REPLACE` invalidates the cached bundle), or in the Streamlit UI: click the three-dot menu > **Restart app**.

---

## Skill slash command (`/setup-exam`) does not show up in the chat

**Cause:** Skill upload didn't complete, or the folder structure got flattened.

**Fix:** Check that the workspace has `/setup-exam/SKILL.md` at the expected path. In CoCo chat:

```
list the skills available in this workspace.
```

If `setup-exam` is missing, re-upload: **CoCo > + > upload folder(s) » `.snowflake/cortex/skills/`** (not just one sub-folder).

Common flattening mistake: uploading `skills/` content directly into the workspace root (without the `.snowflake/cortex/` prefix). Snowsight expects the exact path `.snowflake/cortex/skills/<skill-name>/SKILL.md`.

---

## How to reset everything and start over

Assuming you want to keep the database but scrap one exam's schema:

```sql
-- destructive - makes sure
DROP STREAMLIT <your_database>.QUIZ_<CODE>.SNOWPRO_QUIZ;
DROP SCHEMA   <your_database>.QUIZ_<CODE>;
```

then re-run the **setup prompt** with the same exam. `$setup-exam` will recreate the schema, stages, tables, and ask for the PDF upload again.
