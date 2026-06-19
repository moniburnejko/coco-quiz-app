---
name: quiz-screens
description: "Quiz app behavioral contracts - page flow (st.navigation), screen state machine, history tracking, write-back + cache invalidation, session state, explanation contract, dashboard. Use when building or modifying any page or shared UI module of the app. Triggers: screen flow, page flow, quiz screen, home screen, summary, review page, session state, write-back, explanation, history_item, st.navigation. Do NOT use for question generation (quiz-questions) or styling (quiz-design)."
---

# When to Load

Parent skill `$quiz` routes here for SCREENS intent.

- Building or modifying any page (`pages/*.py`), the entry point (`main.py`), or shared render helpers (`_ui.py`)
- Adding a new feature to the quiz flow
- Debugging page/screen transitions or state issues
- Understanding data flow between pages

# When NOT to Use

- Cortex AI issues -> use `$cortex`
- SiS rendering patterns -> use `$sis`
- Prompt quality audit -> use `$cortex`
- UI styling/badges -> use `$quiz/design`
- Question generation logic -> use `$quiz/questions`

---

# Page Flow

Navigation is native multipage (`st.Page` + `st.navigation`), built in `main.py`. Three pages:

```
main.py  ->  st.navigation([
    pages/quiz.py      "Quiz"   (default)   home -> quiz -> summary  (internal state machine)
    pages/review.py    "Review"             WRONG ANSWERS | LEARNING DASHBOARD  (st.pills sub-tabs)
    pages/admin.py     "Admin"              app config (+ per-call model) · questions manager · Cortex spend · logs
])
```

**Config layer**: runtime behavior toggles live in `QUIZ_CONFIG` (defaults in `_config.py` `CONFIG_DEFAULTS`, DB overrides; `load_config()` cached + `save_config()` in `_data.py`, both with `clear_caches()` on write). Gates used below: `hints_enabled`, `debrief_enabled` (both in `CONFIG_DEFAULTS`), and the **per-call-group model** keys `model_generation` / `model_explanation` / `model_meta` - these **default inline to `CORTEX_MODEL`** via `.get(key, CORTEX_MODEL)` (NOT a `CONFIG_DEFAULTS` entry), so they always track the base model instead of pinning a second hardcoded default; see Admin App config. (There is no `explanations_default` config - the explanation + deep-dive are always on-demand. There is **no** `default_round_size` or `pass_threshold_override` config - round size is set on Home each round, and the pass threshold is the fixed `PASS_THRESHOLD` study proxy, not user-tunable.)

**Config (de)serialization - COPY this; it is the #1 config crash.** `QUIZ_CONFIG.config_value` is `VARIANT`. The round-trip is **`PARSE_JSON(?)` on write + `json.loads` on read** - and it goes through the **one `save_config()` helper**. Do NOT inline a config MERGE in a page, and **NEVER `TO_VARIANT(json.dumps(value))`** - that double-encodes (stores `"\"ai\""`, reads back the literal `'"ai"'`) and then `options.index('"ai"')` raises `ValueError`. (`PARSE_JSON` here is in a MERGE `USING` sub-select, which is allowed - the `$sis` "no PARSE_JSON inside `VALUES(`" rule is only about `INSERT … VALUES`.) VARIANT is kept (matches the DDL + the Step-1g `grounding_mode` seed); a plain `VARCHAR` column would also work but needs no schema change, so don't.
```python
# _data.py
def save_config(key, value):
    get_active_session().sql(
        f"MERGE INTO {SCHEMA}.QUIZ_CONFIG t USING (SELECT ? AS k, PARSE_JSON(?) AS v) s ON t.config_key = s.k "
        f"WHEN MATCHED THEN UPDATE SET config_value = s.v, updated_at = CURRENT_TIMESTAMP() "
        f"WHEN NOT MATCHED THEN INSERT (config_key, config_value) VALUES (s.k, s.v)",
        params=[key, json.dumps(value)],         # json.dumps → PARSE_JSON round-trips; NOT TO_VARIANT(json.dumps(...))
    ).collect()
    clear_caches()

@st.cache_data(show_spinner=False)
def load_config():
    cfg = dict(CONFIG_DEFAULTS)
    for r in get_active_session().sql(f"SELECT config_key, config_value FROM {SCHEMA}.QUIZ_CONFIG").collect():
        v = r["CONFIG_VALUE"]
        if isinstance(v, str):
            try: v = json.loads(v)            # VARIANT string comes back JSON-encoded; decode once
            except Exception: pass
        cfg[r["CONFIG_KEY"]] = v
    return cfg
```
**Guard every config-seeded widget default** - a stored value that isn't in the options list must never crash the page (`ValueError`/`StreamlitAPIException`). Use a helper, never a bare `options.index(cfg[key])`:
```python
def cfg_index(options, value, default=0):       # _ui.py
    return options.index(value) if value in options else default
# st.selectbox("Model", MODEL_OPTIONS, index=cfg_index(MODEL_OPTIONS, cfg.get("model_generation", CORTEX_MODEL)))
```

**`_ui.py` - the ONE shared style/helpers module (COPY + ADAPT).** Every page imports these; **no raw `st.title` / `st.markdown("## …")` / `st.markdown("**…**")` / hardcoded category labels in page code** (`$quiz/design` - Shared UI module + UI-text-case rule).
```python
# _ui.py - shared style + helpers. The ONLY place titles / section headers / labels / badges / case live.
import streamlit as st

def md(text) -> str:                                  # $-escape so dynamic text never renders as LaTeX
    return ("" if text is None else str(text)).replace("$", r"\$")

def page_title(text):       st.markdown(f"## {text}")            # page/tab entry title - NEVER st.title
def section_header(label):  st.markdown(f"**{str(label).upper()}**")  # in-page section - NEVER st.subheader

def render_domain_badge(name):  return f":blue-badge[{(name or '').upper()}]"
def render_difficulty_badge(diff):
    d = (diff or "medium").lower()
    return {"easy": ":green-badge[EASY]", "hard": ":red-badge[HARD]"}.get(d, ":orange-badge[MEDIUM]")

def render_docs_link(url):
    if url and url.strip():  st.markdown(f"📖 [Snowflake Documentation]({url.strip()})")

def cfg_index(options, value, default=0):             # guarded selectbox index - never bare .index()
    return options.index(value) if value in options else default

# UI-text-case rule ($quiz/design): difficulty/domain pills show UPPERCASE via st.pills(format_func=str.upper)
# and return the RAW value (no normalizer). These helpers cover the rest - the Source filter option list,
# and DB-enum display OUTSIDE pills (badges, table cells/headers).
SOURCE_PILLS = ["QUESTION BANK", "AI GENERATED"]                       # Home source-filter options (display = value)
SOURCE_LABELS = {"MANUAL": "MANUAL", "AI_GENERATED": "AI GENERATED"}   # DB enum -> display label
def source_label(v):   return SOURCE_LABELS.get(v, (v or "").replace("_", " ").upper())
def column_label(col): return (col or "").replace("_", " ").upper()    # QUESTION_ID -> "QUESTION ID" (table headers)
```

**Entry point (`main.py`)**: `st.set_page_config` (first `st.` call) -> `init_session_state()` -> shared sidebar title -> `st.navigation(pages).run()`. Pages share `st.session_state` (it persists across page switches).

**Inside `pages/quiz.py`** the three screens are an internal state machine driven by `st.session_state["screen"]` (`home` / `quiz` / `summary`) - they are NOT separate pages.

**Inside `pages/review.py`** each tab body opens with `page_title()` (NEVER `st.title`) and uses `section_header()` for its sections (`$quiz/design`).

Cross-page redirects: set the target state, then `st.switch_page("pages/<target>.py")`.

---

# Home Screen (`pages/quiz.py`, screen == "home")

4 control groups in order, each with a bold UPPERCASE label (`st.markdown("**LABEL**")`) and `label_visibility="collapsed"` on the widget:

1. **QUESTIONS** - `st.slider("Questions", 1, 100, value=…)` so **any** count 1-100 is pickable. **NOT `st.pills`** (a fixed list like 5/10/15/20 is wrong) and not `number_input`. Initial `value` from `round_size` if set, else the `_config.py` default. The round size is set here per round - it is NOT an Admin config value.
2. **DOMAINS** - pills multi-select over the `DOMAIN_NAME`s with `format_func=str.upper` (UPPERCASE labels, raw names returned), initial selection from `domain_filter` if set
3. **DIFFICULTY** - pills over the raw values `["mixed","easy","medium","hard"]` with `format_func=str.upper`, guard against None, initial value from `difficulty` if set
4. **SOURCE** - pills multi-select over `SOURCE_PILLS` (`["QUESTION BANK","AI GENERATED"]`, already display-cased), mapped by membership to `"mix"/"db"/"ai"`

**Defaults + reset-to-default (`DEFAULT_ROUND_CONFIG` in `_config.py`).** Each control **seeds its initial value from the matching session key** (`round_size`, `domain_filter`, `difficulty`, `question_source`) when present - so a cross-page redirect that sets those keys pre-fills the round - and falls back to `DEFAULT_ROUND_CONFIG` = `{"round_size": <config default>, "domain_filter": [] (all domains), "difficulty": "medium", "question_source": "ai"}` otherwise. **On round end** (the summary's "CONFIGURE NEW ROUND" button) reset those four keys back to `DEFAULT_ROUND_CONFIG`, so a finished round never leaks its settings into the next one.

The explanation is on-demand per question (a button after answering), never auto-loaded.

**Grounding guard** (top of Home, `cke`/`custom` mode): if `not docs_available()` (`$cortex`/`_search.py`), show a red "install/grant the Snowflake Documentation CKE - the app can't generate until it's reachable" message and **disable Start Round**. The app must never generate from built-in knowledge. (In `none` mode there is no guard - generation is intentionally ungrounded.)

**Start Round** button: saves settings to session state, builds topic schedule via `_build_topic_schedule()` (from `_questions.py`), loads first question with spinner, sets `screen="quiz"`, reruns.

---

# Quiz Screen (`pages/quiz.py`, screen == "quiz")

**Lazy load**: At top of the quiz screen, if `question` is None, load via `get_question()` with spinner, then rerun for clean render.

**Layout**: progress bar -> domain/difficulty badges -> h4 question text -> answer input -> submit -> result + explanation -> navigation

**Answer input**: `st.radio()` for single-answer (`index=None`, disabled once answered). For multi-answer, render each option as an independent `st.checkbox` with a stable key (`cb_A`, `cb_B`, …); read the selection from session state after rendering; disable all once answered. On "NEXT", clear the `cb_*` keys via the flag-at-top reset (queue `["cb_A", …, "cb_E"]` in `_op_clear_keys`, rerun, pop at the top - see `$sis` widget lifecycle).

**Socratic hint (BEFORE answering; gate `hints_enabled`)**: a secondary "💡 HINT" button near the answer input, visible ONLY while `answered == False`. First click → `call_cortex_json(prompt, "hint", model=model_for("explanation"))`, show `hint_1` (prefixed `💡`) and **relabel the button to "NEED MORE HELP?"**; second click reveals `hint_2` (prefixed `💡💡`), then hide the button. **Render order:** always render the already-revealed hint(s) FIRST, then - on a click - hide the button and show a spinner *below* the visible hint while the next generates (render-then-fetch, never fetch-then-render), with a single `st.rerun()` after, so an already-shown hint stays on screen. The hint is **grounded like every other generation** - in `cke`/`custom` mode embed retrieved `<doc_context>` in the hint prompt and derive the hints from it, never the model's built-in knowledge (`$cortex`); the prompt MUST instruct: hints narrow the concept space (level 1) or eliminate ONE distractor with reasoning (level 2) and must NEVER name or imply the correct option; embed question/options per the untrusted-content delimiting rule. State: `hint` (None/{}/dict), `hint_level` (0/1/2); once answered, the button disappears (the explanation takes over); record `hint_used = hint_level > 0` in the history item; reset both on Next.

**Submit**: Records result in `round_history` (incl. `hint_used`), increments counters, sets `answered=True`, reruns. Does NOT call Cortex.

**After submission** - render **stacked and full-width, top to bottom** (NOT a `st.columns(2)` row):
1. the **result badge**
2. the **"💡 AI EXPLANATION"** button (full-width, secondary) → loads the explanation **on demand** (Explanation Contract below); while it generates, hide the button and show a labelled spinner (On-demand generation UX below); once loaded, the expander renders **here**
3. **"NEXT"** (primary, full-width; **"FINISH ROUND"** on the last question) at the **very bottom - beneath the whole explanation expander** - so the next action sits directly under what the user just read

The explanation is NEVER auto-generated, and the button appears for **correct answers too** (to learn why the distractors are wrong). "NEXT" stays pinned at the bottom whether or not the explanation is open. Do NOT render "NEXT" and "FINISH ROUND" together. Do NOT add per-option markup (✓, strikethrough) - the explanation handles details.
- Correct: `:green-badge[✅ CORRECT]`
- Incorrect: `:red-badge[❌ INCORRECT]` + newline + `Correct answer: **A**, **C**` (bold letters only, not full option text)

**Sidebar "END ROUND"**: When `screen == "quiz"`, the sidebar shows an "END ROUND" button (secondary, full-width). Clicking sets `_pending_finish = True` and does a natural rerender.

**Button click safety**: guard slow-action buttons (Start Round, Submit, Next, Finish) with the `_transitioning` flag so a double-click can't double-fire during the rerun:
```python
if st.button("NEXT", disabled=st.session_state.get("_transitioning", False)):
    st.session_state["_transitioning"] = True
    # ... action ...
    st.session_state["_transitioning"] = False
    st.rerun()
```
Pair this with the spinner + single-`st.rerun()` rule in `$sis`.

**On-demand generation UX (hint, explanation, deep dive, Round Summary - MANDATORY).** When a button triggers a slow Cortex call: (1) render any already-generated content **first**; (2) **hide the button(s) that trigger that generation** while it runs - never leave a sibling sitting there dimmed; (3) show exactly **one** labelled `st.spinner("Generating <thing>…")`; (4) generate; (5) a single `st.rerun()`. Same pattern for all four.

---

# Explanation Contract

**On-demand only** - generated when the user clicks **"💡 AI EXPLANATION"**, never automatically. State machine in `st.session_state["explanation"]`: `None` (not requested yet - just show the button), `{}` (tried and failed; do NOT retry), `{dict}` (success; render the expander). On "NEXT": reset to `None`.

`_generate_explanation()` calls `call_cortex_json(prompt, "explanation", model=model_for("explanation"))` - the `RESPONSE_FORMATS["explanation"]` schema (`$cortex`) guarantees `why_correct` (array), `why_wrong` (object), `mnemonic`, `doc_search`. No fence parsing; retry only on `None`.

**Model routing (MANDATORY) for the learning loop** (`$cortex` `model_for`): explanation, **hint**, and **deep dive** pass `model=model_for("explanation")`; the **Round Summary debrief** passes `model=model_for("meta")`. Every `call_cortex_json`/`call_cortex` in these handlers takes the `model=` arg - omitting it silently pins the call to `CORTEX_MODEL` and the Admin model selector does nothing.

**Same flow for correct AND incorrect answers** - clicking the button opens `st.expander("💡 AI EXPLANATION", expanded=True)` containing, in order:
- `st.container(border=True)` **✅ WHY CORRECT** - `why_correct` bullet list
- `st.container(border=True)` **WHY WRONG** - one line per *other* option (on a correct answer this is exactly the value: learn why the distractors are wrong)
- `st.info()` mnemonic
- the doc link **only**: `📖 [Snowflake Documentation]({doc_url})` - **NO "📚 From the docs" heading, NO `DOCUMENT_TITLE` caption, and NO raw `CHUNK` excerpt** (the chunk renders an internal name + a truncated definition full of `¶` glyphs - drop it entirely; grounding still *uses* the chunk to write `why_correct`/`why_wrong`, the UI just shows the clean link)
- the **Deep dive** control (below)

The 📖 doc link appears **only inside this expander, only after the button is clicked** - never auto-shown (not even for correct answers). Render every dynamic field (`why_correct`, `why_wrong`, `mnemonic`) through the `_ui.py` `md()` escaper before `st.markdown`/`st.info` (`$quiz/design` - escaping dynamic text). Store `mnemonic` + `doc_url` on `current_history_item` for the review log.

**Doc grounding is MANDATORY in `cke`/`custom` mode (`$cortex`)**: retrieve `chunks = search_docs(question_text)` once; if `[]`, broaden once, else **fail visibly** (no built-in). Embed the top chunk(s) in the prompt as `<doc_context>` to ground the *prose*. **The displayed 📖 link reuses the question's stored `DOC_URL`** (set at generation by re-ranking the retrieved chunks to the `question_text` - `$quiz/questions`), NOT a fresh `chunks[0]["SOURCE_URL"]` from this retrieval - re-searching here can surface a different page than the one the question was written from. Only when the question carries no `DOC_URL` (e.g. a manual bank row with none stored) fall back to this retrieval's top `SOURCE_URL`. This is a **teaching** call - use the **teaching grounding style** (`$cortex` "Grounded ≠ parroting"): ground every claim in the chunks, but **EXPLAIN the concept in your own words**; do NOT use the bare *"answer ONLY from the provided documentation"* line, do NOT quote the docs line-by-line, and do NOT write every bullet as "the documentation says…". **`none` mode only**: the prompt asks for `doc_search` ("2-3 words, no URLs/commas") → `https://docs.snowflake.com/en/search?q={query}`.

**Explanation prompt content** - the schema guarantees shape, the prompt controls quality. Lead with: *"You are a SnowPro tutor. Explain comprehensively and holistically WHY the correct answer is right and each distractor is wrong - teach the underlying concept so it sticks. Ground every claim in `<doc_context>` but write in your own words; cite at most one short passage."* Then the fields:
```
"why_correct": ["First reason - a real explanation of the mechanism, not 'the docs say X'", "Second reason with technical detail", "Optional third"],
"why_wrong": {"X": "one sentence explaining the actual misconception behind option X", "Y": "..."},
"mnemonic": "a memorable phrase or acronym for the correct answer",
"doc_search": "exactly 2-3 words for Snowflake docs search. No URLs. No commas. Max 3 words."
```

## Deep dive (inside the expander, below the explanation)

A single **"🔬 DEEP DIVE"** button (`width='stretch'`, full width of the expander) - **NO option picker** → `call_cortex_json(prompt, "deep_dive", model=model_for("explanation"))` - an in-depth breakdown of the **question's topic** (how it works / when to use / exam traps), not a single answer option. `$cortex` `deep_dive` schema: `summary`, `how_it_works[]`, `when_to_use`, `exam_traps[]`. Render in `st.container(border=True)` with bold sub-labels + bullets. **Grounded** like the explanation, and the **same teaching style** (`$cortex` "Grounded ≠ parroting"): reuse the retrieved `<doc_context>`, broaden once then **fail visibly** on empty - never built-in; embed the **question + its topic/domain** per the delimiting rule (not a selected option); EXPLAIN the topic in your own words (do not parrot the docs), carry **more depth** than the base explanation; render through the `_ui.py` `md()` escaper. State: `deep_dive` (None/{}/dict), reset on Next.

---

# Reference code (COPY + ADAPT - the high-fidelity-risk handlers)

The prose above is the contract; this is the **reference implementation of the parts that are repeatedly gotten wrong** (home slider, hint state machine, on-demand spinner, stacked after-submit layout, deep-dive-in-expander, sidebar End Round). **Copy these handlers and adapt** (swap `EXAM_NAME`, wire your `_cortex`/`_search`/`_data` helpers) - do not re-derive them from the prose. The Step-8 UX-conformance gate checks that the generated `quiz.py` matches these shapes.

```python
# pages/quiz.py - reference for render_home / render_quiz (hard parts). LETTERS = ["A","B","C","D","E"]

def render_home():
    cfg = load_config(); domains = load_domains()
    if grounding_required() and not docs_available():
        st.markdown(":red-badge[DOC GROUNDING UNAVAILABLE] Install/grant the Snowflake Documentation CKE."); st.stop()
    st.markdown(f"## {EXAM_NAME} Quiz")
    dflt = DEFAULT_ROUND_CONFIG                                          # _config.py: defaults + reset-to-default
    st.markdown("**QUESTIONS**")
    round_size = st.slider("Questions", 1, 100, value=st.session_state.get("round_size", dflt["round_size"]),
                           key="sl_round_size", label_visibility="collapsed")          # SLIDER, not pills
    st.markdown("**DOMAINS**")
    dom = st.pills("Domains", [d["DOMAIN_NAME"] for d in domains], selection_mode="multi",
                   format_func=str.upper,                               # UPPERCASE label, raw DOMAIN_NAME returned
                   default=st.session_state.get("domain_filter") or dflt["domain_filter"],
                   key="pills_domains", label_visibility="collapsed")
    st.markdown("**DIFFICULTY**")
    diff = st.pills("Difficulty", ["mixed","easy","medium","hard"], format_func=str.upper,   # raw value returned
                    default=st.session_state.get("difficulty") or dflt["difficulty"],
                    key="pills_diff", label_visibility="collapsed") or dflt["difficulty"]
    st.markdown("**SOURCE**")
    src_default = {"db": ["QUESTION BANK"], "ai": ["AI GENERATED"]}.get(                 # seed labels from the value
        st.session_state.get("question_source", dflt["question_source"]), SOURCE_PILLS)
    src_sel = st.pills("Source", SOURCE_PILLS, selection_mode="multi",
                       default=src_default, key="pills_src", label_visibility="collapsed") or ["AI GENERATED"]
    source = "mix" if len(src_sel) == 2 else ("db" if "QUESTION BANK" in src_sel else "ai")
    if st.button("START ROUND", type="primary", width='stretch',
                 disabled=st.session_state.get("_transitioning", False)):
        st.session_state["_transitioning"] = True
        for k, v in [("round_size", round_size), ("domain_filter", dom or []), ("difficulty", diff),
                     ("question_source", source), ("round_history", []), ("correct_count", 0),
                     ("total_count", 0), ("q_index", 0), ("question", None), ("answered", False),
                     ("selected", []), ("explanation", None), ("hint", None), ("hint_level", 0),
                     ("deep_dive", None), ("debrief", None), ("_results_written", False)]:
            st.session_state[k] = v
        st.session_state["_topic_schedule"] = _build_topic_schedule(domains, dom or [], round_size)
        with st.spinner("Loading first question…"):
            st.session_state["question"] = get_question()
        st.session_state["screen"] = "quiz"; st.session_state["_transitioning"] = False; st.rerun()


def render_quiz():
    for k in st.session_state.pop("_op_clear_keys", []):           # flag-at-top widget reset
        st.session_state.pop(k, None)
    cfg = load_config()
    q = st.session_state.get("question")
    if q is None:
        with st.spinner("Loading question…"):
            st.session_state["question"] = get_question()
        st.rerun()

    with st.sidebar:                                              # END ROUND - always present in quiz
        if st.button("END ROUND", width='stretch', key="btn_end_round"):
            st.session_state["_pending_finish"] = True
    if st.session_state.get("_pending_finish") and st.session_state.get("answered"):
        _write_back_results(); st.session_state["screen"] = "summary"
        st.session_state["_pending_finish"] = False; st.rerun()

    idx, total = st.session_state.get("q_index", 0), st.session_state.get("round_size", 10)
    answered = st.session_state.get("answered", False)
    st.progress(idx / total, text=f"Question {idx+1} of {total}")
    st.markdown(f"{render_domain_badge(q.get('DOMAIN_NAME',''))}  {render_difficulty_badge(q.get('DIFFICULTY','medium'))}")
    st.markdown(f"#### {md(q.get('QUESTION_TEXT',''))}")
    options = {lt: q.get(f"OPTION_{lt}") for lt in LETTERS if q.get(f"OPTION_{lt}")}

    # answer input (radio single / checkbox multi), disabled once answered - NO per-option ✅/❌ markup
    if not q.get("IS_MULTI"):
        chosen = st.radio("Answer", [f"{lt}) {md(t)}" for lt, t in options.items()], index=None,
                          disabled=answered, key=f"radio_{idx}", label_visibility="collapsed")
        selected = [chosen.split(")")[0]] if chosen else []
    else:
        st.markdown(":gray-badge[SELECT ALL THAT APPLY]")
        selected = [lt for lt in options if st.checkbox(f"{lt}) {md(options[lt])}", key=f"cb_{lt}", disabled=answered)]

    if not answered:
        # HINT - state machine: render revealed hints FIRST, then the button (relabels, hides at level 2)
        if cfg.get("hints_enabled", True):
            hint, lvl = st.session_state.get("hint"), st.session_state.get("hint_level", 0)
            if isinstance(hint, dict):
                if lvl >= 1 and hint.get("hint_1"): st.markdown(f"💡 {md(hint['hint_1'])}")
                if lvl >= 2 and hint.get("hint_2"): st.markdown(f"💡💡 {md(hint['hint_2'])}")
            if lvl < 2:
                if st.button("💡 HINT" if lvl == 0 else "NEED MORE HELP?", key="btn_hint"):
                    with st.spinner("Thinking of a hint…"):          # on-demand: button replaced by spinner
                        _generate_hint(q)
                    st.session_state["hint_level"] = lvl + 1; st.rerun()
        if st.button("SUBMIT", type="primary", width='stretch',
                     disabled=st.session_state.get("_transitioning", False)):
            if not selected:
                st.markdown(":orange-badge[Please select an answer]")
            else:
                _record_submit(q, selected, options); st.rerun()    # builds full-text history item, sets answered
    else:
        correct = [c.strip() for c in q.get("CORRECT_ANSWER","").split(",")]
        if set(st.session_state.get("selected", [])) == set(correct):
            st.markdown(":green-badge[✅ CORRECT]")
        else:
            st.markdown(":red-badge[❌ INCORRECT]")
            st.markdown("Correct answer: " + ", ".join(f"**{lt}**" for lt in correct))

        # STACKED, top→bottom: AI explanation button → expander (incl. deep dive) → Next pinned at the very bottom
        expl = st.session_state.get("explanation")
        if expl is None:
            if st.button("💡 AI EXPLANATION", width='stretch'):
                with st.spinner("Generating explanation…"):
                    _generate_explanation(q, options)
                st.rerun()
        if isinstance(expl, dict) and expl:
            _render_explanation_expander(q, options)    # WHY CORRECT/WRONG + mnemonic + 📖 link + 🔬 DEEP DIVE (topic, no picker)

        is_last = (idx + 1) >= total
        if st.button("FINISH ROUND" if is_last else "NEXT", type="primary", width='stretch',
                     disabled=st.session_state.get("_transitioning", False)):
            _advance(is_last)    # write-back if last/pending, else load next + reset hint/explanation/deep_dive
```

Notes the gate enforces: round size is a **slider**; the hint **state machine** (no double-dump, button hides at level 2, prior hints persist); **END ROUND** in the sidebar; the explanation is **stacked** with **NEXT pinned last**; **DEEP DIVE lives inside `_render_explanation_expander`** (the quiz screen, on the question's topic) - **never** in the summary; no per-option ✅/❌ markup; the missed-question summary shows **full-text** correct answers only.

---

# History Item Schema

Each submitted answer is appended to `round_history`:

| Field | Type | Notes |
|-------|------|-------|
| `question_id` | int/None | From QUESTION_ID field |
| `domain_id` | str | VARCHAR in the schema |
| `domain_name` | str | |
| `difficulty` | str | |
| `question_text` | str | |
| `correct_answer` | str | Letter(s) - e.g. `"C"` or `"A,D"` |
| `option_texts` | dict | `{"A": "...", "B": "..."}` - critical for correct answer display |
| `selected` | str | Comma-joined selected letters |
| `selected_labels` | list | `["A) full text", ...]` |
| `is_correct` | bool | |
| `hint_used` | bool | True if any hint level was revealed |
| `_topic` | str | From `_current_topic` session state |
| `mnemonic` | str | Empty initially; filled after explanation |
| `doc_url` | str | cke/custom: the question's stored `DOC_URL` (fallback: fresh `chunks[0]` SOURCE_URL); none mode: built from `doc_search` |

`option_texts` is critical - without it, correct answer display shows only letters.

---

# Summary Screen (`pages/quiz.py`, screen == "summary")

- `PASS_THRESHOLD = 75` lives in `_config.py`. All pass/fail logic, badge text, chart threshold rules, and delta calculations MUST reference this constant - never hardcode `75` or `75.0` in multiple places.
- Title: "Round Complete!" (pass >= PASS_THRESHOLD) or "Round Complete" (fail)
- 2-metric row: SCORE (`correct/total`), ACCURACY (`pct%`)
- Pass: `:green-badge[PASSED] above {PASS_THRESHOLD}% threshold`
- Fail: `:orange-badge[NOT YET] {gap}% to go - keep practicing!` (include encouragement)
- **The missed questions** go inside an **`st.expander("TO REMEMBER", expanded=True)`** (NOT "WRONG ANSWERS", NOT collapsed) - each as an `st.container(border=True)` card with domain/difficulty badges, the **question text**, and **only the correct answer, in full text** (`"B) …full option text… & D) …"`, built from the card's `option_texts` + correct letters). **Do NOT show the user's pick, and NEVER show bare letters** (a card reading "Your answer: D / Correct: B" is the exact anti-pattern to avoid - it tells the user nothing). Add the **mnemonic** (`st.info` 🧠) only when one was generated for it this round (omit if empty/None). All dynamic text via the `md()` `$`-escaper (`$quiz/design`).
- Perfect score: `:green-badge[PERFECT SCORE] No wrong answers this round.` (no expander)

**Round Summary (on-demand; gate `debrief_enabled`; only when ≥1 wrong answer)**: a **"ROUND SUMMARY"** button - NOT auto-generated. On click, **hide all summary buttons and show one spinner "Generating round summary…"** (On-demand generation UX, Quiz Screen - don't leave "CONFIGURE NEW ROUND" dimmed beside it) → `call_cortex_json(prompt, "debrief", model=model_for("meta"))` with per-question domain/topic/correctness/`hint_used` from `round_history` → open an **expander titled "ROUND SUMMARY"** with clean formatting: `st.container(border=True)` holding **PATTERNS** (bullets) and **PRIORITY ACTIONS** (max 3 bullets), then `one_thing` as a highlighted **🎯 FOCUS** line in its own bordered container (NOT `st.info` - that's reserved for the mnemonic per `$quiz/design`). **Grounding scope:** meta-analysis over the user's OWN round performance, **addressed to the user in the second person** ("you keep missing X", "you should review Y") per the `$cortex` voice rule - names weak domains/topics + study actions but MUST NOT assert new Snowflake facts or emit doc links; the one runtime generation path exempt from doc-grounding (`$cortex`). State `debrief` (None=not requested / {}=failed / dict=success), reset on round start. A perfect round shows no Round Summary button. Render every dynamic field (PATTERNS, PRIORITY ACTIONS, `one_thing`) through the `md()` `$`-escaper before `st.markdown` (`$quiz/design`).

**Action buttons (by outcome)** - `ROUND SUMMARY` applies only when there is ≥1 wrong answer.
- **Perfect** (all correct): **"CONFIGURE NEW ROUND"** only.
- **Passed, some wrong**: **"ROUND SUMMARY"** + **"CONFIGURE NEW ROUND"**.
- **Failed**: **"ROUND SUMMARY"** + **"CONFIGURE NEW ROUND"**.
- Threshold = `PASS_THRESHOLD` (the fixed study proxy - not user-overridable; there is no `pass_threshold_override`). All buttons set state and call `st.rerun()`. **"CONFIGURE NEW ROUND" resets the round-config keys to `DEFAULT_ROUND_CONFIG`** (`round_size` / `domain_filter` / `difficulty` / `question_source`) before `screen="home"`, so the next round starts from defaults (Home Screen), never the finished round's settings.

---

# Review: Wrong Answers (`pages/review.py`)

Query `QUIZ_REVIEW_LOG` via the cached loader in `_data.py` (`@st.cache_data` with NO ttl, `get_active_session()` inside - see `$sis` Caching). Do NOT query directly in the page code. Freshness comes from `clear_caches()` at write time, not from a ttl.

Filters (BOTH always present): a **domain `st.pills`** (multi-select, empty = all) over the distinct `DOMAIN_NAME` values, PLUS a **date-range `st.date_input`** - **always shown**, even when all wrong answers fall on one day (do NOT hide it when `min == max`; pass that single date as both `value` ends + `min_value`/`max_value`). Keep a row when (no domain selected OR its `DOMAIN_NAME` is in the selection) AND its cast `LOGGED_AT` date is within `[start, end]`. Filtering is **client-side in Python over the cached frame** - the loader takes no params and never filters server-side. Show a `st.caption` with the filtered count.

Wrong answer cards: `st.container(border=True)` with domain badge + difficulty badge + date (`:gray-badge[YYYY-MM-DD]`) badge, question text, **correct answer in FULL TEXT**, mnemonic (`st.info` 🧠) only-when-present, doc link only-when-present.

**Correct answer - FULL TEXT, NEVER a bare letter.** Render `**Correct answer:** {md(CORRECT_ANSWER)}` straight from the `QUIZ_REVIEW_LOG.CORRECT_ANSWER` column. That column is **already stored resolved** as `"{letter}) {full_text}"` (joined with ` & ` for multi-answer) by the Write-Back Contract - so the card does a **plain passthrough with NO letter→option lookup** (the review log has no `OPTION_*` columns to look up anyway). A card must never read just `Correct: A`; it must read e.g. `Correct answer: B) Time Travel lets you query historical data`. Do NOT show the user's pick (the log never stores it). Escape through `md()` so a `$` in the answer text doesn't render as LaTeX (`$quiz/design`).

**Mnemonic - render ONLY when a real one exists; guard the literal string `"None"`.** A wrong answer the user never opened the explanation on has an empty `mnemonic` (the column is `VARCHAR`, not VARIANT), and a Python `None` coerced via `str()` surfaces as the literal text `"None"`. Render the `st.info("🧠 …")` box **only** when the value is truthy AND not the literal `"None"` - e.g. `mnem = (row["MNEMONIC"] or "").strip()` then `if mnem and mnem.lower() != "none": st.info(f"🧠 {md(mnem)}")`. This prevents a `🧠 None` box. `st.info` is mnemonic-only (`$quiz/design` - never for status). The doc link follows the same only-when-present guard and uses the standard format `📖 [Snowflake Documentation]({url})`.

**Date handling** (filters + dashboard): values from the cached loader are Snowflake datetimes - cast with `datetime.date(raw.year, raw.month, raw.day)` before feeding any widget or doing date arithmetic. The Wrong-Answers date filter is an **`st.date_input` range** (always rendered - see above); `st.date_input` returns a 1- or 2-tuple mid-selection, so **defensively unpack** both ends before comparing. Compare each row's cast `LOGGED_AT` date against the selected `[start, end]` in Python. For any `TIMESTAMP_LTZ` range query in SQL, pass dates as `strftime("%Y-%m-%d")` strings with an exclusive upper bound (`< end + 1 day`) to include the full last day. Never pass a `datetime.date` to `st.slider` (`$sis`).

## Reference code (COPY + ADAPT - Review Wrong-Answers handler)

The prose above is the contract; this is the **reference implementation of the Review Wrong-Answers handler**. **Copy this handler and adapt** - do not re-derive it from the prose. The Step-8 UX gate checks `review.py` against this shape. It is `st.pills`-driven tab content in `pages/review.py`; `load_review_log`/`clear_caches` are the `_data.py` loaders (no ttl), `md`/`render_domain_badge`/`render_difficulty_badge` are the `_ui.py` helpers (`$quiz/design`); `LETTERS = ["A","B","C","D","E"]`.

```python
# pages/review.py - reference for the WRONG ANSWERS sub-tab. LETTERS = ["A","B","C","D","E"]
import datetime

def _cast_date(raw):                                            # Snowflake datetime → date before any widget/math
    return datetime.date(raw.year, raw.month, raw.day)

def render_wrong_answers():
    rows = load_review_log()       # cached (@st.cache_data, NO ttl, get_active_session() inside) → list[Row], newest first
    if not rows:                   # list[Row] - read fields by row["UPPER"], never .get()/attr on a Row ($sis item 20)
        st.markdown(":gray-badge[No wrong answers logged yet - finish a round to populate this.]"); return

    # FILTERS - both always rendered. Domain pills (empty = all) + date-range (shown even when min == max).
    dom_opts = sorted({r["DOMAIN_NAME"] for r in rows if r["DOMAIN_NAME"]})
    doms = st.pills("Filter by domain", dom_opts, selection_mode="multi",
                    default=[], key="wa_domain", label_visibility="collapsed")

    dates = [_cast_date(r["LOGGED_AT"]) for r in rows]
    min_d, max_d = min(dates), max(dates)                       # safe - rows is non-empty past the guard above
    rng = st.date_input("Date range", value=(min_d, max_d),
                        min_value=min_d, max_value=max_d, key="wa_dates")
    start = rng[0] if isinstance(rng, (list, tuple)) and len(rng) >= 1 else min_d
    end   = rng[1] if isinstance(rng, (list, tuple)) and len(rng) >= 2 else start   # mid-selection 1-tuple guard

    filtered = [r for r in rows                                 # client-side filter over the cached list[Row]
                if (not doms or r["DOMAIN_NAME"] in doms)
                and start <= _cast_date(r["LOGGED_AT"]) <= end]
    st.caption(f"{len(filtered)} wrong answer(s)")

    for r in filtered:
        with st.container(border=True):
            d = _cast_date(r["LOGGED_AT"]).strftime("%Y-%m-%d")
            st.markdown(f"{render_domain_badge(r['DOMAIN_NAME'] or '')}  "
                        f"{render_difficulty_badge(r['DIFFICULTY'] or 'medium')}  :gray-badge[{d}]")
            st.markdown(f"#### {md(r['QUESTION_TEXT'])}")
            st.markdown(f"**Correct answer:** {md(r['CORRECT_ANSWER'])}")   # FULL TEXT, stored resolved - no letter lookup
            mnem = (r["MNEMONIC"] or "").strip()
            if mnem and mnem.lower() != "none":                  # guard None, "", AND the literal "None" → no 🧠 None box
                st.info(f"🧠 {md(mnem)}")
            doc = (r["DOC_URL"] or "").strip()
            if doc:                                              # 📖 link only when present (never auto-shown)
                st.markdown(f"📖 [Snowflake Documentation]({doc})")
```

Notes the gate enforces: the correct answer is the **full-text `CORRECT_ANSWER` passthrough** (never a bare letter, no `OPTION_*` lookup); the mnemonic `st.info` 🧠 box renders **only** when the value is truthy and not the literal `"None"`; **both** filters are present and the **date input is always rendered** (even when `min == max`); the loader is the **cached `load_review_log()`** (no direct query in page code) and all filtering is **client-side in Python**; every dynamic field routes through `md()`; cards are `st.container(border=True)` with the date `:gray-badge[…]`; no `st.success`/`st.warning`/`st.error`.

---

# Review: Learning Dashboard (`pages/review.py`)

**Empty state**: if 0 sessions, show info message and return.

**Cached queries** (all in `_data.py`, `@st.cache_data` with NO ttl, `get_active_session()` inside):
- `load_session_stats()` - sessions, avg_score, total_questions
- `load_recent_sessions()` - last 10 with session labels (e.g. "#1 . 31/03")
- `load_domain_errors()` - error count per domain

**Layout**: 3 metrics (Sessions, Questions, Readiness with delta) -> Score per Session chart -> Errors by Domain chart.

**Readiness metric**: `st.metric("Readiness", f"{avg_score:.1f}%", delta=f"{delta_val:+.1f}% vs pass")`. Value is a numeric percentage, NOT a badge - `st.metric()` does not render Markdown badges. Hide delta when at threshold: `delta=... if abs(delta_val) >= 0.1 else None`.

**Charts:** build every chart (colors, axis types/formatting, label limits, threshold rules) per `$quiz/design` - that skill is the single source for all visual rules; reference the `_config.py` color constants and never hardcode hexes or axis specs here. Two charts on this dashboard: **Score per Session** (from `load_recent_sessions()`) and **Errors by Domain** (from `load_domain_errors()`).

---

# Admin Page (`pages/admin.py` - core)

Four **`st.tabs`** - **App config · Questions manager · Cortex spend · Logs** (not one long scrolling page); the four subsections below are the tab contents in order. Single-user app → visible to the owner; when multi-user lands, gate via restricted caller's rights (fail-closed) - do NOT build RBAC now.

Admin's widget / pagination / pending-confirm keys - `_qm_filters`, `_qm_select_all`, `_qm_limit`, `_qm_mode`/`_qm_edit_id`, `pending_delete`, `pending_reset` - are **page-local**: initialized/guarded with `.get()` defaults in `admin.py`, NOT added to the core `init_session_state` contract. The core contract carries only cross-page shared state.

## Data tables - shared display contract

The three admin tables (Questions bank, review log, session log) share one display contract so they look and scroll the same. Build each with **`st.data_editor`** (never a raw `st.dataframe`), `width='stretch'`, fed a **`.to_pandas()` DataFrame with UPPERCASE columns** (the one loader-type exception to `list[Row]`, because `column_config` needs a frame; `$sis` loader convention).

- **Column order** is explicit via `column_order=[...]`, independent of the SELECT order.
- **Display headers** come from `_ui.py` `column_label(col)` set in `column_config` (`QUESTION_ID` shows as "QUESTION ID"; underscore to space, uppercase). Display only - the DataFrame keys stay UPPERCASE for access.
- **Per-column** `width` ("small"/"medium"/"large") and `disabled=True` for every read-only column.
- **Pin the leading key columns** so they stay visible while scrolling wide tables: pass `pinned=True` to the `column_config` entry for the `select` checkbox (where present) and the id column; the rest scroll horizontally.
- **Row id shows as a contiguous 1..N index, not the raw key.** `AUTOINCREMENT` ids are unique and increasing but not gap-free (a stored id like 1, 2, 3, 101 looks wrong), so compute the display number at read time with `ROW_NUMBER() OVER (ORDER BY <timestamp column>) AS "#"` and keep the real id only as the hidden key for edit/delete (`WHERE question_id = ?`).
- **Editable cells** (only where specified, e.g. the review log's `mnemonic` + `doc_url`): make ONLY those columns editable, keep every other column `disabled=True`, read the edited frame back from the `st.data_editor` return value, and persist each changed row with a bind-param `UPDATE` keyed by the row id, then `clear_caches()`. An editable table has its own loader (edits must write back), separate from any display-only loader.

```python
# _ui.py - column_config builder. spec = [(COL, "small"/"medium"/"large", editable, pinned), ...]
import streamlit as st
def table_column_config(spec):
    return {col: st.column_config.Column(column_label(col), width=w, disabled=not editable, pinned=pinned)
            for (col, w, editable, pinned) in spec}

# _data.py - the EDITABLE review-log loader: .to_pandas() with UPPERCASE aliases + a write-back helper.
@st.cache_data(show_spinner=False)
def load_review_log_editable():
    return get_active_session().sql(
        f'SELECT log_id AS "LOG_ID", logged_at AS "LOGGED_AT", domain_name AS "DOMAIN_NAME", '
        f'difficulty AS "DIFFICULTY", question_text AS "QUESTION_TEXT", correct_answer AS "CORRECT_ANSWER", '
        f'mnemonic AS "MNEMONIC", doc_url AS "DOC_URL" '
        f"FROM {SCHEMA}.QUIZ_REVIEW_LOG ORDER BY logged_at DESC"
    ).to_pandas()

def write_review_log_edit(log_id, mnemonic, doc_url):          # one row, bind params
    get_active_session().sql(
        f"UPDATE {SCHEMA}.QUIZ_REVIEW_LOG SET mnemonic = ?, doc_url = ? WHERE log_id = ?",
        params=[mnemonic, doc_url, int(log_id)],
    ).collect()
    clear_caches()

# pages/admin.py - editable review-log: only MNEMONIC + DOC_URL editable; persist the changed rows.
def render_review_log_table(df):                                # df = load_review_log_editable(), post-filter
    edited = st.data_editor(
        df, key="rl_editor", hide_index=True, width='stretch',
        column_order=["LOG_ID","LOGGED_AT","DOMAIN_NAME","DIFFICULTY","QUESTION_TEXT","CORRECT_ANSWER","MNEMONIC","DOC_URL"],
        column_config={
            "LOG_ID": st.column_config.Column(column_label("LOG_ID"), pinned=True, disabled=True),
            **table_column_config([("LOGGED_AT","small",False,False), ("DOMAIN_NAME","medium",False,False),
                ("DIFFICULTY","small",False,False), ("QUESTION_TEXT","large",False,False),
                ("CORRECT_ANSWER","large",False,False), ("MNEMONIC","large",True,False), ("DOC_URL","large",True,False)]),
        },
    )
    changed = [r for _, r in edited.iterrows()
               if (r["MNEMONIC"] or "") != (df.loc[df["LOG_ID"] == r["LOG_ID"], "MNEMONIC"].iloc[0] or "")
               or (r["DOC_URL"] or "") != (df.loc[df["LOG_ID"] == r["LOG_ID"], "DOC_URL"].iloc[0] or "")]
    if changed:
        for r in changed:
            write_review_log_edit(r["LOG_ID"], r["MNEMONIC"], r["DOC_URL"])
        st.toast(f"Updated {len(changed)} row(s)"); st.rerun()
```

## 1. App config

**Two toggles** - `hints_enabled`, `debrief_enabled` ("ROUND SUMMARY"). Plus **AI model per call-group** (3 `st.selectbox`, options from `_config.py` `MODEL_OPTIONS = ["claude-haiku-4-5", "claude-sonnet-4-6", "claude-opus-4-8"]`):

| Selector (label) | Config key | Covers |
|---|---|---|
| Question generation | `model_generation` | `get_question`/`generate_ai_question`, Admin Generate batch |
| Explanations & study aids | `model_explanation` | explanation, hint, deep dive |
| Meta-analysis | `model_meta` | Round Summary debrief |

Each defaults to `CORTEX_MODEL` (`load_config().get(f"model_{group}", CORTEX_MODEL)`); the call sites read it via `model_for(group)` and pass it to `call_cortex`/`call_cortex_json` (`$cortex`). The Cortex-spend "by model" chart reflects these choices.

**Do NOT show:** `grounding_mode` (fixed at setup, `$setup-exam` Step 1g - not a runtime toggle, so don't surface it at all), `pass_threshold_override` (the official threshold doesn't change - removed), `default_round_size` (set on Home before each round - redundant), and `question_source` / `difficulty` (these are **per-round Home choices**, never global config - they are never re-added here as selectboxes; App config holds ONLY the two toggles + the three model selectboxes). The only grounding UI here is the guard: in `cke`/`custom` mode, if `docs_available()` is False, a red caption - "The doc grounding service is unavailable - install/grant the Snowflake Documentation CKE; the app can't generate until it's reachable." Every change → the **`save_config()` helper** (the `PARSE_JSON` round-trip from Config layer - NEVER inline a config MERGE here, NEVER `TO_VARIANT(json.dumps())`) → `clear_caches()` (clears `docs_available`/`search_docs` too) → `st.toast`. The three model `st.selectbox`es seed their `index` via `cfg_index(MODEL_OPTIONS, cfg.get(f"model_{group}", CORTEX_MODEL))` (never a bare `.index()` - see Config layer).

## 2. Questions manager

Two nested `st.tabs` - **Bank** and **Generate**.

**Bank** - KPIs above an editable table (the vendors02 reference pattern):
- **KPIs as `st.metric`, NOT a table**: Total questions · MANUAL · AI_GENERATED · domains covered (in `st.columns`, `$quiz/design` KPI cards). Backed by **`load_bank_stats()`** - a no-ttl cached coverage loader (per domain × difficulty × source) in `_data.py`, registered in `clear_caches()` (`$sis` item 24). The editable table below uses its own filtered loader (next).
- **Editable / filterable / deletable table**:
  1. **Filters** in an `st.expander("Filters", expanded=True)` - domain / difficulty / source pills - then a **"Search"** button commits them to a `_qm_filters` session dict (don't query live off widget state). A "Reset filters" button.
  2. **`Select All` / `Clear`** buttons above the table (toggle a `_qm_select_all` flag).
  3. **`st.data_editor`** with a leading `select` `st.column_config.CheckboxColumn`; **every other column `disabled`**; read the selection back as `edited[edited["select"]]`. The table's source is a **no-ttl cached loader returning a pandas DataFrame** (`.to_pandas()`) - `data_editor` preserves its checkbox selection across reruns ONLY when its input is byte-identical (a ttl that expired mid-edit would wipe the selection - `$sis` caching), and `.to_pandas()` columns come back UPPERCASE so there is **no `Row` access at all** - selected-row fields are read as `row["QUESTION_TEXT"]`, never `.get()`/attr on a `Row` (`$sis` scan item 20). Apply the **Data tables display contract** above: `column_order` = select · question id · domain name · question text · correct answer · option a-e · difficulty · source; `column_label` headers; `pinned=True` on the `select` checkbox and the id column; per-column `width`; and a `ROW_NUMBER()` display index in place of the raw `question_id`.
  4. **`Load 10 more`** (page cap 10 - don't render 1000 rows; paginate via a `_qm_limit` that grows by 10).
  5. Below the table: **`Edit` · `Delete` · `Add`**.
     - **Edit** (enabled when exactly 1 row selected): loads that row into the form below → UPDATE by `question_id`.
     - **Add**: opens the same form **empty** → INSERT `source='MANUAL'`.
     - The form (Edit + Add) is an **`st.form(clear_on_submit=True)`** so all fields commit together on submit and the form resets after Insert (fixes the per-field Cmd+Enter + "form stays filled" problem): question `st.text_area`, options A-E `st.text_input`, `correct_answer` `st.multiselect` **restricted to the NON-EMPTY option values** (read inside the form on submit, so no per-field Enter), difficulty pills; `is_multi` derived = `len(correct) > 1`.
     - **Delete** (enabled when ≥1 selected): **two-step confirm** - a bordered `pending_delete` panel ("Delete N question(s)?") with **Confirm / Cancel** (vendors02 `pending_*` pattern, NOT a type-`DELETE` text gate) → `DELETE FROM QUIZ_QUESTIONS WHERE question_id IN (?, …)`.
  - **Hard rules** (`$sis` scan item 21): every write via bind params (`?`, NEVER f-string); length caps in the form AND by truncation (question 2000, options 500); `correct_answer` ⊆ non-empty options; ≥2 options. Every write → `clear_caches()` → `st.toast`.
  - **Editor-state discipline (the hard part - follow vendors02 `01_ai_recommendations.py`):** keep a signature of the rendered slice (tuple of `QUESTION_ID`s). On a **non-append** change (filters/Search/Delete changed the set) drop the `data_editor` widget key and reset `_qm_select_all` so the checkbox column re-seeds cleanly; on a **pure append** (Load-10-more extends the slice) keep the selection. Without this, selection jumps on every Load-more/Search.
  - **Edit vs Add form (one form, a mode flag):** `_qm_mode` ∈ `"add"`/`"edit"` (+ `_qm_edit_id` for edit). **Add** renders the empty `st.form(clear_on_submit=True)` → INSERT. **Edit** seeds the form from the selected row by writing the field values into the widget `session_state` keys at the **top of the run, before the widgets render** (flag-at-top - `$sis` widget lifecycle); **never pass both `value=`/`default=` and also set the `session_state` key** (raises "created with a default value but also had its value set"). Switching Add↔Edit clears the prior field keys via the same flag-at-top reset.

**Generate** - Generate batch with options: **count** (`st.slider`, 1-20 - each item is its own grounded LLM call, so the batch is capped to stay well under the statement timeout; never an unbounded `st.number_input`), **difficulty** (pills, incl. "mixed"), **domain** (`st.selectbox`/multiselect over `EXAM_DOMAINS`), and **model** (`st.selectbox` over `MODEL_OPTIONS`, `index` seeded via `cfg_index(MODEL_OPTIONS, cfg.get("model_generation", CORTEX_MODEL))`) → generates via the **same grounded `_questions.py` path** (`$quiz/questions`: in `cke`/`custom` mode each question embeds retrieved `<doc_context>` as the primary source with `key_facts` as supporting scope, answers ONLY from the docs, fails visibly on empty retrieval - never built-in) → INSERT `source='AI_GENERATED'` → `clear_caches()`. **Feedback is mandatory**: run the loop under one `st.spinner`, then on completion fire an `st.toast` **and** render a transient `:green-badge[Added N question(s)]` line (NEVER `st.success`/`st.warning` - `$quiz/design`); a `st.caption` notes larger batches take longer and cost more. The batch passes its chosen model through `call_cortex_json(..., model=…)`.

## 3. Cortex spend (graceful - distinguish "no grant" from "no data")

`_cortex.py` sets a `QUERY_TAG` (JSON: `app`, `feature`, `model`) per call. Read `SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AISQL_USAGE_HISTORY` (the **current** view - `CORTEX_FUNCTIONS_USAGE_HISTORY` is deprecated/"no longer updated" per Snowflake docs; columns `USAGE_TIME` / `MODEL_NAME` / `FUNCTION_NAME` / `TOKENS` / `TOKEN_CREDITS` / **`QUERY_TAG`** - the view **echoes the per-call `QUERY_TAG` directly**, so feature attribution needs **no** `QUERY_HISTORY` join) inside try/except and branch **structurally**, not "any-exception → GRANT" (an ACCOUNTADMIN holds `IMPORTED PRIVILEGES` by default, so blaming every empty/error result on permissions shows a false GRANT banner):
- **Success path** - the query returned. If the result is **empty** → a plain caption: "No Cortex spend recorded yet - ACCOUNT_USAGE lags up to ~2 h, or no AI calls have run." (This is the ACCOUNTADMIN-on-a-fresh-account case; ACCOUNTADMIN holds `IMPORTED PRIVILEGES` by default, so **never** show the GRANT banner here.) Otherwise render the charts.
- **Exception path** - inspect the error. Only when its message signals a **privilege/visibility problem on the SNOWFLAKE share** (e.g. it contains `Insufficient privileges`, `not authorized`, or `does not exist or not authorized` - the symptom of a role lacking `IMPORTED PRIVILEGES`) → the info banner with the exact `GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <role>;` (substitute the live `CURRENT_ROLE()`). For any **other** exception → a generic "couldn't read Cortex spend: {error}" caption, NOT the GRANT banner.

**Charts** - both aggregate `SUM(TOKEN_CREDITS)` over rows **scoped to THIS app**. The view holds the whole account's Cortex usage, so always filter to the app's own tag, and parse with **`TRY_PARSE_JSON` (never bare `PARSE_JSON`** - other queries leave non-JSON/empty `QUERY_TAG`s that would raise). The loader's base query:
```sql
SELECT model_name,
       TRY_PARSE_JSON(query_tag):feature::VARCHAR AS feature,
       token_credits
FROM SNOWFLAKE.ACCOUNT_USAGE.CORTEX_AISQL_USAGE_HISTORY
WHERE TRY_PARSE_JSON(query_tag):app::VARCHAR = 'snowpro_quiz'   -- the value _cortex.py writes
```
- **Spend by model** - `GROUP BY MODEL_NAME` (the haiku/sonnet/opus split, reflecting the App-config model choices).
- **Spend by feature** - `GROUP BY` the parsed `feature` (question / explanation / hint / deep_dive / debrief / batch). `FUNCTION_NAME` can't do this (it is `AI_COMPLETE`/`COMPLETE` for every call) - the `QUERY_TAG` `feature` is the grouping key.

Caption: ACCOUNT_USAGE has reporting latency (up to ~2 h; this view only covers usage from 2025-11-17 on). When doc grounding is on, add a "docs search" line (Cortex Search query compute is billed to the consumer; small per query).

## 4. Logs (was "Tools")

A read-only log viewer plus a reset - the vendors02 logs page, no download:
- **Both log tables shown**: `QUIZ_REVIEW_LOG` via `load_review_log()` and `QUIZ_SESSION_LOG` via **`load_session_log()`** - a new no-ttl cached loader for the full session-log table (`load_recent_sessions()` is the dashboard's last-10 aggregate, NOT this); **add `load_session_log` to `_data.py` and register it in `clear_caches()`** (`$sis` item 24, else it dangles). Read-only `st.dataframe`, newest first. **No `st.download_button`.**
- **Filters + paging** (`QUIZ_REVIEW_LOG`): a **domain `st.multiselect`** (empty = all) applied **in Python over the cached `list[Row]`** (`[r for r in rows if not doms or r["DOMAIN_NAME"] in doms]` - `load_review_log()` returns `.collect()` Rows, NOT a DataFrame), plus `Load N more` paging via a page-local `_log_limit` (don't render thousands of rows). `st.dataframe` accepts the `list[Row]` slice directly. The session-log table is shown as-is, newest first.
- **Loader return-type convention** (`$sis` Caching): every cached loader returns a **`.collect()` `list[Row]`** (read fields by `row["UPPER"]`, never `.get()`/attr - `$sis` scan item 20) - `load_review_log`, `load_session_log`, `load_recent_sessions`, `load_domain_errors`, `load_bank_stats`, `load_domains`, `load_cortex_spend`. **The exceptions are the `st.data_editor` tables** - `load_questions_page` (Questions bank) and `load_review_log_editable` (the editable review-log table) - which return **`.to_pandas()`** because `st.data_editor`/`column_config` structurally require a DataFrame. (`load_review_log` stays `list[Row]` for the read-only Review wrong-answers cards; the editable admin table uses the separate `load_review_log_editable`.) Do NOT "normalize" them - a DataFrame handler against a `list[Row]` loader (or vice-versa) crashes on the first `.empty`/`.iterrows`/`[col]` call.
- **Reset all logs** - a **frameless/borderless button** (`st.button(..., type="tertiary")`) below the tables, with **no expander and no "DANGER ZONE" label**, then a **two-step confirm** (a bordered `pending_reset` panel with Confirm / Cancel - NOT a type-`DELETE` text gate). Confirm runs `DELETE FROM` on the **two log tables only** (`QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`) - **NEVER `DROP`**, consistent with governance - then `clear_caches()` + `st.toast`. The reset lives **in this Logs tab**, never in App config.

## Reference code (COPY + ADAPT - Admin handlers)

The prose above is the contract; this is the **reference implementation of the Admin handlers**. **Copy these handlers and adapt** - do not re-derive them from the prose. The Step-8 UX gate checks `admin.py` against these shapes. All four are `st.tabs` contents in `pages/admin.py`; `LETTERS = ["A","B","C","D","E"]`; `save_config`/`load_config`/`cfg_index`/`clear_caches` are the Config-layer helpers; `SCHEMA`/`MODEL_OPTIONS`/`CORTEX_MODEL` are `_config.py` constants.

```python
# 1. APP CONFIG - ONLY two toggles + three model selectboxes. No source/round/difficulty/grounding/threshold.
def render_app_config():
    cfg = load_config()
    st.markdown("**FEATURES**")
    hints   = st.toggle("Hints (before answering)", value=cfg.get("hints_enabled", True), key="cfg_hints")
    debrief = st.toggle("Round Summary (end of round)", value=cfg.get("debrief_enabled", True), key="cfg_debrief")
    st.markdown("**AI MODEL PER CALL-GROUP**")
    picks = {}
    for label, group in [("Question generation", "generation"),
                         ("Explanations & study aids", "explanation"),
                         ("Meta-analysis (Round Summary)", "meta")]:
        picks[group] = st.selectbox(label, MODEL_OPTIONS,
            index=cfg_index(MODEL_OPTIONS, cfg.get(f"model_{group}", CORTEX_MODEL)),   # guarded, never bare .index()
            key=f"cfg_model_{group}")
    if grounding_required() and not docs_available():
        st.markdown(":red-badge[DOC GROUNDING UNAVAILABLE] Install/grant the Snowflake Documentation CKE; "
                    "the app can't generate until it's reachable.")
    if st.button("Save", type="primary"):
        save_config("hints_enabled", hints)                # PARSE_JSON round-trip; NEVER TO_VARIANT(json.dumps())
        save_config("debrief_enabled", debrief)
        for group, model in picks.items():
            save_config(f"model_{group}", model)
        st.toast("Settings saved")                         # save_config already clears caches


# 2. QUESTIONS MANAGER - nested tabs; Bank = KPIs + editable/filterable/deletable table.
def render_questions_manager():
    bank_tab, gen_tab = st.tabs(["Bank", "Generate"])
    with bank_tab:    render_bank()
    with gen_tab:     render_generate()

def _is_append(prev, sig):                                  # append = prev is a prefix of the new id slice
    return len(sig) >= len(prev) and sig[:len(prev)] == prev

@st.cache_data(show_spinner=False)
def load_questions_page(flt, limit):                        # the ONE .to_pandas() loader - st.data_editor needs a DataFrame
    where, params = [], []
    for col, key in [("domain_name", "dom"), ("difficulty", "diff"), ("source", "src")]:
        vals = flt.get(key) or []
        if vals:
            where.append(f"{col} IN ({','.join(['?'] * len(vals))})"); params += vals
    clause = ("WHERE " + " AND ".join(where)) if where else ""
    return get_active_session().sql(
        f"SELECT question_id, domain_name, difficulty, source, question_text, "
        f"option_a, option_b, option_c, option_d, option_e, correct_answer "
        f"FROM {SCHEMA}.QUIZ_QUESTIONS {clause} ORDER BY question_id DESC LIMIT {int(limit)}",
        params=params).to_pandas()                          # UPPERCASE cols; register in clear_caches()

def render_bank():
    rows = load_bank_stats()                                # cached coverage loader → list[Row] (domain × difficulty × source, CNT)
    cols = st.columns(4)                                    # KPIs are st.metric, NOT a table - derive them from the rows
    cols[0].metric("Total",  sum(r["CNT"] for r in rows))
    cols[1].metric("Manual", sum(r["CNT"] for r in rows if r["SOURCE"] == "MANUAL"))
    cols[2].metric("AI",     sum(r["CNT"] for r in rows if r["SOURCE"] == "AI_GENERATED"))
    cols[3].metric("Domains", len({r["DOMAIN_NAME"] for r in rows}))

    with st.expander("Filters", expanded=True):             # commit filters on Search, don't query live off widgets
        f_dom  = st.pills("Domain", [d["DOMAIN_NAME"] for d in load_domains()], selection_mode="multi", key="qm_f_dom")
        f_diff = st.pills("Difficulty", ["easy","medium","hard"], selection_mode="multi", key="qm_f_diff")
        f_src  = st.pills("Source", ["MANUAL","AI_GENERATED"], selection_mode="multi", key="qm_f_src")
        fc = st.columns(2)
        if fc[0].button("Search"):
            st.session_state["_qm_filters"] = {"dom": f_dom, "diff": f_diff or [], "src": f_src or []}
            st.session_state["_qm_limit"] = 10
        if fc[1].button("Reset filters"):
            st.session_state.pop("_qm_filters", None); st.session_state["_qm_limit"] = 10

    flt   = st.session_state.get("_qm_filters", {})
    limit = st.session_state.get("_qm_limit", 10)
    df = load_questions_page(flt, limit)                    # cached, .to_pandas() → UPPERCASE cols, no Row access
    df.insert(0, "select", False)

    sig  = tuple(df["QUESTION_ID"].tolist())                # editor-state discipline (vendors02)
    prev = st.session_state.get("_qm_sig")
    if prev is not None and not _is_append(prev, sig):      # non-append change → re-seed checkbox column cleanly
        st.session_state.pop("editor_qm", None); st.session_state["_qm_select_all"] = False
    st.session_state["_qm_sig"] = sig

    sc = st.columns(2)
    if sc[0].button("Select all"): st.session_state["_qm_select_all"] = True;  st.session_state.pop("editor_qm", None)
    if sc[1].button("Clear"):      st.session_state["_qm_select_all"] = False; st.session_state.pop("editor_qm", None)
    if st.session_state.get("_qm_select_all"): df["select"] = True

    edited = st.data_editor(df, key="editor_qm", hide_index=True, width='stretch',
        column_config={"select": st.column_config.CheckboxColumn("", default=False)},
        disabled=[c for c in df.columns if c != "select"])  # every column but the checkbox is read-only
    sel = edited[edited["select"]]                          # read selection back as a DataFrame slice

    if len(df) >= limit and st.button("Load 10 more"):
        st.session_state["_qm_limit"] = limit + 10; st.rerun()

    bc = st.columns(3)
    if bc[0].button("Edit", disabled=len(sel) != 1):        # enabled only when exactly 1 row is selected
        st.session_state["_qm_mode"] = "edit"
        st.session_state["_qm_edit_id"] = int(sel.iloc[0]["QUESTION_ID"])
        st.session_state["_qm_seed"]    = sel.iloc[0].to_dict(); st.rerun()
    if bc[1].button("Add"):
        st.session_state["_qm_mode"] = "add"; st.session_state.pop("_qm_seed", None); st.rerun()
    if bc[2].button("Delete", disabled=len(sel) < 1):
        st.session_state["pending_delete"] = [int(x) for x in sel["QUESTION_ID"]]

    if st.session_state.get("pending_delete"):              # two-step confirm, NOT a type-DELETE gate
        ids = st.session_state["pending_delete"]
        with st.container(border=True):
            st.markdown(f"**Delete {len(ids)} question(s)?**")
            dc = st.columns(2)
            if dc[0].button("Confirm", type="primary"):
                marks = ",".join(["?"] * len(ids))          # bind params, never f-string the ids
                get_active_session().sql(
                    f"DELETE FROM {SCHEMA}.QUIZ_QUESTIONS WHERE question_id IN ({marks})", params=ids).collect()
                clear_caches(); st.session_state.pop("pending_delete", None)
                st.toast(f"Deleted {len(ids)}"); st.rerun()
            if dc[1].button("Cancel"):
                st.session_state.pop("pending_delete", None); st.rerun()

    if st.session_state.get("_qm_mode"):
        _render_question_form()

def _render_question_form():
    mode = st.session_state["_qm_mode"]
    seed = st.session_state.pop("_qm_seed", None)           # flag-at-top: seed widget KEYS once, before they render
    if seed is not None:                                    # (never also pass value=/default= to the same widget)
        st.session_state["qf_text"] = seed.get("QUESTION_TEXT", "")
        for lt in LETTERS: st.session_state[f"qf_{lt}"] = seed.get(f"OPTION_{lt}") or ""
        st.session_state["qf_diff"]    = seed.get("DIFFICULTY", "medium")
        st.session_state["qf_correct"] = [c.strip() for c in (seed.get("CORRECT_ANSWER") or "").split(",") if c.strip()]
    st.markdown(f"**{'EDIT QUESTION' if mode == 'edit' else 'ADD QUESTION'}**")
    with st.form("qform", clear_on_submit=True):            # one form → all fields commit together, no per-field Enter
        text = st.text_area("Question", key="qf_text", max_chars=2000)
        opts = {lt: st.text_input(f"Option {lt}", key=f"qf_{lt}", max_chars=500) for lt in LETTERS}
        diff = st.pills("Difficulty", ["easy","medium","hard"], key="qf_diff")
        # a form can't live-filter options as the user types; offer all letters, enforce ⊆ non-empty options ON SUBMIT
        correct = st.multiselect("Correct answer(s)", LETTERS, key="qf_correct")
        fc = st.columns(2)
        submitted = fc[0].form_submit_button("Save", type="primary")
        cancelled = fc[1].form_submit_button("Cancel")
    if cancelled:
        _close_question_form(); st.rerun()
    if submitted:
        present = [lt for lt in LETTERS if (opts[lt] or "").strip()]      # validate the committed values
        correct = [c for c in correct if c in present]
        if len(present) < 2 or not correct:
            st.markdown(":orange-badge[Need ≥2 options and ≥1 correct answer among the filled options]"); return
        _save_question(mode, st.session_state.get("_qm_edit_id"), text.strip(), opts, correct, diff)  # bind params
        clear_caches(); _close_question_form()
        st.toast("Saved" if mode == "edit" else "Added"); st.rerun()

def _close_question_form():                                 # flag-at-top reset of every form key
    for k in ["_qm_mode", "_qm_edit_id", "qf_text", "qf_diff", "qf_correct", *[f"qf_{lt}" for lt in LETTERS]]:
        st.session_state.pop(k, None)

def render_generate():
    cfg = load_config()
    if grounding_required() and not docs_available():
        st.markdown(":red-badge[DOC GROUNDING UNAVAILABLE] Can't generate until the CKE is reachable."); return
    n     = st.slider("Count", 1, 20, 5, key="gen_n")       # SLIDER, capped - each item is a grounded LLM call
    diff  = st.pills("Difficulty", ["mixed","easy","medium","hard"], default="mixed", key="gen_diff") or "mixed"
    doms  = st.multiselect("Domains", [d["DOMAIN_NAME"] for d in load_domains()], key="gen_dom")
    model = st.selectbox("Model", MODEL_OPTIONS,
        index=cfg_index(MODEL_OPTIONS, cfg.get("model_generation", CORTEX_MODEL)), key="gen_model")
    st.caption("Larger batches take longer and cost more - each question is its own grounded generation.")
    if st.button("Generate batch", type="primary", disabled=st.session_state.get("_gen_busy", False)):
        st.session_state["_gen_busy"] = True; made = 0
        with st.spinner(f"Generating {n} question(s)…"):
            for _ in range(n):
                if generate_ai_question(diff, doms or None, model=model):   # same grounded _questions.py path
                    made += 1
        clear_caches(); st.session_state["_gen_busy"] = False
        st.session_state["_gen_result"] = made
        st.toast(f"Generated {made} question(s)"); st.rerun()
    if "_gen_result" in st.session_state:                   # transient feedback - never st.success ($quiz/design)
        st.markdown(f":green-badge[Added {st.session_state.pop('_gen_result')} question(s)] to the bank.")


# 3. CORTEX SPEND - branch STRUCTURALLY: success-but-empty ≠ permission error.
def render_spend():
    try:
        rows = load_cortex_spend()      # cached → list[Row]; SELECT over CORTEX_AISQL_USAGE_HISTORY scoped to this app
                                        # via WHERE TRY_PARSE_JSON(query_tag):app::VARCHAR = 'snowpro_quiz' (see §3 query)
    except Exception as e:
        msg = str(e).lower()
        if any(sig in msg for sig in ("insufficient privileges", "not authorized", "does not exist")):
            role = get_active_session().get_current_role().strip('"')
            st.markdown(":orange-badge[NO ACCESS] Grant ACCOUNT_USAGE to read Cortex spend:")
            st.code(f"GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE {role};", language="sql")
        else:
            st.caption(f"Couldn't read Cortex spend: {e}")      # any OTHER error → caption, NOT the GRANT banner
        return
    if not rows:                        # ACCOUNTADMIN holds IMPORTED PRIVILEGES by default → empty ≠ no-grant
        st.caption("No Cortex spend recorded yet - ACCOUNT_USAGE lags up to ~2 h, or no AI calls have run."); return
    # ...build a DataFrame inline from `rows` for the two Altair charts, SUM(token_credits):
    #    by MODEL_NAME (haiku/sonnet/opus) and by the parsed QUERY_TAG :feature - per $quiz/design,
    #    same list[Row]→chart pattern as the Learning Dashboard...
    st.caption("ACCOUNT_USAGE lags up to ~2 h.")


# 4. LOGS - both tables (filtered), then a FRAMELESS reset with a two-step confirm. Reset lives HERE, not App config.
def render_logs():
    st.markdown("**REVIEW LOG**")
    rows = load_review_log()                                # cached → list[Row] (.collect()), newest first
    doms = st.multiselect("Domain", sorted({r["DOMAIN_NAME"] for r in rows if r["DOMAIN_NAME"]}), key="log_dom")
    view = [r for r in rows if not doms or r["DOMAIN_NAME"] in doms]   # filter in Python over the cached list
    lim  = st.session_state.get("_log_limit", 50)
    st.dataframe(view[:lim], hide_index=True, width='stretch')   # st.dataframe accepts list[Row]; NO download_button
    if len(view) > lim and st.button("Load 50 more", key="log_more"):
        st.session_state["_log_limit"] = lim + 50; st.rerun()
    st.markdown("**SESSION LOG**")
    st.dataframe(load_session_log(), hide_index=True, width='stretch')   # list[Row]

    if st.button("Reset all logs", type="tertiary", key="log_reset"):        # frameless, no DANGER ZONE
        st.session_state["pending_reset"] = True
    if st.session_state.get("pending_reset"):
        with st.container(border=True):
            st.markdown("**Delete all review + session log rows?**")
            rc = st.columns(2)
            if rc[0].button("Confirm", type="primary", key="reset_yes"):
                s = get_active_session()
                s.sql(f"DELETE FROM {SCHEMA}.QUIZ_REVIEW_LOG").collect()      # DELETE the two log tables - NEVER DROP
                s.sql(f"DELETE FROM {SCHEMA}.QUIZ_SESSION_LOG").collect()
                clear_caches(); st.session_state.pop("pending_reset", None)
                st.toast("Logs reset"); st.rerun()
            if rc[1].button("Cancel", key="reset_no"):
                st.session_state.pop("pending_reset", None); st.rerun()
```

Notes the gate enforces: App config exposes **only** the two toggles + three model selectboxes (no source/round/difficulty); the Questions Bank has the **editable `st.data_editor` table** with the checkbox column + editor-state signature; every config/question/delete write uses **bind params** + `clear_caches()`; the spend panel **branches structurally** (empty-on-success → caption, privilege-signal → GRANT, other → caption); the log **reset is in the Logs tab**, frameless, two-step. Batch generation uses a **slider** and always fires **toast + transient badge** feedback.

---

# Write-Back Contract

On round end (`_write_back_results()` in `pages/quiz.py`):

**0. Write exactly once per round (idempotency guard - MANDATORY).** Re-entry is normal - the Finish button can fire twice during a rerun, or a failure *after* the INSERTs can send the user back to re-click - and each re-entry would otherwise re-INSERT every row. Guard with a **plain boolean**, not a round id: at the very top of `_write_back_results()`, `if st.session_state.get("_results_written"): return`; otherwise set `st.session_state["_results_written"] = True` **before** the INSERTs. **Start Round is the SOLE reset site** - it sets `_results_written = False`; no other path may clear it (any "re-enter summary without a new round" path must leave it set). Pair with the `_transitioning`/`_pending_finish` button guards (Quiz Screen) so a slow Finish can't double-fire.

1. For each wrong answer in `round_history`: INSERT to `QUIZ_REVIEW_LOG` with bind params
   - `correct_answer` must be **resolved** before INSERT - store `"{letter}) {full_text}"`, not the raw letter. Pattern:
     ```python
     letters = h["correct_answer"].split(",")
     texts = h["option_texts"]
     correct_full = " & ".join(f"{lt.strip()}) {texts.get(lt.strip(), '')}" for lt in letters)
     ```
   - Fields: domain_id, domain_name, difficulty, question_text, correct_answer, mnemonic, doc_url

2. One session summary: INSERT to `QUIZ_SESSION_LOG`
   - Fields: exam_code (from `_config.py`), round_size, correct_count, score_pct, domain_filter, difficulty
   - `round_size` = the **configured** round size from `st.session_state["round_size"]`, NOT `total_count` (which is how many questions were actually answered - may differ if user ends early)

3. **Call `clear_caches()`** (from `_data.py`) as the **last** step, immediately after the INSERTs - the Review page loaders have no ttl, so without this the dashboard would not see the new round until a full session restart. `clear_caches()` must not be able to raise: every `.clear()` inside it names a loader actually defined in `_data.py` (`$sis` scan item 24), so a write can't commit and then crash the handler.

Always bind params (`?` qmark - `session.sql(sql, params=[...])`; never `:1`, see `$sis`), never f-string interpolation of values.

---

# Session State Contract

All keys initialized in `init_session_state()` in `main.py` (state is shared across pages):

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `screen` | str | `"home"` | Quiz-page state machine: home/quiz/summary |
| `question` | dict\|None | `None` | Current question (UPPERCASE keys from DB/AI) |
| `answered` | bool | `False` | Whether current question is answered |
| `selected` | list | `[]` | Selected option letters |
| `explanation` | None/{}/ dict | `None` | None=not tried, {}=failed, dict=success |
| `q_index` | int | `0` | 0-based question index in round |
| `round_size` | int | `10` | Questions per round; round-config default from `DEFAULT_ROUND_CONFIG` |
| `round_history` | list | `[]` | List of history_item dicts |
| `difficulty` | str | `"medium"` | mixed/easy/medium/hard; round-config default from `DEFAULT_ROUND_CONFIG` (`_config.py`) |
| `domain_filter` | list | `[]` | Selected domain names; empty=all; from `DEFAULT_ROUND_CONFIG` |
| `question_source` | str | `"ai"` | mix/db/ai; round-config default from `DEFAULT_ROUND_CONFIG` |
| `_current_topic` | str | `""` | Current topic from schedule |
| `_review_page` | str | `"WRONG ANSWERS"` | Active review sub-tab |
| `correct_count` | int | `0` | Correct answers this round |
| `total_count` | int | `0` | Total answered this round |
| `current_history_item` | dict\|None | `None` | Ref to last appended history item |
| `last_cortex_error` | str\|None | `None` | Debug: last Cortex error |
| `last_ai_parse_error` | str\|None | `None` | Debug: last parse guard trip |
| `_transitioning` | bool | `False` | Button click safety flag |
| `_topic_schedule` | list | `[]` | Shuffled (domain,topic) pairs for the round |
| `_pending_finish` | bool | `False` | Finish pending flag for deferred write-back |
| `_results_written` | bool | `False` | Write-once guard for `_write_back_results()`; set `True` before the INSERTs, reset to `False` only on Start Round (prevents duplicate review/session rows on re-click or post-write crash) |
| `_op_clear_keys` | list | `[]` | Widget keys to pop at top of next run (flag-at-top reset) |
| `hint` | None/{}/ dict | `None` | Socratic hint (None=not tried, {}=failed, dict=success) |
| `hint_level` | int | `0` | 0=none, 1=hint_1 shown, 2=hint_2 shown |
| `deep_dive` | None/{}/ dict | `None` | Deep-dive result for the question's **topic** (no option picker); reset on Next |
| `debrief` | None/{}/ dict | `None` | On-demand Round Summary (None=not requested, {}=failed, dict=success); generated from the Round Summary button, reset on round start |

Navigation is owned by `st.navigation` (no `nav_pills` / `_current_page` / `_redirect_to_quiz` keys); cross-page redirects set the target state, then call `st.switch_page("pages/quiz.py")`.

---

# UX-Conformance Gate (run before deploy - separate from the `$sis` scan)

The `$sis` pre-deploy scan certifies the app **runs** and is **SQL-safe** (imports resolve, no `NameError`, binds, cache freshness). It does **NOT** certify that the generated screens match the UX contracts above - so an app can pass the scan 100% while shipping a slider-less Home, a bare-letter answer, a config crash, and a tableless Questions manager. This gate closes that gap: it is a **conformance checklist read statically against the generated `app/` files**, owned here because `$quiz/screens` owns the screen contracts.

**How to run** (Step 8 item 9, after the `$sis` scan): run the gate from **this section read fresh** (not a remembered checklist), and **re-read every generated file from disk this turn** (`pages/quiz.py`, `pages/review.py`, `pages/admin.py`, `_data.py`, `_cortex.py`, `_config.py`, `main.py`) - never from memory, and **writing a file earlier is not reading it** (re-read each from disk). **Report EVERY check below, not a subset** - an abbreviated "N/N pass" reconstructed from memory is not acceptable (the gate has many checks; emit a row for each). A "possible loop" warning during the re-reads is an **expected false positive** - re-read everything before emitting any verdict. Each check is decided by a static read; output one row per check (**# · check · PASS/FAIL · `file:line`**). **Deploy only when every check PASSES**; on any FAIL, fix → re-read → re-run the affected checks (do NOT deploy on a FAIL, exactly like the scan). Most checks are mechanical (grep-able); the few marked *(judgment)* need a short read of the handler. The reference implementations these check against are the `# Reference code (COPY + ADAPT)` blocks above - a FAIL usually means the generator paraphrased instead of copying. **If you cannot re-read every generated file from disk THIS turn** (file-not-found / `I/O` / read-only or degraded session / broken workspace mount), you **cannot run this gate** - **STOP, do not certify, do not deploy**, and report the access failure rather than certifying from memory or a prior turn's reads.

### Quiz screen (`pages/quiz.py`)
1. **Round size = slider.** FAIL if round size uses `st.pills` or `st.number_input` (must be `st.slider(1, 100, …)`). *(Home reference)*
2. **Hint state machine.** FAIL if the hint button isn't gated on `hint_level < 2`, or never relabels `"💡 HINT"` → `"NEED MORE HELP?"`, or reveals `hint_1` and `hint_2` together (revealed hints prefixed `💡` / `💡💡`). *(Quiz reference)*
3. **Hints persist on reveal.** FAIL if already-revealed hint(s) are not rendered **before** the button/spinner (render-then-fetch), so a click can't blank a shown hint.
4. **On-demand generation.** FAIL if hint / explanation / deep-dive auto-generate (no button), or the button stays visible beside the spinner instead of being replaced by one `st.spinner`.
5. **END ROUND in the sidebar.** FAIL if there is no `st.sidebar` "END ROUND" button while `screen == "quiz"`.
6. **No per-option ✅/❌.** FAIL if answer options are annotated with per-option correct/incorrect markup, strikethrough, or color after submit (the result is a single badge + the correct-answer line only).
7. **DEEP DIVE exists, inside the explanation expander.** FAIL if there is **no** `🔬 DEEP DIVE` control inside `_render_explanation_expander` (it must exist), OR if a deep-dive button/section renders anywhere outside it (it is topic-level, **no** per-option picker) - and see check 11.
8. **NEXT pinned last.** FAIL if "NEXT"/"FINISH ROUND" renders before the explanation expander, or both render at once (mutually exclusive, full-width, at the very bottom).
9. **Explanation on-demand for all.** FAIL if the "💡 AI EXPLANATION" button is hidden for correct answers (it appears for correct + incorrect alike).

### Summary screen (`pages/quiz.py`, `screen == "summary"`)
10. **TO REMEMBER, not WRONG ANSWERS.** FAIL if the missed-questions expander isn't `st.expander("TO REMEMBER", expanded=True)`, OR shows **bare letters**, OR shows the user's pick (must be the correct answer in **full text** only).
11. **No deep dive in the summary.** FAIL if any `deep`/`🔬` reference appears in the summary section (deep dive is a quiz-screen concept only).
12. **"ROUND SUMMARY" naming.** FAIL if a **button or section-heading label** reads "Round Brief" or "Debrief" (the on-demand button + expander heading must read "ROUND SUMMARY"; the internal `debrief` state key and code comments are fine - check rendered label text only). The debrief is on-demand, gated by `debrief_enabled`, only when ≥1 wrong.

### Review page (`pages/review.py`)
13. **Full-text correct answer.** FAIL if the wrong-answer card shows a bare letter instead of the full-text `CORRECT_ANSWER` passthrough. *(judgment - read the card)*
14. **Both filters present.** FAIL if there is no domain `st.pills` filter, or no always-rendered `st.date_input` range (must render even when `min == max`).
15. **Mnemonic guard.** FAIL if `st.info(🧠 …)` renders without the `mnem and mnem.lower() != "none"` guard.

### Admin page (`pages/admin.py`)
16. **Four tabs.** FAIL if Admin isn't four `st.tabs` - App config · Questions manager · Cortex spend · Logs.
17. **App config is minimal.** FAIL if App config renders any selector for `question_source` / `round_size`/`default_round_size` / `difficulty` / `grounding_mode` / `pass_threshold` (it holds ONLY the two toggles + three model selectboxes).
18. **Config writes are safe.** FAIL if any config write uses `TO_VARIANT(json.dumps(` or an inline config `MERGE`/`INSERT` in the page (must route through `save_config()` / `PARSE_JSON(?)`), OR if a config-seeded widget uses a bare `options.index(` instead of `cfg_index(`.
19. **Questions manager has the editable table.** FAIL if it isn't nested `st.tabs(["Bank","Generate"])`, OR the Bank tab lacks an `st.data_editor` with a `select` `CheckboxColumn` (+ disabled other columns), OR renders the KPIs as a table instead of `st.metric`.
20. **Batch count = slider.** FAIL if the Generate batch count uses `st.number_input`, or the slider max exceeds 20.
21. **Batch feedback.** FAIL if batch generation doesn't run under an `st.spinner` AND fire an `st.toast` AND show a transient `:green-badge[Added N …]` line.
22. **Spend branches structurally.** FAIL if the Cortex-spend tab reads `METERING_DAILY_HISTORY` or the deprecated `CORTEX_FUNCTIONS_USAGE_HISTORY` (must be `CORTEX_AISQL_USAGE_HISTORY`), OR uses a generic `except → GRANT` instead of the three-way branch (empty-on-success → caption; privilege signal → GRANT with the live `CURRENT_ROLE()`; other error → caption).
23. **Logs reset is here and frameless.** FAIL if the "Reset all logs" button isn't in the Logs tab (never App config), isn't `type="tertiary"`, or uses a type-`DELETE` text gate instead of a bordered two-step `pending_reset` Confirm/Cancel; Logs also carries a domain filter applied in Python.

### Cross-cutting
24. **No exam-code caption.** FAIL if `main.py`/`pages/quiz.py` renders the exam code as a subtitle/caption under a page title.
25. **Loader return-type contract.** FAIL if any of `load_review_log` / `load_session_log` / `load_recent_sessions` / `load_domain_errors` / `load_bank_stats` / `load_domains` / `load_cortex_spend` is consumed with DataFrame ops (`.empty` / `.iterrows` / `.dropna` / `.isin` / `.head`) - they return `.collect()` `list[Row]`; only `load_questions_page` and `load_review_log_editable` are `.to_pandas()` DataFrames (the `st.data_editor` tables).
26. **Grounding style** *(judgment - read the prompts)*. The explanation / hint / deep-dive calls are **teaching** calls: ground in the retrieved `<doc_context>` but explain in the model's own words, at most one short cited passage. FAIL if any of these prompts in `_cortex.py` instead instructs strict fact-extraction - e.g. "answer ONLY from the provided documentation", "do not use prior knowledge", "quote/excerpt the docs" - OR fails to tell the model to explain/teach in its own words. (The strict fact-extraction phrasing belongs ONLY to question/batch/flashcard generation - `$cortex`, "Grounded ≠ parroting".)
27. **Status via badges only.** FAIL if `st.success` / `st.warning` / `st.error` appears anywhere, or `st.info` is used for anything other than the mnemonic 🧠 box (`$quiz/design`).

**Verdict:** all PASS → "UX-conformance gate clean." Any FAIL → "Fix [list] before deploy" with `file:line` + the reference block to copy. A clean `$sis` scan + a clean gate are **both** required to deploy.

---

## Output

Code that conforms to page flow contracts, session state schema, and write-back + cache-invalidation patterns.
