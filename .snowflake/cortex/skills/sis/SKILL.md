---
name: sis
description: "Streamlit-in-Snowflake runtime deltas for this app (warehouse runtime) + the MANDATORY pre-deploy scan. Use when writing app code that touches caching/widgets/session, or before any deploy. Triggers: SiS, streamlit in snowflake, get_active_session, cache_data, clear_caches, ttl, widget reset, showErrorDetails, config.toml, pre-deploy, scan, before deploying, deploy checklist. Do NOT use for general Streamlit authoring (bundled developing-with-streamlit-in-snowflake), app screen/state contracts (quiz-screens), or visual styling (quiz-design)."
---

> **Thin wrapper.** For general Streamlit authoring (widgets, layout, caching, theming) use the bundled CoCo skill **`developing-with-streamlit-in-snowflake`**; for deploy mechanics, **`snowflake-apps`**. This skill keeps only the project deltas for **Streamlit-in-Snowflake** plus the mandatory pre-deploy scan. The **`warehouse` runtime** pins a supported Streamlit (currently ~1.52.2) and installs deps from the Snowflake Anaconda channel (`environment.yml`).

# When to Use

- Writing or reviewing any SiS app code (`main.py`, `_*.py`, `pages/*.py`) - apply the gotchas below.
- **Before every deploy** - run the pre-deploy scan; deploy only on a clean scan.

# When NOT to Use

- App screen flow / state / write-back → `$quiz/screens`
- Question generation → `$quiz/questions`
- Visual styling, theme, charts → `$quiz/design`
- Cortex AI call or prompt issues → `$cortex`

---

# SiS runtime gotchas

The SiS-specific traps that bite. Everything else, defer to the bundled skill.

## Sessions and caching

`get_active_session()` (from `snowflake.snowpark.context`) is the single owner-rights session. Module-level is fine for DML / AI_COMPLETE (`_cortex.py`, `_data.py`); **inside an `@st.cache_data` function you MUST call it again in the body** - the module-level object isn't available in cached context (runtime error otherwise).

```python
session = get_active_session()            # module-level: DML / AI_COMPLETE

@st.cache_data(show_spinner=False)
def load_domains():
    _session = get_active_session()       # required inside cached functions
    return _session.sql("SELECT ...").collect()
```

**No `ttl` on any loader.** A ttl that expires mid-session silently swaps a stateful widget's input DataFrame and resets the widget (selections vanish, filters jump). Cache for the whole session and invalidate explicitly: `_data.py` defines `clear_caches()` (clears every loader) and **every DB write calls it** before the UI reads again.

```python
def clear_caches():                       # must clear EVERY @st.cache_data loader the app defines
    load_domains.clear(); load_session_stats.clear(); load_config.clear()
    load_recent_sessions.clear(); load_domain_errors.clear(); load_review_log.clear()
    load_session_log.clear(); load_bank_stats.clear()           # Admin Logs + Questions-manager KPIs
    load_questions_page.clear(); load_cortex_spend.clear()      # Admin Questions table + Cortex-spend
    load_review_log_editable.clear()                            # Admin editable review-log table
    from _search import docs_available, search_docs   # function-local: _search imports _data, so a
    docs_available.clear(); search_docs.clear()        # module-level import here would be circular
```
The list must stay in sync with the loaders actually defined - every `.clear()` here names a real `@st.cache_data` function (`$sis` pre-deploy scan item 24), the docs caches in `_search.py` included (`$cortex`, `$quiz/screens` Admin), and every loader added is added here (no ttl, so this is the only freshness mechanism).

Cache keys carry real variability as plain hashable args; heavy/unhashable args (session, SQL text, params) are `_`-prefixed so the hasher ignores them.

## Widget lifecycle

**You cannot mutate a widget's `session_state` key after the widget is instantiated in the current run** (`StreamlitAPIException`). Two legal resets:

- **Flag-at-top** (when the work between click and reset is a spinner-wrapped DB call): queue the keys on success, rerun, pop them at the very top of the next run before the widgets render.
  ```python
  for k in st.session_state.pop("_op_clear_keys", []):   # top of page, before widgets
      st.session_state.pop(k, None)
  # ... success handler: st.session_state["_op_clear_keys"] = [...]; st.rerun()
  ```
- **`on_click` callback** (when the reset is cheap): callbacks run before the rerun, so mutating the key there is legal.

**Don't set both `default=` and a `session_state` value** for one widget - initialize the key in `init_session_state()` and set `st.session_state["key"]` directly (passing both raises "created with a default value but also had its value set").

**Guard `None`:** widgets can return `None` during rerun cycles - always use an explicit `key=`, read the default from `session_state`, and fall back if `None`:
```python
difficulty = st.pills("Difficulty", options=[...], default=st.session_state["difficulty"], key="difficulty_pills")
if difficulty is None:
    difficulty = st.session_state["difficulty"]
```

## Rerun discipline

No fixed `st.rerun()` budget. Per handler: a button doing slow work (DB write, AI call) wraps it in `st.spinner()`, sets new state, ends with a **single** `st.rerun()` - never mid-handler, never twice. `st.experimental_rerun()` is deprecated; always `st.rerun()`. `@st.fragment` is supported and preferred for self-contained interactive regions (reruns only the fragment), but it doesn't fix state bugs - inputs feeding stateful widgets must still be constant across reruns (see caching).

## Multipage state

`st.session_state` is scoped to the browser session, so it **persists across `st.navigation` page switches** - all keys are initialized once in `main.py` (`init_session_state()`); pages share them. A filter set on one page is still set when the user returns; clear stale state explicitly if that surprises the flow. Cross-page redirect: set the target state, then `st.switch_page("pages/quiz.py")`.

**`st.navigation` + a `pages/` dir → set `[client] showSidebarNavigation = false`.** Without it, the pinned SiS Streamlit can render its **native** page navigation (the lowercase `pages/` filenames) in the sidebar *alongside* the `st.navigation` menu. Setting `showSidebarNavigation = false` (config.toml) leaves only the explicit `st.navigation` titles. The shared sidebar caption (`EXAM_NAME`) is rendered above the nav in `main.py`.

## Still-constrained on SiS

- **No `unsafe_allow_html`.** Inline HTML/CSS is blocked by platform CSP (no external `<script src>`, no dynamic eval, no external iframes). Visual styling lives in `.streamlit/config.toml` (`$quiz/design`) and native components.
- **`.applymap(`** - removed in pandas 3.0; use `.map(` / `.map_index(`.
- **`config.toml`:** `showErrorDetails = "none"` - the string `"none"`, **not** `false` (deprecated `false` maps to "stacktrace" and still leaks tracebacks to viewers); and `[client] showSidebarNavigation = false` when using `st.navigation` + a `pages/` dir (see Multipage state).
- **Uppercase columns:** Snowflake returns UPPERCASE column names; normalize every `.as_dict()`: `{k.upper(): v for k, v in row.as_dict().items()}` (lowercase keys return `None` silently).

## SQL safety

Only `DATABASE`, `SCHEMA`, `CORTEX_MODEL`, and `RESPONSE_FORMATS` constants may be f-string-interpolated into SQL. Every user-derived value (`domain_id`, difficulty, dates, Admin input) uses bind params.

**Bind-param style is `?` (qmark), never `:1`.** Snowpark's `session.sql(sql, params=[...])` uses positional `?` placeholders bound left-to-right from the `params` list - `session.sql("... WHERE domain_id = ? AND difficulty = ?", params=[d, diff])`. The `:1, :2` numeric style does NOT work through `session.sql(params=…)` and silently fails or errors at runtime. (`:1` positional binds apply to server-side `EXECUTE IMMEDIATE … USING`; `:name` is a connector-level style - this app uses neither.)

---

# Pre-deploy scan

**MANDATORY before every deploy** (`COPY FILES` to `STAGE_SIS_APP` + `CREATE STREAMLIT`) and after any code change. **Re-read ALL app files from disk in this turn** - `main.py`, every `_*.py`, every `pages/*.py`, `.streamlit/config.toml` - then check each item across the whole project. **Never scan from memory or from a summary of the files** - a remembered scan certifies code you didn't actually look at, which can't catch a `NameError`/`ImportError`. **Writing a file earlier in this session is NOT reading it** - generation and verification are independent passes; a file you just wrote, or one trimmed/edited out of context, MUST be re-read from disk, never reconstructed from memory. You may emit **no** PASS/FAIL verdict, and deploy **nothing**, until you hold a fresh read of **every** app file THIS turn - a partial set of reads topped up from memory is **not a scan**. **A "consecutive tool calls / possible loop" warning while you re-read the files is an EXPECTED false positive** - reading all N app files in a row is required and is not a loop; do not let it stop you short. Every PASS/FAIL row must cite a real `file:line` you read this turn; on FAIL show the file, line, and offending snippet. **Deploy only when every item passes.**

**If you cannot re-read every app file from disk THIS turn** - file-not-found, `I/O` error, a read-only/degraded session, a broken workspace mount, expired auth - then you **cannot run this scan**: **STOP, do not certify, do not deploy**, and report the access failure plus its remediation (re-open the workspace in a healthy interactive session as the configured role). A scan you can't ground in a fresh read is not a scan - reconstructing it from memory or a prior turn's reads certifies code you cannot see.

**The scan never introduces symbols.** If it edits any code (e.g. fixing a flagged item), re-read the changed file and re-run the affected items - never add a function/import/`.clear()` reference to a name that isn't defined in the source.

**Resolve an unresolved name by REMOVING the dangling reference, never by authoring a definition to satisfy it.** When item 24 finds a reference to an undefined name, the fix is to delete the reference (the stray `.clear()` line, the bad import), NOT to invent a definition so it resolves - that passes the scan and byte-compiles while keeping a symbol the design never had. **If the scan's set of defined symbols grows, the scan has failed** - verify the symbol genuinely belongs to the architecture (`$quiz` module map) before keeping any new definition.

### SQL and data safety
1. **SQL injection** - every `session.sql(f"...")`: only the four constants above in f-strings; all runtime values via bind params (see SQL safety).
2. **Parameterized INSERT, `?` style** - `INSERT INTO ... VALUES (?, ?, …)` with a params list; no value interpolation inside `VALUES (`. Flag any `:1`/`:2`/`:name` placeholder - Snowpark `session.sql(params=…)` is `?`-only (see SQL safety).
3. **No `PARSE_JSON` inside `VALUES (`** - use bind params instead. BUT `PARSE_JSON(?)` in a MERGE/SELECT **`USING`** sub-select is correct and **required** for config writes (`save_config()`, `$quiz/screens`): config writes use `PARSE_JSON(?)` with `params=[json.dumps(value)]`, **never `TO_VARIANT(json.dumps(...))`** (double-encodes → read returns `'"ai"'` → `options.index()` crashes). Also flag any config-seeded widget default doing a bare `options.index(cfg[key])` instead of the `cfg_index()` guard.
4. **`SELECT DISTINCT` + `IS NOT NULL`** - every `SELECT DISTINCT` filters out NULLs.

### Cortex / AI_COMPLETE  (see `$cortex`)
5. **Dollar-quoting** - `$${safe_prompt}$$`, not single quotes (apostrophes break it).
6. **`$$` sanitization** - `safe_prompt = prompt.replace("$$", "$ $")` before interpolation.
7. **Structured output** - every JSON call goes through `call_cortex_json` with a `RESPONSE_FORMATS` schema; the only `json.loads` on AI output is the single guard inside that helper. No fence-stripping.
8. **No `from snowflake.cortex import complete`** - call `AI_COMPLETE` via `session.sql()`.

### Streamlit compatibility  (see gotchas above)
9. **No `st.experimental_rerun()`** - use `st.rerun()`.
10. **No `.applymap(`** - use `.map(`.
11. **No `unsafe_allow_html`** - styling is in `config.toml` + native components.

### Session, cache, config
12. **`get_active_session()` inside every `@st.cache_data`** - not the module-level session.
13. **No `ttl` on loaders + `clear_caches()`** defined and called after every INSERT/UPDATE.
14. **`st.set_page_config(layout="centered", …)`** - first `st.` call in `main.py`, and ONLY there (never `layout="wide"`, never in a page/module).
15. **`.streamlit/config.toml`** - `[client] showErrorDetails = "none"` (string, not `false`), `toolbarMode = "minimal"`, and `showSidebarNavigation = false` (suppresses the native lowercase page nav - see Multipage state) all present.
16. **Screen transitions** - every slow handler (Start Round, Submit, Finish, Next) wraps work in `st.spinner()` and ends with a single `st.rerun()`.

### Date handling  (see `$quiz/screens` Review)
17. **Date cast from `.collect()`** - Snowflake datetimes cast to `datetime.date(raw.year, raw.month, raw.day)` before a widget or arithmetic.
18. **Date range query** - dates passed as `strftime("%Y-%m-%d")` strings; exclusive upper bound (`< end + 1 day`).
19. **No `st.slider` with a `datetime.date`** min/max - use `st.date_input`.

### Column names
20. **Column normalization + no raw-`Row` access** - every `.as_dict()` result uppercased before access (`{k.upper(): v for k, v in row.as_dict().items()}`); and **never `.get()` or attribute-access a Snowpark `Row`** (`row.get("X")` / `row.X` raise `AttributeError: Row object has no attribute …`). Read a `Row` only via `row["UPPER_COL"]` or the uppercased dict. Flag every `.get(`/attr on a value that came from `.collect()` / a dataframe-selection row (e.g. `st.dataframe(..., on_select=…)` selections → convert to an uppercased dict first). **Loader column/alias agreement:** every key a consumer reads from a loader's row (`row["KEY"]` or an uppercased-dict key) must match a column/alias that loader's own `SELECT` actually produces, **case-exact** - a loader aliasing `… AS total_credits` (→ `TOTAL_CREDITS`) but read as `row["CREDITS"]`, or one returning UPPERCASE keys but read as `stats["sessions"]`, raises `KeyError`/silent `None` at runtime. Cross-check each loader's `SELECT` alias list against every key its callers read (the `sessions`/`CREDITS` class; an imported constant that no module defines - e.g. `from _config import SCORE_COLOR` - is caught by item 24).

### Untrusted input
21. **Admin/flag inputs hardened** - all writes bind-param'd; form values length-capped (question 2000, options 500, comment 500); `correct_answer` ⊆ non-empty options; any stored/user-editable text embedded in a prompt is wrapped in data delimiters (`$cortex` - untrusted content).
22. **Doc-grounding (CKE) isolation + mandate** *(N/A only in `grounding_mode = none`)* - all CKE access via `_search.py` (single caller), each call try/except→`[]`; in `cke`/`custom` mode consumers do NOT fall back to built-in knowledge - on empty retrieval they broaden once then **fail visibly** (return `None`); ungrounded generation exists ONLY in `none` mode; chunks delimited (`<doc_context>`); and **no `SEARCH_PREVIEW` in any app module** (runtime uses the Python `snowflake.core` API, from the `snowflake` package).

### Rendering
23. **No raw `$` in rendered dynamic text** - model/DB strings (question, options, explanation, deep-dive, mnemonics, summary + review cards) pass through the `_ui.py` `md()` escaper before `st.markdown`/`st.write`/`st.info`; an unescaped `$…$` renders as LaTeX in Streamlit. Flag any dynamic text rendered without `md()`.

### Static resolution (catches the `NameError`/`ImportError` a static read must find - no execution needed)
24. **Name & import resolution** - every referenced name resolves to a definition. Build the set of defined names per module, then confirm **every** cross-module reference is in it - in all forms: each `from _x import (a, b, …)` name is defined in `_x.py`; each `<loader>.clear()` in `clear_caches()` names an `@st.cache_data` function defined in `_data.py`; each `module.attr` / `module.func()` access resolves to a definition in that module; no call to a helper that exists in no module. (A loader named in `clear_caches()` but never defined in `_data.py` fails here.)
25. **Write-once side effects** - every DB-writing handler reachable from a button (`_write_back_results()`, Admin INSERT/UPDATE/DELETE) is guarded against duplicate execution: a single-write flag (or delete-before-insert keyed by the round/row) so a double-click or a post-write exception cannot insert the same rows twice; and `clear_caches()` is the **last** statement of the write path and cannot raise (else the writes commit but the handler crashes and the user re-clicks → duplicates). See `$quiz/screens` Write-Back Contract.

**Output:** a table, one row per item (# · item · PASS/FAIL/N·A · `file:line` snippet). Verdict - all pass → "Clean. Proceed to deploy."; any FAIL → "Fix items [list] before deploying," each with file+line and a one-line fix.

**A clean scan is necessary but NOT sufficient.** This scan checks that the app *runs* and is *SQL-safe* - it deliberately does **not** check screen/UX-contract conformance (that's `$quiz/screens`, out of this skill's scope). An app can pass every item here while the Home has no slider, answers show bare letters, the config crashes, and the Questions manager has no table. **Before deploy, also run the `$quiz/screens` UX-conformance gate** - both must be clean. (A clean scan alone is not sufficient.)

---

## Output

SiS-compatible code following the runtime gotchas, or a pre-deploy scan report with PASS/FAIL per item and a clear deploy / no-deploy verdict.
