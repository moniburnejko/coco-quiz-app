---
name: sis
description: "Streamlit-in-Snowflake (container runtime) coding patterns and mandatory pre-deploy scan. Use when writing SiS app code or before deploying. Triggers: SiS, streamlit in snowflake, deploy, pre-deploy, widget state, scan, get_active_session, cache_data, clear_caches, container runtime"
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

## Routing

This parent does no work itself. Match the request against the intent table, then **open the sub-skill file and follow it**:

- PATTERNS intent → load `.snowflake/cortex/skills/sis/patterns/SKILL.md` and follow it.
- PRE-DEPLOY intent → load `.snowflake/cortex/skills/sis/pre-deploy/SKILL.md` and follow it.

When preparing to deploy, always consult `patterns` alongside `pre-deploy`.

## Bundled skills this defers to

For general Streamlit development use the bundled **`developing-with-streamlit`** skill; for SPCS/app deploy mechanics, **`deploy-to-spcs`** / **`snowflake-apps`** — all available natively in CoCo, no upload. The sub-skills here carry only project deltas for the container runtime.

## Capabilities

- **Patterns**: Session management, caching without ttl + explicit invalidation, widget lifecycle (flag-at-top reset), multipage state, scoped reruns (`@st.fragment`), date handling, column normalization, button click safety
- **Pre-Deploy**: MANDATORY 22-item scan across all app files catching SQL injection, cache/config pitfalls, untrusted-input handling, doc-grounding isolation, runtime errors

## Output

SiS-compatible code or a 22-item pre-deploy scan report with PASS/FAIL per item.
