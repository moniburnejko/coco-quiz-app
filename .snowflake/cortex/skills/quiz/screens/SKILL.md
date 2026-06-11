---
name: quiz-screens
description: "Quiz app behavioral contracts — page flow (st.navigation), screen state machine, history tracking, write-back + cache invalidation, session state, explanation contract, dashboard. Use when building or modifying any page or shared UI module of the app. Triggers: screen flow, page flow, quiz screen, home screen, summary, review page, session state, write-back, explanation, history_item, st.navigation. Do NOT use for question generation (quiz-questions), styling (quiz-style), or optional features (quiz-features)."
---

# When to Load

Parent skill `$quiz` routes here for SCREENS intent.

- Building or modifying any page (`pages/*.py`), the entry point (`main.py`), or shared render helpers (`_ui.py`)
- Adding a new feature to the quiz flow
- Debugging page/screen transitions or state issues
- Understanding data flow between pages

# When NOT to Use

- Cortex AI issues -> use `$cortex/patterns`
- SiS rendering patterns -> use `$sis/patterns`
- Prompt quality audit -> use `$cortex/prompt-audit`
- UI styling/badges -> use `$quiz/style`
- Question generation logic -> use `$quiz/questions`

---

# Page Flow

Navigation is native multipage (`st.Page` + `st.navigation`), built in `main.py`. Two base pages; optional features add **their own pages**, never extra tabs:

```
main.py  ->  st.navigation([
    pages/quiz.py      "Quiz"   (default)   home -> quiz -> summary  (internal state machine)
    pages/review.py    "Review"             WRONG ANSWERS | LEARNING DASHBOARD  (st.pills sub-tabs)
    pages/<feature>.py                      only when the feature was requested
])
```

**Entry point (`main.py`)**: `st.set_page_config` (first `st.` call) -> `init_session_state()` -> shared sidebar title -> `st.navigation(pages).run()`. Pages share `st.session_state` (it persists across page switches).

**Inside `pages/quiz.py`** the three screens are an internal state machine driven by `st.session_state["screen"]` (`home` / `quiz` / `summary`) — they are NOT separate pages, because they share one round lifecycle.

**Inside `pages/review.py`** the two tabs are `st.pills` with `st.divider()` and `st.title()` per tab, driven by `_review_page`.

Cross-page redirects (e.g. recommendations -> quiz): set the target state, then `st.switch_page("pages/quiz.py")`.

---

# Home Screen (`pages/quiz.py`, screen == "home")

5 control groups in order, each with a bold UPPERCASE label (`st.markdown("**LABEL**")`) and `label_visibility="collapsed"` on the widget:

1. **QUESTIONS** — number_input (1-100)
2. **DOMAINS** — pills multi-select from EXAM_DOMAINS
3. **DIFFICULTY** — pills (mixed/easy/medium/hard), guard against None
4. **SOURCE** — pills multi-select (`["QUESTION BANK", "AI GENERATED"]`), mapped internally to `"mix"/"db"/"ai"`
5. **Enable AI explanations** — toggle

**Start Round** button: saves settings to session state, builds topic schedule via `_build_topic_schedule()` (from `_questions.py`), loads first question with spinner, sets `screen="quiz"`, reruns.

---

# Quiz Screen (`pages/quiz.py`, screen == "quiz")

**Lazy load**: At top of the quiz screen, if `question` is None, load via `get_question()` with spinner, then rerun for clean render.

**Layout**: progress bar -> domain/difficulty badges -> h4 question text -> answer input -> submit -> result + explanation -> navigation

**Answer input**: `st.radio()` for single-answer (index=None, disabled once answered). Independent `st.checkbox()` per option for multi-answer (see `$sis/patterns` Multi-Answer Checkboxes).

**Submit**: Records result in `round_history`, increments counters, sets `answered=True`, reruns. Does NOT call Cortex.

**After submission**: Shows result badge and correct answer info. Do NOT add per-option markup (✓, strikethrough) — let the AI explanation handle details.
- Correct: `:green-badge[✅ CORRECT]`
- Incorrect: `:red-badge[❌ INCORRECT]` + newline + `Correct answer: **A**, **C**` (bold letters only, not full option text)

If explanations enabled, generates explanation lazily (see Explanation Contract below).

**Navigation after answer**: Main content area shows ONLY "Next" button (primary, full-width) — or "Finish Round" (primary) on the last question. Do NOT show a "Finish" button next to "Next" — it causes accidental round termination.

**Sidebar "End Round"**: When `screen == "quiz"`, the sidebar shows an "End Round" button (secondary, full-width). This lets the user finish early without it competing with "Next" in the main area. Clicking sets `_pending_finish = True` and does a natural rerender.

---

# Explanation Contract

**State machine** in `st.session_state["explanation"]`:
- `None` — not yet attempted; call Cortex
- `{}` — tried and failed (sentinel; do NOT retry)
- `{dict}` — success; render

`_generate_explanation()` calls `call_cortex_json(prompt, "explanation")` — the `RESPONSE_FORMATS["explanation"]` schema (see `$cortex/patterns`) guarantees the keys `why_correct` (array), `why_wrong` (object), `mnemonic`, `doc_search`. No fence parsing, no key-existence paranoia; retry only on `None`.

**For correct answers**: call `_generate_explanation()` same as for incorrect — the full explanation is needed to extract `doc_search`. Then show ONLY `📖 [Snowflake Documentation]({doc_url})` (no expander, no why_correct/why_wrong). Do NOT skip the Cortex call — without it, the doc link defaults to a generic `https://docs.snowflake.com` which is useless. Also store `mnemonic` and `doc_url` in `current_history_item` for review log.

**For incorrect answers**: `st.expander("💡 AI EXPLANATION", expanded=False)` containing:
- `📖 [Snowflake Documentation]({doc_url})` at top
- `st.container(border=True)` with **✅ WHY CORRECT** as bullet list (`why_correct` is a JSON array)
- `st.container(border=True)` with **WHY WRONG** per option
- `st.info()` with mnemonic

**doc_search -> URL**: Prompt asks for `doc_search` ("exactly 2-3 words, no URLs, no commas, max 3 words"), code converts: `https://docs.snowflake.com/en/search?q={query}`

**Explanation prompt content** — the schema guarantees shape, the prompt controls quality. The prompt MUST still include format examples so the content is concise and specific:
```
"why_correct": ["First key reason (Snowflake-specific)", "Second reason with technical detail", "Optional third reason"],
"why_wrong": {"X": "one sentence why option X is wrong", "Y": "one sentence why Y is wrong"},
"mnemonic": "a memorable phrase or acronym to remember the correct answer",
"doc_search": "exactly 2-3 words for Snowflake docs search (e.g. 'Cortex Search', 'AI_COMPLETE'). No URLs. No commas. Max 3 words."
```

On "Next": reset explanation to `None`.

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
| `correct_answer` | str | Letter(s) — e.g. `"C"` or `"A,D"` |
| `option_texts` | dict | `{"A": "...", "B": "..."}` — critical for correct answer display |
| `selected` | str | Comma-joined selected letters |
| `selected_labels` | list | `["A) full text", ...]` |
| `is_correct` | bool | |
| `_topic` | str | From `_current_topic` session state |
| `mnemonic` | str | Empty initially; filled after explanation |
| `doc_url` | str | Empty initially; filled from `doc_search` |

`option_texts` is critical — without it, correct answer display shows only letters.

---

# Summary Screen (`pages/quiz.py`, screen == "summary")

- `PASS_THRESHOLD = 75` lives in `_config.py`. All pass/fail logic, badge text, chart threshold rules, and delta calculations MUST reference this constant — never hardcode `75` or `75.0` in multiple places.
- Title: "Round Complete!" (pass >= PASS_THRESHOLD) or "Round Complete" (fail)
- 2-metric row: SCORE (`correct/total`), ACCURACY (`pct%`)
- Pass: `:green-badge[PASSED] above {PASS_THRESHOLD}% threshold`
- Fail: `:orange-badge[NOT YET] {gap}% to go - keep practicing!` (include encouragement)
- Wrong answers in `st.container(border=True)` cards with domain/difficulty badges
- Perfect score: `:green-badge[PERFECT SCORE] No wrong answers this round.`
- Two buttons: "Retry Same Config" (same settings, rebuilds topic schedule), "Configure New Round" (back to home). Both set `screen` and call `st.rerun()` to transition immediately.

---

# Review: Wrong Answers (`pages/review.py`)

Query `QUIZ_REVIEW_LOG` via the cached loader in `_data.py` (`@st.cache_data` with NO ttl, `get_active_session()` inside — see `$sis/patterns` Caching). Do NOT query directly in the page code. Freshness comes from `clear_caches()` at write time, not from a ttl.

Filters: domain pills (multi, empty=all) + date range slider (integer offset, not date objects).

Wrong answer cards: `st.container(border=True)` with domain badge + difficulty badge + date badge, question text, correct answer, mnemonic caption, doc link caption.

---

# Review: Learning Dashboard (`pages/review.py`)

**Empty state**: if 0 sessions, show info message and return.

**Cached queries** (all in `_data.py`, `@st.cache_data` with NO ttl, `get_active_session()` inside):
- `load_session_stats()` — sessions, avg_score, total_questions
- `load_recent_sessions()` — last 10 with session labels (e.g. "#1 . 31/03")
- `load_domain_errors()` — error count per domain

**Layout**: 3 metrics (Sessions, Questions, Readiness with delta) -> Score per Session chart -> Errors by Domain chart.

**Readiness metric**: `st.metric("Readiness", f"{avg_score:.1f}%", delta=f"{delta_val:+.1f}% vs pass")`. Value is a numeric percentage, NOT a badge — `st.metric()` does not render Markdown badges. Hide delta when at threshold: `delta=... if abs(delta_val) >= 0.1 else None`.

**MANDATORY: read `$quiz/style` Color Scheme section before building any chart.** All chart colors, axis formatting, and label limits are defined there.

**Score per Session chart**:
- X-axis: `LABEL:N` (NOMINAL, NOT quantitative) with `sort=None` to preserve chronological order. Labels format: `#{session_id} · {date}` (e.g., `#1 · 06/04`). Do NOT use `:Q` — it causes float interpolation (1.0, 1.1, 1.2...).
- Y-axis: `SCORE_PCT:Q` with `scale=alt.Scale(domain=[0, 100])`
- Line: `mark_line(point=True, color="#29b5e8")` (Snowflake blue from `$quiz/style`)
- Threshold rule: dashed gray line at PASS_THRESHOLD

**Errors by Domain chart**:
- X-axis: `ERROR_COUNT:Q` with `axis=alt.Axis(tickMinStep=1, title=None)` (integer ticks, no title)
- Y-axis: `DOMAIN_NAME:N` with `sort="-x", axis=alt.Axis(labelLimit=500, title=None)` (full domain names, no title)
- Color: `#F1914C` (orange from `$quiz/style`, NOT red)

---

# Optional Feature Pages

Optional features (`$quiz/features`) are generated as **separate pages** (`pages/exam_simulation.py`, `pages/flashcards.py`, `pages/recommendations.py`) and appended to the `st.navigation` list in `main.py` only when requested. They read the same `_data.py` loaders and shared `_ui.py` helpers — no new tables except where a feature's spec says so.

---

# Write-Back Contract

On round end (`_write_back_results()` in `pages/quiz.py`):

1. For each wrong answer in `round_history`: INSERT to `QUIZ_REVIEW_LOG` with bind params
   - `correct_answer` must be **resolved** before INSERT — store `"{letter}) {full_text}"`, not the raw letter. Pattern:
     ```python
     letters = h["correct_answer"].split(",")
     texts = h["option_texts"]
     correct_full = " & ".join(f"{lt.strip()}) {texts.get(lt.strip(), '')}" for lt in letters)
     ```
   - Fields: domain_id, domain_name, difficulty, question_text, correct_answer, mnemonic, doc_url

2. One session summary: INSERT to `QUIZ_SESSION_LOG`
   - Fields: exam_code (from `_config.py`), round_size, correct_count, score_pct, domain_filter, difficulty
   - `round_size` = the **configured** round size from `st.session_state["round_size"]`, NOT `total_count` (which is how many questions were actually answered — may differ if user ends early)

3. **Call `clear_caches()`** (from `_data.py`) immediately after the INSERTs — the Review page loaders have no ttl, so without this the dashboard would not see the new round until a full session restart.

Always bind params (`:1, :2, ...`), never f-string interpolation of values.

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
| `use_explanations` | bool | `True` | AI explanations toggle |
| `correct_count` | int | `0` | Correct answers this round |
| `total_count` | int | `0` | Total answered this round |
| `current_history_item` | dict\|None | `None` | Ref to last appended history item |
| `last_cortex_error` | str\|None | `None` | Debug: last Cortex error |
| `last_ai_parse_error` | str\|None | `None` | Debug: last parse guard trip |
| `_transitioning` | bool | `False` | Button click safety flag |
| `_topic_schedule` | list | `[]` | Shuffled (domain,topic) pairs for the round |
| `_ai_recommendations` | dict\|None | `None` | Cached AI study recommendations |
| `_rec_cache_key` | str\|None | `None` | Cache key = f"rec_{sessions}" |
| `_pending_finish` | bool | `False` | Finish pending flag for deferred write-back |
| `_op_clear_keys` | list | `[]` | Widget keys to pop at top of next run (flag-at-top reset) |

Page navigation state (`nav_pills`, `_current_page`, `_redirect_to_quiz`) is GONE — `st.navigation` owns the current page, and redirects use `st.switch_page("pages/quiz.py")` directly after setting the target state.

---

## Output

Code that conforms to page flow contracts, session state schema, and write-back + cache-invalidation patterns.
