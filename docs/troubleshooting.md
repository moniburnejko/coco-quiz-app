# Troubleshooting

Snowsight-specific pitfalls, ordered by how frequently they come up during first-time setup.

If none of these match: paste the **fix prompt** from [prompts.md](prompts.md) with your symptom. The agent will run a diagnostic.

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

## `AI_PARSE_DOCUMENT` fails with "file not accessible"

**Cause:** `STAGE_QUIZ_DATA` was created without `ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')` or without `DIRECTORY = (ENABLE = TRUE)`. Both are required.

This bites only when you recycle a stage created by hand; `$setup-exam` emits the correct DDL.

---

## Study-guide PDF won't stage / is too large

**Cause:** The PDF is added to the **workspace file tree** and CoCo stages it onto `STAGE_QUIZ_DATA` via `COPY FILES` (no manual stage upload). An oversized scanned PDF can be slow to stage or parse, or hit workspace file limits.

**Fix options:**
- Compress the PDF first (most study guides are under 10 MB; a 300 MB file is almost always scanned images — run it through a PDF optimiser).
- After CoCo stages it, it verifies with `LIST @STAGE_QUIZ_DATA` — confirm the file appears there before extraction proceeds.

---

## Agent proceeds past the PDF checkpoint without confirming the file

**Cause:** Snowflake CoCo sometimes optimistically assumes the PDF is in place when the chat is active.

**Fix:** Cut it off with "stop - did you `COPY FILES` the PDF and verify `LIST @STAGE_QUIZ_DATA`?" The agent backs up, stages the file, runs `LIST`, and reports actual contents. If the file isn't in the workspace it asks you to add it.

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

---

## The app deploys but shows a blank screen / Python traceback

**Cause:** A regression that slipped past the pre-deploy scan. Usually one of:
- `CORTEX_MODEL` constant not defined (or an import between app modules broken);
- A session-state key read before being initialised;
- A Cortex JSON helper called on a `None`.

**Fix:** Paste the **fix prompt** with the error text. The agent re-runs `$sis`, which catches these categories. Do not redeploy on a failing scan.

---

## Pre-deploy scan passes but the app logs a `KeyError` at runtime

**Cause:** The `response_format` schema does not cover a key the rendering code reads - the call returns a schema-conformant dict, but the code expects a field the schema never asked for.

**Fix:** Run `$cortex` on the offending prompt (item 1 checks schema-vs-code key coverage):

```
run $cortex on the question generation prompt. focus on key name mismatches and JSON completeness.
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
run $quiz/screens and $sis on the current app files. focus on the finish-round handler and the QUIZ_SESSION_LOG INSERT.
```

---

## Questions/explanations aren't grounded in docs (no real citations)

**Cause:** The exam's `grounding_mode` is `cke`/`custom` but the Snowflake Documentation CKE (or your custom Cortex Search service) isn't reachable — uninstalled, no grant, or unreachable from your region. In `cke`/`custom` mode grounding is **mandatory**: the app will not generate from built-in knowledge, so when the service is down the Home screen disables **Start Round** and shows an install/grant message rather than producing ungrounded content. (`none` mode — non-Snowflake exams, chosen at setup — is the only ungrounded path.)

**Fix:**
1. Confirm the free **Snowflake Documentation** listing is installed (Snowsight » Data Products » Marketplace) and your role can query it:
   ```sql
   SELECT SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
     'SNOWFLAKE_DOCUMENTATION.SHARED.CKE_SNOWFLAKE_DOCS_SERVICE',
     '{"query": "time travel", "columns": ["DOCUMENT_TITLE"], "limit": 1}');
   ```
   Empty/error → install the listing or get access granted. If the imported database has a different name, set `DOCS_SEARCH_SERVICE` in `_config.py`.
2. Confirm the exam's `grounding_mode` (shown **read-only** on the Admin → App config tab). It is fixed at setup — `cke` / `custom` / `none` — with **no runtime toggle**; to change it, re-run `$setup-exam` Step 1g.
3. Cross-region: if your account's region can't reach the shared service, the runtime probe fails and the app surfaces the unreachable-service message, blocking generation until access is restored (enable cross-region inference, or use a service your region can reach).

---

## Everything looks fine but the Streamlit app is using an old version

**Cause:** SiS caches app bundles by stage URL — after the files on `STAGE_SIS_APP` change, the running app doesn't auto-refresh.

**Fix:** the agent re-copies the changed files and re-runs `CREATE OR REPLACE STREAMLIT` (`OR REPLACE` invalidates the cached bundle):

```sql
CREATE OR REPLACE STREAMLIT ... FROM '@...STAGE_SIS_APP' MAIN_FILE = 'main.py' ...;
```

…or in the Streamlit UI: three-dot menu > **Restart app**.

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
