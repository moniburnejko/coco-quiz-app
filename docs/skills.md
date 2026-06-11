# Skills

Custom Snowflake CoCo skills in this project live under `.snowflake/cortex/skills/` and are uploaded once per workspace via **CoCo chat > + > Upload Folder(s)**.

There are **13 skill files**: 2 top-level standalone pipelines, 3 parent "routers", and 8 sub-skills. CoCo activates skills by matching your message against each skill's `description` (every description also lists "Do NOT use for…" anti-triggers to prevent cross-firing); you can also invoke explicitly via slash command (`/setup-exam`, `/cortex`, `/sis`, `/quiz`). Each parent router's **body** contains the dispatch instructions — "for intent X, load `<sub-skill path>` and follow it" — which is how CoCo's own bundled router skills work (routing is prose, not frontmatter).

This pack is a **thin layer over CoCo's bundled skills**: `$cortex/*` defers to the built-in `cortex-ai-functions` (full Cortex AI reference), `$sis/*` defers to `developing-with-streamlit` (general Streamlit patterns) and `deploy-to-spcs`/`snowflake-apps` (deploy mechanics). The bundled `skill-development` skill can lint this pack. Bundled skills ship natively with CoCo in Snowsight - no upload needed.

---

## /setup-exam - standalone

**Scope:** the 10-step end-to-end pipeline that takes a study-guide PDF from upload to deployed Streamlit app.

**When to use:**
- First run of the workspace - sets everything up for a new exam;
- Adding another certification - creates a parallel `QUIZ_<NEW_CODE>` schema without touching the previous one.

**What it does:**
- Collects exam metadata (name, code), PDF filename, optional CSV/JSON filename, optional feature list;
- Creates the schema, both stages (`STAGE_QUIZ_DATA` with `SNOWFLAKE_SSE` + `DIRECTORY`, `STAGE_SIS_APP`), all 4 tables, and the CSV/JSON file format;
- Stops for manual PDF upload; calls `AI_PARSE_DOCUMENT` + `AI_COMPLETE` to populate `EXAM_DOMAINS` (domains, weights, topics, `key_facts`);
- Either loads the CSV/JSON question bank (via `$adapt-questions` if columns need remapping) or generates ~30 AI questions per domain in batches of 10 (when you want to run the quiz faster);
- Updates `AGENTS.md` in place with new exam code, schema, PDF filename;
- Reads `AGENTS.md` + all `$quiz/*` sub-skills, generates the decomposed multipage `app/` project (entry point, `_*.py` modules, `pages/`, configs) in the workspace;
- Runs the `$sis/pre-deploy` scan across all app files, fixes until clean;
- Deploys: by default the user previews with **Run** and clicks **Deploy** in the workspace (Path A); scripted fallback = upload `app/` to `STAGE_SIS_APP` + `CREATE STREAMLIT` on the container runtime (Path B); warehouse fallback when no compute pool exists (Path C).

**Built-in stopping points:** input collection, upload PDF, domain approval, pre-deploy gate, deploy path choice, final report. The agent never proceeds past these without user confirmation.

---

## /adapt-questions - standalone

**Scope:** schema adaptation for user-supplied question-bank files (CSV or JSON) that don't match the `QUIZ_QUESTIONS` target schema directly.

**When to use:**
- Invoked from `$setup-exam` step 6 when a CSV has been uploaded;
- Invoked directly: "scan my questions.json for compatibility" or "import this CSV".

**What it does:**
- Inspects source columns via SQL (`SELECT $1..$N FROM @STAGE_QUIZ_DATA/<file> ...`) - no bash, no local filesystem;
- Maps source columns to the target schema (`domain_id`, `difficulty`, `question_text`, `option_a..e`, `correct_answer`, `is_multi`, `source`);
- Picks the loading strategy (direct `COPY INTO`, transform via `INSERT SELECT`, or `AI_COMPLETE`-assisted mapping for messy sources);
- Handles answer-key formats (letter, full text, index);
- Backfills `domain_name` from `EXAM_DOMAINS` via join.

---

## /cortex - parent router

Dispatches to one of two sub-skills depending on keywords in your message: **patterns** (for calling/debugging) or **prompt-audit** (for quality review).

### /cortex/patterns

**Scope:** calling conventions, error handling, and a 5-step diagnostic for `AI_COMPLETE` and `AI_PARSE_DOCUMENT`.

**When to use:**
- Writing any SQL that calls a Cortex AI function;
- Diagnosing "file not accessible", "model not found", NULL responses, parse errors;
- Verifying stage prerequisites (`SNOWFLAKE_SSE`, `DIRECTORY=TRUE`).

**What it covers:**
- `AI_COMPLETE`: `$$...$$` dollar-quoting, `$$` sanitisation before interpolation, **structured outputs** (`response_format` schemas in `RESPONSE_FORMATS`, `call_cortex_json()` helper — guaranteed schema-conformant JSON, no fence parsing).
- `AI_PARSE_DOCUMENT`: correct `TO_FILE(...)` + options object, common mistakes (no `BUILD_SCOPED_FILE_URL`, no `PARSE_JSON` wrap, no per-page pagination), stage DDL.
- `AI_EXTRACT`: noted as an optional alternative for structured extraction (`$setup-exam` Step 5b).
- 5-step diagnostic runbook: basic connectivity, model access, cross-region parameter, structured output, available models.
- Defers to the bundled `cortex-ai-functions` skill for the full Cortex AI reference.

### /cortex/prompt-audit

**Scope:** 7-item audit checklist for any `AI_COMPLETE` prompt in the app modules.

**When to use:**
- `call_cortex_json` returns `None` or a dict missing expected content;
- AI explanations are shallow, generic, or missing per-option reasoning;
- A prompt was just edited and needs a sanity check.

**What it checks:**
- Structured output requested (`response_format` schema covers every key the code reads);
- Content completeness (difficulty adherence, grounding from `key_facts`);
- Deduplication block uses `_get_shown_texts()` correctly;
- Injection safety (user-derived values never f-string-interpolated into prompts);
- `doc_search` pattern (no hallucinated URLs).

---

## /sis - parent router

Dispatches to **patterns** (for writing code) or **pre-deploy** (for the mandatory scan).

### /sis/patterns

**Scope:** Streamlit-in-Snowflake coding rules for the container runtime — any app-module work.

**When to use:**
- Writing or modifying any SiS code;
- Debugging runtime errors (widget state drift, date bugs, cache staleness);
- Reviewing generated code for SiS compatibility.

**What it covers:**
- `get_active_session()` placement and cache scope;
- Caching without `ttl` + explicit `clear_caches()` invalidation after writes;
- Widget lifecycle (flag-at-top reset, `on_click`), multipage `session_state`, `@st.fragment` scoped reruns;
- Still-constrained APIs (CSP / `unsafe_allow_html`, `.applymap` removed in pandas 3, `st.experimental_rerun`);
- Date handling, column-name normalisation, button click safety, SQL safety.
- Defers to the bundled `developing-with-streamlit` skill for general Streamlit patterns.

### /sis/pre-deploy

**Scope:** **mandatory** 20-item scan across all app files (`main.py`, `_*.py`, `pages/*.py`, `config.toml`), run before every deploy. Catches the top runtime-failure classes before they reach production.

**When to use:**
- Before every deploy - no exceptions;
- After any code change, before re-deploying (Workspaces Deploy or stage upload);
- When reviewing generated code for SiS compatibility.

**What it checks:** SQL-injection safety, `AI_COMPLETE` dollar-quoting, structured-output usage, cache discipline (no `ttl` + `clear_caches()` after writes), `config.toml` settings, `st.set_page_config` placement, screen transitions, date handling, column-name conventions, and more. All must pass.

---

## /quiz - parent router

Dispatches across four sub-skills depending on what you are working on: **screens**, **questions**, **style**, or **features**.

### /quiz/screens

**Scope:** behavioural contracts for the quiz app screens - home, quiz, summary, review, dashboard.

**When to use:** building or modifying any screen; adding a new feature to the flow; debugging screen transitions or state.

**What it covers:** screen flow diagram, session-state key table, `history_item` schema, write-back to `QUIZ_REVIEW_LOG` and `QUIZ_SESSION_LOG` on round end, explanation rendering contract (collapsed by default, expands to `why_correct` / `why_wrong` / `mnemonic` / `doc_url`), dashboard chart specs (score-per-session line chart, domain error bar chart, readiness score).

### /quiz/questions

**Scope:** question-selection, generation, validation, deduplication.

**When to use:** loading questions from DB or generating via AI; building the topic schedule; debugging coverage or quality issues.

**What it covers:** `DIFFICULTY_GUIDE` constant (required), topic schedule (`_build_topic_schedule`), deduplication via `_get_shown_texts()` from `round_history`, fallback chain (DB → AI on miss), retry/backoff on parse failure, answer shuffling, JSON schema for AI-generated questions with 2–5 options + multi-answer flag.

### /quiz/style

**Scope:** UI conventions - badge colours, section labels, chart colours, CSS rules, button style.

**When to use:** any visual or layout work; reviewing visual consistency across screens; adding new badges / cards / sections.

**What it covers:** `EXAM_NAME` constant, badge colour palette, section-label pattern, chart colour constants (`#29b5e8` blue, `#F1914C` orange), axis formatting, docs-link format, card layout, button style variants.

### /quiz/features

**Scope:** optional feature implementations - exam simulation mode, flashcards, quick stats, spaced repetition, achievement badges, AI study recommendations.

**When to use:** **only** when the user explicitly requests a feature in their prompt. Never trigger by default.

**What it covers:** per-feature: description, UI location, data dependencies (new tables if any), screen hook points, session-state additions.

---

## Skill dependency graph

```
$setup-exam ----┬--> $adapt-questions -> $cortex/patterns (if AI-assisted mapping)
                ├--> $cortex/patterns (AI_PARSE_DOCUMENT, AI_COMPLETE structured outputs)
                ├--> $sis/pre-deploy -> $cortex/patterns (dollar-quoting, response_format)
                └--> $quiz/screens, $quiz/questions, $quiz/style, $quiz/features

$cortex/prompt-audit - runs against prompts produced by $setup-exam and $quiz/questions

Bundled CoCo skills this pack defers to (built-in, no upload):
  $cortex/* -> cortex-ai-functions          $sis/* -> developing-with-streamlit
  deploy    -> deploy-to-spcs / snowflake-apps        authoring/lint -> skill-development
```