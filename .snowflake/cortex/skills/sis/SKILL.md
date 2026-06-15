---
name: sis
description: "Streamlit-in-Snowflake container-runtime deltas for this app + the MANDATORY pre-deploy scan. Use when writing app code that touches caching/widgets/session, or before any deploy. Triggers: SiS, streamlit in snowflake, get_active_session, cache_data, clear_caches, ttl, widget reset, showErrorDetails, config.toml, pre-deploy, scan, before deploying, deploy checklist. Do NOT use for general Streamlit authoring (bundled developing-with-streamlit-in-snowflake), app screen/state contracts (quiz-screens), or visual styling (quiz-design)."
---

> **Thin wrapper.** For general Streamlit authoring (widgets, layout, caching, theming) use the bundled CoCo skill **`developing-with-streamlit-in-snowflake`**; for deploy mechanics, **`deploy-to-spcs`** / **`snowflake-apps`**. This skill keeps only the project deltas for the **container runtime** (`SYSTEM$ST_CONTAINER_RUNTIME_PY3_11`, Streamlit ≥1.50, Python 3.11) plus the mandatory pre-deploy scan. The warehouse fallback (Path C) caps Streamlit at 1.52.2 — the gotchas still apply, but PyPI-only packages do not.

# When to Use

- Writing or reviewing any SiS app code (`main.py`, `_*.py`, `pages/*.py`) — apply the gotchas below.
- **Before every deploy** — run the pre-deploy scan; deploy only on a clean scan.

# When NOT to Use

- App screen flow / state / write-back → `$quiz/screens`
- Question generation → `$quiz/questions`
- Visual styling, theme, charts → `$quiz/design`
- Cortex AI call or prompt issues → `$cortex`

---

# Container-runtime gotchas

The SiS-specific traps that bite — what CoCo's general Streamlit knowledge doesn't cover. Everything else, defer to the bundled skill.

## Sessions and caching

`get_active_session()` (from `snowflake.snowpark.context`) is the single owner-rights session. Module-level is fine for DML / AI_COMPLETE (`_cortex.py`, `_data.py`); **inside an `@st.cache_data` function you MUST call it again in the body** — the module-level object isn't available in cached context (runtime error otherwise).

```python
session = get_active_session()            # module-level: DML / AI_COMPLETE

@st.cache_data(show_spinner=False)
def load_domains():
    _session = get_active_session()       # required inside cached functions
    return _session.sql("SELECT ...").collect()
```

**No `ttl` on any loader.** A ttl that expires mid-session silently swaps a stateful widget's input DataFrame and resets the widget (selections vanish, filters jump) — a miserable, time-dependent bug. Cache for the whole session and invalidate explicitly: `_data.py` defines `clear_caches()` (clears every loader) and **every DB write calls it** before the UI reads again.

```python
def clear_caches():
    load_domains.clear(); load_session_stats.clear()
    load_recent_sessions.clear(); load_domain_errors.clear(); load_review_log.clear()
```

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

**Don't set both `default=` and a `session_state` value** for one widget — initialize the key in `init_session_state()` and set `st.session_state["key"]` directly (passing both raises "created with a default value but also had its value set").

**Guard `None`:** widgets can return `None` during rerun cycles — always use an explicit `key=`, read the default from `session_state`, and fall back if `None`:
```python
difficulty = st.pills("Difficulty", options=[...], default=st.session_state["difficulty"], key="difficulty_pills")
if difficulty is None:
    difficulty = st.session_state["difficulty"]
```

## Rerun discipline

No fixed `st.rerun()` budget on the container runtime. Per handler: a button doing slow work (DB write, AI call) wraps it in `st.spinner()`, sets new state, ends with a **single** `st.rerun()` — never mid-handler, never twice. `st.experimental_rerun()` is deprecated; always `st.rerun()`. `@st.fragment` is supported and preferred for self-contained interactive regions (reruns only the fragment), but it doesn't fix state bugs — inputs feeding stateful widgets must still be constant across reruns (see caching).

## Multipage state

`st.session_state` is scoped to the browser session, so it **persists across `st.navigation` page switches** — all keys are initialized once in `main.py` (`init_session_state()`); pages share them. A filter set on one page is still set when the user returns; clear stale state explicitly if that surprises the flow. Cross-page redirect: set the target state, then `st.switch_page("pages/quiz.py")`.

## Still-constrained on the container runtime

- **No `unsafe_allow_html`.** Inline HTML/CSS is blocked by platform CSP (no external `<script src>`, no dynamic eval, no external iframes). Visual styling lives in `.streamlit/config.toml` (`$quiz/design`) and native components.
- **`.applymap(`** — removed in pandas 3.0; use `.map(` / `.map_index(`.
- **`config.toml`:** `showErrorDetails = "none"` — the string `"none"`, **not** `false` (deprecated `false` maps to "stacktrace" and still leaks tracebacks to viewers).
- **Uppercase columns:** Snowflake returns UPPERCASE column names; normalize every `.as_dict()`: `{k.upper(): v for k, v in row.as_dict().items()}` (lowercase keys return `None` silently).

## SQL safety

Only `DATABASE`, `SCHEMA`, `CORTEX_MODEL`, and `RESPONSE_FORMATS` constants may be f-string-interpolated into SQL. Every user-derived value (`domain_id`, difficulty, dates, Admin input) uses bind params (`:1, :2, …`).

---

# Pre-deploy scan

**MANDATORY before every deploy** (Workspaces Deploy or stage upload) and after any code change. Read ALL app files in full — `main.py`, every `_*.py`, every `pages/*.py`, `.streamlit/config.toml` — then check each item across the whole project. Report PASS/FAIL per item; on FAIL show the file, line, and offending snippet. **Deploy only when every item passes.**

### SQL and data safety
1. **SQL injection** — every `session.sql(f"...")`: only the four constants above in f-strings; all runtime values via bind params (see SQL safety).
2. **Parameterized INSERT** — `INSERT INTO ... VALUES (:1, :2, …)` with a params list; no value interpolation inside `VALUES (`.
3. **No `PARSE_JSON` inside `VALUES (`** — use bind params instead.
4. **`SELECT DISTINCT` + `IS NOT NULL`** — every `SELECT DISTINCT` filters out NULLs.

### Cortex / AI_COMPLETE  (see `$cortex`)
5. **Dollar-quoting** — `$${safe_prompt}$$`, not single quotes (apostrophes break it).
6. **`$$` sanitization** — `safe_prompt = prompt.replace("$$", "$ $")` before interpolation.
7. **Structured output** — every JSON call goes through `call_cortex_json` with a `RESPONSE_FORMATS` schema; the only `json.loads` on AI output is the single guard inside that helper. No fence-stripping.
8. **No `from snowflake.cortex import complete`** — call `AI_COMPLETE` via `session.sql()`.

### Streamlit compatibility  (see gotchas above)
9. **No `st.experimental_rerun()`** — use `st.rerun()`.
10. **No `.applymap(`** — use `.map(`.
11. **No `unsafe_allow_html`** — styling is in `config.toml` + native components.

### Session, cache, config
12. **`get_active_session()` inside every `@st.cache_data`** — not the module-level session.
13. **No `ttl` on loaders + `clear_caches()`** defined and called after every INSERT/UPDATE.
14. **`st.set_page_config(layout="centered", …)`** — first `st.` call in `main.py`, and ONLY there (never `layout="wide"`, never in a page/module).
15. **`.streamlit/config.toml`** — `[client] showErrorDetails = "none"` (string, not `false`) and `toolbarMode = "minimal"` present.
16. **Screen transitions** — every slow handler (Start Round, Submit, Finish, Next) wraps work in `st.spinner()` and ends with a single `st.rerun()`.

### Date handling  (see `$quiz/screens` Review)
17. **Date cast from `.collect()`** — Snowflake datetimes cast to `datetime.date(raw.year, raw.month, raw.day)` before a widget or arithmetic.
18. **Date range query** — dates passed as `strftime("%Y-%m-%d")` strings; exclusive upper bound (`< end + 1 day`).
19. **No `st.slider` with a `datetime.date`** min/max — use `st.date_input`.

### Column names
20. **Column normalization** — every `.as_dict()` result uppercased before access.

### Untrusted input
21. **Admin/flag inputs hardened** — all writes bind-param'd; form values length-capped (question 2000, options 500, comment 500); `correct_answer` ⊆ non-empty options; any stored/user-editable text embedded in a prompt is wrapped in data delimiters (`$cortex` — untrusted content).
22. **Doc-grounding (CKE) isolation + fallback** *(only if the app uses doc grounding; else N/A)* — all CKE access via `_search.py` (single caller), each call try/except→`[]`, every consumer has a non-grounded fallback, chunks delimited (`<doc_context>`), and **no `SEARCH_PREVIEW` in any app module** (runtime uses the Python `snowflake.core` API).

**Output:** a table, one row per item (# · item · PASS/FAIL/N·A · `file:line` snippet). Verdict — all pass → "Clean. Proceed to deploy."; any FAIL → "Fix items [list] before deploying," each with file+line and a one-line fix.

---

## Output

SiS-compatible code following the container-runtime gotchas, or a pre-deploy scan report with PASS/FAIL per item and a clear deploy / no-deploy verdict.
