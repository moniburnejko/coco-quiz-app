# SnowPro Core certification quiz app - Streamlit in Snowflake

> ## ⛔ How to run this project — do NOT improvise
> To set up an exam you **MUST** open `.snowflake/cortex/skills/setup-exam/SKILL.md` and execute it **step by step, top to bottom**, honoring every mandatory STOP. Read the whole skill first; create nothing before its Step 2.
> The sections below are **reference context** (what the app is, the data model, constraints) — they are **NOT the build procedure** and are not detailed enough to improvise from. Improvising from this overview instead of loading the skill causes known failures (stage without encryption, unfilled placeholders, deploy without compute pool + PyPI EAI).

## What this project is

A context package for Snowflake CoCo that, given a study guide PDF, autonomously creates the Snowflake schema, extracts exam domains, loads an optional question bank, builds and deploys a multipage Streamlit-in-Snowflake quiz app, and tracks learning progress across sessions. Invoked via `$setup-exam` — **which you must read and follow, not summarize.**

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
| compute_pool | `<your_compute_pool>` (container runtime) |
| external_access_integration | `pypi_access_integration` (container runtime — lets it install pandas/altair from PyPI) |
| runtime   | `container` (default) / `warehouse` (fallback) |
| deps_file | `pyproject.toml` (container) / `environment.yml` (warehouse) |

**Preconditions** (user-set before running `$setup-exam`): replace `<your_database>`, `<your_warehouse>`, `<your_role>` with actual object names; replace `<your_compute_pool>` too unless you plan the warehouse fallback (`$setup-exam` Step 9 Path C). The role must have `CREATE SCHEMA` on `database`. **The default container runtime needs TWO extra things — a compute pool (`USAGE`) and a PyPI external access integration** (so it can install pandas/altair; the base image has only Python/Streamlit/Snowpark). `$setup-exam` Step 1f checks both up front and gives you the DDL if missing. The warehouse fallback needs neither. `$setup-exam` stops if any required `<...>` placeholder remains unfilled.

**Outputs** (populated in the table above by `$setup-exam`): `schema` (= `<database>.QUIZ_<EXAM_CODE>`) and `exam_code` (from the study guide PDF). `$setup-exam` is idempotent (`IF NOT EXISTS`) and never drops. Each exam gets its own schema - never share a schema between exams.

**Project defaults** (customizable but have working values): stages, `app_name`, `main_file`, `runtime`, `deps_file`. Change only if you need to.

All sections reference these values. Never hardcode environment names elsewhere in this file.

---

## Domain model

Five tables, all in `{database}.{schema}`. Full DDL lives in `$setup-exam` Step 3. (A sixth, QUIZ_FLAGS, exists only when the flag-a-question feature is enabled.)

### EXAM_DOMAINS
Populated once per exam from the study guide PDF via `AI_PARSE_DOCUMENT` + `AI_COMPLETE`.
columns: `domain_id`, `domain_name`, `weight_pct`, `topics`, `key_facts`.

### QUIZ_QUESTIONS
The optional question bank. Loaded from CSV in `$setup-exam` Step 6 if the user has one; otherwise stays empty and is seeded post-build (Admin "Generate batch", worksheet recipe, scheduled task/Automation). NEVER auto-generated during setup. Not to be confused with runtime AI generation, which bypasses this table.
columns: `question_id`, `domain_id`, `domain_name`, `difficulty`, `question_text`, `is_multi`, `option_a..e`, `correct_answer`, `source`, `created_at`.

### QUIZ_REVIEW_LOG
Per-question wrong-answer history, written by the app at round end. Drives the Review page, domain error analysis, and (optionally) misconception analysis.
columns: `log_id`, `logged_at`, `domain_id`, `domain_name`, `difficulty`, `question_text`, `correct_answer`, `selected_answer`, `mnemonic`, `doc_url`, `misconception`.

### QUIZ_SESSION_LOG
Per-round summary, written by the app at round end. Drives the Learning Dashboard progress metrics. Remedial rounds write nothing (by design).
columns: `session_id`, `session_ts`, `exam_code`, `round_size`, `correct_count`, `score_pct`, `domain_filter`, `difficulty`.

### QUIZ_CONFIG
Runtime app configuration (key-value, VARIANT), edited from the Admin page. Defaults live in `_config.py` `CONFIG_DEFAULTS`; DB values override them via the cached `load_config()`. Keys include the learning-loop toggles and `docs_grounding` (`auto`/`on`/`off` — grounding in the Snowflake Documentation CKE; see `$cortex`).
columns: `config_key`, `config_value`, `updated_at`.

---

## Cortex LLM

Preferred model: `claude-sonnet-4-6`. Store as constant `CORTEX_MODEL` in `_config.py`.

Accounts that cannot reach the chosen model in-region must enable cross-region inference (once per account, as ACCOUNTADMIN): `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';` (`'AWS_GLOBAL'` is a narrower alternative; the legacy `'AWS_US'` still works but is narrowest). Accounts created after 2026-03-09 default to `ANY_REGION` and may need no change.

For calling patterns, dollar-quoting, structured outputs (`response_format`), diagnostics, and prompt-quality audit: see `$cortex`.

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
| `pages/quiz.py` | QUIZ page: home → quiz → summary state machine (hints, contrast, debrief, remedial round) |
| `pages/review.py` | REVIEW page: wrong-answer history + learning dashboard |
| `pages/admin.py` | ADMIN page: app config (QUIZ_CONFIG), question manager + Generate batch, bank stats, Cortex spend, tools |
| `pages/<feature>.py` | Generated ONLY when the user requests an optional feature (`$quiz/features`) - e.g. `exam_simulation.py`, `flashcards.py`, `recommendations.py` |

### Pages and navigation

Navigation is native multipage via `st.Page` + `st.navigation`, built in `main.py`. `st.session_state` is shared across pages.

**QUIZ page**: home → quiz → summary. Home configures round settings (questions, domains, difficulty, source, explanations). Quiz presents questions with lazy loading and AI explanations. Summary shows score, pass/fail, and wrong answer cards.

**REVIEW page** (sub-tabs): Wrong Answers shows filtered history with domain/date filters. Learning Dashboard shows session metrics and charts (score trend, error distribution). Optional features from `$quiz/features` add their own pages, not tabs.

For screen contracts, session state, history schema, and write-back: see `$quiz/screens`.
For UI styling, badges, section labels, and chart colors: see `$quiz/design`.

---

## Available skills

| Skill | Invoke | Purpose | Sub-skills |
|-------|--------|---------|------------|
| `$cortex` | Cortex AI work | Structured outputs, injection delimiting, CKE grounding, diagnostics, prompt audit | (standalone) |
| `$sis` | SiS code or deploy | Container-runtime gotchas + mandatory pre-deploy scan | (standalone) |
| `$quiz` | app code work | Screen contracts, question generation, UI styling, optional features | `$quiz/screens`, `$quiz/questions`, `$quiz/design`, `$quiz/features` |
| `$setup-exam` | new exam | Full 10-step pipeline (schema, stages, tables, domains, questions, app build, deploy) | (standalone) |
| `$adapt-questions` | question bank import | Schema mapping, loading strategies, domain coverage | (standalone) |

### Skill dependencies

- `$setup-exam` uses `$sis` (pre-deploy scan), `$quiz/*` (app generation), optionally `$adapt-questions` (CSV/JSON schema mismatch)
- `$adapt-questions` depends on `$cortex` (if Strategy D uses AI_COMPLETE)
- `$sis` depends on `$cortex` (dollar-quoting, JSON parsing)
- `$cortex` validates prompts authored during `$setup-exam`

### Global skills

CoCo in Snowsight ships with built-in skills, available natively from any workspace - no upload needed. This project's skills are a **thin layer over them**: `$cortex/*` defers to `cortex-ai-functions` (full Cortex AI reference), `$sis/*` defers to `developing-with-streamlit` (general Streamlit patterns) and `deploy-to-spcs`/`snowflake-apps` (deploy mechanics). Consult the bundled skills for anything not covered by the project deltas.

---

## Security and governance

1. **Isolation**: all DDL/DML in `{database}.{schema}` only. Never cross schemas.
2. **No DROP** on existing objects. Never `DROP SCHEMA`, `DROP DATABASE`, `DROP TABLE`.
3. **Idempotent DDL**: `CREATE TABLE IF NOT EXISTS`, `CREATE STAGE IF NOT EXISTS`. `CREATE OR REPLACE` is allowed for `STREAMLIT` and `FILE FORMAT` only.
4. **Parameterized SQL** for all user-derived values. Never interpolate widget values into f-string SQL.
5. **AI_COMPLETE**: dollar-quote the prompt, sanitize any `$$` in interpolated content to `$ $`. `CORTEX_MODEL` is a hardcoded constant.
6. **Schema-per-exam** is the mandatory isolation boundary the agent enforces. The user may additionally create a Git branch per exam.
