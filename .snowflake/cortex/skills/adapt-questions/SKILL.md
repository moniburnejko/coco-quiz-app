---
name: adapt-questions
description: "Adapt a CSV or JSON question-bank file to the QUIZ_QUESTIONS schema — inspect the source, map columns, pick a loading strategy, transform. Triggers: adapt questions, question bank, CSV mapping, schema compatibility, column mapping, question import, transform questions. Do NOT use for runtime AI generation (quiz-questions) or the full setup pipeline (setup-exam)."
---

# Adapt Question Bank

## When to Use
- From `$setup-exam` Step 6 when an uploaded CSV/JSON needs remapping, or directly: *"adapt my questions.csv for the quiz schema"*, *"import questions.json"*.

## When NOT to Use
- Runtime AI generation → `$quiz/questions`. Full exam setup → `$setup-exam`. App code → `$quiz/screens`.

## Prerequisites
- `{database}`, `{schema}` from `AGENTS.md`; `QUIZ_QUESTIONS` already exists.
- The source file is ALREADY on `@{database}.{schema}.STAGE_QUIZ_DATA` (the agent can't `PUT` from Snowsight — the user uploads via the UI).
- **Tool `ask_user_question`** at STOP points; if unavailable, present the same options as a numbered list and wait.

## Target

`QUIZ_QUESTIONS` — full column DDL in `$setup-exam` Step 3. **Minimum required in the source:** `domain_id`, `question_text`, `option_a`, `option_b`, `correct_answer`. Auto/skip: `question_id`, `created_at`. Backfilled post-load: `domain_name` (from `EXAM_DOMAINS`). Defaulted if absent: `difficulty='medium'`, `is_multi=FALSE`, `source='MANUAL'`.

## Step 1 — Inspect the source (SQL against the stage; no bash)

Peek at the file to list columns/keys, sample values, types, and NULL patterns.
- **CSV:** `SELECT $1, $2, … FROM @{database}.{schema}.STAGE_QUIZ_DATA/<file> (FILE_FORMAT => (TYPE=CSV, SKIP_HEADER=0, FIELD_OPTIONALLY_ENCLOSED_BY='"')) LIMIT 10;` — row 1 is the header; widen `$N` if >12 columns; add `FIELD_DELIMITER` for non-comma; `INFER_SCHEMA` optionally.
- **JSON:** read `$1` with `TYPE=JSON`; if it's a single array (not line-delimited), `TABLE(FLATTEN(INPUT => $1))` to inspect; list all keys.

Report columns/keys, sample values, types, NULL/empty patterns.

## Step 2 — Map to the schema  ⚠️ STOP

Match each source column by: exact name (case-insensitive) → common synonyms (`question`→`question_text`, `answer`→`correct_answer`, `domain`→`domain_id`, `level`→`difficulty`, `type`→`is_multi`) → content inference (a TRUE/FALSE column with no other match → `is_multi`).

Present a mapping table with a **Status** per row — DIRECT (1:1), TRANSFORM (needs conversion), BACKFILL (filled post-load), DEFAULT, AUTO (skip), MISSING (required but absent → problem), EXTRA (ignore).

**Domain-id check:** `SELECT DISTINCT domain_id FROM {database}.{schema}.EXAM_DOMAINS ORDER BY domain_id;` and compare. Flag mismatches ("source domain_ids don't match the target"); if the file is from a different exam version, say so explicitly — the ids may not align.

`ask_user_question`: **Approve** (→ Step 3) / **Edit mapping** (change one, re-present) / **Abort** (STOP with what to fix). Do not proceed until answered.

## Step 3 — Choose a loading strategy  ⚠️ STOP

| Strategy | When |
|---|---|
| **A. Direct `COPY INTO`** | columns align — right order and types |
| **B. `COPY INTO (SELECT …)`** | order differs / extra columns, data correct |
| **C. Staging table + `INSERT…SELECT`** | type conversions (booleans, truncation, format) |
| **D. Staging + `AI_COMPLETE`** | a required column is inferable from content (e.g. `domain_id` from question text + `EXAM_DOMAINS`) |
| **E. Manual fix** | required columns missing / unrecognizable → give the user the expected CSV template and STOP |

Recommend one with a rationale, then `ask_user_question` to accept or override (show only feasible strategies). CoCo writes the actual `COPY INTO`/transform — here is the one instructive example (Strategy C); the others follow the same shape:

```sql
CREATE TEMPORARY TABLE {database}.{schema}.STG_QUESTIONS ( /* mirror source columns as VARCHAR */ );
COPY INTO {database}.{schema}.STG_QUESTIONS FROM @{database}.{schema}.STAGE_QUIZ_DATA/<file>
  FILE_FORMAT = (TYPE=CSV SKIP_HEADER=1 FIELD_OPTIONALLY_ENCLOSED_BY='"');
INSERT INTO {database}.{schema}.QUIZ_QUESTIONS
  (domain_id, difficulty, question_text, is_multi, option_a, option_b, option_c, option_d, option_e, correct_answer, source)
SELECT domain_id,
       CASE WHEN difficulty IN ('easy','medium','hard') THEN difficulty ELSE 'medium' END,
       LEFT(question_text, 2000), IFF(UPPER(is_multi) IN ('TRUE','YES','1'), TRUE, FALSE),
       LEFT(option_a,500), LEFT(option_b,500), LEFT(option_c,500), LEFT(option_d,500), LEFT(option_e,500),
       correct_answer, COALESCE(source, 'MANUAL')
FROM {database}.{schema}.STG_QUESTIONS;
DROP TABLE {database}.{schema}.STG_QUESTIONS;
```

Strategy D classifies `domain_id` in batches via `AI_COMPLETE` against `EXAM_DOMAINS` (calling pattern → `$cortex`). Adjust `FIELD_DELIMITER` / `ENCODING` for non-standard delimiters or encodings (UTF-8 BOM, Latin-1, semicolons, tabs).

## Step 4 — Post-load

- Backfill `domain_name` from `EXAM_DOMAINS` where NULL.
- Verify: total row count; NULLs in required columns; domain coverage (`LEFT JOIN EXAM_DOMAINS … GROUP BY domain_id`).

## Step 5 — Report

Rows loaded, rows skipped/failed, the domain-coverage table, and warnings (NULL `domain_name` = id mismatch, truncated text). If id mismatches are significant, suggest re-running with **Strategy D** to re-map via AI.

## Output

Question bank loaded and validated against `QUIZ_QUESTIONS`, with a domain-coverage report.
