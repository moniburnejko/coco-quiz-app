# Skills

Custom Snowflake CoCo skills live under `.snowflake/cortex/skills/` and are uploaded once per workspace via **CoCo chat » + » Upload Folder(s)**.

**9 skill files** — 4 invocable top-level skills plus a `quiz` router with 4 sub-skills. CoCo activates a skill by matching your message against its `description` (each also lists "Do NOT use for…" anti-triggers to stop cross-firing); you can also invoke explicitly with `/`. The `quiz` parent routes via its **body** ("for intent X, load `<sub-skill>` and follow it") — the same prose-routing CoCo's own bundled routers use.

| Invoke | Kind | Carries |
|--------|------|---------|
| `/setup-exam` | standalone | the end-to-end pipeline (PDF → schema → domains → bank → app → deploy) |
| `/adapt-questions` | standalone | map a CSV/JSON question bank to the `QUIZ_QUESTIONS` schema |
| `/cortex` | standalone | AI deltas: structured outputs, injection delimiting, CKE grounding, diagnostics, prompt audit |
| `/sis` | standalone | container-runtime gotchas + the mandatory pre-deploy scan |
| `/quiz` | router → `screens` · `questions` · `design` · `features` | building/modifying the app |

**Thin layer over CoCo's bundled skills** (built-in, no upload): `$cortex` defers to **`cortex-ai-function-studio`** + **`document-intelligence`** (full Cortex AI / doc-parsing reference); `$sis` defers to **`developing-with-streamlit-in-snowflake`** (general Streamlit) and **`deploy-to-spcs`** / **`snowflake-apps`** (deploy mechanics). The bundled **`skill-development`** skill can lint this pack. Our skills carry only the project deltas — decisions, conventions, gotchas, and the app's contracts.

---

## /setup-exam — standalone

**Scope:** the 10-step pipeline that takes a study-guide PDF from upload to a deployed Streamlit app.

**When:** first run of a workspace, or adding another certification (a fresh `QUIZ_<NEW_CODE>` schema; the previous exam untouched).

Collects exam metadata + file names + optional features + look; validates AGENTS.md placeholders and (only for the container opt-in) runs a deploy preflight for a compute pool + PyPI EAI; creates the schema, stages (`STAGE_QUIZ_DATA` with `SNOWFLAKE_SSE`+`DIRECTORY`, verified via `DESCRIBE`), all 5 tables, and the file format; stops for the manual PDF upload; extracts `EXAM_DOMAINS` (domains/weights/topics/`key_facts`); loads the CSV bank if provided (else leaves it empty — no build-time generation); updates AGENTS.md; reads `$quiz/*` + `$sis` + `$cortex` and generates the decomposed `app/`; runs the `$sis` pre-deploy scan; deploys on the **default warehouse runtime** — copies `app/` to `STAGE_SIS_APP` with `COPY FILES` + `CREATE STREAMLIT` (container is an opt-in, deployed via Workspaces Run+Deploy). Also owns the **data-model DDL** and the **Advanced options** (opt-in: quality model / self-verify / Automations / container runtime).

**Built-in stops:** input collection, PDF upload, domain approval, pre-deploy gate, final report.

## /adapt-questions — standalone

**Scope:** adapt a user-supplied CSV/JSON question bank that doesn't match `QUIZ_QUESTIONS` directly. Invoked from `$setup-exam` Step 6 or directly. Inspects the source on the stage, maps columns (with a Status per column), runs a domain-id compatibility check, picks a loading strategy (A direct COPY INTO → E manual fix), executes, backfills `domain_name`, and reports coverage. Two mandatory stops (mapping approval, strategy choice).

## /cortex — standalone

**Scope:** the project's Cortex AI deltas (not a general AI-function reference — that's bundled `cortex-ai-function-studio`). Covers: `AI_COMPLETE` **structured outputs** (`response_format` + `RESPONSE_FORMATS`, the `call_cortex_json` helper — guaranteed schema-conformant JSON, no fence parsing); `$$` dollar-quoting + sanitization; **untrusted-content delimiting** (prompt-injection defense); the **Cortex Search (CKE) `_search.py`** isolation pattern (Python `snowflake.core` at runtime, graceful fallback); a trimmed **diagnostics** runbook; and a **prompt-audit** checklist.

## /sis — standalone

**Scope:** Streamlit-in-Snowflake container-runtime deltas + the **mandatory pre-deploy scan** (run before every deploy). Gotchas: `get_active_session()` inside cached functions; no-`ttl` caching + `clear_caches()`; widget lifecycle (flag-at-top reset, `None`-guards); rerun discipline; multipage state; CSP / `unsafe_allow_html`; `.applymap`→`.map`; `showErrorDetails="none"`; uppercase columns; SQL bind-params. The scan checks every item across all app files; deploy only on a clean pass.

## /quiz — router

Dispatches across four sub-skills. Also carries the **app module map** (which `app/` file owns what).

### /quiz/screens
Behavioural contracts: page flow (`st.navigation`), the home/quiz/summary state machine, the session-state key table, `history_item` schema, the learning loop (Socratic hint, explanation + contrast, AI debrief, fail-only remedial round), write-back + `clear_caches()`, the Admin page, and the multi-answer/button-safety/date-handling widget patterns.

### /quiz/questions
Question selection/generation: `DIFFICULTY_GUIDE`, topic schedule, deduplication via `round_history`, fallback chain (DB → AI), retry, answer shuffling, hybrid doc-grounded generation.

### /quiz/design
**The single source for every visual rule** — theme/`config.toml` keys, badge palette, chart colors + axis formatting, cards, buttons, titles, docs-link. Other skills reference it; they never re-specify a color or chart rule.

### /quiz/features
Optional features (only when explicitly requested): exam simulation, flashcards, quick stats, spaced repetition, achievement badges, AI study recommendation, misconception analysis, flag-a-question. Each spec gives behavior/data/state; visuals follow `$quiz/design`.

---

## Skill dependency graph

```
/setup-exam ──┬── /adapt-questions ── /cortex   (if AI-assisted column mapping)
              ├── /cortex                        (AI_COMPLETE, structured outputs, CKE)
              ├── /sis                           (pre-deploy scan; gotchas the app must satisfy)
              └── /quiz  →  screens · questions · design · features

Bundled CoCo skills this pack defers to (built-in, no upload):
  /cortex → cortex-ai-function-studio + document-intelligence
  /sis    → developing-with-streamlit-in-snowflake, deploy-to-spcs / snowflake-apps
  lint    → skill-development
```
