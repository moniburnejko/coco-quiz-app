# Skills

Custom Cortex Code skills in this project live under `.snowflake/cortex/skills/` and are uploaded once per workspace via **Cortex Code chat > + > Upload Folder(s)**.

There are **13 skill files**: 2 top-level standalone pipelines, 3 parent "routers" that dispatch to intent-specific sub-skills, and 8 sub-skills. The parent routers are what you invoke via slash command (`/cortex`, `/sis`, `/quiz`). They read your intent keywords and forward to the right sub-skill automatically.

Global Cortex Code skills (like `cortex-ai-functions`) ship natively with Cortex Code in Snowsight - no upload needed.

---

## /setup-exam - standalone

**Scope:** the 10-step end-to-end pipeline that takes a study-guide PDF from upload to deployed Streamlit app.

**When to use:**
- first run of the workspace - sets everything up for a new exam;
- adding another certification - creates a parallel `QUIZ_<NEW_CODE>` schema without touching the previous one.

**What it does:**
- collects exam metadata (name, code), PDF filename, optional CSV/JSON filename, optional feature list;
- creates the schema, both stages (`STAGE_QUIZ_DATA` with `SNOWFLAKE_SSE` + `DIRECTORY`, `STAGE_SIS_APP`), all 4 tables, and the CSV/JSON file format;
- stops for manual PDF upload; calls `AI_PARSE_DOCUMENT` + `AI_COMPLETE` to populate `EXAM_DOMAINS` (domains, weights, topics, `key_facts`);
- either loads the CSV/JSON question bank (via `$adapt-questions` if columns need remapping) or generates ~30 AI questions per domain in batches of 10 (when you want to run the quiz faster);
- updates `AGENTS.md` in place with new exam code, schema, PDF filename;
- reads `AGENTS.md` + all `$quiz/*` sub-skills, generates `quiz.py` + `environment.yml` in the workspace;
- runs `$sis/pre-deploy` 22-item scan, fixes until clean;
- stops for manual app-file upload to `STAGE_SIS_APP`, then runs `CREATE STREAMLIT` (Path A). Optionally the user uses the workspace right-click "Deploy as Streamlit App" (Path B - unverified).

**Built-in stopping points:** input collection, upload PDF, domain approval, pre-deploy gate, deploy path choice, final report. The agent never proceeds past these without user confirmation.

---

## /adapt-questions - standalone

**Scope:** schema adaptation for user-supplied question-bank files (CSV or JSON) that don't match the `QUIZ_QUESTIONS` target schema directly.

**When to use:**
- invoked from `$setup-exam` step 6 when a CSV has been uploaded;
- invoked directly: "scan my questions.json for compatibility" or "import this CSV".

**What it does:**
- inspects source columns via SQL (`SELECT $1..$N FROM @STAGE_QUIZ_DATA/<file> ...`) - no bash, no local filesystem;
- maps source columns to the target schema (`domain_id`, `difficulty`, `question_text`, `option_a..e`, `correct_answer`, `is_multi`, `source`);
- picks the loading strategy (direct `COPY INTO`, transform via `INSERT SELECT`, or `AI_COMPLETE`-assisted mapping for messy sources);
- handles answer-key formats (letter, full text, index);
- backfills `domain_name` from `EXAM_DOMAINS` via join.

---

## /cortex - parent router

Dispatches to one of two sub-skills depending on keywords in your message: **patterns** (for calling/debugging) or **prompt-audit** (for quality review).

### /cortex/patterns

**Scope:** calling conventions, error handling, and a 5-step diagnostic for `AI_COMPLETE` and `AI_PARSE_DOCUMENT`.

**When to use:**
- writing any SQL that calls a Cortex AI function;
- diagnosing "file not accessible", "model not found", NULL responses, parse errors;
- verifying stage prerequisites (`SNOWFLAKE_SSE`, `DIRECTORY=TRUE`).

**What it covers:**
- `AI_COMPLETE`: `$$...$$` dollar-quoting, `$$` sanitisation before interpolation, error handling wrapper, VARIANT-string response decoding, `parse_cortex_json()` for markdown fences + double-encoding.
- `AI_PARSE_DOCUMENT`: correct `TO_FILE(...)` + options object, common mistakes (no `BUILD_SCOPED_FILE_URL`, no `PARSE_JSON` wrap, no per-page pagination), stage DDL.
- `AI_EXTRACT`: noted as an alternative for structured extraction.
- 5-step diagnostic runbook: basic connectivity, model access, cross-region parameter, JSON output, available models.

### /cortex/prompt-audit

**Scope:** 8-item audit checklist for any `AI_COMPLETE` prompt in `quiz.py`.

**When to use:**
- `parse_cortex_json` raises `KeyError` or returns `None` for expected fields;
- AI explanations are shallow, generic, or missing per-option reasoning;
- a prompt was just edited and needs a sanity check.

**What it checks:**
- JSON output reliability (required keys, no unexpected fences, no double-encoding traps);
- content completeness (difficulty adherence, grounding from `key_facts`);
- deduplication block uses `_get_shown_texts()` correctly;
- injection safety (user-derived values never f-string-interpolated into prompts);
- prompt length + token budget for the selected model.

---

## /sis - parent router

Dispatches to **patterns** (for writing code) or **pre-deploy** (for the mandatory scan).

### /sis/patterns

**Scope:** Streamlit-in-Snowflake v1.52.* coding rules for any `quiz.py` work.

**When to use:**
- writing or modifying any SiS code;
- debugging runtime errors (rerun loops, widget state drift, date bugs, cache misses);
- reviewing generated code for SiS compatibility.

**What it covers:**
- `get_active_session()` placement and cache scope;
- `st.rerun()` budget (must be exactly 6, at specific call sites);
- unsupported APIs in SiS 1.52 (`st.fragment`, `st.connection`, `unsafe_allow_html=True`);
- `session_state` reliability - mutable objects, rerun timing, safe patterns;
- date handling, column-name normalisation, page config, table rendering with pandas.

### /sis/pre-deploy

**Scope:** **mandatory** 22-item scan run before every `CREATE STREAMLIT`. Catches the top runtime-failure classes before they reach production.

**When to use:**
- before every deploy - no exceptions;
- after any code change, before re-uploading to `STAGE_SIS_APP`;
- when reviewing generated code for SiS compatibility.

**What it checks:** SQL-injection safety, `AI_COMPLETE` dollar-quoting, `parse_cortex_json` usage, `st.rerun()` count and placement, SiS-incompatible APIs, session-state patterns, date handling, column-name conventions, page config, and 12 more. All must pass.

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

## skill dependency graph

```
$setup-exam ----┬--> $adapt-questions -> $cortex/patterns (if AI-assisted mapping)
                ├--> $cortex/patterns (AI_PARSE_DOCUMENT, AI_COMPLETE)
                ├--> $sis/pre-deploy -> $cortex/patterns (dollar-quoting check)
                └--> $quiz/screens, $quiz/questions, $quiz/style, $quiz/features

$cortex/prompt-audit - runs against prompts produced by $setup-exam and $quiz/questions
```