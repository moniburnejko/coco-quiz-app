---
name: setup-exam
description: "Automated 9-step exam setup pipeline for Cortex Code in Snowsight. Creates Snowflake schema, extracts domains from PDF, generates questions, builds quiz.py, and deploys. Works for first exam or adding a new one. Triggers: setup exam, new exam, new certification, new study guide PDF, add exam, switch exam, create exam"
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

This skill runs inside **Cortex Code in Snowsight**. Assumptions:
- No bash / shell / git / `snow` CLI.
- No `PUT file://...` from this agent — the workspace cannot read local filesystem.
- Files are uploaded **manually by the user** via Snowsight UI (Data » Databases » Stages » + Files, or Ingestion » Add Data » Load files into a Stage).
- Generated artefacts (`quiz.py`, `environment.yml`) are written as **files in this workspace** by the agent, then moved to a stage by the user (or deployed via workspace "Deploy as Streamlit App" if that path is confirmed working).
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

## Step 1 — Collect inputs

### 1a — Validate AGENTS.md environment config

Read the `snowflake environment` table from `AGENTS.md`. Check the `database`, `warehouse`, and `role` rows.

If any of these values still contains `<...>` placeholder syntax (e.g. `<your_database>`, `<your_warehouse>`, `<your_role>`), **STOP** and ask the user:

> "Before I can set up the exam, please fill in the `snowflake environment` table in `AGENTS.md`:
> - replace `<your_database>` with the database name where the schema will live
> - replace `<your_warehouse>` with the warehouse name to use
> - replace `<your_role>` with the role name that has `CREATE SCHEMA` on that database
>
> Leave `schema` and `exam_code` as is — I will fill those in once we know the exam code.
>
> Let me know when done and I will re-read AGENTS.md."

Wait for user confirmation, then re-read AGENTS.md and re-check. Do not proceed to 1b until `database`, `warehouse`, `role` are concrete values (no angle brackets).

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
- **No, generate via AI** → continue to Step 1d; question_source default will be `'ai'` in quiz.py.

### 1d — Additional requirements

Ask: "Any additional requirements or customizations? (e.g., specific UI features, AI study recommendations, different scoring)"

If the user mentions features not already in AGENTS.md, ask clarifying questions about requirements BEFORE proceeding to Step 2.

Wait for the user's response before proceeding.

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

**Tables** — all 4 with exact DDL below:

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
```

Rationale for 4 tables: QUIZ_REVIEW_LOG stores per-question wrong answers (Review tab + domain error analysis). QUIZ_SESSION_LOG stores per-round summaries (score, round size — needed for progress metrics). Rounds with 0 wrong answers have no QUIZ_REVIEW_LOG rows, so merging would lose session data.

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
- domain_id (integer, starting from 1)
- domain_name (string, exact name from the guide)
- weight_pct (number, percentage weight - must sum to 100 across all domains)
- topics (JSON array of topic strings covered in this domain)

Study guide content:
{doc_content}$$
)::VARCHAR;
```

Parse the response and INSERT each domain into EXAM_DOMAINS.

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

## Step 6 — Load or generate questions

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

### 6b — If no CSV — generate via AI (question bank pre-load)

Generate questions grounded on `key_facts` from EXAM_DOMAINS.

**Batch size**: MAX 10 questions per AI_COMPLETE call. Larger batches cause JSON truncation (model hits output token limits → invalid JSON).

For each domain:
1. Read domain's `key_facts` and `topics` from EXAM_DOMAINS.
2. Generate questions in batches of 10:
   - Each batch: specify domain, difficulty mix (roughly 3 easy + 4 medium + 3 hard), and a subset of topics to cover.
   - Run 3 batches per domain → ~30 questions per domain.
   - If JSON parse fails (truncation), retry the batch with 5 questions.
3. INSERT each question with `source = 'AI_GENERATED'`, `domain_id`, `domain_name`, `difficulty`.

**Target**: ~30 questions per domain (150 total for 5 domains).

For question format, char limits, and JSON schema: see `$quiz/questions` AI Question JSON Format section.

Note: This pre-loaded bank is for the "From question bank" source mode. When users select "AI Generated" at runtime, questions are generated live (not from this bank).

### 6c — Verify

```sql
SELECT COUNT(*) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS;
SELECT COUNT(DISTINCT domain_id) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS;
SELECT COUNT(*) FROM {database}.QUIZ_<CODE>.QUIZ_QUESTIONS WHERE domain_name IS NULL;
```

If the user chose "No CSV" in Step 1c and 6b was skipped (e.g., user wants only runtime AI), QUIZ_QUESTIONS may legitimately be 0 rows. Confirm with the user before proceeding.

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
- Environment: snowsight workspace section (platform constraints)
- Domain model section (4 tables named — the column details live in this skill)
- Cortex LLM section (model constant + cross-region note)
- App overview section (screen flow, function list)
- Available skills index
- Security and governance principles

### What to update

1. **Schema and exam code**: update both rows in the environment table.
2. **Title**: update to new exam name.
3. **Additional requirements**: if user requested features not in AGENTS.md, append new subsections. Do not modify existing subsections.

## Step 8 — Build quiz.py and environment.yml in workspace

1. Read the updated AGENTS.md fully.
2. **MANDATORY: read ALL quiz skills before generating code:**
   - `$quiz/screens` — screen flow, session state, explanation contract, write-back, dashboard chart specs
   - `$quiz/questions` — DIFFICULTY_GUIDE (REQUIRED constant), answer shuffling, validation, retry logic
   - `$quiz/style` — EXAM_NAME constant, badge colors, chart colors (#29b5e8 blue, #F1914C orange), axis formatting, docs link format
   - If user requested optional features: also read `$quiz/features`
3. Generate `quiz.py` per the complete specification.
   - If the user chose "No CSV" in Step 1c → default `question_source = 'ai'` in the home screen.
4. Generate `environment.yml` with exactly this content:

   ```yaml
   name: snowpro_quiz
   channels:
     - snowflake
   dependencies:
     - streamlit=1.52.*
     - pandas
     - altair
   ```

5. **Run the pre-deploy scan from the `$sis/pre-deploy` skill — all 22 items must pass.** Fix any issues and re-scan until clean.

Both files should be written as files in the current workspace (not inside `.snowflake/cortex/skills/`). The user will deploy them from workspace in Step 9.

## Step 9 — Deploy Streamlit app

Two supported deploy paths. The agent cannot execute `PUT`; the user drives file transfer via UI.

### Path A — Deploy via STAGE_SIS_APP (default, verified)

1. Present instructions to the user:

   > "Deploy ready. Please:
   > 1. In Snowsight, go to **Data » Databases » {database} » QUIZ_<CODE> » Stages » STAGE_SIS_APP**.
   > 2. Click **+ Files**, drag-drop both `quiz.py` and `environment.yml` from your workspace.
   > 3. Confirm they appear in the stage.
   > 4. Let me know when upload is complete."

2. Verify:
   ```sql
   LIST @{database}.QUIZ_<CODE>.STAGE_SIS_APP;
   ```
   Must return both files. If anything is missing, ask the user to retry.

3. Execute:
   ```sql
   CREATE OR REPLACE STREAMLIT {database}.QUIZ_<CODE>.SNOWPRO_QUIZ
     FROM '@{database}.QUIZ_<CODE>.STAGE_SIS_APP'
     MAIN_FILE = '/quiz.py'
     QUERY_WAREHOUSE = {warehouse};
   ```

### Path B — Deploy from Workspace ("Deploy as Streamlit App")

⚠️ **UNVERIFIED** — confirm with the user whether this path is acceptable.

If the user prefers: right-click `quiz.py` in the workspace file tree → **Deploy as Streamlit App**. Set warehouse, database, schema (`QUIZ_<CODE>`), and app name (`SNOWPRO_QUIZ`). Snowsight handles the internal stage.

After deploy, verify:
```sql
SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA {database}.QUIZ_<CODE>;
```

### Choosing between paths

**Use `ask_user_question` tool (if available):**

> "Ready to deploy. Which path do you prefer?"

| Option | Description |
|--------|-------------|
| **Path A — via stage (default)** | You upload files to STAGE_SIS_APP, I run `CREATE STREAMLIT`. Fully scripted. |
| **Path B — workspace Deploy** | You right-click `quiz.py` in workspace → Deploy as Streamlit App. Faster if it works as expected. |

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
- ⚠️ After Step 4: Wait for manual upload confirmation; verify via `LIST @stage`.
- ⚠️ After Step 5a.1 (conditional): If conflicting domain structures found, let user choose.
- ⚠️ After Step 5d: Domain verification. Approve/Re-extract/Abort. Do NOT proceed until user responds.
- ⚠️ After Step 8 scan: All 22 items from `$sis/pre-deploy` must PASS. Do NOT deploy on any FAIL. (No `ask_user_question` — pass/fail gate.)
- ⚠️ Step 9: Path A vs Path B deploy choice; wait for upload confirmation if Path A.
- ⚠️ After Step 10b: Derive app URL from `CURRENT_ORGANIZATION_NAME()` + `CURRENT_ACCOUNT_NAME()`, NOT from `CURRENT_ACCOUNT()`.
- ⚠️ After Step 10: Deployment report. Done/Review.

**Resume rule:** Upon user approval, proceed directly to next step without re-asking.

---

# Important Notes

- **Never drop or modify the previous exam's schema.** Both exams coexist in separate schemas.
- **All SQL uses the new schema.** Double-check every query references `{database}.QUIZ_<CODE>`.
- **Dollar-quoting for AI_COMPLETE prompts.** Sanitize any `$$` in interpolated content to `$ $`.
- **If any step fails**, diagnose the issue, fix it, and retry. Do not skip steps.
- **Manual upload is the only way to get files onto stages in Snowsight** — the agent cannot execute `PUT`. Always wait for user confirmation + `LIST @stage` check.
- **Schema-per-exam** is the only isolation mechanism in this variant. No git, no branches.
- **If a question bank needs schema adaptation**, invoke `$adapt-questions` before Step 6 loading.

---

## Output

A fully deployed quiz app in a dedicated schema, with domains extracted from the PDF, questions loaded (from CSV if provided, otherwise AI-generated), and passing pre-deploy scan.
