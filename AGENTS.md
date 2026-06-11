# SnowPro Core certification quiz app - Streamlit in Snowflake

## What this project is

A context package for Snowflake CoCo that, given a study guide PDF, autonomously creates the Snowflake schema, extracts exam domains, generates or loads questions, builds and deploys a 4-screen Streamlit-in-Snowflake quiz app, and tracks learning progress across sessions. Invoked via `$setup-exam`.

---

## Snowflake environment

Do not touch any database or schema other than the one configured below:

| setting   | value                                |
|-----------|--------------------------------------|
| database  | `<your_database>`                    |
| schema    | `<your_database>.QUIZ_<EXAM_CODE>`   |
| warehouse | `<your_warehouse>`                   |
| role      | `<your_role>`                        |
| exam_code | `<EXAM_CODE>` (e.g. `COF-C03`)       |
| stage     | `STAGE_QUIZ_DATA`                    |
| app stage | `STAGE_SIS_APP`                      |
| app_name  | `SNOWPRO_QUIZ`                       |
| main_file | `main.py`                            |
| compute_pool | `<your_compute_pool>`             |
| runtime   | `container` (default) / `warehouse` (fallback) |
| deps_file | `pyproject.toml` (container) / `environment.yml` (warehouse) |

**Preconditions** (user-set before running `$setup-exam`): replace `<your_database>`, `<your_warehouse>`, `<your_role>` with actual object names; replace `<your_compute_pool>` too unless you plan the warehouse fallback (`$setup-exam` Step 9 Path C). The role must have `CREATE SCHEMA` on `database` (and `USAGE` on the compute pool for the container runtime). `$setup-exam` stops if any required `<...>` placeholder remains unfilled.

**Outputs** (populated in the table above by `$setup-exam`): `schema` (= `<database>.QUIZ_<EXAM_CODE>`) and `exam_code` (from the study guide PDF). `$setup-exam` is idempotent (`IF NOT EXISTS`) and never drops. Each exam gets its own schema - never share a schema between exams.

**Project defaults** (customizable but have working values): stages, `app_name`, `main_file`, `runtime`, `deps_file`. Change only if you need to.

All sections reference these values. Never hardcode environment names elsewhere in this file.

---

## Domain model

Four tables, all in `{database}.{schema}`. Full DDL lives in `$setup-exam` Step 3.

### EXAM_DOMAINS
Populated once per exam from the study guide PDF via `AI_PARSE_DOCUMENT` + `AI_COMPLETE`.
columns: `domain_id`, `domain_name`, `weight_pct`, `topics`, `key_facts`.

### QUIZ_QUESTIONS
Pre-loaded from CSV (if user has one) or AI-generated in `$setup-exam` Step 6 grounded on `key_facts`. Not to be confused with runtime AI generation, which bypasses this table.
columns: `question_id`, `domain_id`, `domain_name`, `difficulty`, `question_text`, `is_multi`, `option_a..e`, `correct_answer`, `source`, `created_at`.

### QUIZ_REVIEW_LOG
Per-question wrong-answer history, written by the app at round end. Drives the Review tab and domain error analysis.
columns: `log_id`, `logged_at`, `domain_id`, `domain_name`, `difficulty`, `question_text`, `correct_answer`, `mnemonic`, `doc_url`.

### QUIZ_SESSION_LOG
Per-round summary, written by the app at round end. Drives the Learning Dashboard progress metrics.
columns: `session_id`, `session_ts`, `exam_code`, `round_size`, `correct_count`, `score_pct`, `domain_filter`, `difficulty`.

---

## Cortex LLM

Preferred model: `claude-sonnet-4-6`. Store as constant `CORTEX_MODEL` in `_config.py`.

Accounts that cannot reach the chosen model in-region must enable cross-region inference (once per account, as ACCOUNTADMIN): `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';` (`'AWS_GLOBAL'` is a narrower alternative; the legacy `'AWS_US'` still works but is narrowest). Accounts created after 2026-03-09 default to `ANY_REGION` and may need no change.

For calling patterns, dollar-quoting, JSON parsing, and diagnostics: see `$cortex/patterns`.
For prompt quality audit: see `$cortex/prompt-audit`.

---

## App overview

The generated app is a **decomposed multipage Streamlit project** under `app/`. Read the modules directly for HOW things work. This section describes WHAT the app does and where each responsibility lives.

### Module map

| File | Responsibility |
|------|----------------|
| `main.py` | Entry point: `st.set_page_config` (first `st.` call), `init_session_state()`, shared sidebar title, builds `st.navigation([...])` from enabled pages and `.run()`s it |
| `_config.py` | Constants: `EXAM_NAME`, `EXAM_CODE`, `CORTEX_MODEL`, `PASS_THRESHOLD`, `DIFFICULTY_GUIDE`, color constants |
| `_cortex.py` | `call_cortex(prompt)` + JSON-returning Cortex helpers (dollar-quoting, error handling) |
| `_data.py` | Cached loaders - `load_domains()`, `load_session_stats()`, `load_recent_sessions()`, `load_domain_errors()` (each calls `get_active_session()` inside) - plus `clear_caches()` invalidation |
| `_questions.py` | `parse_topics`, `_build_topic_schedule`, `generate_ai_question`, `get_question`, answer shuffling, dedup via `_get_shown_texts` |
| `_ui.py` | Shared render helpers: badges, cards, explanation expander, docs link |
| `pages/quiz.py` | QUIZ page: home → quiz → summary state machine |
| `pages/review.py` | REVIEW page: wrong-answer history + learning dashboard |
| `pages/<feature>.py` | Generated ONLY when the user requests an optional feature (`$quiz/features`) - e.g. `exam_simulation.py`, `flashcards.py`, `recommendations.py` |

### Pages and navigation

Navigation is native multipage via `st.Page` + `st.navigation`, built in `main.py`. `st.session_state` is shared across pages.

**QUIZ page**: home → quiz → summary. Home configures round settings (questions, domains, difficulty, source, explanations). Quiz presents questions with lazy loading and AI explanations. Summary shows score, pass/fail, and wrong answer cards.

**REVIEW page** (sub-tabs): Wrong Answers shows filtered history with domain/date filters. Learning Dashboard shows session metrics and charts (score trend, error distribution). Optional features from `$quiz/features` add their own pages, not tabs.

For screen contracts, session state, history schema, and write-back: see `$quiz/screens`.
For UI styling, badges, section labels, and chart colors: see `$quiz/style`.

---

## Available skills

| Skill | Invoke | Purpose | Sub-skills |
|-------|--------|---------|------------|
| `$cortex` | Cortex AI work | Calling patterns, diagnostics, prompt auditing | `$cortex/patterns`, `$cortex/prompt-audit` |
| `$sis` | SiS code or deploy | Coding patterns, mandatory pre-deploy scan | `$sis/patterns`, `$sis/pre-deploy` |
| `$quiz` | app code work | Screen contracts, question generation, UI styling, optional features | `$quiz/screens`, `$quiz/questions`, `$quiz/style`, `$quiz/features` |
| `$setup-exam` | new exam | Full 10-step pipeline (schema, stages, tables, domains, questions, app build, deploy) | (standalone) |
| `$adapt-questions` | question bank import | Schema mapping, loading strategies, domain coverage | (standalone) |

### Skill dependencies

- `$setup-exam` uses `$sis/pre-deploy` (pre-deploy scan), `$quiz/*` (app generation), optionally `$adapt-questions` (CSV/JSON schema mismatch)
- `$adapt-questions` depends on `$cortex/patterns` (if Strategy D uses AI_COMPLETE)
- `$sis/pre-deploy` depends on `$cortex/patterns` (dollar-quoting, JSON parsing)
- `$cortex/prompt-audit` validates prompts authored during `$setup-exam`

### Global skills

CoCo in Snowsight ships with built-in skills (including `cortex-ai-functions` reference for all Cortex AI functions). They are available natively from any workspace - no upload needed. Consult them when using functions not covered by `$cortex/patterns`.

---

## Security and governance

1. **Isolation**: all DDL/DML in `{database}.{schema}` only. Never cross schemas.
2. **No DROP** on existing objects. Never `DROP SCHEMA`, `DROP DATABASE`, `DROP TABLE`.
3. **Idempotent DDL**: `CREATE TABLE IF NOT EXISTS`, `CREATE STAGE IF NOT EXISTS`. `CREATE OR REPLACE` is allowed for `STREAMLIT` and `FILE FORMAT` only.
4. **Parameterized SQL** for all user-derived values. Never interpolate widget values into f-string SQL.
5. **AI_COMPLETE**: dollar-quote the prompt, sanitize any `$$` in interpolated content to `$ $`. `CORTEX_MODEL` is a hardcoded constant.
6. **Schema-per-exam** is the mandatory isolation boundary the agent enforces. The user may additionally create a Git branch per exam.
