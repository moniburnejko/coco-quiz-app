---
name: quiz-screens
description: "Quiz app behavioral contracts — screen flow, question selection, history tracking, write-back, session state, explanation, dashboard, AI recommendations. Use when building or modifying any screen in quiz.py. Triggers: screen flow, quiz screen, home screen, summary, review tab, session state, write-back, explanation, history_item, render_"
parent_skill: quiz
---

# When to Load

Parent skill `$quiz` routes here for SCREENS intent.

- Building or modifying any screen in quiz.py
- Adding a new feature to the quiz flow
- Debugging screen transitions or state issues
- Understanding data flow between screens

# When NOT to Use

- Cortex AI issues -> use `$cortex/patterns`
- SiS rendering patterns -> use `$sis/patterns`
- Prompt quality audit -> use `$cortex/prompt-audit`
- UI styling/badges -> use `$quiz/style`
- Question generation logic -> use `$quiz/questions`

---

# Screen Flow

The app has two top-level pages (sidebar pills: QUIZ / REVIEW):

```
QUIZ page:
  home -> quiz -> summary
  (if Exam Simulation enabled: sub-pills PRACTICE | EXAM SIMULATION in sidebar)

REVIEW page (sub-pills):
  WRONG ANSWERS | LEARNING DASHBOARD
  (optional features from $quiz/features may add tabs: AI STUDY RECOMMENDATION, FLASHCARDS)
```

**Entry point**: `init_session_state()` -> `load_domains()` -> sidebar navigation -> screen routing. Sidebar uses `st.pills()` with key `nav_pills`. Review uses a second `st.pills()` with `st.divider()` and `st.title()` per tab. REVIEW sub-pills are dynamic — base tabs are WRONG ANSWERS + LEARNING DASHBOARD; optional features (`$quiz/features`) may add more.

Reference: see entry point block at bottom of `quiz.py`

---

# Home Screen

5 control groups in order, each with a bold UPPERCASE label (`st.markdown("**LABEL**")`) and `label_visibility="collapsed"` on the widget:

1. **QUESTIONS** — number_input (1-100)
2. **DOMAINS** — pills multi-select from EXAM_DOMAINS
3. **DIFFICULTY** — pills (mixed/easy/medium/hard), guard against None
4. **SOURCE** — pills multi-select (`["QUESTION BANK", "AI GENERATED"]`), mapped internally to `"mix"/"db"/"ai"`
5. **Enable AI explanations** — toggle

**Start Round** button: saves settings to session state, builds topic schedule via `_build_topic_schedule()`, loads first question with spinner, sets `screen="quiz"`, reruns.

Reference: see `render_home()` in `quiz.py`

---

# Quiz Screen

**Lazy load**: At top of `render_quiz`, if `question` is None, load via `get_question()` with spinner, then rerun for clean render.

**Layout**: progress bar -> domain/difficulty badges -> h4 question text -> answer input -> submit -> result + explanation -> navigation

**Answer input**: `st.radio()` for single-answer (index=None, disabled once answered). Independent `st.checkbox()` per option for multi-answer.

**Submit**: Records result in `round_history`, increments counters, sets `answered=True`, reruns. Does NOT call Cortex.

**After submission**: Shows result badge and correct answer info. Do NOT add per-option markup (✓, strikethrough) — let the AI explanation handle details.
- Correct: `:green-badge[✅ CORRECT]`
- Incorrect: `:red-badge[❌ INCORRECT]` + newline + `Correct answer: **A**, **C**` (bold letters only, not full option text)

If explanations enabled, generates explanation lazily (see Explanation Contract below).

**Navigation after answer**: Main content area shows ONLY "Next" button (primary, full-width) — or "Finish Round" (primary) on the last question. Do NOT show a "Finish" button next to "Next" — it causes accidental round termination.

**Sidebar "End Round"**: When `screen == "quiz"`, the sidebar shows an "End Round" button (secondary, full-width). This lets the user finish early without it competing with "Next" in the main area. Clicking sets `_pending_finish = True` and does a natural rerender.

Reference: see `render_quiz()` in `quiz.py`

---

# Explanation Contract

**State machine** in `st.session_state["explanation"]`:
- `None` — not yet attempted; call Cortex
- `{}` — tried and failed (sentinel; do NOT retry)
- `{dict}` — success; render

**For correct answers**: call `_generate_explanation()` same as for incorrect — the full explanation is needed to extract `doc_search`. Then show ONLY `📖 [Snowflake Documentation]({doc_url})` (no expander, no why_correct/why_wrong). Do NOT skip the Cortex call — without it, the doc link defaults to a generic `https://docs.snowflake.com` which is useless. Also store `mnemonic` and `doc_url` in `current_history_item` for review log.

**For incorrect answers**: `st.expander("💡 AI EXPLANATION", expanded=False)` containing:
- `📖 [Snowflake Documentation]({doc_url})` at top
- `st.container(border=True)` with **✅ WHY CORRECT** as bullet list (`why_correct` is a JSON array)
- `st.container(border=True)` with **WHY WRONG** per option
- `st.info()` with mnemonic

**doc_search -> URL**: Prompt asks for `doc_search` ("exactly 2-3 words, no URLs, no commas, max 3 words"), code converts: `https://docs.snowflake.com/en/search?q={query}`

**Explanation prompt format** — the prompt MUST include format examples in the JSON template to control output quality:
```
"why_correct": ["First key reason (Snowflake-specific)", "Second reason with technical detail", "Optional third reason"],
"why_wrong": {"X": "one sentence why option X is wrong", "Y": "one sentence why Y is wrong"},
"mnemonic": "a memorable phrase or acronym to remember the correct answer",
"doc_search": "exactly 2-3 words for Snowflake docs search (e.g. 'Cortex Search', 'AI_COMPLETE'). No URLs. No commas. Max 3 words."
```
These format examples guide the AI to produce concise, bullet-point explanations — not verbose paragraphs.

On "Next": reset explanation to `None`.

Reference: see `_generate_explanation()` in `quiz.py`

---

# History Item Schema

Each submitted answer is appended to `round_history`:

| Field | Type | Notes |
|-------|------|-------|
| `question_id` | int/None | From QUESTION_ID field |
| `domain_id` | int | |
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

# Summary Screen

- Define `PASS_THRESHOLD = 75` as a constant at module level. All pass/fail logic, badge text, chart threshold rules, and delta calculations MUST reference this constant — never hardcode `75` or `75.0` in multiple places.
- Title: "Round Complete!" (pass >= PASS_THRESHOLD) or "Round Complete" (fail)
- 2-metric row: SCORE (`correct/total`), ACCURACY (`pct%`)
- Pass: `:green-badge[PASSED] above {PASS_THRESHOLD}% threshold`
- Fail: `:orange-badge[NOT YET] {gap}% to go - keep practicing!` (include encouragement)
- Wrong answers in `st.container(border=True)` cards with domain/difficulty badges
- Perfect score: `:green-badge[PERFECT SCORE] No wrong answers this round.`
- Two buttons: "Retry Same Config" (same settings, rebuilds topic schedule), "Configure New Round" (back to home). Both set `screen` and call `st.rerun()` to transition immediately.

Reference: see `render_summary()` in `quiz.py`

---

# Review: Wrong Answers

Query `QUIZ_REVIEW_LOG` via a cached function (`@st.cache_data(ttl=60)` with `get_active_session()` inside), like all other data loaders. Do NOT query directly in the render function.

Filters: domain pills (multi, empty=all) + date range slider (integer offset, not date objects).

Wrong answer cards: `st.container(border=True)` with domain badge + difficulty badge + date badge, question text, correct answer, mnemonic caption, doc link caption.

Reference: see `render_review()` in `quiz.py`

---

# Review: Learning Dashboard

**Empty state**: if 0 sessions, show info message and return.

**Cached queries** (all `@st.cache_data(ttl=60)` with `get_active_session()` inside):
- `load_session_stats()` — sessions, avg_score, total_questions
- `load_recent_sessions()` — last 10 with session labels (e.g. "#1 . 31/03")
- `load_domain_errors()` — error count per domain

**Layout**: 3 metrics (Sessions, Questions, Readiness with delta) -> Score per Session chart -> Errors by Domain chart.

**Readiness metric**: `st.metric("Readiness", f"{avg_score:.1f}%", delta=f"{delta_val:+.1f}% vs pass")`. Value is a numeric percentage, NOT a badge — `st.metric()` does not render markdown badges. Hide delta when at threshold: `delta=... if abs(delta_val) >= 0.1 else None`.

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

Reference: see `render_dashboard()` in `quiz.py`

---

# Review: AI Study Recommendations

**OPTIONAL feature** — full spec in `$quiz/features` (Feature 6). Only implement if user requests it. When enabled, adds "AI STUDY RECOMMENDATION" to REVIEW sub-pills.

---

# Write-Back Contract

On round end (`_write_back_results()`):

1. For each wrong answer in `round_history`: INSERT to `QUIZ_REVIEW_LOG` with bind params
   - `correct_answer` must be **resolved** before INSERT — store `"{letter}) {full_text}"`, not the raw letter. Pattern:
     ```python
     letters = h["correct_answer"].split(",")
     texts = h["option_texts"]
     correct_full = " & ".join(f"{lt.strip()}) {texts.get(lt.strip(), '')}" for lt in letters)
     ```
   - Fields: domain_id, domain_name, difficulty, question_text, correct_answer, mnemonic, doc_url

2. One session summary: INSERT to `QUIZ_SESSION_LOG`
   - Fields: exam_code (from constant), round_size, correct_count, score_pct, domain_filter, difficulty
   - `round_size` = the **configured** round size from `st.session_state["round_size"]`, NOT `total_count` (which is how many questions were actually answered — may differ if user ends early)

Always bind params (`:1, :2, ...`), never f-string interpolation of values.

Reference: see `_write_back_results()` in `quiz.py`

---

# Session State Contract

All keys initialized in `init_session_state()` with defaults:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `screen` | str | `"home"` | Current quiz screen: home$quiz/summary |
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
| `nav_pills` | str | `"QUIZ"` | Sidebar pills widget key (init here, NOT via default param) |
| `_current_page` | str | `"QUIZ"` | QUIZ/REVIEW navigation state |
| `_current_topic` | str | `""` | Current topic from schedule |
| `_review_page` | str | `"WRONG ANSWERS"` | Active review tab |
| `_redirect_to_quiz` | bool | `False` | Start Focused Session redirect flag |
| `use_explanations` | bool | `True` | AI explanations toggle |
| `correct_count` | int | `0` | Correct answers this round |
| `total_count` | int | `0` | Total answered this round |
| `current_history_item` | dict\|None | `None` | Ref to last appended history item |
| `last_cortex_error` | str\|None | `None` | Debug: last Cortex error |
| `last_ai_response` | str\|None | `None` | Debug: last raw AI response |
| `last_ai_parse_error` | str\|None | `None` | Debug: last JSON parse error |
| `_transitioning` | bool | `False` | Button click safety flag |
| `_topic_schedule` | list | `[]` | Shuffled (domain,topic) pairs for the round |
| `_ai_recommendations` | dict\|None | `None` | Cached AI study recommendations |
| `_rec_cache_key` | str\|None | `None` | Cache key = f"rec_{sessions}" |
| `_pending_finish` | bool | `False` | Finish pending flag for deferred write-back |

---

## Output

Code that conforms to screen flow contracts, session state schema, and write-back patterns.
