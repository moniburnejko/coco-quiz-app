---
name: cortex-patterns
description: "Cortex AI function patterns — AI_COMPLETE dollar-quoting, response parsing, AI_PARSE_DOCUMENT, stage requirements, 5-step diagnostics. Use when calling or debugging Cortex AI functions. Triggers: AI_COMPLETE, AI_PARSE_DOCUMENT, cortex error, model not found, file not accessible, dollar-quoting, parse_cortex_json"
parent_skill: cortex
---

# When to Load

Parent skill `$cortex` routes here for PATTERNS intent.

- Calling `AI_COMPLETE` (question generation, explanation generation, key_facts extraction)
- Calling `AI_PARSE_DOCUMENT` (PDF parsing in Phase 1)
- Debugging Cortex errors: "file not accessible", "model not found", NULL responses
- Setting up stages for Cortex AI functions
- Before deploying quiz.py - verify Cortex connectivity

# When NOT to Use

- Prompt quality audit -> use `$cortex/prompt-audit`
- SiS rendering patterns -> use `$sis/patterns`
- Pre-deploy scan -> use `$sis/pre-deploy`

---

# AI_COMPLETE

## Dollar-quoting

All prompts passed to AI_COMPLETE must use `$$...$$` quoting, not single quotes. Before interpolation, sanitize:

```python
safe_prompt = prompt.replace("$$", "$ $")
sql = f"SELECT AI_COMPLETE('{CORTEX_MODEL}', $${safe_prompt}$$)"
```

`CORTEX_MODEL` is a hardcoded constant - safe to interpolate. Never interpolate user-derived values.

## Error handling

```python
def call_cortex(prompt):
    try:
        safe_prompt = prompt.replace("$$", "$ $")
        rows = session.sql(f"SELECT AI_COMPLETE('{CORTEX_MODEL}', $${safe_prompt}$$)").collect()
        if not rows or rows[0][0] is None:
            return None
        return str(rows[0][0])
    except Exception as e:
        st.session_state["last_cortex_error"] = str(e)
        return None
```

## Response parsing

AI_COMPLETE responses may have these encoding issues:
1. **Plain JSON** - works directly with `json.loads()`
2. **Markdown fences** - response wrapped in ` ```json ... ``` ` - strip fences before parsing
3. **Double-encoded** - first `json.loads()` returns a string, needs second parse
4. **VARIANT string wrapping** - Snowflake returns AI_COMPLETE results as VARIANT. When cast to `str()`, the value may arrive with surrounding JSON quotes: `'"```json\n{...}"'`. The `call_cortex()` function must strip this outer encoding before returning:

```python
raw = str(rows[0][0])
# Strip VARIANT string encoding
if raw.startswith('"'):
    try:
        decoded = json.loads(raw)
        if isinstance(decoded, str):
            raw = decoded
    except Exception:
        pass
return raw
```

Always use `parse_cortex_json()` - never `json.loads()` directly. The `parse_cortex_json()` function must handle all 4 cases with separate try blocks (so a failure in one path does not skip the others).

---

# AI_PARSE_DOCUMENT

## Correct pattern

`AI_PARSE_DOCUMENT` takes a FILE object (via `TO_FILE()`) and an options OBJECT. Returns VARIANT.

Common mistakes:

| Mistake | Fix |
|---------|-----|
| Using `BUILD_SCOPED_FILE_URL()` | Use `TO_FILE('@stage', 'file.pdf')` — returns FILE type, not VARCHAR |
| Passing mode as string `'LAYOUT'` | Pass as OBJECT: `{'mode': 'LAYOUT'}` |
| Wrapping in `PARSE_JSON()` | Do NOT — result is already VARIANT |
| Paginating manually (page by page) | Do NOT — one call returns the full document |

```sql
-- get full document content in one call
SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@{database}.{schema}.STAGE_QUIZ_DATA', '<pdf_filename>'),
    {'mode': 'LAYOUT'}
):content::VARCHAR AS doc_content;

-- with subquery for further processing
SELECT result:content::VARCHAR AS doc_content
FROM (
    SELECT AI_PARSE_DOCUMENT(
        TO_FILE('@{database}.{schema}.STAGE_QUIZ_DATA', '<pdf_filename>'),
        {'mode': 'LAYOUT'}
    ) AS result
);
```

**Options OBJECT keys:**

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `mode` | STRING | `'OCR'` | `'OCR'` or `'LAYOUT'` |
| `page_split` | BOOLEAN | `false` | Return pages as separate array elements |
| `page_filter` | ARRAY | — | Process specific page ranges, e.g. `[{'start': 0, 'end': 10}]` |

## Stage requirements

`AI_PARSE_DOCUMENT` requires server-side encryption and directory enabled on the stage:

```sql
CREATE STAGE IF NOT EXISTS {database}.{schema}.STAGE_QUIZ_DATA
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  DIRECTORY = (ENABLE = TRUE);
```

- `SNOWFLAKE_SSE` - required for Cortex to read files. #1 cause of "file not accessible" errors after PDF upload.
- `DIRECTORY = TRUE` - required for file path enumeration.

If stage exists without these settings, recreate it:
```sql
DROP STAGE IF EXISTS {database}.{schema}.STAGE_QUIZ_DATA;
CREATE STAGE {database}.{schema}.STAGE_QUIZ_DATA
  ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')
  DIRECTORY = (ENABLE = TRUE);
```
Then re-upload files via Snowsight UI.

---

# Diagnostics

Run these tests when AI functions fail. Report pass/fail for each.

## Step 1 - Basic connectivity

```sql
SELECT AI_COMPLETE('claude-sonnet-4-6', $Tell me the current Snowflake region in one word.$);
```

Expected: non-empty string. Fail: `not allowed to access this endpoint` or NULL.

## Step 2 - Model access

```sql
SELECT AI_COMPLETE('claude-sonnet-4-6', $Say "ok" in JSON exactly: {"status":"ok"}$);
```

Expected: string containing `{"status":"ok"}`. Fail: `Model not found` or NULL.

## Step 3 - Cross-region parameter

```sql
SHOW PARAMETERS LIKE 'CORTEX_ENABLED_CROSS_REGION' IN ACCOUNT;
```

Expected: value = `AWS_US`. If `DISABLED`: AI_COMPLETE will fail for EU accounts.

Fix (requires ACCOUNTADMIN):
```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'AWS_US';
```

## Step 4 - JSON parsing

```sql
SELECT AI_COMPLETE('claude-sonnet-4-6', $Return only valid JSON with this exact structure, no markdown fences:
{"question_text": "What is a virtual warehouse?", "option_a": "A compute cluster", "option_b": "A storage unit", "option_c": "A database schema", "option_d": "A role", "correct_answer": "A"}$);
```

Expected: JSON object (possibly with markdown fences - `parse_cortex_json` handles that).

## Step 5 - Available models

```sql
SELECT * FROM SNOWFLAKE.ML_FUNCTIONS.MODELS WHERE MODEL_NAME LIKE 'claude%' ORDER BY MODEL_NAME;
```

Expected: list of available claude models in this region.

## Reporting

| Step | Status | Notes |
|------|--------|-------|
| 1 - Basic connectivity | PASS / FAIL | |
| 2 - Model access | PASS / FAIL | |
| 3 - Cross-region | PASS / FAIL | current value |
| 4 - JSON output | PASS / FAIL | |
| 5 - Models list | INFO | |

If Step 3 fails: provide the ALTER ACCOUNT fix and ask user to confirm ACCOUNTADMIN role before running.

---

# AI_EXTRACT (Alternative for Structured Extraction)

AI_EXTRACT is an optional alternative for extracting structured fields (domain names, topics, weights) from documents. It returns JSON directly without a two-step parse-then-analyze pipeline.

| Approach | Best For | Cost |
|----------|----------|------|
| AI_PARSE_DOCUMENT + AI_COMPLETE | Full text extraction + free-form analysis (key_facts) | 0.5-3.33 credits/1000 pages + AI_COMPLETE cost |
| AI_EXTRACT | Targeted structured fields (domain names, weights, topics) | 5 credits/million tokens |

The current project uses AI_PARSE_DOCUMENT + AI_COMPLETE for all PDF processing. AI_EXTRACT is documented here as a reference for future optimization.

See the built-in `cortex-ai-functions` skill in Cortex Code Snowsight for full AI_EXTRACT documentation including TO_FILE path handling and response format options.

---

# Global Skills Reference

Cortex Code in Snowsight ships with a built-in `cortex-ai-functions` skill that provides comprehensive reference documentation for all Cortex AI functions including AI_EXTRACT, AI_CLASSIFY, AI_FILTER, and the Document Intelligence workflow. It is available natively — no upload needed. Consult it when using a Cortex AI function not covered in this project skill.

---

## Output

Correct AI_COMPLETE/AI_PARSE_DOCUMENT calling patterns applied. Diagnostics report if troubleshooting.
