---
name: setup-exam
description: "Automated 10-step exam setup pipeline for Snowflake CoCo in Snowsight. Creates the schema, extracts domains from a study-guide PDF, loads an optional question bank, builds the multipage `app/` Streamlit project, and deploys it on a virtual warehouse. First exam or adding another. Triggers: setup exam, new exam, new certification, new study guide PDF, add exam, switch exam, create exam"
---

# When to Use

Setting up a new Snowflake (or any) certification exam, or switching to another. Each exam gets its own schema — nothing is overwritten.

Example: *"I have SnowProGenAIStudyGuide.pdf — create a quiz app for this exam."* · *"Switch to the SnowPro Specialty: Gen AI exam."*

# When NOT to Use

- The exam is **fully set up** already — schema + deployed app (`SHOW SCHEMAS LIKE 'QUIZ_<CODE>' IN DATABASE {database};` + `SHOW STREAMLITS`). *(A **partial** schema from an interrupted or reloaded run is a resume, not a re-setup — Step 1a probes and continues.)*
- Fixing bugs in an existing quiz → `$cortex` or `$sis`.

---

# Environment

Runs inside **CoCo in Snowsight**: no bash/git/`snow` CLI, no `PUT` (the agent can't read the local filesystem). The user only **drops files into the workspace** (the PDF/CSV, plus the generated `app/` project the agent writes); the agent then **copies them onto stages with `COPY FILES`** — the PDF onto `STAGE_QUIZ_DATA` (Step 4) and the `app/` onto `STAGE_SIS_APP`, deploying via `CREATE STREAMLIT` on the warehouse runtime (Step 9). No manual stage upload. Isolation is **schema-per-exam** (`QUIZ_<CODE>`) — no git branch.

Read `{database}`, `{warehouse}`, `{role}` from the `snowflake environment` table in `AGENTS.md`; every SQL placeholder below substitutes those. Never hardcode exam names/codes — extract the name from the PDF and ALWAYS confirm the code with the user.

**Tool — `ask_user_question`** (at STOP points): present fixed options, wait for the choice. If unavailable, present the same options as a numbered list and wait.

---

# Advanced options (opt-in)

All OFF by default — enabled only if the user asks in Step 1d. **Without an explicit request, behave exactly as if this section did not exist.** Preview-dependent items must be verified in-account.

| Option | Values | What it does |
|--------|--------|--------------|
| model_profile | `default` / `quality` | `default` = `claude-sonnet-4-6`. `quality` = `claude-opus-4-7` set as `CORTEX_MODEL` in `_config.py` — stronger reasoning for hard distractors, slower + markedly pricier. `claude-opus-4-8` is **Public Preview** — use only on explicit request. Do NOT change the AGENTS.md default. |
| self_verify | `off` / `on` | Adds Step 8.5 (byte-compile the generated modules). Needs a CoCo session with code execution (Cloud Agents); skipped gracefully otherwise. |
| automations | `off` / `on` | Recurring unattended maintenance via CoCo **Automations** (Preview) — report-only recipe in `docs/customization.md` §5c. |

---

# Instructions

> **Step 0 — read this entire skill before acting.** Run no SQL, create no object, generate no file until you've read every step. Each STOP prevents a known costly failure (stage without `SNOWFLAKE_SSE` → `AI_PARSE_DOCUMENT` fails; unfilled `<...>` placeholders → wrong objects). **Do NOT improvise from the AGENTS.md overview.** You create nothing before Step 2.

## Step 1 — Collect inputs

### 1a — Validate AGENTS.md config (mandatory guard)

**Echo the full `snowflake environment` table back to the user**, then scan every row for `<...>` placeholders. If any remain (`<your_database>`, `<your_warehouse>`, `<your_role>`, …), **STOP immediately** — create nothing. Tell the user exactly which placeholders are unfilled and what each means. Wait, re-read AGENTS.md, re-scan. Do not proceed while any required `<...>` remains. *(An unfilled placeholder makes the pipeline create wrong or missing objects.)*

Then confirm the session matches AGENTS.md: `SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE();` — if it differs, ask whether to `USE` the AGENTS.md values or update AGENTS.md.

**AGENTS.md env is the source of truth for the target schema.** Every statement targets exactly the `schema` named there. If a code you derive later (Step 1b, or the PDF title in Step 5) implies a *different* schema, that's a contradiction — STOP and reconcile env first; never silently write into a different schema than env names.

**Resume safely — this skill re-runs from the top after any chat reload, with NO memory of prior turns.** See what already exists, then resume from the first incomplete step (never redo finished work, never write outside the env schema). `SHOW SCHEMAS LIKE 'QUIZ_%' IN DATABASE {database};` — env schema **absent** → fresh run (start at Step 2); **present** → probe before redoing anything:
```sql
SELECT COUNT(*) AS domains, COUNT(key_facts) AS facts FROM {database}.QUIZ_<CODE>.EXAM_DOMAINS;
SELECT COUNT(*) AS questions FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS;
SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA {database}.QUIZ_<CODE>;
```
Domains populated → skip Step 5; questions loaded → skip Step 6; app exists → you're at Step 10. A user's *"you already did X — continue"* is a claim to **verify with these probes against the env schema**, not to trust blindly; if the work sits in a different `QUIZ_*` schema than env names, reconcile env before continuing.

### 1b — Target exam

Extract the exam name from the prompt, then **propose the exam code you recognize for it** and ask only to confirm — don't ask open-endedly for a code you already know. The code is *not* free to get wrong: it becomes a durable schema identifier (`QUIZ_<CODE>`) and exam codes are revised across versions (e.g. `COF-C01` → `C02` → `C03`), possibly past your training cutoff — so a confident-but-stale guess must never pass silently. *"**{exam_name}** is **{proposed_code}** — confirm, or give the current code."* Ask the user to **verify it against their study guide's title page** — that page is authoritative, and Step 5 re-confirms the code from the parsed document as a backstop. If you genuinely don't recognize the exam, ask outright. Wait for confirmation before using it.

### 1c — Input files

You do **not** need filenames here — the agent reads the real names off the workspace stage with `LIST` in Step 4, so don't ask the user to type a name from memory. Ask only **what they have** (this selects the mode):
- **Study-guide PDF (mandatory):** confirm they have one **in the workspace** (they drop it into the file tree — the agent stages it in Step 4; no manual stage upload). No PDF → **STOP** (required for `AI_PARSE_DOCUMENT` → `EXAM_DOMAINS`).
- **Question bank CSV/JSON (optional):** `ask_user_question` Yes/No (No → `question_source` defaults to `'ai'`). The Yes/No sets the mode; the filename is resolved by `LIST`, not asked.

### 1d — Additional requirements + advanced mode

Ask: *"Any additional requirements? (optional features, AI study recommendations, different scoring — or advanced mode: quality model / self-verify / Automations)"*. Map advanced requests to the **Advanced options** section above (set at Step 8 / 8.5 / 10; never change AGENTS.md defaults). If the user names features not in AGENTS.md, clarify requirements before Step 2. Wait for the response.

### 1e — Look & feel

`ask_user_question`: **Default look** (clean Snowflake-blue — no further questions) or **Custom**. Custom → a short one-question-at-a-time dialog (light/dark base; accent color; corner roundness; font stack; sidebar tint), mapped ONLY to native `[theme]`/`[theme.sidebar]` keys per the `$quiz/design` theming contract — **never CSS, `unsafe_allow_html`, or external fonts (CSP)**. Confirm the palette in words. Store the choice for Step 8.

### 1f — Deploy prerequisites

**Nothing to check** — `pandas`/`altair` come from the Snowflake Anaconda channel via `environment.yml`. Skip to 1g.

### 1g — Doc grounding mode (decided here, fixed for the exam)

Generation (questions, explanations, hints, deep-dive) is **grounded — the app NEVER falls back to the model's built-in knowledge.** Set `grounding_mode` ONCE here (stored in `QUIZ_CONFIG`, fixed for this exam's app — there is no runtime toggle):

- **Snowflake exam → `cke` (default).** Grounds in the **Snowflake Documentation CKE** (`SNOWFLAKE_DOCUMENTATION.SHARED.CKE_SNOWFLAKE_DOCS_SERVICE`). **Hard gate** — probe it; if unreachable, **STOP**: tell the user to install the free listing (Data Products » Marketplace » Snowflake Documentation; needs `IMPORT SHARE`/ACCOUNTADMIN), then re-probe. Do NOT build until it's reachable. It's free and installable on **trial accounts**. (One ad-hoc `SEARCH_PREVIEW` is the probe; the app itself uses the Python API — `$cortex`.)
  ```sql
  SELECT SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
    'SNOWFLAKE_DOCUMENTATION.SHARED.CKE_SNOWFLAKE_DOCS_SERVICE',
    '{"query": "virtual warehouse", "columns": ["DOCUMENT_TITLE"], "limit": 1}');
  ```
- **Non-Snowflake exam (AWS/GCP/…) → `none` or `custom`.** No Snowflake-docs CKE applies. `ask_user_question`: **`none`** = ungrounded AI generation — confirm the user accepts the tradeoff (modern models know this material decently, but ungrounded raises factual-error risk that CKE essentially eliminates); or **`custom`** = a private Cortex Search service the user built over their own docs (ask for its `db.schema.service` name). See `docs/customization.md` "Using a different exam".

Seed the choice (replaces any prior value); for `custom`, also seed the `DOCS_SEARCH_SERVICE` override (matches the `_config.py` constant — `$cortex`):
```sql
MERGE INTO {database}.QUIZ_<CODE>.QUIZ_CONFIG t
USING (SELECT 'grounding_mode' AS k, TO_VARIANT('<cke|custom|none>') AS v) s ON t.config_key = s.k
WHEN MATCHED THEN UPDATE SET config_value = s.v
WHEN NOT MATCHED THEN INSERT (config_key, config_value) VALUES (s.k, s.v);
```

## Step 2 — Create the schema

`<EXAM_CODE>` → hyphens to underscores (e.g. `COF-C03` → `QUIZ_COF_C03`). The configured role has full permissions on `{database}` — no grants.
```sql
CREATE SCHEMA IF NOT EXISTS {database}.QUIZ_<EXAM_CODE>;
USE SCHEMA {database}.QUIZ_<EXAM_CODE>;
```

## Step 3 — Create stages, tables, file format (this skill is the sole owner of the data model)

**Stages** — `STAGE_QUIZ_DATA` MUST have SSE + directory for `AI_PARSE_DOCUMENT`. **Copy this DDL verbatim — do NOT write `CREATE STAGE` from memory** (a bare `CREATE STAGE` defaults to client-side encryption → `AI_PARSE_DOCUMENT` fails "Client Side Encryption is not supported"):
```sql
CREATE STAGE IF NOT EXISTS {database}.QUIZ_<CODE>.STAGE_QUIZ_DATA
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  DIRECTORY = (ENABLE = TRUE);
CREATE STAGE IF NOT EXISTS {database}.QUIZ_<CODE>.STAGE_SIS_APP;
```
Verify immediately, and drop+recreate (with the exact DDL) if encryption isn't `SNOWFLAKE_SSE` before any upload:
```sql
DESCRIBE STAGE {database}.QUIZ_<CODE>.STAGE_QUIZ_DATA;  -- confirm TYPE = SNOWFLAKE_SSE and DIRECTORY enabled = true
```

**Tables** — all 5, exact DDL:
```sql
CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.EXAM_DOMAINS (
    domain_id    VARCHAR PRIMARY KEY,
    domain_name  VARCHAR NOT NULL,
    weight_pct   FLOAT NOT NULL,
    topics       VARIANT,
    key_facts    VARCHAR
);

CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.QUIZ_QUESTIONS (
    question_id    NUMBER AUTOINCREMENT PRIMARY KEY,
    domain_id      VARCHAR NOT NULL,
    domain_name    VARCHAR,
    difficulty     VARCHAR DEFAULT 'medium',
    question_text  VARCHAR(2000) NOT NULL,
    is_multi       BOOLEAN DEFAULT FALSE,
    option_a       VARCHAR(500)  NOT NULL,
    option_b       VARCHAR(500)  NOT NULL,
    option_c       VARCHAR(500),
    option_d       VARCHAR(500),
    option_e       VARCHAR(500),
    correct_answer VARCHAR NOT NULL,
    source         VARCHAR DEFAULT 'MANUAL',
    created_at     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);

CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.QUIZ_REVIEW_LOG (
    log_id         NUMBER AUTOINCREMENT PRIMARY KEY,
    logged_at      TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
    domain_id      VARCHAR,
    domain_name    VARCHAR,
    difficulty     VARCHAR,
    question_text  VARCHAR,
    correct_answer VARCHAR,
    mnemonic       VARCHAR(500),
    doc_url        VARCHAR(500)
);

CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.QUIZ_SESSION_LOG (
    session_id     NUMBER AUTOINCREMENT PRIMARY KEY,
    session_ts     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
    exam_code      VARCHAR NOT NULL,
    round_size     NUMBER NOT NULL,
    correct_count  NUMBER NOT NULL,
    score_pct      FLOAT NOT NULL,
    domain_filter  VARCHAR,
    difficulty     VARCHAR
);

CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.QUIZ_CONFIG (
    config_key   VARCHAR PRIMARY KEY,
    config_value VARIANT,
    updated_at   TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);
```
`QUIZ_REVIEW_LOG` = per-wrong-answer history; `QUIZ_SESSION_LOG` = per-round summary (a perfect round writes a session row but no review rows, so they can't merge); `QUIZ_CONFIG` = runtime config (defaults in `_config.py`, DB overrides). The **Flashcards** feature adds a `FLASHCARD_PROGRESS` table when enabled (`$quiz/features`); other deferred features (e.g. Misconception → `ALTER TABLE … ADD COLUMN selected_answer, misconception`; Flag → `QUIZ_FLAGS`) are in `docs/future-features.md`.

**File format:**
```sql
CREATE FILE FORMAT IF NOT EXISTS {database}.QUIZ_<CODE>.FF_CSV
  TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

## Step 4 — Stage the study guide (agent-driven — no manual stage upload)

The user only drops the study-guide PDF (and an optional question-bank CSV) into the **workspace** file tree — never the stage UI. The agent copies it from the workspace's internal stage onto `STAGE_QUIZ_DATA` with `COPY FILES` (the same mechanism as the Step 9 app deploy).

1. **Find the file(s) on the workspace stage** — the workspace name varies, so confirm the URI by listing it; the `.pdf` is the study guide, a `.csv`/`.json` is the bank:
```sql
LIST 'snow://workspace/USER$.PUBLIC."<workspace_name>"/versions/live/';
```
2. **Copy onto `STAGE_QUIZ_DATA` and verify.** `STAGE_QUIZ_DATA` is the SSE+directory stage from Step 3; the copied file is stored under the target stage's `SNOWFLAKE_SSE` encryption, which is exactly what `AI_PARSE_DOCUMENT` requires:
```sql
COPY FILES INTO @{database}.QUIZ_<CODE>.STAGE_QUIZ_DATA
  FROM 'snow://workspace/USER$.PUBLIC."<workspace_name>"/versions/live/'
  FILES = ('<pdf_filename>');   -- + '<csv_filename>' if a bank CSV was provided
ALTER STAGE {database}.QUIZ_<CODE>.STAGE_QUIZ_DATA REFRESH;
LIST @{database}.QUIZ_<CODE>.STAGE_QUIZ_DATA;
```
**STOP** if the PDF isn't in the workspace — ask the user to add it to the file tree and re-list. Ambiguous (multiple PDFs) → `ask_user_question`. The `LIST` output is the source of truth for the names (never typed from memory); carry the resolved `<pdf_filename>` (and `<csv_filename>`) — now on `STAGE_QUIZ_DATA` — into Steps 5–6.

## Step 5 — Extract domains from the PDF

**Parse once into a transient table, then reuse it — survives chat reloads.** `AI_PARSE_DOCUMENT` is billed per call, so parse the PDF (`<pdf_filename>` from Step 4's `LIST`) **exactly once** and keep the result; every later statement — and any resumed session — reads from it instead of re-parsing. A CTE or `TEMPORARY` table is lost across separate statements / a chat reload; a **`TRANSIENT` table persists across reloads with no Fail-safe overhead** — right for regenerable parse output (just re-parse if it's ever lost). Parse-only-if-empty guard:
```sql
CREATE TRANSIENT TABLE IF NOT EXISTS {database}.QUIZ_<CODE>._DOC_CONTENT (doc_content VARCHAR);
INSERT INTO {database}.QUIZ_<CODE>._DOC_CONTENT
SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@{database}.QUIZ_<CODE>.STAGE_QUIZ_DATA', '<pdf_filename>'),
    {'mode': 'LAYOUT'}):content::VARCHAR
WHERE NOT EXISTS (SELECT 1 FROM {database}.QUIZ_<CODE>._DOC_CONTENT);
```
**Ground every domain, topic, and `key_facts` value in this parsed content — never hand-author them from your own knowledge** (that silently swaps the real study guide for a remembered, possibly-stale blueprint). Every prompt below reads `(SELECT doc_content FROM {database}.QUIZ_<CODE>._DOC_CONTENT)`. This is **build-time PDF grounding** — a separate regime from the runtime `grounding_mode`/CKE contract (Step 1g), which governs the deployed app's question/explanation/hint/etc. generation.

**Confirm the exam code against the document** — the parsed title page is authoritative (model knowledge can be stale; see Step 1b). Compare it to `<EXAM_CODE>`. On a **mismatch** (e.g. the guide says `DEA-C02` but the schema is `QUIZ_DEA_C01`) → **STOP**; on the user's confirmation, recover WITHOUT a mid-pipeline `DROP`: create `QUIZ_<NEW_CODE>`, re-run the Step 3 DDL there, copy the already-uploaded file server-side (no re-upload), rebuild `_DOC_CONTENT` in the new schema, update AGENTS.md (Step 7 bounds), then OFFER to drop the empty wrong-code schema:
```sql
COPY FILES INTO @{database}.QUIZ_<NEW_CODE>.STAGE_QUIZ_DATA FROM @{database}.QUIZ_<OLD_CODE>.STAGE_QUIZ_DATA;
-- only after re-extraction succeeds, with user approval (safe ONLY because it never received domains/questions):
DROP SCHEMA IF EXISTS {database}.QUIZ_<OLD_CODE>;
```

**Before extracting**, scan the parsed content (`SELECT doc_content FROM {database}.QUIZ_<CODE>._DOC_CONTENT`) for multiple/transition blueprints (effective dates, "old vs new"). If conflicting structures exist, `ask_user_question` to pick (default: the one effective today) — guides published during a transition often contain both.

**Extract domains — schema-constrained, one statement.** Structured output is the project standard for ALL JSON (→ `$cortex`): pass a `response_format` schema so the result is guaranteed-conformant — never prompt-only "return JSON". Read the TEMP table and FLATTEN straight into `EXAM_DOMAINS`:
```sql
INSERT INTO {database}.QUIZ_<CODE>.EXAM_DOMAINS (domain_id, domain_name, weight_pct, topics)
SELECT d.value:domain_id::VARCHAR, d.value:domain_name::VARCHAR, d.value:weight_pct::FLOAT, d.value:topics
FROM TABLE(FLATTEN(PARSE_JSON(AI_COMPLETE(
    model => 'claude-sonnet-4-6',
    prompt => CONCAT($$List EVERY exam domain for <EXAM_CODE> with its exact name, weight % (numbers summing to 100), and topics, as JSON. If the guide shows old + new blueprints, use the one effective today.$$, CHR(10),
                     (SELECT doc_content FROM {database}.QUIZ_<CODE>._DOC_CONTENT)),
    model_parameters => {},
    response_format => {'type':'json','schema':{'type':'object','properties':{'domains':{'type':'array','items':{'type':'object','properties':{
        'domain_id':{'type':'string'},'domain_name':{'type':'string'},'weight_pct':{'type':'number'},
        'topics':{'type':'array','items':{'type':'string'}}},'required':['domain_id','domain_name','weight_pct','topics']}}},'required':['domains']}}
)):domains)) d;
```
*(Messy PDF? `AI_EXTRACT` is a one-call keyed-JSON alternative — bundled `cortex-ai-function-studio`. AI_COMPLETE stays the default.)*

**Extract `key_facts` per domain** — reuse the SAME `_DOC_CONTENT` (free text, so no `response_format`); ONE `UPDATE` covers every domain, each using its own `domain_name`, **never re-parsing**:
```sql
UPDATE {database}.QUIZ_<CODE>.EXAM_DOMAINS d
SET key_facts = AI_COMPLETE(model => 'claude-sonnet-4-6',
    prompt => CONCAT($$Extract a comprehensive plain-text list of testable facts (definitions, limits, best practices, feature names) for the $$, d.domain_name,
                     $$ domain of {exam_name} (<EXAM_CODE>).$$, CHR(10),
                     (SELECT doc_content FROM {database}.QUIZ_<CODE>._DOC_CONTENT)));
```
Verify each `key_facts` is non-null.

**Capture the exam structure** for `_config.py` (Step 8): from the same `_DOC_CONTENT`, extract the exam's **question count** and **time limit (minutes)** — one schema-constrained `AI_COMPLETE` returning `{question_count:int|null, time_limit_min:int|null}` — and bake the result into `_config.py` as `EXAM_QUESTION_COUNT` / `EXAM_TIME_LIMIT_MIN`. The **Exam Simulation** feature reads them (`$quiz/features`); capture them now even if no feature is requested, because `_DOC_CONTENT` is transient (gone in later sessions). Null is fine — the feature confirms with the user when needed.

**Verify + ⚠️ STOP:**
```sql
SELECT COUNT(*) , SUM(weight_pct) FROM {database}.QUIZ_<CODE>.EXAM_DOMAINS;
SELECT domain_name, LENGTH(key_facts) FROM {database}.QUIZ_<CODE>.EXAM_DOMAINS ORDER BY domain_id;
```
Expected: N domains, weights sum to 100, all `key_facts` non-empty. Present the results, then `ask_user_question`: **Approve** (→ Step 6) / **Re-extract** (clear `EXAM_DOMAINS`, re-run) / **Abort** (STOP). Do not proceed until answered.

## Step 6 — Load the question bank (optional)

**With a CSV** (uploaded in Step 4): if `SELECT COUNT(*) FROM QUIZ_QUESTIONS` already has rows (e.g. `$adapt-questions` ran), skip to verify. Else `COPY INTO QUIZ_QUESTIONS FROM @…STAGE_QUIZ_DATA/<csv> FILE_FORMAT = …FF_CSV;` then backfill `domain_name` from `EXAM_DOMAINS`. If columns/types differ from the target schema, run `$adapt-questions` first.

**Without a CSV — the bank starts empty; do NOT mass-generate at build time** (slow, burns the token budget before the user sees the app, and confuses — "are these the only questions?"). The app is fully functional on runtime AI questions (`question_source` defaults to `'ai'`), and **every runtime AI question is saved to the bank** (`$quiz/questions` — Bank persistence), so it **fills organically as the user practices** — "Question Bank" mode then serves stored questions with no AI call. The user can also seed it deliberately: CSV/JSON + `$adapt-questions`, the Admin **Generate batch** button, the worksheet recipe (`docs/customization.md` §6), or a scheduled task/Automation.

Verify (0 rows without a CSV is legitimate — state it, don't treat as error):
```sql
SELECT COUNT(*), COUNT(DISTINCT domain_id), COUNT(*) FILTER (WHERE domain_name IS NULL) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS;
```

## Step 7 — Update AGENTS.md

Edit `AGENTS.md` within these boundaries.
**CAN edit:** schema name + exam code (env table), the title line, and append a new subsection if the user requested a feature not already documented.
**MUST NOT edit:** database/warehouse/role (user preconditions), the Environment section, the data-model summary, the Cortex-LLM section, the skills index, or Security & governance.

## Step 8 — Build the `app/` project in the workspace

1. Read the updated `AGENTS.md`.
2. **MANDATORY — read these skills BEFORE writing any code** (they're generation rules, not a post-hoc linter — reading them first is what makes the scan pass):
   - `$sis` — gotchas (no-`ttl` cache + `clear_caches()`, widget lifecycle, SQL safety) + the pre-deploy scan your code must ALREADY pass
   - `$cortex` — `call_cortex_json` + `response_format` (no fence parsing), injection delimiting, the **mandatory grounding contract** (no built-in fallback), and the `_search.py` CKE helper (generate it unless `grounding_mode = none` — Step 1g; requires the `snowflake` package in `environment.yml`)
   - `$quiz/screens` (flow, state, learning-loop contracts, write-back), `$quiz/questions` (`DIFFICULTY_GUIDE`, shuffling, validation, retry), `$quiz/design` (`EXAM_NAME`, the theming contract + chart rules), and `$quiz/features` only if a feature was requested
   - For general Streamlit / AISQL, the bundled `developing-with-streamlit-in-snowflake` / `cortex-ai-function-studio`. `$sis`/`$cortex` carry only the project deltas.

   **Generation rules — apply WHILE writing:** bind params (only `DATABASE`/`SCHEMA`/`CORTEX_MODEL`/`RESPONSE_FORMATS` in f-strings); `@st.cache_data` with no `ttl` + `clear_caches()` after every write; NO `unsafe_allow_html`; every JSON AI call via `call_cortex_json` + `response_format`; `config.toml` `showErrorDetails = "none"` (string); `st.set_page_config(layout="centered")` first `st.` call in `main.py` and nowhere else.

3. Generate the decomposed project (file responsibilities → `$quiz`; the app module map is in `$quiz`):
   ```
   app/
     main.py            _config.py   _cortex.py   _data.py   _questions.py   _ui.py   _search.py
     pages/  quiz.py   review.py   admin.py        # + <feature>.py ONLY for requested features
     .streamlit/config.toml   environment.yml   snowflake.yml
   ```
   **Generate `snowflake.yml` + `.streamlit/config.toml` FIRST** — `snowflake.yml` at the project root is what makes the Workspace recognize the folder as a Streamlit app (no "Convert to Streamlit app" click). If the user chose "No CSV", default `question_source = 'ai'`.

4. **`main.py`**: `st.set_page_config(layout="centered", …)` first; `init_session_state()`; shared sidebar title; build navigation explicitly (with `st.navigation` the `pages/` dir is NOT auto-discovered):
   ```python
   pages = [st.Page("pages/quiz.py", title="Quiz", default=True),
            st.Page("pages/review.py", title="Review"),
            st.Page("pages/admin.py", title="Admin")]
   # append st.Page("pages/<feature>.py", ...) ONLY for generated features
   st.navigation(pages).run()
   ```

5. **`environment.yml`** (warehouse runtime, Snowflake Anaconda channel):
   ```yaml
   name: snowpro_quiz
   channels: [snowflake]
   dependencies: [streamlit, pandas, altair, snowflake]
   ```
   `snowflake` (the Snowflake Python API, unpinned) provides `snowflake.core`, which the CKE doc-grounding helper (`_search.py`, `$cortex`) imports — omit it and the app dies at load with `ModuleNotFoundError: No module named 'snowflake.core'`.

6. **`.streamlit/config.toml`** — `[client] showErrorDetails = "none"` (the string, NOT `false`, which leaks tracebacks) + `toolbarMode = "minimal"`. The `[theme]`/`[theme.sidebar]` block comes from **`$quiz/design`** (the canonical default theme lives there; or the user's Step 1e custom palette) — never author theme values here from memory.

7. **`snowflake.yml`** — `identifier` = `app_name` from AGENTS.md; **list EVERY generated file in `artifacts`** (an incomplete list = a partial / broken deploy):
   ```yaml
   definition_version: 2
   entities:
     quiz_app:
       type: streamlit
       identifier: SNOWPRO_QUIZ
       stage: STAGE_SIS_APP
       query_warehouse: {warehouse}
       main_file: main.py
       artifacts: [main.py, _config.py, _cortex.py, _data.py, _questions.py, _ui.py, _search.py, pages/, environment.yml, .streamlit/config.toml]
   ```
   Add each `pages/<feature>.py` to `artifacts`. Do NOT set `pages_dir` (navigation is `st.navigation`-controlled) or `execute_as`/`run_mode` (caller's-rights Preview; this app is owner-rights).

8. **Run the `$sis` pre-deploy scan across ALL app files as a final confirmation.** If you applied the item-2 rules it reports 0 failures — that's the target; >0 means a rule was skipped during generation (fix + re-scan). ⚠️ Do NOT deploy on any FAIL.

## Step 8.5 — Self-verify (OPTIONAL — advanced mode)

Only if **self-verify** was enabled (Step 1d) and the session can execute code (Cloud Agents). Byte-compile every module (`python -m py_compile app/main.py app/_*.py app/pages/*.py`) — Snowflake-bound modules can't run outside SiS, so compile/parse only; on failure fix → re-run the `$sis` scan → repeat. If the session can't execute code, say so and skip. Supplements the Step 8 scan, never replaces it.

## Step 9 — Deploy (warehouse, fully scriptable — no manual upload)

The agent can't click the Workspaces UI or `PUT`, but the workspace files already live on an internal stage — so the agent copies them onto `STAGE_SIS_APP` with SQL and creates the app on the **warehouse runtime** — works on trial accounts.

**1. Copy `app/` from the workspace stage → `STAGE_SIS_APP`.** Workspace files sit at `snow://workspace/USER$.PUBLIC."<workspace_name>"/versions/live/app/` — **confirm the exact URI first** (the workspace name varies), then copy, preserving the `pages/` and `.streamlit/` subfolders:
```sql
LIST 'snow://workspace/USER$.PUBLIC."<workspace_name>"/versions/live/app/';  -- confirm URI + files

COPY FILES INTO @{database}.QUIZ_<CODE>.STAGE_SIS_APP
  FROM 'snow://workspace/USER$.PUBLIC."<workspace_name>"/versions/live/app/'
  FILES = ('main.py','_config.py','_cortex.py','_data.py','_questions.py','_ui.py','_search.py','environment.yml');
COPY FILES INTO @{database}.QUIZ_<CODE>.STAGE_SIS_APP/pages/
  FROM 'snow://workspace/USER$.PUBLIC."<workspace_name>"/versions/live/app/pages/'
  FILES = ('quiz.py','review.py','admin.py');  -- + each <feature>.py
COPY FILES INTO @{database}.QUIZ_<CODE>.STAGE_SIS_APP/.streamlit/
  FROM 'snow://workspace/USER$.PUBLIC."<workspace_name>"/versions/live/app/.streamlit/'
  FILES = ('config.toml');
```

**2. Verify the stage** (root + `pages/` + `.streamlit/` all present):
```sql
LIST @{database}.QUIZ_<CODE>.STAGE_SIS_APP;
```

**3. Create (or replace) the app on the warehouse runtime** — packages come from the Snowflake Anaconda channel via `environment.yml`; no internet:
```sql
CREATE OR REPLACE STREAMLIT {database}.QUIZ_<CODE>.SNOWPRO_QUIZ
  FROM '@{database}.QUIZ_<CODE>.STAGE_SIS_APP' MAIN_FILE = 'main.py'
  QUERY_WAREHOUSE = {warehouse};
```
**Redeploy after an edit:** save the file in the workspace, re-run the relevant `COPY FILES` (it overwrites same-named files) + `CREATE OR REPLACE STREAMLIT`; confirm with `LIST` that the changed file's size updated before assuming it took.

Verify:
```sql
SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA {database}.QUIZ_<CODE>;
```

## Step 10 — Verify + report

Confirm `SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA {database}.QUIZ_<CODE>;` returns 1 row and the previous exam's schema is untouched.

**App URL — do NOT use `CURRENT_ACCOUNT()`** (that's the account locator, not the URL slug). Use org + account name:
```sql
SELECT LOWER(CURRENT_ORGANIZATION_NAME()) AS org, LOWER(CURRENT_ACCOUNT_NAME()) AS account;
```
→ `https://app.snowflake.com/{org}/{account}/#/streamlit-apps/{database}.QUIZ_<CODE>.SNOWPRO_QUIZ`. If either function is NULL, ask the user for their Snowsight base URL (the part up to `/#/`).

Report: exam name + code, schema, domains extracted (N), questions loaded (N or 0=AI-only), app name, **app URL**, features implemented, advanced options active. Then `ask_user_question`: **Done** / **Review**.

---

# Stopping Points

`ask_user_question` (or numbered-list fallback) at each:
- **1a** — halt on unfilled `<...>` placeholders; resume when filled.
- **1c** — confirm a PDF exists; Yes/No on an optional CSV (no filenames — resolved at Step 4's `LIST`).
- **1e** — default vs custom look; if custom, run the dialog + confirm palette before Step 2.
- **1f** — warehouse runtime needs nothing checked; skip to 1g.
- **4** — `LIST` the workspace stage, `COPY FILES` the PDF (+ optional CSV) onto `STAGE_QUIZ_DATA`, verify with `LIST`, resolve the real filenames for Steps 5–6. STOP only if the PDF isn't in the workspace.
- **5 (conditional)** — pick among conflicting domain structures.
- **5 verify** — Approve / Re-extract / Abort.
- **8 scan** — every `$sis` item must PASS; do NOT deploy on any FAIL.
- **8.5** (if self-verify) — modules compile-clean, else fix → re-scan.
- **9** — copy `app/` → `STAGE_SIS_APP` (`COPY FILES`) + `CREATE STREAMLIT` (warehouse runtime). Verify with `SHOW STREAMLITS`.
- **10** — report; Done / Review.

**Resume rule:** on approval, proceed without re-asking.

---

# Important Notes

- **Never drop or modify the previous exam's schema** — exams coexist in separate schemas (`QUIZ_<CODE>` is the only isolation; no git here).
- **Every query references `{database}.QUIZ_<CODE>`** — double-check.
- **Env schema is authoritative; resume by probing, not assuming** — after a chat reload, target only the schema AGENTS.md env names, check what already exists (Step 1a), resume from the first incomplete step, and verify any "you already did X" claim against that schema — never redo work or write into a different `QUIZ_*` schema.
- **Files reach stages via `COPY FILES` from the workspace stage** (no `PUT` in Snowsight) — the user drops files into the workspace file tree; the agent copies them onto `STAGE_QUIZ_DATA` (PDF/CSV, Step 4) and `STAGE_SIS_APP` (the `app/`, Step 9), then `LIST`s to verify.
- **If a step fails**, diagnose, fix, retry — don't skip.
- **Bank needs schema adaptation?** → `$adapt-questions` before Step 6.

## Output

A deployed quiz app in a dedicated schema: domains extracted from the PDF, the bank loaded (CSV) or empty (AI-only), the `app/` project generated and passing the `$sis` scan.
