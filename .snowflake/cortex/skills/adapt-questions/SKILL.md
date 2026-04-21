---
name: adapt-questions
description: "Adapt CSV or JSON question bank files to the QUIZ_QUESTIONS table schema. Inspects source format, maps columns, chooses loading strategy, executes transformation. Triggers: adapt questions, question bank, CSV mapping, schema compatibility, column mapping, question import, transform questions"
---

# Adapt Question Bank

## When to Use

- Invoked from `$setup-exam` Step 1 when user requests a schema compatibility scan
- Invoked directly: "Adapt my questions CSV for the quiz schema"
- Any time a user has a CSV or JSON file with questions that needs loading into QUIZ_QUESTIONS

Example prompts:
- "Scan quiz_questions_core_c02.csv for compatibility"
- "Import my questions from questions.json"
- "Adapt this CSV for the quiz schema"

## When NOT to Use

- AI question generation at runtime -> use `$quiz/questions`
- Full exam setup pipeline -> use `$setup-exam`
- Quiz.py code changes -> use `$quiz/screens`

---

# Prerequisites

- Read `{database}`, `{schema}` from the environment table in `AGENTS.md`
- Target schema and QUIZ_QUESTIONS table must already exist
- **Source file must already be uploaded by the user to `@{database}.{schema}.STAGE_QUIZ_DATA`** via Snowsight UI (Data » Stages » + Files). This skill does not upload files — the agent cannot `PUT` from Snowsight.

---

# Tools

### ask_user_question

**Description:** Present the user with a fixed list of options and wait for their selection.

**When to use:** At mandatory stopping points where the user must choose between defined options (approvals, strategy selections). Do NOT use for free-text input.

**Fallback:** If `ask_user_question` is not available, present the same options as a numbered list and wait for the user's text response.

---

# Target Schema Reference

```
question_id    NUMBER AUTOINCREMENT  -- skip, auto-generated
domain_id      VARCHAR NOT NULL      -- REQUIRED
domain_name    VARCHAR               -- backfilled from EXAM_DOMAINS
difficulty     VARCHAR DEFAULT 'medium'  -- required, map or default
question_text  VARCHAR(2000) NOT NULL   -- REQUIRED
is_multi       BOOLEAN DEFAULT FALSE    -- required, map or default
option_a       VARCHAR(500) NOT NULL    -- REQUIRED
option_b       VARCHAR(500) NOT NULL    -- REQUIRED
option_c       VARCHAR(500)             -- optional
option_d       VARCHAR(500)             -- optional
option_e       VARCHAR(500)             -- optional
correct_answer VARCHAR NOT NULL         -- REQUIRED
source         VARCHAR DEFAULT 'MANUAL' -- optional
created_at     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP() -- skip, auto-generated
```

**Minimum required in source**: `domain_id`, `question_text`, `option_a`, `option_b`, `correct_answer`

---

# Instructions

## Step 1 - Inspect source file (via SQL against stage)

The file is already on `STAGE_QUIZ_DATA`. Use SQL to peek at contents — no bash.

**For CSV:**

Peek at the first 10 rows including header (read as raw columns, no file format):
```sql
SELECT $1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12
FROM @{database}.{schema}.STAGE_QUIZ_DATA/<filename>
(FILE_FORMAT => (TYPE = CSV, SKIP_HEADER = 0, FIELD_OPTIONALLY_ENCLOSED_BY = '"'))
LIMIT 10;
```

- Row 1 = header → identify column names.
- Rows 2–10 = sample data.
- If columns > 12, increase `$1..$N`. If delimiter is not comma, add `FIELD_DELIMITER => ';'` (or tab, etc.) to the inline `FILE_FORMAT`.
- For schema inference, optionally run `INFER_SCHEMA(LOCATION => '@...{filename}', FILE_FORMAT => '{database}.{schema}.FF_CSV')`.

**For JSON:**

```sql
SELECT $1
FROM @{database}.{schema}.STAGE_QUIZ_DATA/<filename>
(FILE_FORMAT => (TYPE = JSON))
LIMIT 5;
```
- If the file is a single JSON array (not line-delimited), load it as VARIANT and `FLATTEN` to inspect:
  ```sql
  SELECT value FROM TABLE(FLATTEN(INPUT => $1))
  FROM @{database}.{schema}.STAGE_QUIZ_DATA/<filename>
  (FILE_FORMAT => (TYPE = JSON))
  LIMIT 5;
  ```
- Identify structure: flat array of objects vs nested. List all keys found.

**Report**: columns/keys found, sample values per column, data types, NULL/empty patterns.

## Step 2 - Map to QUIZ_QUESTIONS schema

Match source columns to target columns using this priority:

1. **Exact name match** (case-insensitive): `question_text` -> `question_text`
2. **Common synonyms**: `question` -> `question_text`, `answer` -> `correct_answer`, `domain` -> `domain_id`, `level` -> `difficulty`, `type` -> `is_multi`
3. **Content-based inference**: column with TRUE/FALSE values and no other match -> likely `is_multi`

Present mapping as a table:

| Source Column | Target Column | Status | Notes |
|---|---|---|---|
| question_id | (skip) | AUTO | auto-generated in target |
| domain_id | domain_id | DIRECT | |
| question_text | question_text | DIRECT | check max 2000 chars |
| is_multi | is_multi | TRANSFORM | "FALSE"/"TRUE" -> BOOLEAN |
| (missing) | domain_name | BACKFILL | from EXAM_DOMAINS post-load |
| (missing) | created_at | DEFAULT | CURRENT_TIMESTAMP() |

**Status values**: DIRECT (1:1 map), TRANSFORM (needs conversion), BACKFILL (filled post-load from another table), DEFAULT (uses column default), AUTO (auto-generated, skip), MISSING (required but absent — problem), EXTRA (in source, not needed — ignore)

**Domain ID compatibility check**: Compare source domain_ids against target EXAM_DOMAINS:
```sql
SELECT DISTINCT domain_id FROM {database}.{schema}.EXAM_DOMAINS ORDER BY domain_id;
```
- If source domain_ids don't match target -> flag: "Source domain_ids do not match target EXAM_DOMAINS."
- If file is from a different exam version (detected from filename) -> flag explicitly: "These questions were created for {old_exam}. Domain IDs may not align with {target_exam} domains."

**⚠️ MANDATORY STOPPING POINT — Use `ask_user_question` tool (if available):**

Present the mapping table above, then ask:

> "Column mapping complete. {N} columns mapped, {M} transforms needed, {K} issues flagged."

| Option | Description |
|--------|-------------|
| **Approve** | Mapping looks correct — proceed to strategy selection |
| **Edit mapping** | I want to change a column mapping before proceeding |
| **Abort** | Cancel — the source file needs manual fixes first |

**Routing logic:**
- If user chooses **Approve** → proceed to Step 3
- If user chooses **Edit mapping** → ask which mapping to change, update, and re-present
- If user chooses **Abort** → **STOP** with guidance on what to fix

⚠️ **STOP**: Do NOT proceed until user responds.

## Step 3 - Choose loading strategy

| Strategy | When to use | Complexity |
|---|---|---|
| A. Direct COPY INTO | Columns align perfectly, correct order and types | Lowest |
| B. COPY INTO with SELECT | Column order differs or extra columns present, but all data correct | Low |
| C. Staging table + INSERT...SELECT | Type conversions needed (booleans, truncation, format changes) | Medium |
| D. Staging table + AI_COMPLETE | Missing columns inferable from content (e.g., domain_id from question text + EXAM_DOMAINS) | High |
| E. Manual intervention | Too many required columns missing or unrecognizable format | N/A |

**Decision rules:**
- Source matches existing 12-column CSV format (same order as target) -> **Strategy A**
- Columns present but different order or with extras -> **Strategy B**
- Boolean format differs ("Yes"/"No", "1"/"0"), text needs truncation -> **Strategy C**
- `domain_id` missing but `question_text` present -> **Strategy D** (AI classifies into domains)
- `question_text` or both `option_a`/`option_b` missing -> **Strategy E** (ask user to fix source)

**Use `ask_user_question` tool (if available):**

Present the recommended strategy with rationale, then ask:

> "Recommended loading strategy: **Strategy {X}** — {rationale}."

| Option | Description |
|--------|-------------|
| **Accept** | Use the recommended strategy |
| **Strategy A** | Direct COPY INTO |
| **Strategy B** | COPY INTO with SELECT reordering |
| **Strategy C** | Staging table + INSERT...SELECT transforms |
| **Strategy D** | Staging table + AI_COMPLETE enrichment |
| **Strategy E** | Manual intervention — show expected format |

Only show strategies that are technically feasible given the mapping results. Mark the recommended one.

**Routing logic:**
- If user chooses **Accept** → execute the recommended strategy in Step 4
- If user chooses a specific strategy → execute that strategy in Step 4
- If user chooses **Strategy E** → provide the expected CSV template and **STOP**

⚠️ **STOP**: Wait for user confirmation before executing.

## Step 4 - Execute chosen strategy

**Strategy A: Direct COPY INTO**

(File must already be on `STAGE_QUIZ_DATA` — user uploaded via Snowsight UI in the setup-exam flow.)

```sql
CREATE FILE FORMAT IF NOT EXISTS {database}.{schema}.FF_IMPORT
  TYPE = 'CSV' SKIP_HEADER = 1 FIELD_OPTIONALLY_ENCLOSED_BY = '"';

COPY INTO {database}.{schema}.QUIZ_QUESTIONS (
    domain_id, difficulty, question_text, is_multi,
    option_a, option_b, option_c, option_d, option_e,
    correct_answer, source
)
FROM (
    SELECT $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12
    FROM @{database}.{schema}.STAGE_QUIZ_DATA/<filename>
)
FILE_FORMAT = {database}.{schema}.FF_IMPORT;
```

**Strategy B: COPY INTO with SELECT** (reorder columns as needed)
```sql
COPY INTO {database}.{schema}.QUIZ_QUESTIONS (domain_id, difficulty, question_text, ...)
FROM (
    SELECT $N, $M, $K, ...  -- reordered column positions
    FROM @{database}.{schema}.STAGE_QUIZ_DATA/<filename>
)
FILE_FORMAT = {database}.{schema}.FF_IMPORT;
```

**Strategy C: Staging table + transform**
```sql
CREATE TEMPORARY TABLE {database}.{schema}.STG_QUESTIONS (
    -- mirror source columns as VARCHAR
);
COPY INTO {database}.{schema}.STG_QUESTIONS ...;
INSERT INTO {database}.{schema}.QUIZ_QUESTIONS (domain_id, difficulty, question_text, is_multi, option_a, option_b, option_c, option_d, option_e, correct_answer, source)
SELECT
    domain_id,
    CASE WHEN difficulty IN ('easy','medium','hard') THEN difficulty ELSE 'medium' END,
    LEFT(question_text, 2000),
    IFF(UPPER(is_multi) IN ('TRUE','YES','1'), TRUE, FALSE),
    LEFT(option_a, 500), LEFT(option_b, 500), LEFT(option_c, 500),
    LEFT(option_d, 500), LEFT(option_e, 500),
    correct_answer,
    COALESCE(source, 'MANUAL')
FROM {database}.{schema}.STG_QUESTIONS;
DROP TABLE {database}.{schema}.STG_QUESTIONS;
```

**Strategy D: Staging + AI_COMPLETE enrichment**

After staging load, classify domain_id using AI_COMPLETE in batches:
- Group questions by original domain_id (or lack thereof)
- For each group, sample 5 questions and classify against current EXAM_DOMAINS
- If consistent mapping found, apply to entire group
- For unclassifiable questions, set domain_id to the closest match with a warning

**Strategy E**: Report what's missing. Provide expected CSV format as a template for the user to fix.

**File format notes**: Adjust `FIELD_DELIMITER`, `RECORD_DELIMITER`, `ENCODING` in FILE FORMAT if source uses non-standard delimiters or encoding (UTF-8 BOM, Latin-1, semicolons, tabs).

## Step 5 - Post-load operations

Always run after any successful load:

1. **Backfill domain_name:**
```sql
UPDATE {database}.{schema}.QUIZ_QUESTIONS q
SET q.domain_name = d.domain_name
FROM {database}.{schema}.EXAM_DOMAINS d
WHERE q.domain_id = d.domain_id AND q.domain_name IS NULL;
```

2. **Verify required columns:**
```sql
SELECT COUNT(*) AS total_loaded FROM {database}.{schema}.QUIZ_QUESTIONS;
SELECT COUNT(*) AS null_domain_name FROM {database}.{schema}.QUIZ_QUESTIONS WHERE domain_name IS NULL;
SELECT COUNT(*) AS null_question_text FROM {database}.{schema}.QUIZ_QUESTIONS WHERE question_text IS NULL;
SELECT COUNT(*) AS null_correct_answer FROM {database}.{schema}.QUIZ_QUESTIONS WHERE correct_answer IS NULL;
```

3. **Domain coverage:**
```sql
SELECT d.domain_id, d.domain_name,
       COUNT(q.question_id) AS question_count
FROM {database}.{schema}.EXAM_DOMAINS d
LEFT JOIN {database}.{schema}.QUIZ_QUESTIONS q ON d.domain_id = q.domain_id
GROUP BY d.domain_id, d.domain_name
ORDER BY d.domain_id;
```

## Step 6 - Report

Present a summary:
- Total rows loaded
- Rows skipped or failed (if any)
- Domain coverage table (which domains have how many questions)
- Warnings: NULL domain_names (domain_id mismatch), truncated text, unmapped domain_ids
- If domain_id mismatches are significant, suggest: "Consider re-running `$adapt-questions` with Strategy D to re-map domain_ids using AI."

---

# Stopping Points

All stopping points below use `ask_user_question` (if available) to present structured options. If the tool is not available, present the same options as a numbered list and wait for the user's text response.

- ⚠️ After Step 2: Mapping report — MANDATORY. Uses `ask_user_question` with Approve/Edit/Abort options. Do NOT proceed until user responds.
- ⚠️ After Step 3: Strategy selection. Uses `ask_user_question` to let user accept or override recommended strategy. Do NOT proceed until user responds.
- After Step 6: Inform user of results. (No `ask_user_question` — informational only.)

**Resume rule:** Upon user approval, proceed directly to next step without re-asking.

---

## Output

Question bank loaded and validated against QUIZ_QUESTIONS schema, with domain coverage report.
