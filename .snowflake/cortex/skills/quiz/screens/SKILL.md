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

**Config (de)serialization — COPY this; it is the #1 config crash.** `QUIZ_CONFIG.config_value` is `VARIANT`. The round-trip is **`PARSE_JSON(?)` on write + `json.loads` on read** — and it goes through the **one `save_config()` helper**. Do NOT inline a config MERGE in a page, and **NEVER `TO_VARIANT(json.dumps(value))`** — that double-encodes (stores `"\"ai\""`, reads back the literal `'"ai"'`) and then `options.index('"ai"')` raises `ValueError`. (`PARSE_JSON` here is in a MERGE `USING` sub-select, which is allowed — the `$sis` "no PARSE_JSON inside `VALUES(`" rule is only about `INSERT … VALUES`.) VARIANT is kept (matches the DDL + the Step-1g `grounding_mode` seed); a plain `VARCHAR` column would also work but needs no schema change, so don't.
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
**Guard every config-seeded widget default** — a stored value that isn't in the options list must never crash the page (`ValueError`/`StreamlitAPIException`). Use a helper, never a bare `options.index(cfg[key])`:
```python
def cfg_index(options, value, default=0):       # _ui.py
    return options.index(value) if value in options else default
# st.selectbox("Model", MODEL_OPTIONS, index=cfg_index(MODEL_OPTIONS, cfg.get("model_generation", CORTEX_MODEL)))
```

**Entry point (`main.py`)**: `st.set_page_config` (first `st.` call) -> `init_session_state()` -> shared sidebar title -> `st.navigation(pages).run()`. Pages share `st.session_state` (it persists across page switches).

**Inside `pages/quiz.py`** the three screens are an internal state machine driven by `st.session_state["screen"]` (`home` / `quiz` / `summary`) - they are NOT separate pages.

**Inside `pages/review.py`** the tabs are `st.pills` with `st.divider()` and `st.title()` per tab, driven by `_review_page` - `WRONG ANSWERS` and `LEARNING DASHBOARD`.

Cross-page redirects: set the target state, then `st.switch_page("pages/<target>.py")`.

---

# Home Screen (`pages/quiz.py`, screen == "home")

4 control groups in order, each with a bold UPPERCASE label (`st.markdown("**LABEL**")`) and `label_visibility="collapsed"` on the widget:

1. **QUESTIONS** - `st.slider("Questions", 1, 100, value=…)` so **any** count 1-100 is pickable. **NOT `st.pills`** (a fixed list like 5/10/15/20 is wrong) and not `number_input`. Initial `value` from `round_size` if set, else the `_config.py` default. The round size is set here per round - it is NOT an Admin config value.
2. **DOMAINS** - pills multi-select from EXAM_DOMAINS, initial selection from `domain_filter` if set
3. **DIFFICULTY** - pills (mixed/easy/medium/hard), guard against None, initial value from `difficulty` if set
4. **SOURCE** - pills multi-select (`["QUESTION BANK", "AI GENERATED"]`), mapped internally to `"mix"/"db"/"ai"`

Each control **seeds its initial value from the matching session key** (`round_size`, `domain_filter`, `difficulty`) when present, so a cross-page redirect that sets those keys pre-fills the round.

The explanation is on-demand per question (a button after answering), never auto-loaded.

**Grounding guard** (top of Home, `cke`/`custom` mode): if `not docs_available()` (`$cortex`/`_search.py`), show a red "install/grant the Snowflake Documentation CKE - the app can't generate until it's reachable" message and **disable Start Round**. The app must never generate from built-in knowledge. (In `none` mode there is no guard - generation is intentionally ungrounded.)

**Start Round** button: saves settings to session state, builds topic schedule via `_build_topic_schedule()` (from `_questions.py`), loads first question with spinner, sets `screen="quiz"`, reruns.

---

# Quiz Screen (`pages/quiz.py`, screen == "quiz")

**Lazy load**: At top of the quiz screen, if `question` is None, load via `get_question()` with spinner, then rerun for clean render.

**Layout**: progress bar -> domain/difficulty badges -> h4 question text -> answer input -> submit -> result + explanation -> navigation

**Answer input**: `st.radio()` for single-answer (`index=None`, disabled once answered). For multi-answer, render each option as an independent `st.checkbox` with a stable key (`cb_A`, `cb_B`, …); read the selection from session state after rendering; disable all once answered. On "Next", clear the `cb_*` keys via the flag-at-top reset (queue `["cb_A", …, "cb_E"]` in `_op_clear_keys`, rerun, pop at the top - see `$sis` widget lifecycle).

**Socratic hint (BEFORE answering; gate `hints_enabled`)**: a secondary "💡 Hint" button near the answer input, visible ONLY while `answered == False`. First click → `call_cortex_json(prompt, "hint", model=model_for("explanation"))`, show `hint_1` and **relabel the button to "Need more help?"**; second click reveals `hint_2`, then hide the button. **Render order:** always render the already-revealed hint(s) FIRST, then - on a click - hide the button and show a spinner *below* the visible hint while the next generates (render-then-fetch, never fetch-then-render), with a single `st.rerun()` after, so an already-shown hint stays on screen. The hint is **grounded like every other generation** - in `cke`/`custom` mode embed retrieved `<doc_context>` in the hint prompt and derive the hints from it, never the model's built-in knowledge (`$cortex`); the prompt MUST instruct: hints narrow the concept space (level 1) or eliminate ONE distractor with reasoning (level 2) and must NEVER name or imply the correct option; embed question/options per the untrusted-content delimiting rule. State: `hint` (None/{}/dict), `hint_level` (0/1/2); once answered, the button disappears (the explanation takes over); record `hint_used = hint_level > 0` in the history item; reset both on Next.

**Submit**: Records result in `round_history` (incl. `hint_used`), increments counters, sets `answered=True`, reruns. Does NOT call Cortex.

**After submission** - render **stacked and full-width, top to bottom** (NOT a `st.columns(2)` row):
1. the **result badge**
2. the **"💡 AI explanation"** button (full-width, secondary) → loads the explanation **on demand** (Explanation Contract below); while it generates, hide the button and show a labelled spinner (On-demand generation UX below); once loaded, the expander renders **here**
3. **"Next"** (primary, full-width; **"Finish Round"** on the last question) at the **very bottom - beneath the whole explanation expander** - so the next action sits directly under what the user just read

The explanation is NEVER auto-generated, and the button appears for **correct answers too** (to learn why the distractors are wrong). Next stays pinned at the bottom whether or not the explanation is open. Do NOT render "Next" and "Finish" together. Do NOT add per-option markup (✓, strikethrough) - the explanation handles details.
- Correct: `:green-badge[✅ CORRECT]`
- Incorrect: `:red-badge[❌ INCORRECT]` + newline + `Correct answer: **A**, **C**` (bold letters only, not full option text)

**Sidebar "End Round"**: When `screen == "quiz"`, the sidebar shows an "End Round" button (secondary, full-width). Clicking sets `_pending_finish = True` and does a natural rerender.

**Button click safety**: guard slow-action buttons (Start Round, Submit, Next, Finish) with the `_transitioning` flag so a double-click can't double-fire during the rerun:
```python
if st.button("Next", disabled=st.session_state.get("_transitioning", False)):
    st.session_state["_transitioning"] = True
    # ... action ...
    st.session_state["_transitioning"] = False
    st.rerun()
```
Pair this with the spinner + single-`st.rerun()` rule in `$sis`.

**On-demand generation UX (hint, explanation, deep dive, Round Summary - MANDATORY).** When a button triggers a slow Cortex call: (1) render any already-generated content **first**; (2) **hide the button(s) that trigger that generation** while it runs - never leave a sibling sitting there dimmed; (3) show exactly **one** labelled `st.spinner("Generating <thing>…")`; (4) generate; (5) a single `st.rerun()`. Same pattern for all four.

---

# Explanation Contract

**On-demand only** - generated when the user clicks **"💡 AI explanation"**, never automatically. State machine in `st.session_state["explanation"]`: `None` (not requested yet - just show the button), `{}` (tried and failed; do NOT retry), `{dict}` (success; render the expander). On "Next": reset to `None`.

`_generate_explanation()` calls `call_cortex_json(prompt, "explanation", model=model_for("explanation"))` - the `RESPONSE_FORMATS["explanation"]` schema (`$cortex`) guarantees `why_correct` (array), `why_wrong` (object), `mnemonic`, `doc_search`. No fence parsing; retry only on `None`.

**Model routing (MANDATORY) for the learning loop** (`$cortex` `model_for`): explanation, **hint**, and **deep dive** pass `model=model_for("explanation")`; the **Round Summary debrief** passes `model=model_for("meta")`. Every `call_cortex_json`/`call_cortex` in these handlers takes the `model=` arg - omitting it silently pins the call to `CORTEX_MODEL` and the Admin model selector does nothing.

**Same flow for correct AND incorrect answers** - clicking the button opens `st.expander("💡 AI EXPLANATION", expanded=True)` containing, in order:
- `st.container(border=True)` **✅ WHY CORRECT** - `why_correct` bullet list
- `st.container(border=True)` **WHY WRONG** - one line per *other* option (on a correct answer this is exactly the value: learn why the distractors are wrong)
- `st.info()` mnemonic
- the doc link **only**: `📖 [Snowflake Documentation]({doc_url})` - **NO "📚 From the docs" heading, NO `DOCUMENT_TITLE` caption, and NO raw `CHUNK` excerpt** (the chunk renders an internal name + a truncated definition full of `¶` glyphs - drop it entirely; grounding still *uses* the chunk to write `why_correct`/`why_wrong`, the UI just shows the clean link)
- the **Deep dive** control (below)

The 📖 doc link appears **only inside this expander, only after the button is clicked** - never auto-shown (not even for correct answers). Render every dynamic field (`why_correct`, `why_wrong`, `mnemonic`) through the `_ui.py` `md()` escaper before `st.markdown`/`st.info` (`$quiz/design` - escaping dynamic text). Store `mnemonic` + `doc_url` on `current_history_item` for the review log.

**Doc grounding is MANDATORY in `cke`/`custom` mode (`$cortex`)**: retrieve `chunks = search_docs(question_text)` once; if `[]`, broaden once, else **fail visibly** (no built-in). Embed the top chunk(s) in the prompt as `<doc_context>`; `doc_url = chunks[0]["SOURCE_URL"]`. This is a **teaching** call — use the **teaching grounding style** (`$cortex` "Grounded ≠ parroting"): ground every claim in the chunks, but **EXPLAIN the concept in your own words**; do NOT use the bare *"answer ONLY from the provided documentation"* line, do NOT quote the docs line-by-line, and do NOT write every bullet as "the documentation says…". **`none` mode only**: the prompt asks for `doc_search` ("2-3 words, no URLs/commas") → `https://docs.snowflake.com/en/search?q={query}`.

**Explanation prompt content** - the schema guarantees shape, the prompt controls quality. Lead with: *"You are a SnowPro tutor. Explain comprehensively and holistically WHY the correct answer is right and each distractor is wrong — teach the underlying concept so it sticks. Ground every claim in `<doc_context>` but write in your own words; cite at most one short passage."* Then the fields:
```
"why_correct": ["First reason — a real explanation of the mechanism, not 'the docs say X'", "Second reason with technical detail", "Optional third"],
"why_wrong": {"X": "one sentence explaining the actual misconception behind option X", "Y": "..."},
"mnemonic": "a memorable phrase or acronym for the correct answer",
"doc_search": "exactly 2-3 words for Snowflake docs search. No URLs. No commas. Max 3 words."
```

## Deep dive (inside the expander, below the explanation)

A single **"🔬 Deep dive"** button - **NO option picker** → `call_cortex_json(prompt, "deep_dive", model=model_for("explanation"))` - an in-depth breakdown of the **question's topic** (how it works / when to use / exam traps), not a single answer option. `$cortex` `deep_dive` schema: `summary`, `how_it_works[]`, `when_to_use`, `exam_traps[]`. Render in `st.container(border=True)` with bold sub-labels + bullets. **Grounded** like the explanation, and the **same teaching style** (`$cortex` "Grounded ≠ parroting"): reuse the retrieved `<doc_context>`, broaden once then **fail visibly** on empty - never built-in; embed the **question + its topic/domain** per the delimiting rule (not a selected option); EXPLAIN the topic in your own words (do not parrot the docs), carry **more depth** than the base explanation; render through the `_ui.py` `md()` escaper. State: `deep_dive` (None/{}/dict), reset on Next.

---

# Reference code (COPY + ADAPT — the high-fidelity-risk handlers)

The prose above is the contract; this is the **reference implementation of the parts that are repeatedly gotten wrong** (home slider, hint state machine, on-demand spinner, stacked after-submit layout, deep-dive-in-expander, sidebar End Round). **Copy these handlers and adapt** (swap `EXAM_NAME`, wire your `_cortex`/`_search`/`_data` helpers) — do not re-derive them from the prose. The Step-8 UX-conformance gate checks that the generated `quiz.py` matches these shapes.

```python
# pages/quiz.py — reference for render_home / render_quiz (hard parts). LETTERS = ["A","B","C","D","E"]

def render_home():
    cfg = load_config(); domains = load_domains()
    if grounding_required() and not docs_available():
        st.markdown(":red-badge[DOC GROUNDING UNAVAILABLE] Install/grant the Snowflake Documentation CKE."); st.stop()
    st.markdown(f"## {EXAM_NAME} Quiz")
    st.markdown("**QUESTIONS**")
    round_size = st.slider("Questions", 1, 100, value=st.session_state.get("round_size", 10),
                           key="sl_round_size", label_visibility="collapsed")          # SLIDER, not pills
    st.markdown("**DOMAINS**")
    dom = st.pills("Domains", [d["DOMAIN_NAME"] for d in domains], selection_mode="multi",
                   default=st.session_state.get("domain_filter") or [], key="pills_domains", label_visibility="collapsed")
    st.markdown("**DIFFICULTY**")
    diff = st.pills("Difficulty", ["mixed","easy","medium","hard"],
                    default=st.session_state.get("difficulty","mixed") or "mixed",
                    key="pills_diff", label_visibility="collapsed") or "mixed"
    st.markdown("**SOURCE**")
    src_sel = st.pills("Source", ["QUESTION BANK","AI GENERATED"], selection_mode="multi",
                       default=["QUESTION BANK","AI GENERATED"], key="pills_src", label_visibility="collapsed") or ["AI GENERATED"]
    source = "mix" if len(src_sel) == 2 else ("db" if "QUESTION BANK" in src_sel else "ai")
    if st.button("Start Round", type="primary", use_container_width=True,
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

    with st.sidebar:                                              # END ROUND — always present in quiz
        if st.button("End Round", use_container_width=True, key="btn_end_round"):
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

    # answer input (radio single / checkbox multi), disabled once answered — NO per-option ✅/❌ markup
    if not q.get("IS_MULTI"):
        chosen = st.radio("Answer", [f"{lt}) {md(t)}" for lt, t in options.items()], index=None,
                          disabled=answered, key=f"radio_{idx}", label_visibility="collapsed")
        selected = [chosen.split(")")[0]] if chosen else []
    else:
        st.markdown(":gray-badge[SELECT ALL THAT APPLY]")
        selected = [lt for lt in options if st.checkbox(f"{lt}) {md(options[lt])}", key=f"cb_{lt}", disabled=answered)]

    if not answered:
        # HINT — state machine: render revealed hints FIRST, then the button (relabels, hides at level 2)
        if cfg.get("hints_enabled", True):
            hint, lvl = st.session_state.get("hint"), st.session_state.get("hint_level", 0)
            if isinstance(hint, dict):
                if lvl >= 1 and hint.get("hint_1"): st.markdown(f"💡 {md(hint['hint_1'])}")
                if lvl >= 2 and hint.get("hint_2"): st.markdown(f"💡 {md(hint['hint_2'])}")
            if lvl < 2:
                if st.button("💡 Hint" if lvl == 0 else "Need more help?", key="btn_hint"):
                    with st.spinner("Thinking of a hint…"):          # on-demand: button replaced by spinner
                        _generate_hint(q)
                    st.session_state["hint_level"] = lvl + 1; st.rerun()
        if st.button("Submit", type="primary", use_container_width=True,
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
            if st.button("💡 AI explanation", use_container_width=True):
                with st.spinner("Generating explanation…"):
                    _generate_explanation(q, options)
                st.rerun()
        if isinstance(expl, dict) and expl:
            _render_explanation_expander(q, options)    # WHY CORRECT/WRONG + mnemonic + 📖 link + 🔬 Deep dive (topic, no picker)

        is_last = (idx + 1) >= total
        if st.button("Finish Round" if is_last else "Next", type="primary", use_container_width=True,
                     disabled=st.session_state.get("_transitioning", False)):
            _advance(is_last)    # write-back if last/pending, else load next + reset hint/explanation/deep_dive
```

Notes the gate enforces: round size is a **slider**; the hint **state machine** (no double-dump, button hides at level 2, prior hints persist); **End Round** in the sidebar; the explanation is **stacked** with **Next pinned last**; **Deep dive lives inside `_render_explanation_expander`** (the quiz screen, on the question's topic) — **never** in the summary; no per-option ✅/❌ markup; the missed-question summary shows **full-text** correct answers only.

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
| `doc_url` | str | Empty initially; filled from `doc_search` |

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

**Round Summary (on-demand; gate `debrief_enabled`; only when ≥1 wrong answer)**: a **"Round Summary"** button - NOT auto-generated. On click, **hide all summary buttons and show one spinner "Generating round summary…"** (On-demand generation UX, Quiz Screen - don't leave "Configure New Round" dimmed beside it) → `call_cortex_json(prompt, "debrief", model=model_for("meta"))` with per-question domain/topic/correctness/`hint_used` from `round_history` → open an **expander** with clean formatting: `st.container(border=True)` holding **PATTERNS** (bullets) and **PRIORITY ACTIONS** (max 3 bullets), then `one_thing` as a highlighted **🎯 FOCUS** line in its own bordered container (NOT `st.info` - that's reserved for the mnemonic per `$quiz/design`). **Grounding scope:** meta-analysis over the user's OWN round performance - names weak domains/topics + study actions but MUST NOT assert new Snowflake facts or emit doc links; the one runtime generation path exempt from doc-grounding (`$cortex`). State `debrief` (None=not requested / {}=failed / dict=success), reset on round start. A perfect round shows no Round Summary button. Render every dynamic field (PATTERNS, PRIORITY ACTIONS, `one_thing`) through the `md()` `$`-escaper before `st.markdown` (`$quiz/design`).

**Action buttons (by outcome)** - `Round Summary` applies only when there is ≥1 wrong answer.
- **Perfect** (all correct): **"Configure New Round"** only.
- **Passed, some wrong**: **"Round Summary"** + **"Configure New Round"**.
- **Failed**: **"Round Summary"** + **"Configure New Round"**.
- Threshold = `PASS_THRESHOLD` (the fixed study proxy - not user-overridable; there is no `pass_threshold_override`). All buttons set state and call `st.rerun()`.

---

# Review: Wrong Answers (`pages/review.py`)

Query `QUIZ_REVIEW_LOG` via the cached loader in `_data.py` (`@st.cache_data` with NO ttl, `get_active_session()` inside - see `$sis` Caching). Do NOT query directly in the page code. Freshness comes from `clear_caches()` at write time, not from a ttl.

Filters: domain pills (multi, empty=all) + a **date-range `st.date_input`** - **always shown**, even when all wrong answers fall on one day (do NOT hide it when `min == max`; pass that single date as both `value` ends + `min_value`/`max_value`). Keep rows whose cast `LOGGED_AT` date is within `[start, end]`.

Wrong answer cards: `st.container(border=True)` with domain badge + difficulty badge + date badge, question text, correct answer, mnemonic caption, doc link caption.

**Date handling** (filters + dashboard): values from `.collect()` are Snowflake datetimes - cast with `datetime.date(raw.year, raw.month, raw.day)` before feeding any widget or doing date arithmetic. The Wrong-Answers date filter is an **`st.date_input` range** (always rendered - see above); compare each row's cast `LOGGED_AT` date against the selected `[start, end]` in Python. For any `TIMESTAMP_LTZ` range query in SQL, pass dates as `strftime("%Y-%m-%d")` strings with an exclusive upper bound (`< end + 1 day`) to include the full last day. Never pass a `datetime.date` to `st.slider` (`$sis`).

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

## 1. App config

**Two toggles** - `hints_enabled`, `debrief_enabled` ("Round Summary"). Plus **AI model per call-group** (3 `st.selectbox`, options from `_config.py` `MODEL_OPTIONS = ["claude-haiku-4-5", "claude-sonnet-4-6", "claude-opus-4-8"]`):

| Selector (label) | Config key | Covers |
|---|---|---|
| Question generation | `model_generation` | `get_question`/`generate_ai_question`, Admin Generate batch |
| Explanations & study aids | `model_explanation` | explanation, hint, deep dive |
| Meta-analysis | `model_meta` | Round Summary debrief |

Each defaults to `CORTEX_MODEL` (`load_config().get(f"model_{group}", CORTEX_MODEL)`); the call sites read it via `model_for(group)` and pass it to `call_cortex`/`call_cortex_json` (`$cortex`). The Cortex-spend "by model" chart reflects these choices.

**Do NOT show:** `grounding_mode` (fixed at setup, `$setup-exam` Step 1g - not a runtime toggle, so don't surface it at all), `pass_threshold_override` (the official threshold doesn't change - removed), `default_round_size` (set on Home before each round - redundant), and `question_source` / `difficulty` (these are **per-round Home choices**, never global config - re-adding them here as selectboxes is the exact app-v4 regression to avoid; App config holds ONLY the two toggles + the three model selectboxes). The only grounding UI here is the guard: in `cke`/`custom` mode, if `docs_available()` is False, a red caption - "The doc grounding service is unavailable - install/grant the Snowflake Documentation CKE; the app can't generate until it's reachable." Every change → the **`save_config()` helper** (the `PARSE_JSON` round-trip from Config layer — NEVER inline a config MERGE here, NEVER `TO_VARIANT(json.dumps())`) → `clear_caches()` (clears `docs_available`/`search_docs` too) → `st.toast`. The three model `st.selectbox`es seed their `index` via `cfg_index(MODEL_OPTIONS, cfg.get(f"model_{group}", CORTEX_MODEL))` (never a bare `.index()` — see Config layer).

## 2. Questions manager

Two nested `st.tabs` - **Bank** and **Generate**.

**Bank** - KPIs above an editable table (the vendors02 reference pattern):
- **KPIs as `st.metric`, NOT a table**: Total questions · MANUAL · AI_GENERATED · domains covered (in `st.columns`, `$quiz/design` KPI cards). Backed by **`load_bank_stats()`** - a no-ttl cached coverage loader (per domain × difficulty × source) in `_data.py`, registered in `clear_caches()` (`$sis` item 24). The editable table below uses its own filtered loader (next).
- **Editable / filterable / deletable table**:
  1. **Filters** in an `st.expander("Filters", expanded=True)` - domain / difficulty / source pills - then a **"Search"** button commits them to a `_qm_filters` session dict (don't query live off widget state). A "Reset filters" button.
  2. **`Select All` / `Clear`** buttons above the table (toggle a `_qm_select_all` flag).
  3. **`st.data_editor`** with a leading `select` `st.column_config.CheckboxColumn`; **every other column `disabled`**; read the selection back as `edited[edited["select"]]`. The table's source is a **no-ttl cached loader returning a pandas DataFrame** (`.to_pandas()`) - `data_editor` preserves its checkbox selection across reruns ONLY when its input is byte-identical (a ttl that expired mid-edit would wipe the selection - `$sis` caching), and `.to_pandas()` columns come back UPPERCASE so there is **no `Row` access at all** - selected-row fields are read as `row["QUESTION_TEXT"]`, never `.get()`/attr on a `Row` (`$sis` scan item 20).
  4. **`Load 10 more`** (page cap 10 - don't render 1000 rows; paginate via a `_qm_limit` that grows by 10).
  5. Below the table: **`Edit` · `Delete` · `Add`**.
     - **Edit** (enabled when exactly 1 row selected): loads that row into the form below → UPDATE by `question_id`.
     - **Add**: opens the same form **empty** → INSERT `source='MANUAL'`.
     - The form (Edit + Add) is an **`st.form(clear_on_submit=True)`** so all fields commit together on submit and the form resets after Insert (fixes the per-field Cmd+Enter + "form stays filled" problem): question `st.text_area`, options A-E `st.text_input`, `correct_answer` `st.multiselect` **restricted to the NON-EMPTY option values** (read inside the form on submit, so no per-field Enter), difficulty pills; `is_multi` derived = `len(correct) > 1`.
     - **Delete** (enabled when ≥1 selected): **two-step confirm** - a bordered `pending_delete` panel ("Delete N question(s)?") with **Confirm / Cancel** (vendors02 `pending_*` pattern, NOT a type-`DELETE` text gate) → `DELETE FROM QUIZ_QUESTIONS WHERE question_id IN (?, …)`.
  - **Hard rules** (`$sis` scan item 21): every write via bind params (`?`, NEVER f-string); length caps in the form AND by truncation (question 2000, options 500); `correct_answer` ⊆ non-empty options; ≥2 options. Every write → `clear_caches()` → `st.toast`.
  - **Editor-state discipline (the hard part - follow vendors02 `01_ai_recommendations.py`):** keep a signature of the rendered slice (tuple of `QUESTION_ID`s). On a **non-append** change (filters/Search/Delete changed the set) drop the `data_editor` widget key and reset `_qm_select_all` so the checkbox column re-seeds cleanly; on a **pure append** (Load-10-more extends the slice) keep the selection. Without this, selection jumps on every Load-more/Search.
  - **Edit vs Add form (one form, a mode flag):** `_qm_mode` ∈ `"add"`/`"edit"` (+ `_qm_edit_id` for edit). **Add** renders the empty `st.form(clear_on_submit=True)` → INSERT. **Edit** seeds the form from the selected row by writing the field values into the widget `session_state` keys at the **top of the run, before the widgets render** (flag-at-top - `$sis` widget lifecycle); **never pass both `value=`/`default=` and also set the `session_state` key** (raises "created with a default value but also had its value set"). Switching Add↔Edit clears the prior field keys via the same flag-at-top reset.

**Generate** - Generate batch with options: **count** (`st.slider`, 1-20 - each item is its own grounded LLM call, so the batch is capped to stay well under the statement timeout; never an unbounded `st.number_input`), **difficulty** (pills, incl. "mixed"), **domain** (`st.selectbox`/multiselect over `EXAM_DOMAINS`), and **model** (`st.selectbox` over `MODEL_OPTIONS`, `index` seeded via `cfg_index(MODEL_OPTIONS, cfg.get("model_generation", CORTEX_MODEL))`) → generates via the **same grounded `_questions.py` path** (`$quiz/questions`: in `cke`/`custom` mode each question embeds retrieved `<doc_context>` as the primary source with `key_facts` as supporting scope, answers ONLY from the docs, fails visibly on empty retrieval - never built-in) → INSERT `source='AI_GENERATED'` → `clear_caches()`. **Feedback is mandatory** (the app-v4 "silent batch" bug): run the loop under one `st.spinner`, then on completion fire an `st.toast` **and** render a transient `:green-badge[Added N question(s)]` line (NEVER `st.success`/`st.warning` - `$quiz/design`); a `st.caption` notes larger batches take longer and cost more. The batch passes its chosen model through `call_cortex_json(..., model=…)`.

## 3. Cortex spend (graceful - distinguish "no grant" from "no data")

`_cortex.py` sets a session `QUERY_TAG` (JSON: app, feature, model) per call. Read `SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY` inside try/except and branch **structurally**, not "any-exception → GRANT" (an ACCOUNTADMIN holds `IMPORTED PRIVILEGES` by default, so blaming every empty/error result on permissions shows a false GRANT banner):
- **Success path** - the query returned. If the result is **empty** → a plain caption: "No Cortex spend recorded yet - ACCOUNT_USAGE lags up to ~2 h, or no AI calls have run." (This is the ACCOUNTADMIN-on-a-fresh-account case; ACCOUNTADMIN holds `IMPORTED PRIVILEGES` by default, so **never** show the GRANT banner here.) Otherwise render the charts.
- **Exception path** - inspect the error. Only when its message signals a **privilege/visibility problem on the SNOWFLAKE share** (e.g. it contains `Insufficient privileges`, `not authorized`, or `does not exist or not authorized` - the symptom of a role lacking `IMPORTED PRIVILEGES`) → the info banner with the exact `GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <role>;` (substitute the live `CURRENT_ROLE()`). For any **other** exception → a generic "couldn't read Cortex spend: {error}" caption, NOT the GRANT banner.

Charts: spend by feature, spend by model (now a haiku/sonnet/opus split, reflecting the App-config model choices). Caption: ACCOUNT_USAGE lags up to ~2 h. When doc grounding is on, add a "docs search" line (Cortex Search query compute is billed to the consumer; small per query).

## 4. Logs (was "Tools")

A read-only log viewer plus a reset - the vendors02 logs page, no download:
- **Both log tables shown**: `QUIZ_REVIEW_LOG` via `load_review_log()` and `QUIZ_SESSION_LOG` via **`load_session_log()`** - a new no-ttl cached loader for the full session-log table (`load_recent_sessions()` is the dashboard's last-10 aggregate, NOT this); **add `load_session_log` to `_data.py` and register it in `clear_caches()`** (`$sis` item 24, else it dangles). Read-only `st.dataframe`, newest first. **No `st.download_button`.**
- **Filters + paging** (`QUIZ_REVIEW_LOG`): a **domain `st.multiselect`** (empty = all) applied in pandas over the cached frame, plus `Load N more` paging via a page-local `_log_limit` (don't render thousands of rows). The session-log table is shown as-is, newest first.
- **Reset all logs** - a **frameless/borderless button** (`st.button(..., type="tertiary")`) below the tables, with **no expander and no "DANGER ZONE" label**, then a **two-step confirm** (a bordered `pending_reset` panel with Confirm / Cancel - NOT a type-`DELETE` text gate). Confirm runs `DELETE FROM` on the **two log tables only** (`QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`) - **NEVER `DROP`**, consistent with governance - then `clear_caches()` + `st.toast`. The reset lives **in this Logs tab**, never in App config (the app-v4 misplacement).

## Reference code (COPY + ADAPT — Admin handlers)

The prose above is the contract; this is the **reference implementation of the Admin parts that one-shot generation gets wrong** (app-v4 shipped App config with the wrong fields, a Questions manager with no editable table, a spend panel that blamed every empty result on permissions, and the log reset on the wrong tab). **Copy these handlers and adapt** — do not re-derive them from the prose. The Step-8 UX gate checks `admin.py` against these shapes. All four are `st.tabs` contents in `pages/admin.py`; `LETTERS = ["A","B","C","D","E"]`; `save_config`/`load_config`/`cfg_index`/`clear_caches` are the Config-layer helpers; `SCHEMA`/`MODEL_OPTIONS`/`CORTEX_MODEL` are `_config.py` constants.

```python
# 1. APP CONFIG — ONLY two toggles + three model selectboxes. No source/round/difficulty/grounding/threshold.
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


# 2. QUESTIONS MANAGER — nested tabs; Bank = KPIs + editable/filterable/deletable table (the app-v4 gap).
def render_questions_manager():
    bank_tab, gen_tab = st.tabs(["Bank", "Generate"])
    with bank_tab:    render_bank()
    with gen_tab:     render_generate()

def _is_append(prev, sig):                                  # append = prev is a prefix of the new id slice
    return len(sig) >= len(prev) and sig[:len(prev)] == prev

def render_bank():
    s = load_bank_stats()                                   # cached coverage loader; KPIs are st.metric, NOT a table
    cols = st.columns(4)
    cols[0].metric("Total", s["total"]);   cols[1].metric("Manual", s["manual"])
    cols[2].metric("AI", s["ai"]);         cols[3].metric("Domains", s["domains"])

    with st.expander("Filters", expanded=True):             # commit filters on Search, don't query live off widgets
        f_dom  = st.multiselect("Domain", [d["DOMAIN_NAME"] for d in load_domains()], key="qm_f_dom")
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

    edited = st.data_editor(df, key="editor_qm", hide_index=True, use_container_width=True,
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
    n     = st.slider("Count", 1, 20, 5, key="gen_n")       # SLIDER, capped — each item is a grounded LLM call
    diff  = st.pills("Difficulty", ["mixed","easy","medium","hard"], default="mixed", key="gen_diff") or "mixed"
    doms  = st.multiselect("Domains", [d["DOMAIN_NAME"] for d in load_domains()], key="gen_dom")
    model = st.selectbox("Model", MODEL_OPTIONS,
        index=cfg_index(MODEL_OPTIONS, cfg.get("model_generation", CORTEX_MODEL)), key="gen_model")
    st.caption("Larger batches take longer and cost more — each question is its own grounded generation.")
    if st.button("Generate batch", type="primary", disabled=st.session_state.get("_gen_busy", False)):
        st.session_state["_gen_busy"] = True; made = 0
        with st.spinner(f"Generating {n} question(s)…"):
            for _ in range(n):
                if generate_ai_question(diff, doms or None, model=model):   # same grounded _questions.py path
                    made += 1
        clear_caches(); st.session_state["_gen_busy"] = False
        st.session_state["_gen_result"] = made
        st.toast(f"Generated {made} question(s)"); st.rerun()
    if "_gen_result" in st.session_state:                   # transient feedback — never st.success ($quiz/design)
        st.markdown(f":green-badge[Added {st.session_state.pop('_gen_result')} question(s)] to the bank.")


# 3. CORTEX SPEND — branch STRUCTURALLY: success-but-empty ≠ permission error. (app-v4 blamed every miss on a grant.)
def render_spend():
    try:
        df = load_cortex_spend()        # cached SELECT over SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY
    except Exception as e:
        msg = str(e).lower()
        if any(sig in msg for sig in ("insufficient privileges", "not authorized", "does not exist")):
            role = get_active_session().get_current_role().strip('"')
            st.markdown(":orange-badge[NO ACCESS] Grant ACCOUNT_USAGE to read Cortex spend:")
            st.code(f"GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE {role};", language="sql")
        else:
            st.caption(f"Couldn't read Cortex spend: {e}")      # any OTHER error → caption, NOT the GRANT banner
        return
    if df.empty:                        # ACCOUNTADMIN holds IMPORTED PRIVILEGES by default → empty ≠ no-grant
        st.caption("No Cortex spend recorded yet — ACCOUNT_USAGE lags up to ~2 h, or no AI calls have run."); return
    # ...render the two charts (spend by feature, spend by model: haiku/sonnet/opus) per $quiz/design...
    st.caption("ACCOUNT_USAGE lags up to ~2 h.")


# 4. LOGS — both tables (filtered), then a FRAMELESS reset with a two-step confirm. Reset lives HERE, not App config.
def render_logs():
    st.markdown("**REVIEW LOG**")
    rdf = load_review_log()                                 # cached, newest first
    doms = st.multiselect("Domain", sorted(rdf["DOMAIN_NAME"].dropna().unique()), key="log_dom")
    view = rdf[rdf["DOMAIN_NAME"].isin(doms)] if doms else rdf
    lim  = st.session_state.get("_log_limit", 50)
    st.dataframe(view.head(lim), hide_index=True, use_container_width=True)   # NO download_button
    if len(view) > lim and st.button("Load 50 more", key="log_more"):
        st.session_state["_log_limit"] = lim + 50; st.rerun()
    st.markdown("**SESSION LOG**")
    st.dataframe(load_session_log(), hide_index=True, use_container_width=True)

    if st.button("Reset all logs", type="tertiary", key="log_reset"):        # frameless, no DANGER ZONE
        st.session_state["pending_reset"] = True
    if st.session_state.get("pending_reset"):
        with st.container(border=True):
            st.markdown("**Delete all review + session log rows?**")
            rc = st.columns(2)
            if rc[0].button("Confirm", type="primary", key="reset_yes"):
                s = get_active_session()
                s.sql(f"DELETE FROM {SCHEMA}.QUIZ_REVIEW_LOG").collect()      # DELETE the two log tables — NEVER DROP
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
| `round_size` | int | `10` | Questions per round |
| `round_history` | list | `[]` | List of history_item dicts |
| `difficulty` | str | `"mixed"` | mixed/easy/medium/hard |
| `domain_filter` | list | `[]` | Selected domain names; empty=all |
| `question_source` | str | `"mix"` | mix/db/ai |
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

## Output

Code that conforms to page flow contracts, session state schema, and write-back + cache-invalidation patterns.
