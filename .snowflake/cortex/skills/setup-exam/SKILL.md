---
name: setup-exam
description: "Automated 10-step exam setup pipeline for Snowflake CoCo in Snowsight. Creates Snowflake schema, extracts domains from PDF, generates questions, builds the multipage `app/` Streamlit project, and deploys it on the container runtime. Works for first exam or adding a new one. Triggers: setup exam, new exam, new certification, new study guide PDF, add exam, switch exam, create exam"
---

# When to Use

Use this skill when setting up a new Snowflake certification exam or switching from one to another. Each exam gets its own Snowflake schema — nothing is overwritten.

Example prompts:
- "I have a new study guide - SnowProGenAIStudyGuide.pdf. Create a quiz app for this exam."
- "Switch to the SnowPro Specialty: Gen AI exam."

# When NOT to Use

- Do not use this skill if the exam schema is already set up (check `SHOW SCHEMAS LIKE 'QUIZ_<CODE>' IN DATABASE {database};`).
- Do not use for fixing bugs in an existing quiz — use `$cortex/patterns`, `$cortex/prompt-audit`, or `$sis/pre-deploy` instead.

---

# Environment: Snowsight Workspace

This skill runs inside **CoCo in Snowsight**. Assumptions:
- No bash / shell / git / `snow` CLI.
- No `PUT file://...` from this agent — the workspace cannot read local filesystem.
- Files are uploaded **manually by the user** via Snowsight UI (Data » Databases » Stages » + Files, or Ingestion » Add Data » Load files into a Stage).
- Generated artefacts (the `app/` Streamlit project — `main.py`, `_*.py` modules, `pages/`, configs) are written as **files in this workspace** by the agent. Deploy is via the Workspaces **Run + Deploy** flow (default) or by the user uploading `app/` to a stage for a scripted `CREATE STREAMLIT` (see Step 9).
- Exam isolation is **schema-per-exam only** (`QUIZ_<CODE>`). No git branch.

---

# Exam Identification

Do NOT hardcode exam names or codes. Instead:
1. Read the PDF study guide — extract the certification name from the document.
2. ALWAYS ask the user to confirm: "I found exam: {exam_name}. What is the exam code? (e.g., COF-C03)"
3. The user provides the definitive exam code.

Snowflake certifications reference: https://learn.snowflake.com/en/certifications/

---

# Prerequisites

Read `{database}`, `{warehouse}`, and `{role}` from the environment table in `AGENTS.md`. All SQL in this skill uses these placeholders — substitute the actual values from that table. If any value is still `<your_...>`, Step 1a halts and prompts the user to fill AGENTS.md before continuing.

---

# Tools

### ask_user_question

**Description:** Present the user with a fixed list of options and wait for their selection.

**When to use:** At mandatory stopping points where the user must choose between defined options (confirmations, approvals, strategy selections). Do NOT use for free-text input.

**Fallback:** If `ask_user_question` is not available, present the same options as a numbered list and wait for the user's text response.

---

# Instructions

> **Step 0 — Read this entire skill before acting.** Do not run any SQL, create any object, or generate any file until you have read every step below. This is a 10-step pipeline with mandatory STOP points; each STOP prevents a known, costly failure (stage without `SNOWFLAKE_SSE` → `AI_PARSE_DOCUMENT` fails; unfilled `<...>` placeholders → wrong/missing objects; container deploy without a compute pool + PyPI EAI → deploy fails at the package server). **Do NOT improvise this pipeline from the AGENTS.md overview** — follow these steps in order. You create nothing before Step 2.

## Step 1 — Collect inputs

### 1a — Validate AGENTS.md environment config (mandatory guard)

**Echo the full `snowflake environment` table values back to the user** (so the check is visible), then scan EVERY row for `<...>` placeholder syntax. Build a list of every value still containing angle brackets — `<your_database>`, `<your_warehouse>`, `<your_role>`, `<your_compute_pool>`, and any others.

If that list is non-empty, **STOP immediately** — do not create anything, do not proceed. Show the user exactly which placeholders remain:

> "I can't start setup yet — the `snowflake environment` table in `AGENTS.md` still has these placeholders: **{list}**. Please replace:
> - `<your_database>` → the database where the schema will live
> - `<your_warehouse>` → the warehouse to use
> - `<your_role>` → the role with `CREATE SCHEMA` on that database
> - `<your_compute_pool>` → the compute pool for the container runtime (you may leave this only if you'll use the warehouse fallback — I'll check in Step 1f)
>
> Leave `schema` and `exam_code` as is — I fill those once we know the exam code. Tell me when done and I'll re-read AGENTS.md."

Wait for confirmation, re-read AGENTS.md, and re-run the scan. **Do not proceed to 1b while any required `<...>` remains.** (This guard is not optional — skipping it was a real failure: the pipeline ran with an unfilled `<your_compute_pool>` and broke at deploy.)

Verify session context matches:
```sql
SELECT CURRENT_ROLE(), CURRENT_WAREHOUSE(), CURRENT_DATABASE();
```
If the session is using a different role/warehouse/database than AGENTS.md declares, ask the user whether to `USE` the AGENTS.md values or update AGENTS.md to match the session.

### 1b — Determine target exam

Extract the exam name from the user's prompt (e.g., "set up SnowPro Core" → "SnowPro Core"). ALWAYS ask the user for the exam code — no hardcoded mapping:

> "I understand the target exam is: **{exam_name}**. What is the exam code? (e.g., COF-C03)"

Wait for the user to confirm both name and code before proceeding.

### 1c — Ask about input files

The agent cannot list local filesystem; ask the user directly what files they have.

**Study guide PDF — MANDATORY:**

> "I need the study guide PDF for {exam_name}. What is the filename (e.g., `SnowProCoreStudyGuide_c03.pdf`)?"

If the user says no PDF is available → **STOP**. The study guide is required for `AI_PARSE_DOCUMENT` → `EXAM_DOMAINS`.

**Question bank CSV/JSON — OPTIONAL:**

**Use `ask_user_question` tool (if available):**

> "Do you have a question bank file (CSV or JSON) for this exam?"

| Option | Description |
|--------|-------------|
| **Yes, I have a file** | I will upload a CSV/JSON with pre-authored questions |
| **No, generate via AI** | Skip question bank — questions will be generated by AI in Step 7 |

**Routing logic:**
- **Yes, I have a file** → ask for filename; continue to Step 1d.
- **No, generate via AI** → continue to Step 1d; question_source default will be `'ai'` on the app's home screen.

### 1d — Additional requirements

Ask: "Any additional requirements or customizations? (e.g., specific UI features, AI study recommendations, different scoring — or advanced mode: quality model profile, self-verify, Automations)"

Advanced options (see AGENTS.md `Advanced options` — all OFF unless explicitly requested here):
- **quality model profile** → set `CORTEX_MODEL = "claude-opus-4-7"` in `_config.py` at Step 8 (do NOT change AGENTS.md defaults).
- **self-verify** → run Step 8.5 after the scan.
- **automations** → after Step 10, point the user to the report-only recipe in `docs/customization.md` section 5c (**Preview** — verify account availability).

If the user mentions features not already in AGENTS.md, ask clarifying questions about requirements BEFORE proceeding to Step 2.

Wait for the user's response before proceeding.

### 1e — App look & feel

**Use `ask_user_question` tool (if available):**

> "Do you want to decide the app's look (colors, style), or use the default theme?"

| Option | Description |
|--------|-------------|
| **Default look** | Clean Snowflake-blue theme (recommended) — no further questions |
| **Custom look** | I'll ask a few quick style questions and theme the app to your taste |

**Routing logic:**
- **Default look** → use the canonical theme from `$quiz/style` in Step 8. Continue.
- **Custom look** → run a short guided dialog (one question at a time): light or dark base; primary/accent color (name or hex); corner roundness (sharp / soft / round); font stack (sans-serif / serif / monospace); sidebar tint (same as app / subtle contrast). Map answers ONLY to native Streamlit `[theme]` / `[theme.sidebar]` keys per the `$quiz/style` theming contract — **never CSS, never `unsafe_allow_html`, no external font files (CSP)**. Confirm the resulting palette back to the user in words before Step 2.

Store the choice for Step 8 (config.toml generation).

### 1f — Deploy prerequisites (container runtime is the default)

The default deploy target is the **container runtime** (Streamlit-in-Workspaces live preview). It has TWO account-level prerequisites that, if missing, make the deploy FAIL at the package server (`Failed to retrieve package... Have you enabled External Access Integration?`). Check BOTH now — early, so the user can fix them while the rest of the pipeline runs — and **STOP** if either is missing and unresolved.

**1. Compute pool** (runs the container):
```sql
SHOW COMPUTE POOLS;
```
The role needs `USAGE` on at least one. If none exists, an admin must create/grant one.

**2. PyPI external access integration** — the container installs `pandas`/`altair` from PyPI; they are NOT in the base image (only Python, Streamlit, Snowpark are), so an EAI is mandatory. Ask whether one exists and is granted to the role. If not, give the user this ACCOUNTADMIN DDL (Snowflake ships the managed network rule — no custom rule needed):
```sql
USE ROLE ACCOUNTADMIN;
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION pypi_access_integration
  ALLOWED_NETWORK_RULES = (snowflake.external_access.pypi_rule)
  ENABLED = TRUE;
GRANT USAGE ON INTEGRATION pypi_access_integration TO ROLE {role};
```
Record the EAI name (the `external_access_integration` value in AGENTS.md) — it is attached to the app at deploy (Step 9).

**⚠️ STOP** if either prerequisite is missing and the user can't resolve it. Offer the **warehouse fallback** (Step 9 Path C): no compute pool, no EAI, `environment.yml` from the Snowflake Anaconda channel, Streamlit 1.52.2. Confirm container vs warehouse before continuing — it sets the deps file (`pyproject.toml` vs `environment.yml`) and the deploy path.

## Step 2 — Create Snowflake schema

Each exam gets a dedicated schema. Replace `<EXAM_CODE>` with the mapped code, hyphens replaced by underscores (e.g. `COF-C03` → `QUIZ_COF_C03`).

```sql
CREATE SCHEMA IF NOT EXISTS {database}.QUIZ_<EXAM_CODE>;
USE SCHEMA {database}.QUIZ_<EXAM_CODE>;
```

The configured role has full permissions on `{database}` — no grants needed.

## Step 3 — Create stages, tables, file format

**Stages** (STAGE_QUIZ_DATA MUST have encryption + directory for AI_PARSE_DOCUMENT):
```sql
CREATE STAGE IF NOT EXISTS {database}.QUIZ_<CODE>.STAGE_QUIZ_DATA
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  DIRECTORY = (ENABLE = TRUE);

CREATE STAGE IF NOT EXISTS {database}.QUIZ_<CODE>.STAGE_SIS_APP;
```

**Copy the stage DDL above verbatim — do NOT write `CREATE STAGE` from memory.** A bare `CREATE STAGE` defaults to client-side encryption, which makes `AI_PARSE_DOCUMENT` fail with "Client Side Encryption is not supported" (a real failure that cost a re-upload and several wrong syntaxes). Immediately verify:

```sql
DESCRIBE STAGE {database}.QUIZ_<CODE>.STAGE_QUIZ_DATA;
-- confirm in the output: encryption TYPE = SNOWFLAKE_SSE, and DIRECTORY enabled = true
```

If encryption is not `SNOWFLAKE_SSE`, drop and recreate with the exact DDL before any upload:
```sql
DROP STAGE {database}.QUIZ_<CODE>.STAGE_QUIZ_DATA;
-- then re-run the CREATE STAGE ... ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE') DIRECTORY = (ENABLE = TRUE);
```

**Tables** — all 5 with exact DDL below:

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
    selected_answer VARCHAR,
    mnemonic       VARCHAR(500),
    doc_url        VARCHAR(500),
    misconception  VARCHAR
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
```

```sql
CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.QUIZ_CONFIG (
    config_key   VARCHAR PRIMARY KEY,
    config_value VARIANT,
    updated_at   TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);
```

Rationale: QUIZ_REVIEW_LOG stores per-question wrong answers (Review page + domain error analysis; `selected_answer`/`misconception` feed the optional misconception-analysis feature). QUIZ_SESSION_LOG stores per-round summaries (score, round size — needed for progress metrics). Rounds with 0 wrong answers have no QUIZ_REVIEW_LOG rows, so merging would lose session data. QUIZ_CONFIG holds runtime app configuration edited from the Admin page (defaults live in `_config.py`; DB values override them). If the user enables the flag-a-question feature, `$quiz/features` Feature 8 adds a QUIZ_FLAGS table.

**File format:**
```sql
CREATE FILE FORMAT IF NOT EXISTS {database}.QUIZ_<CODE>.FF_CSV
  TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"';
```

## Step 4 — User uploads input files to stage (MANUAL)

Present clear instructions to the user. The agent cannot upload files; the user must do this in Snowsight UI.

**PDF upload (required):**

> "Please upload the study guide PDF to the stage:
> 1. In Snowsight, go to **Data » Databases » {database} » QUIZ_<CODE> » Stages » STAGE_QUIZ_DATA**.
> 2. Click **+ Files** (top-right).
> 3. Drag-drop or browse to select `{pdf_filename}`.
> 4. Click **Upload**.
>
> Alternative: **Ingestion » Add Data » Load files into a Stage**.
>
> Let me know when upload is complete."

**CSV upload (only if user has one, from Step 1c):**

> "Do the same for the CSV: upload `{csv_filename}` to the same stage."

Wait for user confirmation. Then verify:

```sql
ALTER STAGE {database}.QUIZ_<CODE>.STAGE_QUIZ_DATA REFRESH;
LIST @{database}.QUIZ_<CODE>.STAGE_QUIZ_DATA;
```

The file list must include the PDF (and CSV if provided). If any expected file is missing → ask the user to re-upload before proceeding.

## Step 5 — Extract domains from PDF

### 5a — Parse PDF content

```sql
SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@{database}.QUIZ_<CODE>.STAGE_QUIZ_DATA', '<pdf_filename>'),
    {'mode': 'LAYOUT'}
):content::VARCHAR AS doc_content;
```

Save the result — reuse it for domain extraction AND key_facts. Do NOT re-call `AI_PARSE_DOCUMENT` per domain.

### 5a.1 — Analyze document structure for date-based changes

BEFORE extracting domains, analyze the full parsed PDF content for:
1. Mentions of effective dates, transition dates, or "effective from" language
2. Multiple conflicting domain structures (e.g., "old exam blueprint" vs "new exam blueprint")
3. Version indicators or restructuring notices

If only one structure is found and there is no ambiguity, use it directly (no `ask_user_question` needed).

If conflicting domain structures exist, **use `ask_user_question` tool (if available):**

> "I found multiple domain structures in this study guide."

| Option | Description |
|--------|-------------|
| **Structure A** | {structure_a_name} (effective {date_a}) — {N} domains |
| **Structure B** | {structure_b_name} (effective {date_b}) — {N} domains |

**Routing logic:**
- Use the structure the user selects.
- Default recommendation: the one effective as of today's date.

This step is critical because study guides are sometimes published during exam transitions and contain both old and new structures.

### 5b — Extract exam domains

Use AI_COMPLETE on the parsed content. The prompt must ask for a JSON array:

```sql
SELECT AI_COMPLETE(
    'claude-sonnet-4-6',
    $$Today's date is {current_date}. If multiple exam blueprints are shown, use only the one effective as of today.

Extract ALL exam domains from this certification study guide.
Return ONLY a valid JSON array. Each object must have:
- domain_id (string, e.g. "1", "2", sequential)
- domain_name (string, exact name from the guide)
- weight_pct (number, percentage weight - must sum to 100 across all domains)
- topics (JSON array of topic strings covered in this domain)

Study guide content:
{doc_content}$$
)::VARCHAR;
```

Parse the response and INSERT each domain into EXAM_DOMAINS.

**Optional alternative — `AI_EXTRACT`** (one call, keyed JSON, no free-form prompt). Offer it only if the AI_COMPLETE extraction struggles (e.g. messy PDF structure); AI_COMPLETE stays the default:

```sql
SELECT AI_EXTRACT(
  text => :doc_content,
  responseFormat => {
    'domains': 'List every exam domain name, in order',
    'weights': 'List each domain percentage weight (numbers summing to 100), same order',
    'topics' : 'For each domain, list the topics it covers, same order'
  });
```

Map the keyed arrays into EXAM_DOMAINS rows (domain_id = position as string). The key_facts extraction (5c) still uses AI_COMPLETE either way.

### 5c — Extract key_facts per domain

**Reuse the `doc_content` from Step 5a.** Store it in a session variable, temporary table, or pass via CTE — do NOT re-call AI_PARSE_DOCUMENT for each domain.

For each domain, run a separate AI_COMPLETE call to extract testable facts:

```sql
UPDATE {database}.QUIZ_<CODE>.EXAM_DOMAINS
SET key_facts = (
    SELECT AI_COMPLETE(
        'claude-sonnet-4-6',
        $$Extract the key testable facts for the "{domain_name}" domain from this study guide.
Focus on facts that could appear as exam questions: definitions, limits, best practices, feature names, SQL syntax.
Return a plain text list, one fact per line. No JSON, no markdown.

Study guide content:
{doc_content}$$
    )::VARCHAR
)
WHERE domain_id = {id};
```

Run one UPDATE per domain. Verify each sets a non-null value before moving to the next.

### 5d — Verify

```sql
SELECT COUNT(*) FROM {database}.QUIZ_<CODE>.EXAM_DOMAINS;
SELECT SUM(weight_pct) FROM {database}.QUIZ_<CODE>.EXAM_DOMAINS;
SELECT domain_name, LENGTH(key_facts) AS facts_len FROM {database}.QUIZ_<CODE>.EXAM_DOMAINS ORDER BY domain_id;
```

Expected: N domains (varies per exam), weights sum to 100, all facts_len > 0.

**⚠️ MANDATORY STOPPING POINT — Use `ask_user_question` tool (if available):**

Present the domain verification results, then ask:

> "Domain extraction complete. {N} domains found, weights sum to {sum}."

| Option | Description |
|--------|-------------|
| **Approve** | Domains look correct — proceed to question loading |
| **Re-extract** | Something is wrong — re-run domain extraction |
| **Abort** | Stop the pipeline — I need to review manually |

**Routing logic:**
- **Approve** → proceed to Step 6.
- **Re-extract** → clear EXAM_DOMAINS and re-run Step 5b-5d.
- **Abort** → **STOP**.

⚠️ **STOP**: Do NOT proceed until user responds.

## Step 6 — Load the question bank (optional)

### 6a — If user has a CSV (from Step 1c) already uploaded in Step 4

Check if rows were already inserted (e.g., by `$adapt-questions`):
```sql
SELECT COUNT(*) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS;
```
If rows exist, skip loading and go to Verify.

Otherwise, load the CSV:
```sql
COPY INTO {database}.QUIZ_<CODE>.QUIZ_QUESTIONS
FROM @{database}.QUIZ_<CODE>.STAGE_QUIZ_DATA/<csv_filename>
FILE_FORMAT = {database}.QUIZ_<CODE>.FF_CSV;
```

If column order/types differ from the target schema, invoke `$adapt-questions` first.

Backfill `domain_name`:
```sql
UPDATE {database}.QUIZ_<CODE>.QUIZ_QUESTIONS q
SET q.domain_name = d.domain_name
FROM {database}.QUIZ_<CODE>.EXAM_DOMAINS d
WHERE q.domain_id = d.domain_id AND q.domain_name IS NULL;
```

### 6b — If no CSV — the bank stays empty (do NOT generate questions now)

**Do NOT generate a question bank during setup.** Build-time generation is slow, burns the user's token budget before they ever see the app, and confuses users ("are these the only questions?"). The app is fully functional without a bank: `question_source` defaults to `'ai'` (runtime generation, one question at a time).

Inform the user (verbatim or close):

> "I'm skipping question-bank pre-generation — the app generates questions live via AI. A populated bank is still worth having, because:
> - **resilience**: if an AI call fails (service interruption, cross-region issue, token limits), the app falls back to bank questions;
> - **speed**: bank questions load instantly, no AI round-trip;
> - **consistency**: a curated, repeatable question set.
>
> You can seed it anytime: (1) upload a CSV/JSON now or later (I'll run `$adapt-questions`), (2) use the **Generate batch** button on the app's Admin page, (3) run the worksheet SQL recipe from `docs/customization.md` (section: Seeding the question bank), or (4) schedule it as a recurring task / Automation."

If the user decides to upload a CSV after all → go back to 6a.

Note: the bank feeds the "From question bank" source mode and the AI-fallback path. Runtime "AI Generated" questions never come from the bank.

### 6c — Verify

```sql
SELECT COUNT(*) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS;
SELECT COUNT(DISTINCT domain_id) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS;
SELECT COUNT(*) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS WHERE domain_name IS NULL;
```

Without a CSV, QUIZ_QUESTIONS legitimately has 0 rows (runtime-AI-only mode) — state this and proceed; do not treat it as an error.

## Step 7 — Update AGENTS.md

Use the Edit tool on `AGENTS.md`. Follow the edit boundaries strictly.

### AGENTS.md Edit Boundaries

**CAN edit:**
- Schema name in the environment table (e.g., `QUIZ_COF_C03` → `QUIZ_GES_C01`)
- Exam code in the environment table (e.g., `COF-C03` → `GES-C01`)
- Title line (exam name)
- New feature specification sections (append to end of relevant section if user requested optional features not in AGENTS.md)

**MUST NOT edit:**
- Database, warehouse, role (user-set values in env table — preconditions)
- Environment: Snowsight Workspace section (platform constraints)
- Domain model section (4 tables named — the column details live in this skill)
- Cortex LLM section (model constant + cross-region note)
- App overview section (screen flow, function list)
- Available skills index
- Security and governance principles

### What to update

1. **Schema and exam code**: update both rows in the environment table.
2. **Title**: update to new exam name.
3. **Additional requirements**: if user requested features not in AGENTS.md, append new subsections. Do not modify existing subsections.

## Step 8 — Build the `app/` Streamlit project in workspace

1. Read the updated AGENTS.md fully.
2. **MANDATORY: read these skills BEFORE writing any code.** They are generation rules, not a post-hoc linter — reading them first is what makes the pre-deploy scan pass on the first try. (Skipping this step produced 10+ scan failures and a costly fix pass in a real run.)
   - `$sis/patterns` — caching (no `ttl` + `clear_caches()`), widget lifecycle, container-runtime constraints, SQL safety
   - `$sis/pre-deploy` — the 21 rules your generated code must ALREADY satisfy
   - `$cortex/patterns` — `call_cortex_json` + `response_format` structured outputs (no fence parsing), untrusted-content delimiting
   - `$quiz/screens` — page flow, session state, explanation/hint/contrast/debrief/remedial contracts, write-back + `clear_caches()`
   - `$quiz/questions` — DIFFICULTY_GUIDE (REQUIRED constant), answer shuffling, validation, retry logic
   - `$quiz/style` — EXAM_NAME constant, theming contract (config.toml keys), chart colors (#29b5e8 blue, #F1914C orange)
   - If the user requested optional features: also read `$quiz/features`
   - For general Streamlit patterns consult the bundled `developing-with-streamlit`; for AISQL, `cortex-ai-functions`. This project's `$sis`/`$cortex` carry only the project deltas.

   **Generation rules — apply WHILE writing (these were the actual first-pass failures):**
   - SQL uses bind params (`:1, :2, …`); only `DATABASE`/`SCHEMA`/`CORTEX_MODEL`/`RESPONSE_FORMATS` constants may be f-string-interpolated.
   - `@st.cache_data` loaders have **no `ttl`**; every DB write calls `clear_caches()`.
   - **NO `unsafe_allow_html`** anywhere — styling lives in `.streamlit/config.toml`.
   - Every JSON AI call uses `call_cortex_json(prompt, fmt_key)` with a `response_format` schema — no markdown-fence parsing.
   - `.streamlit/config.toml` has `showErrorDetails = "none"` (the string, NOT `false`).
   - `st.set_page_config(layout="centered")` is the FIRST `st.` call in `main.py`, and appears nowhere else.
3. Generate a decomposed multipage project under `app/` (NOT a single-file `quiz.py`):

   ```
   app/
     main.py              # entry point: st.set_page_config + init_session_state + st.navigation
     _config.py           # EXAM_NAME, EXAM_CODE, CORTEX_MODEL, PASS_THRESHOLD, DIFFICULTY_GUIDE, color constants
     _cortex.py           # call_cortex() and the JSON-returning Cortex helpers
     _data.py             # cached loaders (domains, session stats, recent sessions, domain errors) + clear_caches()
     _questions.py        # topic schedule, get_question, AI generation, answer shuffling, dedup
     _ui.py               # shared render helpers: badges, cards, explanation expander, docs link
     pages/
       quiz.py            # QUIZ page: home -> quiz -> summary state machine
       review.py          # REVIEW page: wrong answers + learning dashboard
       admin.py           # ADMIN page: app config, question manager, bank stats, spend, tools
     .streamlit/config.toml
     pyproject.toml       # container-runtime deps (default)
     snowflake.yml        # deploy descriptor (container)
   ```

   - Generate `pages/<feature>.py` (e.g. `exam_simulation.py`, `flashcards.py`, `recommendations.py`) ONLY for features the user requested in Step 1d.
   - If the user chose "No CSV" in Step 1c → default `question_source = 'ai'` on the home screen.

   **Generation order: create `snowflake.yml` and `.streamlit/config.toml` FIRST** (items 6–7 below), before the code modules. The Workspace recognizes the folder as a Streamlit app via `snowflake.yml` at the project root — generate it first and the app is recognized immediately; you never need the user to click "Convert to Streamlit app". (The Workspace scaffold's default entry file is `streamlit_app.py`; we use `main.py` and point `main_file: main.py` at it.)

4. **`main.py` responsibilities** (entry point):
   - `st.set_page_config(layout="centered", ...)` MUST be the very first `st.` call.
   - `init_session_state()` — defaults for all session-state keys. `st.session_state` persists across pages; pages share it.
   - Build the page list and run navigation:

     ```python
     pages = [
         st.Page("pages/quiz.py", title="Quiz", default=True),
         st.Page("pages/review.py", title="Review"),
         st.Page("pages/admin.py", title="Admin"),
     ]
     # append feature pages ONLY if that feature was generated, e.g.:
     # pages.append(st.Page("pages/exam_simulation.py", title="Exam Simulation"))
     st.navigation(pages).run()
     ```

   - Shared sidebar content (app title) renders in `main.py`; each page adds its own widgets.
   - With `st.navigation` in the entry point, the `pages/` directory is NOT auto-discovered — navigation is fully controlled by this list. Do not rely on filename-based auto-pages.

5. Generate `pyproject.toml` (container runtime, PyPI deps) with exactly:

   ```toml
   [project]
   name = "snowpro_quiz"
   version = "1.0.0"
   requires-python = "==3.11.*"
   dependencies = [
     "streamlit[snowflake]",
     "pandas",
     "altair",
   ]
   ```

6. Generate `.streamlit/config.toml`. The `[client]` block is fixed; the `[theme]` block comes from Step 1e — the user's custom choices mapped per the `$quiz/style` theming contract, or (default) the canonical theme below:

   ```toml
   [client]
   showErrorDetails = "none"     # "none", NOT false — the deprecated false maps to "stacktrace" and still leaks tracebacks

   [theme]
   base = "light"
   primaryColor = "#29b5e8"
   linkColor = "#1572a1"
   baseRadius = "0.5rem"
   borderColor = "#d6e4ec"
   showWidgetBorder = true
   chartCategoricalColors = ["#29b5e8", "#F1914C", "#36B37E", "#7C5CFC"]

   [theme.sidebar]
   secondaryBackgroundColor = "#eef6fa"
   ```

   Rules: ONLY native `[theme]`/`[theme.sidebar]` keys (full key reference in `$quiz/style`); no CSS, no external `fontFaces` (CSP); `chartCategoricalColors[0..1]` MUST match the chart constants in `_config.py`.

7. Generate `snowflake.yml` — **this is what makes the Workspace treat the folder as a Streamlit app** (also drives Path B / Snowflake CLI). Use `identifier` = the `app_name` from AGENTS.md, attach the PyPI EAI, and **list EVERY generated file in `artifacts`** (an incomplete `artifacts` list = a broken/partial deploy — this was an observed bug):

   ```yaml
   definition_version: 2
   entities:
     quiz_app:
       type: streamlit
       identifier: SNOWPRO_QUIZ              # = app_name in AGENTS.md
       stage: STAGE_SIS_APP
       query_warehouse: {warehouse}
       compute_pool: {compute_pool}
       runtime_name: SYSTEM$ST_CONTAINER_RUNTIME_PY3_11
       external_access_integrations:
         - pypi_access_integration           # required: lets the container install pandas/altair from PyPI
       main_file: main.py
       artifacts:                            # MUST list every generated file (+ each optional-feature page)
         - main.py
         - _config.py
         - _cortex.py
         - _data.py
         - _questions.py
         - _ui.py
         - pages/
         - pyproject.toml
         - .streamlit/config.toml
   ```

   Add `pages/<feature>.py` to `artifacts` for each generated optional-feature page. Do NOT set `pages_dir` (navigation is `st.navigation`-controlled). Do NOT set `execute_as`/`run_mode` (caller's-rights Preview fields; this app uses owner-rights default). **Warehouse fallback (Path C):** omit `compute_pool`, `runtime_name`, and `external_access_integrations`, and swap `pyproject.toml` → `environment.yml` in `artifacts`.

8. `environment.yml` is generated ONLY for the warehouse fallback (Step 9 Path C). Do not emit it on the container path.

9. **Run the `$sis/pre-deploy` scan across ALL app files as a FINAL CONFIRMATION** (`main.py`, `_*.py`, `pages/*.py`, `.streamlit/config.toml`). If you read item 2's skills and applied the generation rules, this reports **0 failures** — that is the target. More than 0 means a rule was skipped during generation; fix and re-scan, but treat repeated failures as a sign you didn't internalize item 2.

All files are written into the current workspace under `app/` (not inside `.snowflake/cortex/skills/`). The user deploys them in Step 9.

## Step 8.5 — Self-verify the generated modules (OPTIONAL — advanced mode)

Run ONLY if the user enabled **self-verify** in Step 1d. Requires a CoCo session that can execute code (Cloud Agents — rolling out since Summit 26). If this session has no code-execution capability, say so explicitly, skip this step, and continue to Step 9.

1. Byte-compile every generated module to catch syntax errors before the user ever clicks Run:
   ```
   python -m py_compile app/main.py app/_config.py app/_cortex.py app/_data.py app/_questions.py app/_ui.py app/pages/*.py
   ```
2. Snowflake-bound modules (`get_active_session`, `AI_COMPLETE`) cannot execute outside SiS — do NOT try to run them; compile/parse checks only.
3. On any failure: fix the module, re-run the `$sis/pre-deploy` scan, then repeat 8.5.
4. Report: list of files checked, PASS/FAIL per file.

This step supplements the Step 8 scan with an execution-level syntax check — it never replaces it.

## Step 9 — Deploy Streamlit app

Three supported deploy paths. **Path A (Workspaces) is the default.** The agent cannot click the UI or execute `PUT`; it instructs the user and verifies with SQL.

### Prerequisites (Paths A and B — already verified in Step 1f)

Container deploy needs BOTH a **compute pool** and the **PyPI external access integration** (`pypi_access_integration`) — both checked in Step 1f. If either is still unresolved, fix it now (Step 1f has the DDL) or fall back to Path C.

### Path A — Workspaces live preview + Deploy (default; Streamlit-in-Workspaces is Public Preview)

1. Present instructions to the user:

   > "The app is generated under `app/` in this workspace. To preview and deploy:
   > 1. Open `app/main.py` and click **Run** (or press Cmd/Ctrl+Enter). This starts a private **dev app** preview in the browser — no stage upload needed. Iterate until it looks right.
   > 2. Click **Deploy** in the project toolbar. In the dialog set: app title `SNOWPRO_QUIZ` (the `app_name` from AGENTS.md), database `{database}`, schema `QUIZ_<CODE>`, **compute pool** `{compute_pool}`, query warehouse `{warehouse}`, and under **Network → External Access Integrations** add `pypi_access_integration` (without it the deploy fails fetching pandas from PyPI — the exact error you'd otherwise hit).
   > 3. Reply 'deployed'."

2. Verify:
   ```sql
   SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA {database}.QUIZ_<CODE>;
   ```

Note: dev-app changes are visible only to the developing user. Other users see the app only after **Deploy** — and after every later edit, only after a re-Deploy.

### Path B — Scripted: stage + CREATE STREAMLIT (container runtime)

1. Ask the user to upload the `app/` files to the stage (**Data » Databases » {database} » QUIZ_<CODE> » Stages » STAGE_SIS_APP » + Files**), preserving the folder layout (`pages/`, `.streamlit/`). Then verify:
   ```sql
   LIST @{database}.QUIZ_<CODE>.STAGE_SIS_APP;
   ```
2. Execute:
   ```sql
   CREATE OR REPLACE STREAMLIT {database}.QUIZ_<CODE>.SNOWPRO_QUIZ
     FROM '@{database}.QUIZ_<CODE>.STAGE_SIS_APP'
     MAIN_FILE = 'main.py'
     RUNTIME_NAME = 'SYSTEM$ST_CONTAINER_RUNTIME_PY3_11'
     COMPUTE_POOL = {compute_pool}
     QUERY_WAREHOUSE = {warehouse}
     EXTERNAL_ACCESS_INTEGRATIONS = (pypi_access_integration);
   ```
   The `EXTERNAL_ACCESS_INTEGRATIONS` clause is required on the container runtime — without it the app can't install pandas/altair from PyPI. (Equivalent from a machine with the Snowflake CLI: `snow streamlit deploy`, driven by the generated `snowflake.yml`.)

### Path C — Warehouse-runtime fallback (no compute pool or no PyPI EAI)

Use when the account has no usable compute pool OR no PyPI external access integration (the warehouse runtime needs neither — it installs from the Snowflake Anaconda channel). Generate `environment.yml` (instead of `pyproject.toml`):

```yaml
name: snowpro_quiz
channels:
  - snowflake
dependencies:
  - streamlit
  - pandas
  - altair
```

Then deploy via the Path B stage flow, but without the container parameters:

```sql
CREATE OR REPLACE STREAMLIT {database}.QUIZ_<CODE>.SNOWPRO_QUIZ
  FROM '@{database}.QUIZ_<CODE>.STAGE_SIS_APP'
  MAIN_FILE = 'main.py'
  QUERY_WAREHOUSE = {warehouse};
```

Note: the warehouse runtime caps Streamlit at 1.52.2 — flag to the user that container-only guidance in `$sis/patterns` does not all apply on this path.

### Choosing between paths

**Use `ask_user_question` tool (if available):**

> "Ready to deploy. Which path do you prefer?"

| Option | Description |
|--------|-------------|
| **Path A — Workspaces (default)** | Run `app/main.py` for a live dev-app preview, then one-click Deploy (compute pool + warehouse). |
| **Path B — scripted via stage** | You upload `app/` to STAGE_SIS_APP, I run `CREATE STREAMLIT` on the container runtime. Fully reproducible. |
| **Path C — warehouse fallback** | No compute pool available: `environment.yml` + warehouse runtime (Streamlit 1.52.2). |

Default: Path A.

## Step 10 — Verify and report

### 10a — Verify deployment

```sql
SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA {database}.QUIZ_<CODE>;
```

Must return 1 row. Confirm the previous exam schema (if any) is untouched.

### 10b — Derive the app URL

**Do NOT construct the URL from `CURRENT_ACCOUNT()`** - that returns the account locator (e.g. `jr65399`), which is not what Snowsight uses in URLs. The correct URL uses the **organization name** + **account name**:

```sql
SELECT
  LOWER(CURRENT_ORGANIZATION_NAME()) AS org,
  LOWER(CURRENT_ACCOUNT_NAME())      AS account;
```

Construct the URL as:

```
https://app.snowflake.com/{org}/{account}/#/streamlit-apps/{database}.QUIZ_<CODE>.SNOWPRO_QUIZ
```

Example: `https://app.snowflake.com/000000/xxxxxxx/#/streamlit-apps/CORTEX_DB.QUIZ_COF_C03.SNOWPRO_QUIZ`

If either function returns NULL (older Snowflake edition or missing privilege), fall back to asking the user: "What is your Snowsight base URL? Open any Snowsight tab and copy the part up to `/#/` from the address bar."

### 10c — Report to the user

Present:
- Exam: {exam_name} ({exam_code})
- Schema: `{database}.QUIZ_<CODE>`
- Domains extracted: N
- Questions loaded: N (or 0 if AI-only)
- Streamlit app: `SNOWPRO_QUIZ`
- **App URL**: (from 10b)
- Any additional features implemented
- Advanced options active, if any (model profile / self-verify result / Automations recipe handed over)

### 10d — Final checkpoint

**Use `ask_user_question` tool (if available):**

> "Deployment complete. All resources created successfully."

| Option | Description |
|--------|-------------|
| **Done** | Looks good - no further action needed |
| **Review** | I want to review something before finalizing |

**Routing logic:**
- **Done** → end skill execution.
- **Review** → ask what they want to review.

---

# Stopping Points

All stopping points below use `ask_user_question` (if available) to present structured options. If the tool is not available, present the same options as a numbered list and wait for the user's text response.

- ⚠️ After Step 1a: Halt if AGENTS.md env config still has `<...>` placeholders; resume when filled.
- ⚠️ After Step 1c: Confirm PDF and ask about optional CSV.
- ⚠️ After Step 1e: Default vs custom look; if custom, run the guided theming dialog and confirm the palette before Step 2.
- ⚠️ After Step 4: Wait for manual upload confirmation; verify via `LIST @stage`.
- ⚠️ After Step 5a.1 (conditional): If conflicting domain structures found, let user choose.
- ⚠️ After Step 5d: Domain verification. Approve/Re-extract/Abort. Do NOT proceed until user responds.
- ⚠️ After Step 8 scan: All items from `$sis/pre-deploy` must PASS across every app file. Do NOT deploy on any FAIL. (No `ask_user_question` — pass/fail gate.)
- ⚠️ After Step 8.5 (only if self-verify enabled): all modules compile-clean; on FAIL fix → re-scan → re-verify. If the session cannot execute code, state it and proceed.
- ⚠️ Step 9: Compute-pool check (`SHOW COMPUTE POOLS`), then Path A / B / C deploy choice; wait for "deployed" (Path A) or upload confirmation (Path B/C).
- ⚠️ After Step 10b: Derive app URL from `CURRENT_ORGANIZATION_NAME()` + `CURRENT_ACCOUNT_NAME()`, NOT from `CURRENT_ACCOUNT()`.
- ⚠️ After Step 10: Deployment report. Done/Review.

**Resume rule:** Upon user approval, proceed directly to next step without re-asking.

---

# Important Notes

- **Never drop or modify the previous exam's schema.** Both exams coexist in separate schemas.
- **All SQL uses the new schema.** Double-check every query references `{database}.QUIZ_<CODE>`.
- **Dollar-quoting for AI_COMPLETE prompts.** Sanitize any `$$` in interpolated content to `$ $`.
- **If any step fails**, diagnose the issue, fix it, and retry. Do not skip steps.
- **Manual upload is the only way to get files onto stages in Snowsight** — the agent cannot execute `PUT`. Always wait for user confirmation + `LIST @stage` check. (Stages are needed for input files in Step 4 and for deploy Paths B/C; deploy Path A needs no stage.)
- **Schema-per-exam** is the only isolation mechanism in this variant. No git, no branches.
- **If a question bank needs schema adaptation**, invoke `$adapt-questions` before Step 6 loading.

---

## Output

A fully deployed quiz app in a dedicated schema, with domains extracted from the PDF, questions loaded (from CSV if provided, otherwise AI-generated), and passing pre-deploy scan.
