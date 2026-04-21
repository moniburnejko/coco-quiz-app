---
name: sis-patterns
description: "Streamlit in Snowflake (SiS v1.52.*) coding patterns — session management, rerun budget, unsupported APIs, date handling, widget state safety. Use when writing any SiS app code. Triggers: SiS, streamlit in snowflake, get_active_session, st.rerun, widget state, cache_data, rerun budget, unsupported API, date handling"
parent_skill: sis
---

# When to Load

Parent skill `$sis` routes here for PATTERNS intent.

- Writing any Streamlit in Snowflake app code
- Debugging runtime errors in SiS (session, rerun, widget state)
- Reviewing generated code for SiS compatibility
- Always consult alongside `$sis/pre-deploy` before deploying

# When NOT to Use

- UI styling/badges/colors -> use `$quiz/style`
- Cortex AI function issues -> use `$cortex/patterns`
- Pre-deploy scan checklist -> use `$sis/pre-deploy`

---

# Session Management

Always `get_active_session()` from `snowflake.snowpark.context`. Never `st.connection("snowflake")`.

**Inside `@st.cache_data` functions**: call `get_active_session()` inside the function body. The module-level session object is not available in cached context.

**Module-level** `session = get_active_session()` is valid for non-cached code (DML, AI_COMPLETE calls).

```python
from snowflake.snowpark.context import get_active_session
session = get_active_session()   # module-level for DML/AI_COMPLETE

@st.cache_data(ttl=300)
def load_domains():
    _session = get_active_session()   # required inside cached function
    return _session.sql(...).collect()
```

---

# Rerun Budget

`st.rerun()` is allowed in exactly **10 locations**. Adding extra reruns causes blank screen flashes; missing reruns cause duplicate buttons or stale UI.

1. **Start Round** — after loading first question with spinner
2. **Lazy load** — top of render_quiz, after question loaded
3. **Retry** — after clearing error state on AI failure screen
4. **Finish pending** — top of render_quiz, after writing results
5. **Submit Answer** — hide submit button, reveal answered state
6. **Next** — clear state for next question
7. **Start Focused Session** — navigate from AI recommendations to quiz
8. **Retry Same Config** — summary screen, reset state + start new round
9. **Configure New Round** — summary screen, go back to home
10. **AI Recommendations generated** — after saving recs to session state, hide the generate button

Every button handler that does slow work (loading questions, writing to DB) must: wrap in `st.spinner()`, then call `st.rerun()` after setting state.

---

# Unsupported APIs

These cause runtime errors in SiS v1.52.*:

- `@st.fragment` / `st.fragment()` — not supported
- `st.experimental_rerun()` — deprecated, use `st.rerun()`
- `st.container(horizontal=True)` — use `st.columns()` instead
- `.applymap()` — removed in pandas >= 2.1, use `.map()`
- `from snowflake.cortex import complete` — use `AI_COMPLETE` via `session.sql()`
- `st.connection("snowflake")` — use `get_active_session()`

---

# Multi-Answer Checkboxes

Render each option as an independent `st.checkbox` with stable key (`cb_A`, `cb_B`, etc.). Read selected by checking session state after rendering. Disable all once answered.

On "Next": delete all `cb_*` keys manually:
```python
for lt in ["A", "B", "C", "D", "E"]:
    st.session_state.pop(f"cb_{lt}", None)
```

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

## Programmatic Navigation (redirect pattern)

For widgets whose value is set programmatically (e.g. sidebar `nav_pills` set by a redirect handler): initialize the key in `init_session_state` and set `st.session_state["key"] = value` directly. Do NOT use the `default` param on the widget — using both causes:

> "The widget with key 'nav_pills' was created with a default value but also had its value set via the Session State API."

---

# Session State Reliability

SiS serializes session_state between reruns. Mutable objects stored directly may not survive — in-place mutations (`.append()`, `.add()`) are NOT detected by the serializer.

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

Only `DATABASE`, `SCHEMA`, and `CORTEX_MODEL` constants in f-string SQL. All user-derived values via bind params (`:1, :2, ...`).

---

## Output

SiS-compatible code following session management, rerun budget, and widget state rules.
