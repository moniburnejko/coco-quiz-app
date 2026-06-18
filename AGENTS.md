# SnowPro Core certification quiz app - Streamlit in Snowflake

> ## ⛔ How to run this project - do NOT improvise
> To set up an exam you **MUST** open `.snowflake/cortex/skills/setup-exam/SKILL.md` and execute it **step by step, top to bottom**, honoring every mandatory STOP. Read the whole skill first; create nothing before its Step 2.
> The sections below are **reference context** (what the app is, the data model, constraints) - they are **NOT the build procedure** and are not detailed enough to improvise from. Improvising from this overview instead of loading the skill produces wrong or broken objects.

## What this project is

A context package for Snowflake CoCo that, given a study guide PDF, autonomously creates the Snowflake schema, extracts exam domains, loads an optional question bank, builds and deploys a multipage Streamlit-in-Snowflake quiz app, and tracks learning progress across sessions. Invoked via `$setup-exam` - **which you must read and follow, not summarize.**

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
| runtime   | `warehouse`                          |
| deps_file | `environment.yml` (Snowflake Anaconda channel) |

**Preconditions** (user-set before running `$setup-exam`): replace `<your_database>`, `<your_warehouse>`, `<your_role>` with actual object names. The role must have `CREATE SCHEMA` on `database`. `pandas`/`altair` come from the Snowflake Anaconda channel. `$setup-exam` stops if any required `<...>` placeholder remains unfilled.

**Outputs** (populated in the table above by `$setup-exam`): `schema` (= `<database>.QUIZ_<EXAM_CODE>`) and `exam_code` (from the study guide PDF). `$setup-exam` is idempotent (`CREATE OR ALTER` for tables, `IF NOT EXISTS` for schema/stages) and never drops a schema or table. Each exam gets its own schema - never share a schema between exams.

**Project defaults** (customizable but have working values): stages, `app_name`, `main_file`, `deps_file` (`environment.yml`).

All sections reference these values. Never hardcode environment names elsewhere in this file.

---

## Domain model

Five tables in `{database}.{schema}`, plus a transient `_DOC_CONTENT` used only during setup (dropped after the PDF is parsed). Full DDL and column details live in `$setup-exam` Step 3 - the single source.

| Table | Purpose |
|-------|---------|
| `EXAM_DOMAINS` | Domains, weights, topics, key_facts - extracted once from the study-guide PDF. |
| `QUIZ_QUESTIONS` | Question bank: CSV/seeded `'MANUAL'` rows plus every runtime AI question persisted as `'AI_GENERATED'`; reused with randomized selection. |
| `QUIZ_REVIEW_LOG` | Per-question wrong-answer history - drives the Review page. |
| `QUIZ_SESSION_LOG` | Per-round summary - drives the Learning Dashboard. |
| `QUIZ_CONFIG` | Runtime key-value app config edited from Admin (defaults in `_config.py`); also stores `grounding_mode`, set once at setup and read-only at runtime. |

---

## Cortex LLM

Preferred model: `claude-sonnet-4-6`. Store as constant `CORTEX_MODEL` in `_config.py`.

Accounts that cannot reach the chosen model in-region must enable cross-region inference (once per account, as ACCOUNTADMIN): `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';` (`'AWS_GLOBAL'` is a narrower alternative; the legacy `'AWS_US'` still works but is narrowest). Accounts created after 2026-03-09 default to `ANY_REGION` and may need no change.

For calling patterns, dollar-quoting, structured outputs (`response_format`), diagnostics, and prompt-quality audit: see `$cortex`.

---

## Available skills

| Skill | Invoke | Purpose | Sub-skills |
|-------|--------|---------|------------|
| `$cortex` | Cortex AI work | Structured outputs, injection delimiting, CKE grounding, diagnostics, prompt audit | (standalone) |
| `$sis` | SiS code or deploy | SiS runtime gotchas (warehouse) + mandatory pre-deploy scan | (standalone) |
| `$quiz` | app code work | App module map + screen contracts, question generation, design (visuals) | `$quiz/screens`, `$quiz/questions`, `$quiz/design` |
| `$setup-exam` | new exam | Full 10-step pipeline (schema, stages, tables, domains, questions, app build, deploy) | (standalone) |
| `$adapt-questions` | question bank import | Schema mapping, loading strategies, domain coverage | (standalone) |

### Skill dependencies

- `$setup-exam` uses `$sis` (pre-deploy scan), `$quiz/*` (app generation), optionally `$adapt-questions` (CSV/JSON schema mismatch)
- `$adapt-questions` depends on `$cortex` (if Strategy D uses AI_COMPLETE)
- `$sis` depends on `$cortex` (dollar-quoting, JSON parsing)
- `$cortex` validates prompts authored during `$setup-exam`

### Global skills

CoCo in Snowsight ships with built-in skills, available natively from any workspace - no upload needed. This project's skills are a **thin layer over them**: `$cortex` defers to `cortex-ai-function-studio` + `document-intelligence` (full Cortex AI / doc-parsing reference), `$sis` defers to `developing-with-streamlit-in-snowflake` (general Streamlit) and `snowflake-apps` (deploy mechanics). Consult the bundled skills for anything not covered by the project deltas.

---

## Security and governance

1. **Isolation**: all DDL/DML in `{database}.{schema}` only. Never cross schemas.
2. **No DROP** on existing objects. Never `DROP SCHEMA`, `DROP DATABASE`, `DROP TABLE`.
3. **Idempotent DDL**: `CREATE OR ALTER TABLE` for the data-model tables (reconciles columns to the definition; never `DROP TABLE`); `CREATE … IF NOT EXISTS` for schema and stages (and the CSV file format, created only when a bank CSV is loaded). `CREATE OR REPLACE` only for `STREAMLIT` (the redeployed app).
4. **Parameterized SQL** for all user-derived values. Never interpolate widget values into f-string SQL.
5. **AI_COMPLETE**: dollar-quote the prompt, sanitize any `$$` in interpolated content to `$ $`. `CORTEX_MODEL` is a hardcoded constant.
6. **Schema-per-exam** is the mandatory isolation boundary the agent enforces. The user may additionally create a Git branch per exam.
