---
name: sis
description: "Streamlit in Snowflake (SiS v1.52.*) coding patterns and mandatory pre-deploy scan. Use when writing SiS app code or before deploying. Triggers: SiS, streamlit in snowflake, deploy, pre-deploy, rerun, widget state, scan, get_active_session, cache_data"
---

# Streamlit in Snowflake

## When to Use

When writing SiS app code or preparing to deploy quiz.py to Snowflake.

## Intent Detection

| Intent | Triggers | Load |
|--------|----------|------|
| PATTERNS | writing code, session, rerun, widget state, dates, cache_data, unsupported API | `patterns/SKILL.md` |
| PRE-DEPLOY | deploy, scan, upload, before deploying, stage copy, push to snowflake | `pre-deploy/SKILL.md` |

## Workflow

```
User request
  ↓
Intent Detection
  ├─→ PATTERNS   → Load patterns/SKILL.md
  └─→ PRE-DEPLOY → Load pre-deploy/SKILL.md
```

**Note:** Always consult `patterns/SKILL.md` alongside `pre-deploy/SKILL.md` when preparing to deploy.

## Capabilities

- **Patterns**: Session management, rerun budget (10 locations), unsupported APIs, multi-answer checkboxes, date handling, column normalization, widget state safety, button click safety
- **Pre-Deploy**: MANDATORY 22-item scan catching SQL injection, SiS incompatibilities, runtime errors

## Output

SiS-compatible code or a 22-item pre-deploy scan report with PASS/FAIL per item.
