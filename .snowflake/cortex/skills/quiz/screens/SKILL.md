---
name: quiz-screens
description: "Quiz app behavioral contracts — page flow (st.navigation), screen state machine, history tracking, write-back + cache invalidation, session state, explanation contract, dashboard. Use when building or modifying any page or shared UI module of the app. Triggers: screen flow, page flow, quiz screen, home screen, summary, review page, session state, write-back, explanation, history_item, st.navigation. Do NOT use for question generation (quiz-questions), styling (quiz-design), or optional features (quiz-features)."
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

Navigation is native multipage (`st.Page` + `st.navigation`), built in `main.py`. Two base pages; **most** optional features add **their own pages**, a few hook into Review/Quiz instead (see Optional Feature Pages):

```
main.py  ->  st.navigation([
    pages/quiz.py      "Quiz"   (default)   home -> quiz -> summary  (internal state machine)
    pages/review.py    "Review"             WRONG ANSWERS | [FLASHCARDS] | LEARNING DASHBOARD  (st.pills sub-tabs; FLASHCARDS only if enabled)
    pages/admin.py     "Admin"              app config, question manager, bank stats, spend, tools
    pages/<feature>.py                      only when the feature was requested
])
```

**Config layer**: runtime behavior toggles live in `QUIZ_CONFIG` (defaults in `_config.py` `CONFIG_DEFAULTS`, DB overrides; `load_config()` cached + `save_config()` in `_data.py`, both with `clear_caches()` on write). Gates used below: `hints_enabled`, `debrief_enabled`, `remedial_enabled`, `default_round_size`, `pass_threshold_override`. (There is no `explanations_default`/`contrast_enabled` config — the explanation + deep-dive are always-available on-demand, so they need no toggle.)

**Entry point (`main.py`)**: `st.set_page_config` (first `st.` call) -> `init_session_state()` -> shared sidebar title -> `st.navigation(pages).run()`. Pages share `st.session_state` (it persists across page switches).

**Inside `pages/quiz.py`** the three screens are an internal state machine driven by `st.session_state["screen"]` (`home` / `quiz` / `summary`) — they are NOT separate pages, because they share one round lifecycle.

**Inside `pages/review.py`** the tabs are `st.pills` with `st.divider()` and `st.title()` per tab, driven by `_review_page` — `WRONG ANSWERS` and `LEARNING DASHBOARD` always, plus `FLASHCARDS` when the flashcards feature is enabled (`$quiz/features` Feature 2).

Cross-page redirects (e.g. recommendations -> quiz): set the target state, then `st.switch_page("pages/quiz.py")`.

---

# Home Screen (`pages/quiz.py`, screen == "home")

4 control groups in order, each with a bold UPPERCASE label (`st.markdown("**LABEL**")`) and `label_visibility="collapsed"` on the widget:

1. **QUESTIONS** — number_input (1-100)
2. **DOMAINS** — pills multi-select from EXAM_DOMAINS
3. **DIFFICULTY** — pills (mixed/easy/medium/hard), guard against None
4. **SOURCE** — pills multi-select (`["QUESTION BANK", "AI GENERATED"]`), mapped internally to `"mix"/"db"/"ai"`

There is **no "AI explanations" toggle** — the explanation is on-demand per question (a button after answering), never auto-loaded.

**Grounding guard** (top of Home, `cke`/`custom` mode): if `not docs_available()` (`$cortex`/`_search.py`), show a red "install/grant the Snowflake Documentation CKE — the app can't generate until it's reachable" message and **disable Start Round**. The app must never generate from built-in knowledge. (In `none` mode there is no guard — generation is intentionally ungrounded.)

**Start Round** button: saves settings to session state, builds topic schedule via `_build_topic_schedule()` (from `_questions.py`), loads first question with spinner, sets `screen="quiz"`, reruns.

---

# Quiz Screen (`pages/quiz.py`, screen == "quiz")

**Lazy load**: At top of the quiz screen, if `question` is None, load via `get_question()` with spinner, then rerun for clean render.

**Layout**: progress bar -> domain/difficulty badges -> h4 question text -> answer input -> submit -> result + explanation -> navigation

**Answer input**: `st.radio()` for single-answer (`index=None`, disabled once answered). For multi-answer, render each option as an independent `st.checkbox` with a stable key (`cb_A`, `cb_B`, …); read the selection from session state after rendering; disable all once answered. On "Next", clear the `cb_*` keys via the flag-at-top reset (queue `["cb_A", …, "cb_E"]` in `_op_clear_keys`, rerun, pop at the top — see `$sis` widget lifecycle).

**Socratic hint (BEFORE answering; gate `hints_enabled`)**: a secondary "💡 Hint" button near the answer input, visible ONLY while `answered == False`. First click → `call_cortex_json(prompt, "hint")`, show `hint_1` and **relabel the button to "Need more help?"**; second click reveals `hint_2`, then hide the button. The hint is **grounded like every other generation** — in `cke`/`custom` mode embed retrieved `<doc_context>` in the hint prompt and derive the hints from it, never the model's built-in knowledge (`$cortex`); the prompt MUST instruct: hints narrow the concept space (level 1) or eliminate ONE distractor with reasoning (level 2) and must NEVER name or imply the correct option; embed question/options per the untrusted-content delimiting rule. State: `hint` (None/{}/dict), `hint_level` (0/1/2); once answered, the button disappears (the explanation takes over); record `hint_used = hint_level > 0` in the history item; reset both on Next.

**Submit**: Records result in `round_history` (incl. `hint_used`), increments counters, sets `answered=True`, reruns. Does NOT call Cortex.

**After submission** — show the result badge, then a **two-button row** (`st.columns(2)`):
- **left = "Next"** (primary; **"Finish Round"** on the last question) → advances immediately
- **right = "💡 AI explanation"** (secondary) → loads the explanation **on demand** (Explanation Contract below) into an expander rendered beneath the row

The explanation is NEVER auto-generated, and the button appears for **correct answers too** (to learn why the distractors are wrong). Both buttons stay visible after the explanation loads (read, then Next). Do NOT render "Next" and "Finish" together — it causes accidental round termination. Do NOT add per-option markup (✓, strikethrough) — the explanation handles details.
- Correct: `:green-badge[✅ CORRECT]`
- Incorrect: `:red-badge[❌ INCORRECT]` + newline + `Correct answer: **A**, **C**` (bold letters only, not full option text)

**Sidebar "End Round"**: When `screen == "quiz"`, the sidebar shows an "End Round" button (secondary, full-width). This lets the user finish early without it competing with "Next" in the main area. Clicking sets `_pending_finish = True` and does a natural rerender.

**Button click safety**: guard slow-action buttons (Start Round, Submit, Next, Finish) with the `_transitioning` flag so a double-click can't double-fire during the rerun:
```python
if st.button("Next", disabled=st.session_state.get("_transitioning", False)):
    st.session_state["_transitioning"] = True
    # ... action ...
    st.session_state["_transitioning"] = False
    st.rerun()
```
Pair this with the spinner + single-`st.rerun()` rule in `$sis`.

---

# Explanation Contract

**On-demand only** — generated when the user clicks **"💡 AI explanation"**, never automatically. State machine in `st.session_state["explanation"]`: `None` (not requested yet — just show the button), `{}` (tried and failed; do NOT retry), `{dict}` (success; render the expander). On "Next": reset to `None`.

`_generate_explanation()` calls `call_cortex_json(prompt, "explanation")` — the `RESPONSE_FORMATS["explanation"]` schema (`$cortex`) guarantees `why_correct` (array), `why_wrong` (object), `mnemonic`, `doc_search`. No fence parsing; retry only on `None`.

**Same flow for correct AND incorrect answers** — clicking the button opens `st.expander("💡 AI EXPLANATION", expanded=True)` containing, in order:
- `st.container(border=True)` **✅ WHY CORRECT** — `why_correct` bullet list
- `st.container(border=True)` **WHY WRONG** — one line per *other* option (on a correct answer this is exactly the value: learn why the distractors are wrong)
- `st.info()` mnemonic
- a **"📚 From the docs"** block: `st.caption(DOCUMENT_TITLE)`, a short `CHUNK` excerpt (~280 chars), and `📖 [Snowflake Documentation]({doc_url})`
- the **Deep dive** control (below)

The 📖 doc link appears **only inside this expander, only after the button is clicked** — never auto-shown (not even for correct answers). Render every dynamic field (`why_correct`, `why_wrong`, `mnemonic`, the `CHUNK` excerpt) through the `_ui.py` `md()` escaper before `st.markdown`/`st.info` (`$quiz/design` — escaping dynamic text). Store `mnemonic` + `doc_url` on `current_history_item` for the review log.

**Doc grounding is MANDATORY in `cke`/`custom` mode (`$cortex`)**: retrieve `chunks = search_docs(question_text)` once; if `[]`, broaden once, else **fail visibly** (no built-in). Embed the top chunk(s) in the prompt as `<doc_context>` so `why_correct`/`why_wrong` are doc-grounded; `doc_url = chunks[0]["SOURCE_URL"]`. **`none` mode only**: the prompt asks for `doc_search` ("2-3 words, no URLs/commas") → `https://docs.snowflake.com/en/search?q={query}`.

**Explanation prompt content** — the schema guarantees shape, the prompt controls quality:
```
"why_correct": ["First key reason (Snowflake-specific)", "Second reason with technical detail", "Optional third"],
"why_wrong": {"X": "one sentence why option X is wrong", "Y": "..."},
"mnemonic": "a memorable phrase or acronym for the correct answer",
"doc_search": "exactly 2-3 words for Snowflake docs search. No URLs. No commas. Max 3 words."
```

## Deep dive (inside the expander, below the explanation)

A picker over the question's options (`st.multiselect` / option pills), **validated to exactly 1 or 2 selections**, + a **"🔬 Deep dive"** button:
- **1 selected** → `call_cortex_json(prompt, "deep_dive")` — an in-depth breakdown of that single option (`$cortex` `deep_dive` schema: `summary`, `how_it_works[]`, `when_to_use`, `exam_traps[]`). Render in `st.container(border=True)` with bold sub-labels + bullets.
- **2 selected** → `call_cortex_json(prompt, "contrast")` — the A-vs-B comparison (`concept_a`, `concept_b`, `differences[]`, `exam_trap`). Render a compact `aspect | A | B` table + `exam_trap` caption, in a bordered container.

Both modes are **grounded** like the explanation: reuse the explanation's retrieved `<doc_context>`; if a 2-compare needs a wider query and retrieval comes back empty, broaden once then **fail visibly** (return `None`) — never built-in. Embed the option text(s) + question per the delimiting rule, render all output through the `_ui.py` `md()` escaper, and carry **more detail** than the base explanation. State: `deep_dive` (None/{}/dict), reset on Next. Default the picker to the correct option (or the user's wrong choice + the correct one for a 2-compare), but any 1-or-2 selection is allowed. Rationale: SnowPro questions are discrimination tasks — "explain one" deepens a concept, "compare two" trains the exact distinction the exam tests.

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
| `hint_used` | bool | True if any hint level was revealed |
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
- **Wrong answers** go inside a **collapsed `st.expander("WRONG ANSWERS", expanded=False)`** — each as an `st.container(border=True)` card with domain/difficulty badges, the **question text**, the **correct answer in full** (`"B) …text… & D) …text…"`, built from the card's `option_texts` + correct letters — NEVER bare letters), and the **mnemonic** (`st.info` 🧠) only when one was generated for it this round (`round_history[i]["mnemonic"]`; omit if empty). All dynamic text via the `md()` `$`-escaper (`$quiz/design`).
- Perfect score: `:green-badge[PERFECT SCORE] No wrong answers this round.` (no expander)

**Round Brief (on-demand; gate `debrief_enabled`; only when ≥1 wrong answer)**: a **"Round Brief"** button — NOT auto-generated. On click → `call_cortex_json(prompt, "debrief")` with per-question domain/topic/correctness/`hint_used` from `round_history` → open an **expander** with clean formatting: `st.container(border=True)` holding **PATTERNS** (bullets) and **PRIORITY ACTIONS** (max 3 bullets), then `one_thing` as a highlighted **🎯 FOCUS** line in its own bordered container (NOT `st.info` — that's reserved for the mnemonic per `$quiz/design`). **Grounding scope:** meta-analysis over the user's OWN round performance — names weak domains/topics + study actions but MUST NOT assert new Snowflake facts or emit doc links; the one runtime generation path exempt from doc-grounding (`$cortex`). State `debrief` (None=not requested / {}=failed / dict=success), reset on round start. A perfect round shows no Round Brief button. Render every dynamic field (PATTERNS, PRIORITY ACTIONS, `one_thing`) through the `md()` `$`-escaper before `st.markdown` (`$quiz/design`).

**Action buttons (practice rounds, by outcome)** — `Round Brief` and `Remedial Round` only apply when there are wrong answers. When `_round_type == "remedial"`, ignore this table: the remedial summary shows only **"Configure New Round"** (see the Remedial round contract).
- **Perfect** (all correct): **"Configure New Round"** only.
- **Passed, some wrong**: **"Round Brief"** + **"Configure New Round"**.
- **Failed, `remedial_enabled` ON**: **"Round Brief"** + **"Remedial Round"** (primary) + **"Configure New Round"**.
- **Failed, `remedial_enabled` OFF**: **"Round Brief"** + **"Configure New Round"**. (Remedial Round is the ONLY button gated by `remedial_enabled` — a failed round always offers at least Round Brief + Configure New Round.)
- No "Retry Same Config" button (removed). Threshold = `pass_threshold_override` if set, else `PASS_THRESHOLD`. All buttons set state and call `st.rerun()`.

**Remedial round contract**: queue = the wrong items from `round_history` (order shuffled; `_shuffle_options` re-applied to every question so option letters move). Sets `_round_type="remedial"`, `_remedial_queue`, resets counters/q_index/history for the remedial pass. During remedial: hints/explanations behave normally; questions count toward nothing — **no `_write_back_results()`, no debrief, no logging** (a re-test of just-seen questions would inflate readiness stats and duplicate review entries). Remedial summary: score + only "Configure New Round" (no chained remedials) — this overrides the by-outcome button table above. `_round_type` resets to `"practice"` on any new round.

---

# Review: Wrong Answers (`pages/review.py`)

Query `QUIZ_REVIEW_LOG` via the cached loader in `_data.py` (`@st.cache_data` with NO ttl, `get_active_session()` inside — see `$sis` Caching). Do NOT query directly in the page code. Freshness comes from `clear_caches()` at write time, not from a ttl.

Filters: domain pills (multi, empty=all) + a **date-range `st.date_input`** — **always shown**, even when all wrong answers fall on one day (do NOT hide it when `min == max`; pass that single date as both `value` ends + `min_value`/`max_value`). Keep rows whose cast `LOGGED_AT` date is within `[start, end]`.

Wrong answer cards: `st.container(border=True)` with domain badge + difficulty badge + date badge, question text, correct answer, mnemonic caption, doc link caption.

**Date handling** (filters + dashboard): values from `.collect()` are Snowflake datetimes — cast with `datetime.date(raw.year, raw.month, raw.day)` before feeding any widget or doing date arithmetic. The Wrong-Answers date filter is an **`st.date_input` range** (always rendered — see above); compare each row's cast `LOGGED_AT` date against the selected `[start, end]` in Python. For any `TIMESTAMP_LTZ` range query in SQL, pass dates as `strftime("%Y-%m-%d")` strings with an exclusive upper bound (`< end + 1 day`) to include the full last day. Never pass a `datetime.date` to `st.slider` (`$sis`).

---

# Review: Learning Dashboard (`pages/review.py`)

**Empty state**: if 0 sessions, show info message and return.

**Cached queries** (all in `_data.py`, `@st.cache_data` with NO ttl, `get_active_session()` inside):
- `load_session_stats()` — sessions, avg_score, total_questions
- `load_recent_sessions()` — last 10 with session labels (e.g. "#1 . 31/03")
- `load_domain_errors()` — error count per domain

**Layout**: 3 metrics (Sessions, Questions, Readiness with delta) -> Score per Session chart -> Errors by Domain chart.

**Readiness metric**: `st.metric("Readiness", f"{avg_score:.1f}%", delta=f"{delta_val:+.1f}% vs pass")`. Value is a numeric percentage, NOT a badge — `st.metric()` does not render Markdown badges. Hide delta when at threshold: `delta=... if abs(delta_val) >= 0.1 else None`.

**Charts:** build every chart (colors, axis types/formatting, label limits, threshold rules) per `$quiz/design` — that skill is the single source for all visual rules; reference the `_config.py` color constants and never hardcode hexes or axis specs here. Two charts on this dashboard: **Score per Session** (from `load_recent_sessions()`) and **Errors by Domain** (from `load_domain_errors()`).

---

# Optional Feature Pages

Optional features (`$quiz/features`) are generated as **separate pages** (`pages/exam_simulation.py`, `pages/recommendations.py`) appended to the `st.navigation` list in `main.py` only when requested. Three features hook into existing pages instead: **Feature 2 (flashcards)** adds a FLASHCARDS tab to the Review page, Feature 7 (misconception analysis) extends the Review page, Feature 8 (flag a question) adds a button to the quiz screen. They read the same `_data.py` loaders and shared `_ui.py` helpers — no new tables except where a feature's spec says so (Feature 2 adds `FLASHCARD_PROGRESS`, Feature 8 adds `QUIZ_FLAGS`).

---

# Admin Page (`pages/admin.py` — core)

Five **`st.tabs`** — **App config · Question manager · Bank stats · Cortex spend · Tools** (not one long scrolling page); the five subsections below are the tab contents in order. Single-user app → visible to the owner; when multi-user lands, gate via restricted caller's rights (fail-closed) — do NOT build RBAC now.

**1. App configuration**: toggles for `hints_enabled`, `debrief_enabled`, `remedial_enabled`; slider `default_round_size` (5–50); `pass_threshold_override` slider with an "exam default (75%)" reset button + warning caption that the official exam threshold does not change. **Grounding** is shown **read-only** — `grounding_mode` is fixed at setup (`$setup-exam` Step 1g), never a runtime toggle (an "off" switch would be a built-in-knowledge backdoor). In `cke`/`custom` mode, if `docs_available()` is False, show a red caption: "The doc grounding service is unavailable — install/grant the Snowflake Documentation CKE; the app can't generate until it's reachable." Every config change → `save_config(key, value)` (MERGE by key, bind params) → `clear_caches()` (clears `docs_available`/`search_docs` too) → `st.toast`.

**2. Question manager**: filter pills (domain / difficulty / source) → cached query → `st.dataframe(..., on_select="rerun", selection_mode="single-row")` → selected row loads into an edit form below (question `st.text_area`, options A–E inputs, `correct_answer` multiselect restricted to NON-EMPTY options, difficulty pills; `is_multi` derived = len(correct) > 1) → UPDATE by `question_id`. "Add new question" = the same form, empty → INSERT with `source='MANUAL'`. **Hard rules**: every write via bind params (NEVER f-string); length caps enforced in the form AND by truncation (question 2000, options 500); `correct_answer` ⊆ non-empty options; ≥2 options. **"Generate batch (AI)"** button: pick domain + difficulty mix → generates 10 questions via the **same grounded `_questions.py` path** (`$quiz/questions`: in `cke`/`custom` mode each question embeds retrieved `<doc_context>` as the primary source with `key_facts` as supporting scope, answers ONLY from the docs, and fails visibly on empty retrieval — never built-in knowledge) → INSERT with `source='AI_GENERATED'` → report count; caption with an approximate-cost note. If Feature 8 is enabled, show OPEN flags next to their questions.

**3. Bank stats**: cached coverage table — questions per domain × difficulty × source, plus flagged count (if Feature 8).

**4. Cortex spend (graceful)**: `_cortex.py` sets a session `QUERY_TAG` (JSON: app, feature, model) and passes the feature per call. The dashboard reads `SNOWFLAKE.ACCOUNT_USAGE.CORTEX_FUNCTIONS_USAGE_HISTORY` inside try/except: on a permissions error render an info banner with the exact statement (`GRANT IMPORTED PRIVILEGES ON DATABASE SNOWFLAKE TO ROLE <role>;`) instead of crashing. Charts: spend by feature, spend by model (sonnet vs opus comparison). Caption: ACCOUNT_USAGE lags up to ~2h. When doc grounding is on, add a "docs search" line (Cortex Search query compute is billed to the consumer; small per query).

**5. Tools**: "Refresh data" (`clear_caches()`); CSV export of `QUIZ_REVIEW_LOG` / `QUIZ_SESSION_LOG` (`st.download_button`); danger zone in an expander — "Reset logs" requires typing `DELETE` to confirm, runs `DELETE FROM` on the two log tables only (NEVER DROP, consistent with governance).

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
| `hint` | None/{}/ dict | `None` | Socratic hint (None=not tried, {}=failed, dict=success) |
| `hint_level` | int | `0` | 0=none, 1=hint_1 shown, 2=hint_2 shown |
| `deep_dive` | None/{}/ dict | `None` | Deep-dive result (explain-1 or compare-2) for the current question |
| `debrief` | None/{}/ dict | `None` | On-demand Round Brief (None=not requested, {}=failed, dict=success); generated from the Round Brief button, reset on round start |
| `_round_type` | str | `"practice"` | practice / remedial |
| `_remedial_queue` | list | `[]` | Wrong items queued for the remedial round |

Navigation is owned by `st.navigation` (no `nav_pills` / `_current_page` / `_redirect_to_quiz` keys); cross-page redirects set the target state, then call `st.switch_page("pages/quiz.py")`.

---

## Output

Code that conforms to page flow contracts, session state schema, and write-back + cache-invalidation patterns.
