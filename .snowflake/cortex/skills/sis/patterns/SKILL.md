---
name: sis-patterns
description: "Streamlit-in-Snowflake coding patterns for the container runtime — session management, caching without ttl + explicit invalidation, widget lifecycle (flag-at-top reset), multipage state, date handling, still-constrained APIs. Use when writing any SiS app code. Triggers: SiS, streamlit in snowflake, get_active_session, cache_data, clear_caches, widget state, st.fragment, config.toml, date handling, container runtime"
parent_skill: sis
---

> **Thin wrapper.** For general Streamlit development patterns use the bundled CoCo skill **`developing-with-streamlit`**. This skill keeps only the project deltas for Streamlit-in-Snowflake on the **container runtime** (Streamlit ≥1.50, Python 3.11).

# When to Load

Parent skill `$sis` routes here for PATTERNS intent.

- Writing any Streamlit-in-Snowflake app code (`main.py`, `_*.py`, `pages/*.py`)
- Debugging runtime errors in SiS (session, widget state, cache staleness)
- Reviewing generated code for SiS compatibility
- Always consult alongside `$sis/pre-deploy` before deploying

# When NOT to Use

- UI styling/badges/colors -> use `$quiz/style`
- Cortex AI function issues -> use `$cortex/patterns`
- Pre-deploy scan checklist -> use `$sis/pre-deploy`

---

# Runtime Context

The app targets the **container runtime** (`SYSTEM$ST_CONTAINER_RUNTIME_PY3_11`): Streamlit ≥1.50 from PyPI (`pyproject.toml`), Python 3.11, compute pool. The warehouse fallback (Path C) caps Streamlit at 1.52.2 — the patterns below still work there, but PyPI-only packages do not.

---

# Session Management

Use `get_active_session()` from `snowflake.snowpark.context` as the single owner-rights session for ALL database work. (The container runtime also supports `st.connection("snowflake")`, but this project standardizes on `get_active_session()` — one session type, no config.)

**Inside `@st.cache_data` functions**: call `get_active_session()` inside the function body. The module-level session object is not available in cached context.

**Module-level** `session = get_active_session()` is valid for non-cached code (DML, AI_COMPLETE calls) — it lives in `_cortex.py` and `_data.py`.

```python
from snowflake.snowpark.context import get_active_session
session = get_active_session()   # module-level for DML/AI_COMPLETE

@st.cache_data(show_spinner=False)
def load_domains():
    _session = get_active_session()   # required inside cached function
    return _session.sql("SELECT ...").collect()
```

---

# Caching: no ttl, explicit invalidation

**Never put a `ttl` on a cached loader that feeds a stateful widget.** A ttl that expires mid-session silently swaps the widget's input DataFrame, which resets the widget (selections vanish, filters jump). The failure looks time-dependent and is miserable to debug.

Rules (all loaders live in `_data.py`):
- `@st.cache_data(show_spinner=False)` with **NO `ttl`** — the rowset stays stable for the whole session.
- The cache key carries the real variability as plain hashable args (filters, page size). Heavy/unhashable args (session, SQL text, params) are `_`-prefixed so the hasher ignores them.
- **Invalidate explicitly after every write** — define and call `clear_caches()`:

```python
def clear_caches():
    """Call after EVERY DB write so dashboards reflect the new data."""
    load_domains.clear()
    load_session_stats.clear()
    load_recent_sessions.clear()
    load_domain_errors.clear()
    load_review_log.clear()
```

The round write-back (`QUIZ_SESSION_LOG` + `QUIZ_REVIEW_LOG` INSERTs) must call `clear_caches()` before navigating to the summary — otherwise the Review dashboard shows stale numbers until a full session restart.

---

# Widget Lifecycle

**You cannot modify a widget's `session_state` key after the widget has been instantiated in the current run** — it raises `StreamlitAPIException`. Two legal reset patterns:

**Flag-at-top (use when the action between click and reset is a spinner-wrapped DB call):** queue the keys on success + rerun; pop them at the top of the next run, BEFORE the widgets are drawn:

```python
# top of the page, BEFORE any of these widgets render
for k in st.session_state.pop("_op_clear_keys", []):
    st.session_state.pop(k, None)

# ... later, in a success handler ...
st.session_state["_op_clear_keys"] = ["cb_A", "cb_B", "cb_C", "cb_D", "cb_E"]
st.rerun()
```

**`on_click` callback (use when the reset is cheap):** callbacks run before the rerun, so mutating the key there is legal:

```python
def clear_name():
    st.session_state["name"] = ""
st.button("Clear", on_click=clear_name)
```

**Programmatic navigation values:** initialize keys in `init_session_state()` and set `st.session_state["key"] = value` directly — do NOT also pass `default=` to the widget (setting both raises the "created with a default value but also had its value set" exception).

---

# Multipage State

`st.session_state` is scoped to the browser session, **not** the page — it persists across `st.navigation` page switches. All keys are initialized once in `main.py` (`init_session_state()`); pages read/write the same state. A filter set on one page is still set when the user returns — clear stale state explicitly if that surprises the flow.

Cross-page redirects (e.g. "Start Focused Session" on the recommendations page → quiz): set the state the target page needs, then `st.switch_page("pages/quiz.py")`.

---

# Scoped Reruns: @st.fragment

`@st.fragment` is supported on the container runtime and **preferred** for self-contained interactive regions (e.g. the answer/submit area): a widget interaction inside the fragment reruns only the fragment, not the whole page — kills the visible full-page flash and the cost of re-running everything above.

A fragment does not fix state bugs by itself — inputs feeding stateful widgets must still be constant across reruns (see Caching above).

---

# Rerun Discipline

No fixed `st.rerun()` budget on the container runtime. The rule is per-handler: a button handler that does slow work (DB write, AI call) wraps the work in `st.spinner()`, sets the new state, and ends with a **single** `st.rerun()`. Never rerun mid-handler, never rerun twice.

`st.experimental_rerun()` is deprecated — always `st.rerun()`.

---

# Still-Constrained on SiS

These remain true on the container runtime:

- **`unsafe_allow_html=True` / CSP** — inline HTML/CSS is fine, but the platform CSP blocks external `<script src>`, dynamic code evaluation, and external iframes. The app uses NO `unsafe_allow_html` (theme lives in `.streamlit/config.toml`; see `$quiz/style`).
- **`.applymap(`** — removed in pandas 3.0; use `.map(` / `.map_index(`.
- **`config.toml`:** `showErrorDetails = "none"` — the string `"none"`, NOT `false` (the deprecated `false` maps to "stacktrace" and still leaks tracebacks to viewers).

---

# Multi-Answer Checkboxes

Render each option as an independent `st.checkbox` with stable key (`cb_A`, `cb_B`, etc.). Read selected by checking session state after rendering. Disable all once answered.

On "Next": clear the `cb_*` keys via the flag-at-top pattern (queue `["cb_A", ..., "cb_E"]` in `_op_clear_keys`, rerun, pop at top).

---

# Date Handling

**From `.collect()`**: Snowflake returns datetime objects, not Python `datetime.date`. Convert:
```python
min_date = datetime.date(raw.year, raw.month, raw.day)
```

**Range queries** against `TIMESTAMP_LTZ` columns: pass dates as formatted strings (`"%Y-%m-%d"`). Use exclusive upper bound (`< end + 1 day`) to include the full last day.

---

# Column Name Normalization

Snowflake returns UPPERCASE column names. Always normalize:
```python
d = {k.upper(): v for k, v in row.as_dict().items()}
```

Accessing with lowercase keys returns `None` silently.

---

# Widget State Safety

All interactive widgets must:
1. Use explicit `key=` parameters
2. Read default values from `st.session_state` before rendering
3. Guard against `None` returns (which happen during rerun cycles)

```python
difficulty = st.pills("Difficulty", options=[...],
    default=st.session_state["difficulty"], key="difficulty_pills")
if difficulty is None:
    difficulty = st.session_state["difficulty"]
```

---

# Session State Reliability

Mutable objects stored in session_state may not survive in-place mutation (`.append()`, `.add()` are not change-detected).

**Rules:**
- Never use separate tracking lists/sets in session_state for dedup
- Use `round_history` as the single source of truth for "what has been shown"
- For DB dedup: collect texts from round_history, pass as `NOT IN` bind params
- For AI dedup: collect texts from round_history, pass as "DO NOT repeat" in prompt

---

# Button Click Safety

Use `_transitioning` flag to prevent duplicate button renders:
```python
if st.button("Next", disabled=st.session_state.get("_transitioning", False)):
    st.session_state["_transitioning"] = True
    # ... action ...
    st.session_state["_transitioning"] = False
    st.rerun()
```

---

# SQL Safety

Only `DATABASE`, `SCHEMA`, `CORTEX_MODEL`, and `RESPONSE_FORMATS` constants in f-string SQL. All user-derived values via bind params (`:1, :2, ...`).

---

## Output

SiS-compatible code following container-runtime session, caching, and widget-lifecycle rules.
