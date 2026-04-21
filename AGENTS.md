# snowpro core certification quiz app - streamlit in snowflake

## what this project is

a context package for cortex code that, given a study guide PDF, autonomously creates the snowflake schema, extracts exam domains, generates or loads questions, builds and deploys a 4-screen streamlit-in-snowflake quiz app, and tracks learning progress across sessions. invoked via `$setup-exam`.

---

## snowflake environment

do not touch any database or schema other than the one configured below:

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
| main_file | `quiz.py`                            |

**preconditions** (user-set before running `$setup-exam`): replace `<your_database>`, `<your_warehouse>`, `<your_role>` with actual object names. the role must have `CREATE SCHEMA` on `database`. `$setup-exam` stops if any `<...>` placeholder remains unfilled.

**outputs** (populated in the table above by `$setup-exam`): `schema` (= `<database>.QUIZ_<EXAM_CODE>`) and `exam_code` (from the study guide PDF). `$setup-exam` is idempotent (`IF NOT EXISTS`) and never drops. each exam gets its own schema - never share a schema between exams.

**project defaults** (customizable but have working values): stages, `app_name`, `main_file`. change only if you need to.

all sections reference these values. never hardcode environment names elsewhere in this file.

---

## domain model

four tables, all in `{database}.{schema}`. full DDL lives in `$setup-exam` Step 3.

### EXAM_DOMAINS
populated once per exam from the study guide PDF via `AI_PARSE_DOCUMENT` + `AI_COMPLETE`.
columns: `domain_id`, `domain_name`, `weight_pct` (sums to 100), `topics` (VARIANT array), `key_facts`.

### QUIZ_QUESTIONS
pre-loaded from CSV (if user has one) or AI-generated in `$setup-exam` Step 6 grounded on `key_facts`. not to be confused with runtime AI generation, which bypasses this table.
columns: `question_id`, `domain_id`, `domain_name`, `difficulty`, `question_text`, `is_multi`, `option_a..e`, `correct_answer`, `source`, `created_at`.

### QUIZ_REVIEW_LOG
per-question wrong-answer history, written by the app at round end. drives the Review tab and domain error analysis.
columns: `log_id`, `logged_at`, `domain_id`, `domain_name`, `difficulty`, `question_text`, `correct_answer`, `mnemonic`, `doc_url`.

### QUIZ_SESSION_LOG
per-round summary, written by the app at round end. drives the Learning Dashboard progress metrics.
columns: `session_id`, `session_ts`, `exam_code`, `round_size`, `correct_count`, `score_pct`, `domain_filter`, `difficulty`.

---

## cortex llm

preferred model: `claude-sonnet-4-6`. store as constant `CORTEX_MODEL` in quiz.py.

EU account requires `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'AWS_US'` (once per account, as ACCOUNTADMIN).

for calling patterns, dollar-quoting, JSON parsing, and diagnostics: see `$cortex/patterns`.
for prompt quality audit: see `$cortex/prompt-audit`.

---

## app overview

quiz.py is the **reference implementation**. read it directly for HOW things work. this section describes WHAT the app does.

### core functions (in order)

1. `init_session_state()` - defaults for all session state keys
2. `load_domains()` - cached, `get_active_session()` inside
3. `load_session_stats()` - cached, aggregate stats from QUIZ_SESSION_LOG
4. `load_recent_sessions()` - cached, last 10 sessions with labels
5. `load_domain_errors()` - cached, error counts per domain
6. `call_cortex(prompt)` - AI_COMPLETE with dollar-quoting and error handling
7. `parse_cortex_json(response)` - safe JSON parsing (markdown fences, double-encoding)
8. `parse_topics(topics_raw)` - VARIANT/JSON topics field parser
9. `_build_topic_schedule(domains, domain_filter, round_size)` - shuffled (domain, topic) pairs
10. `generate_ai_question(domain, difficulty, topic)` - topic-focused AI question generation
11. `get_question(domains, difficulty_filter, domain_filter, question_source)` - next question from schedule
12. `render_home(domains)` - configuration screen
13. `render_quiz(domains)` - question display and answer handling
14. `render_summary()` - round results
15. `render_review()` - wrong answers history
16. `render_dashboard(domains)` - learning analytics

optional features (from `$quiz/features`) may add more functions - e.g., `render_ai_recommendations()`, `render_flashcards()`, `render_sim_quiz()`, `compute_badges()`.

### screens

**QUIZ page** (sidebar pill): home → quiz → summary. Home configures round settings (questions, domains, difficulty, source, explanations). Quiz presents questions with lazy loading and AI explanations. Summary shows score, pass/fail, and wrong answer cards.

**REVIEW page** (sidebar pill, sub-tabs): Wrong Answers shows filtered history with domain/date filters. Learning Dashboard shows session metrics and charts (score trend, error distribution). Optional features from `$quiz/features` may add more tabs.

for screen contracts, session state, history schema, and write-back: see `$quiz/screens`.
for UI styling, badges, section labels, and chart colors: see `$quiz/style`.

---

## available skills

| Skill | Invoke | Purpose | Sub-skills |
|-------|--------|---------|------------|
| `$cortex` | Cortex AI work | Calling patterns, diagnostics, prompt auditing | `$cortex/patterns`, `$cortex/prompt-audit` |
| `$sis` | SiS code or deploy | Coding patterns, mandatory pre-deploy scan | `$sis/patterns`, `$sis/pre-deploy` |
| `$quiz` | quiz.py work | Screen contracts, question generation, UI styling, optional features | `$quiz/screens`, `$quiz/questions`, `$quiz/style`, `$quiz/features` |
| `$setup-exam` | new exam | Full 10-step pipeline (schema, stages, tables, domains, questions, quiz.py, deploy) | (standalone) |
| `$adapt-questions` | question bank import | Schema mapping, loading strategies, domain coverage | (standalone) |

### skill dependencies

- `$setup-exam` uses `$sis/pre-deploy` (pre-deploy scan), `$quiz/*` (quiz.py generation), optionally `$adapt-questions` (CSV/json schema mismatch)
- `$adapt-questions` depends on `$cortex/patterns` (if Strategy D uses AI_COMPLETE)
- `$sis/pre-deploy` depends on `$cortex/patterns` (dollar-quoting, JSON parsing)
- `$cortex/prompt-audit` validates prompts authored during `$setup-exam`

### global skills

cortex code in snowsight ships with built-in skills (including `cortex-ai-functions` reference for all cortex AI functions). they are available natively from any workspace - no upload needed. consult them when using functions not covered by `$cortex/patterns`.

---

## security and governance

1. **isolation**: all DDL/DML in `{database}.{schema}` only. never cross schemas.
2. **no DROP** on existing objects. never `DROP SCHEMA`, `DROP DATABASE`, `DROP TABLE`.
3. **idempotent DDL**: `CREATE TABLE IF NOT EXISTS`, `CREATE STAGE IF NOT EXISTS`. `CREATE OR REPLACE` is allowed for `STREAMLIT` and `FILE FORMAT` only.
4. **parameterized SQL** for all user-derived values. never interpolate widget values into f-string SQL.
5. **AI_COMPLETE**: dollar-quote the prompt, sanitize any `$$` in interpolated content to `$ $`. `CORTEX_MODEL` is a hardcoded constant.
6. **schema-per-exam** is the mandatory isolation boundary the agent enforces. the user may additionally create a git branch per exam.
