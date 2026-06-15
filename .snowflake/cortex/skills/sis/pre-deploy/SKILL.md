---
name: sis-pre-deploy
description: "MANDATORY 22-item pre-deploy scan for the Streamlit-in-Snowflake app (container runtime). Run before EVERY deploy — catches SQL injection, runtime errors, cache and config pitfalls, untrusted-input handling, doc-grounding isolation. Triggers: deploy, pre-deploy, scan, before deploying, push to snowflake, deploy checklist. Do NOT use for writing SiS code (sis-patterns) or Cortex call issues (cortex-patterns)."
---

# When to Load

Parent skill `$sis` routes here for PRE-DEPLOY intent.

- **MANDATORY** before every deploy of the app (Workspaces Deploy or stage upload)
- After any code changes, before re-deploying
- When reviewing generated code for SiS compatibility

# When NOT to Use

- Coding patterns (session, cache, widgets) -> use `$sis/patterns`
- UI styling/badges -> use `$quiz/style`
- Cortex AI issues -> use `$cortex/patterns`

---

# Pre-Deploy Scan

Read ALL app files in full: `main.py`, every `_*.py` module, every `pages/*.py`, and `.streamlit/config.toml`. Then check each of the 22 items below across the whole project. For each item report PASS or FAIL. On FAIL: show the file, line number, and the offending code snippet.

## Scan Items

### SQL and data safety

**1. SQL injection risk**
Find every `session.sql(f"...")` call in every module. Only `DATABASE`, `SCHEMA`, `CORTEX_MODEL`, and `RESPONSE_FORMATS` constants are allowed in f-strings. Any runtime variable (`domain_id`, `difficulty`, dates, user input) must use bind params `:1, :2, …`.
- PASS: user-derived values use bind params
- FAIL: any runtime variable interpolated directly into f-string SQL

**2. Parameterized INSERT**
All `INSERT INTO` statements must use `VALUES (:1, :2, …)` with a params list.
- PASS: bind params used
- FAIL: f-string interpolation of values inside `VALUES (`

**3. PARSE_JSON inside VALUES**
`PARSE_JSON(` must not appear inside a `VALUES (` clause.
- PASS: 0 occurrences
- FAIL: any occurrence (use bind params instead)

**4. SELECT DISTINCT without IS NOT NULL**
Every `SELECT DISTINCT` query must also filter `WHERE column IS NOT NULL`.
- PASS: IS NOT NULL filter present on every SELECT DISTINCT
- FAIL: SELECT DISTINCT without NULL exclusion

### Cortex / AI_COMPLETE

**5. AI_COMPLETE dollar-quoting**
`call_cortex` / `call_cortex_json` must use `$$...$$` quoting, not single-quote `'...'`.
- PASS: `$${safe_prompt}$$` pattern used
- FAIL: single-quote quoting - breaks on apostrophes in question text

**6. `$$` sanitization**
Before interpolating a prompt into `$$...$$`, the code must call `.replace("$$", "$ $")`.
- PASS: `safe_prompt = prompt.replace("$$", "$ $")` present
- FAIL: missing - a `$$` in question text will break the SQL query

**7. Structured output for AI JSON**
Every JSON-expecting AI call goes through `call_cortex_json` with a `response_format` schema from `RESPONSE_FORMATS`. No markdown-fence stripping, no fence-aware parsing, no ad-hoc `json.loads` on free-text completions.
- PASS: all JSON calls use `response_format`; the only `json.loads` on AI output is the single guard inside `call_cortex_json`
- FAIL: prose-only "return JSON" prompt, fence-stripping code, or `json.loads` on a `call_cortex` (free-text) result

**8. `from snowflake.cortex import complete`**
Must not appear. Use `AI_COMPLETE` via `session.sql()` only.
- PASS: 0 occurrences
- FAIL: any occurrence

### Streamlit compatibility

**9. `st.experimental_rerun()`**
Must not appear. Use `st.rerun()`.
- PASS: 0 occurrences
- FAIL: any occurrence (deprecated)

**10. `.applymap(`**
Must not appear.
- PASS: 0 occurrences
- FAIL: any occurrence (removed in pandas 3.0, use `.map(` instead)

**11. `unsafe_allow_html`**
Must not appear anywhere in the app. Theme and styling live in `.streamlit/config.toml` and native components (badges, containers).
- PASS: 0 occurrences
- FAIL: any occurrence

### Session, cache, and config

**12. `get_active_session()` inside `@st.cache_data`**
Every `@st.cache_data` function must call `get_active_session()` inside its own body. Reusing the module-level session in a cached context causes a runtime error.
- PASS: each cached function calls get_active_session() internally
- FAIL: any cached function uses a module-level session variable

**13. Cache discipline: no `ttl`, explicit invalidation**
No `@st.cache_data` loader may set a `ttl` (mid-session expiry resets stateful widgets and silently staleness-flips data). `_data.py` must define `clear_caches()` clearing every loader, and every DB-write path (round write-back, feature writes) must call it.
- PASS: 0 `ttl=` on loaders; `clear_caches()` defined and called after every INSERT/UPDATE
- FAIL: any `ttl=` on a loader, or a write path that does not invalidate

**14. `st.set_page_config` position**
`st.set_page_config(layout="centered", ...)` must be the very first `st.` call in `main.py`, and must appear ONLY in `main.py` (never in pages or modules).
- PASS: first `st.` call in `main.py`; 0 occurrences elsewhere
- FAIL: any `st.` call before it, `layout="wide"`, or a second occurrence in a page

**15. `.streamlit/config.toml` settings**
`[client] showErrorDetails = "none"` (the string — NOT `false`, which still leaks tracebacks) and `toolbarMode = "minimal"` must be present.
- PASS: both set as specified
- FAIL: missing file, `showErrorDetails = false`, or `"full"` left in from debugging

**16. Screen transitions**
Button handlers that do slow work (loading questions, writing to DB) MUST wrap the work in `st.spinner()` and end with a single `st.rerun()` after setting state - otherwise stale widgets render alongside the new screen.
- PASS: Start Round, Submit, Finish, Next all use the `st.spinner()` + single `st.rerun()` pattern
- FAIL: a slow handler sets state without a final `st.rerun()`, or calls `st.rerun()` more than once

### Date handling

**17. Date from `.collect()` without cast**
Any timestamp value from `.collect()` passed to `st.date_input` or date arithmetic must be cast: `datetime.date(raw.year, raw.month, raw.day)`.
- PASS: all collect() dates are cast before use
- FAIL: raw Snowflake datetime object passed directly to a widget

**18. Date range query pattern**
Comparisons against `TIMESTAMP_LTZ`/`TIMESTAMP_NTZ`: dates must be passed as formatted strings (`strftime("%Y-%m-%d")`); end date must use exclusive upper bound (`< end + 1 day`).
- PASS: pattern followed
- FAIL: date object passed directly or inclusive end bound used

**19. `st.slider` with date variable**
Must not receive a `datetime.date` as `min_value` / `max_value`. Use `st.date_input` for date ranges.
- PASS: 0 violations
- FAIL: any st.slider call with a date-typed min/max

### Column names

**20. Column name normalization**
All `.as_dict()` results must be normalized: `{k.upper(): v for k, v in row.as_dict().items()}`. Snowflake returns uppercase column names; accessing them with lowercase keys returns `None`.
- PASS: normalization applied to every as_dict() call
- FAIL: any as_dict() result accessed without uppercasing keys

### Untrusted input hardening

**21. Admin/flag inputs hardened**
All Admin-page and flag writes (question edits, new questions, config saves, flags) use bind params; form values are length-capped before write (question 2000, options 500, comment 500); `correct_answer` is validated against non-empty options; any stored/user-editable text embedded in a prompt is wrapped in data delimiters per `$cortex/patterns` ("Untrusted content inside prompts").
- PASS: all four conditions hold across `pages/admin.py` and every prompt that embeds bank questions
- FAIL: any f-string write, uncapped input, unvalidated answer key, or undelimited stored text in a prompt

**22. Doc-grounding (CKE) isolation + fallback** (only if the app uses doc grounding)
All Cortex Search / CKE access goes through `_search.py` (the single caller); every call is wrapped try/except returning `[]`, and every consumer (generation, explanation) has a non-grounded fallback branch; retrieved chunks are delimited (`<doc_context>`); and **no `SNOWFLAKE.CORTEX.SEARCH_PREVIEW` appears in any app module** (runtime uses the Python `snowflake.core` API).
- PASS: single caller, try/except + fallback everywhere, chunks delimited, no SEARCH_PREVIEW in app code
- FAIL: a direct CKE call outside `_search.py`, a missing fallback branch, undelimited chunks, or `SEARCH_PREVIEW` in an app module
- N/A: app does not use doc grounding

---

## Output

After checking all 22 items, a summary table:

| # | Item | Status | Notes |
|---|------|--------|-------|
| 1 | SQL injection | PASS/FAIL | file:line `snippet` |
| 2 | Parameterized INSERT | PASS/FAIL | |
| 3 | PARSE_JSON in VALUES | PASS/FAIL | |
| 4 | SELECT DISTINCT NULL | PASS/FAIL | |
| 5 | Dollar-quoting | PASS/FAIL | |
| 6 | `$$` sanitization | PASS/FAIL | |
| 7 | Structured output | PASS/FAIL | |
| 8 | cortex import | PASS/FAIL | |
| 9 | experimental_rerun | PASS/FAIL | |
| 10 | applymap | PASS/FAIL | |
| 11 | unsafe_allow_html | PASS/FAIL | |
| 12 | get_active_session in cache | PASS/FAIL | |
| 13 | cache ttl + clear_caches | PASS/FAIL | |
| 14 | set_page_config position | PASS/FAIL | |
| 15 | config.toml settings | PASS/FAIL | |
| 16 | screen transitions | PASS/FAIL | |
| 17 | date cast from collect() | PASS/FAIL | |
| 18 | date range query pattern | PASS/FAIL | |
| 19 | slider date | PASS/FAIL | |
| 20 | column name normalization | PASS/FAIL | |
| 21 | admin/flag inputs hardened | PASS/FAIL | |
| 22 | doc-grounding (CKE) isolation + fallback | PASS/FAIL/N/A | |

**Final verdict:**
- All 22 PASS -> "Clean. Proceed to deploy."
- Any FAIL -> "Fix items [list] before deploying."

For each FAIL item: show the exact file + line number and a 1-line fix suggestion.
