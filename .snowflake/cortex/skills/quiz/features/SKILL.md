---
name: quiz-features
description: "Optional features for the quiz app — exam simulation, flashcards, quick stats, spaced repetition, achievement badges, AI study recommendations. Implement ONLY features explicitly requested by the user. Triggers: exam simulation, timer, flashcard, spaced repetition, smart review, achievement, badge, streak, study recommendation, AI recommendation"
parent_skill: quiz
---

# When to Load

Parent skill `$quiz` routes here for FEATURES intent.

- User explicitly requests an optional feature in their prompt
- Adding new functionality beyond the core quiz flow

# When NOT to Use

- Core quiz screens (home, quiz, summary) -> use `$quiz/screens`
- Question generation logic -> use `$quiz/questions`
- UI styling/badges -> use `$quiz/style`

---

# IMPORTANT

This skill contains **OPTIONAL** features. Do NOT implement any feature unless the user explicitly requests it in their prompt. Each feature is self-contained — implement only the requested ones. Core quiz functionality (home, quiz, summary, wrong answers, learning dashboard) does NOT require this skill.

---

# Feature 1: Exam Simulation Mode

**OPTIONAL** — implement only if user requests exam simulation, timed quiz, mock exam, practice exam, or exam mode.

## What

New mode alongside Practice. Fixed question count matching real exam, countdown timer, all domains weighted by exam blueprint, no explanations during simulation, final report with pass/fail vs real threshold.

## Navigation

QUIZ page gets sub-pills in sidebar (like REVIEW has sub-pills): `PRACTICE | EXAM SIMULATION`. Appears in sidebar only when `_current_page == "QUIZ"`. PRACTICE is the existing flow (home→quiz→summary). EXAM SIMULATION is a separate flow.

```
QUIZ page (with Exam Simulation enabled):
  sidebar sub-pills: PRACTICE | EXAM SIMULATION
  PRACTICE: home -> quiz -> summary (existing, unchanged)
  EXAM SIMULATION: sim_config -> sim_quiz -> sim_results
```

## UI spec

**Sidebar**: `st.pills("Quiz mode", options=["PRACTICE", "EXAM SIMULATION"], ...)` below nav pills, only when on QUIZ page. Use `label_visibility="collapsed"` with bold label above.

**sim_config screen**: Shows exam params as read-only info:
- Exam: {EXAM_NAME} ({EXAM_CODE})
- Questions: {N} (weighted by domain)
- Time limit: {M} minutes
- Pass threshold: {PASS_THRESHOLD}%
- "Start Exam" button (primary, full-width)

**sim_quiz screen**: Same as regular quiz but:
- Prominent timer: `st.metric("Time Remaining", f"{minutes}:{seconds:02d}")` at top
- Progress bar below timer
- NO explanations toggle (always off)
- NO source selection — all from DB+AI weighted by domain
- Timer: check elapsed time on each render (non-blocking), updates on each answer submission
- End conditions: all questions answered OR timer expires

**sim_results screen**: Detailed breakdown:
- Overall: pass/fail badge, score, time used
- Domain table: domain name, questions asked, correct, %, exam weight
- Comparison vs real exam threshold

## Session state keys

`_quiz_mode` (str: "PRACTICE"/"EXAM SIMULATION"), `_sim_start_time` (datetime), `_sim_time_limit` (int, seconds), `_sim_questions` (int), `_sim_screen` (str: "config"/"quiz"/"results")

## Implementation notes

- Timer is non-blocking — check `datetime.now() - _sim_start_time` on each render
- Domain question distribution: `round(weight_pct / 100 * total_questions)` per domain
- Time limit and question count: derive from EXAM_CODE — ask user to confirm if unknown. Common values: COF-C03 = 100q/115min, GES-C01 = 65q/90min
- Reuse existing `get_question()`, `render_quiz()` patterns — just wrap with timer and different config

---

# Feature 2: Flashcard Review

**OPTIONAL** — implement only if user requests flashcards, study cards, or review cards.

## What

New REVIEW sub-tab "FLASHCARDS". Shows wrong answers as flashcards — front = question, click to reveal = correct answer + mnemonic + docs link.

## UI spec

Add "FLASHCARDS" to REVIEW sub-pills: `["WRONG ANSWERS", "LEARNING DASHBOARD", "AI STUDY RECOMMENDATION", "FLASHCARDS"]` (only if this feature is enabled; if AI Study Recommendation is not enabled, omit it from the list).

**Card front**: `st.container(border=True)` with:
- Domain badge + difficulty badge
- Question text (`st.markdown(f"#### {text}")`)
- "Show Answer" button (primary)

**Card back** (after "Show Answer"):
- Correct answer: bold text
- Mnemonic: `st.info(f"🧠 {mnemonic}")`
- Docs link: `st.markdown(f"📖 [Snowflake Documentation]({doc_url})")`
- Two buttons: "Got it ✓" (removes from deck this session) / "Review Again ↻" (keeps in deck)

**Counter**: `st.caption(f"Card {current} of {total} remaining")`

**Empty state**: `:green-badge[All caught up!] No flashcards to review.`

**Data source**: `load_review_log()` (existing cached function)

**Session state keys**: `_flashcard_deck` (list of review log entries), `_flashcard_index` (int), `_flashcard_revealed` (bool)

---

# Feature 3: Quick Stats Sidebar

**OPTIONAL** — implement only if user requests quick stats, live stats, sidebar stats, or gamification.

## What

During active quiz, sidebar shows live mini-stats: current streak, round accuracy, current domain.

## UI spec

Inside `with st.sidebar:` block, below End Round button, only when `screen == "quiz"`:

```python
st.divider()
st.caption("ROUND STATS")
st.metric("Streak", streak_count)
st.metric("Accuracy", f"{round_accuracy}%")
st.markdown(f":blue-badge[{current_domain}]")
```

- `streak_count`: count consecutive correct from end of `round_history`
- `round_accuracy`: `correct_count / total_count * 100` from session state
- `current_domain`: from current question's DOMAIN_NAME

**Data source**: `st.session_state["round_history"]` + current question (live, no DB query)

---

# Feature 4: Spaced Repetition (Smart Review)

**OPTIONAL** — implement only if user requests spaced repetition, smart review, SM-2, intelligent review, or adaptive review.

## What

New question source mode "SMART REVIEW" — prioritizes questions the user got wrong more recently or repeatedly.

## UI spec

Home screen: SOURCE pills get additional option: `["QUESTION BANK", "AI GENERATED", "SMART REVIEW"]`

When SMART REVIEW selected:
- Ignores domain_filter and difficulty settings
- Picks questions based on review priority score
- Priority formula: `score = error_count * (1 / max(days_since_last_error, 1))`
- Source: QUIZ_REVIEW_LOG question_text → match back to QUIZ_QUESTIONS, or re-ask via AI

**Session state keys**: `_smart_review_queue` (list of prioritized question_texts)

**Data source**: QUIZ_REVIEW_LOG (logged_at, question_text), QUIZ_QUESTIONS (for re-fetching full question)

---

# Feature 5: Achievement Badges

**OPTIONAL** — implement only if user requests achievements, badges, milestones, gamification, or streak tracking.

## What

Gamification badges computed from quiz history. Displayed in sidebar.

## Badge definitions

Computed from logs on each render (not stored in DB):

| Badge | Condition | Icon |
|-------|-----------|------|
| First Perfect | Any round with 100% score | 🎯 |
| Century | 100+ questions answered total | 💯 |
| Week Streak | Sessions on 5+ different days in last 7 days | 🔥 |
| Domain Master: {name} | 90%+ accuracy in a domain (min 10 questions) | 🏆 |
| Speed Demon | Completed 20-question round in under 5 minutes | ⚡ |

## UI spec

In sidebar, below navigation pills (always visible, not just during quiz):

```python
st.divider()
st.caption("ACHIEVEMENTS")
for badge in earned_badges:
    st.markdown(f":green-badge[{badge.icon} {badge.name}]")
for badge in locked_badges:
    st.markdown(f":gray-badge[{badge.icon} {badge.name}]")
    st.caption(badge.requirement_text)
```

**Data sources**: QUIZ_SESSION_LOG (round scores, timestamps), QUIZ_REVIEW_LOG (domain accuracy)

---

# Feature 6: AI Study Recommendation

**OPTIONAL** — implement only if user requests study recommendation, AI recommendation, readiness analysis, or exam readiness.

## What

AI-powered exam readiness analysis. REVIEW sub-tab "AI STUDY RECOMMENDATION".

## Navigation

Add "AI STUDY RECOMMENDATION" to REVIEW sub-pills when this feature is enabled.

## Condition

Sessions >= 2 AND error_data is non-empty. Otherwise show caption: "Complete at least 2 quiz sessions to see personalized recommendations."

## Trigger

"Generate AI Study Recommendation" button. Spinner during call. After saving results to session state, call `st.rerun()` so the button is hidden on the next render cycle. The button only shows when `has_cache` is False.

## Prompt inputs

Session count, avg score, total questions, domain error counts, wrong question samples (last 30 per domain via `load_wrong_question_samples()`), domain topic lists.

## JSON output schema

```json
{
  "exam_readiness": {"ready": false, "gap_pct": 12.5, "message": "..."},
  "weak_domains": [{"domain_name": "...", "error_count": 5, "recommendation": "..."}],
  "weak_topics": [{"domain_name": "...", "topic": "...", "recommendation": "...", "doc_search": "..."}],
  "study_plan": ["step 1", "step 2", "step 3"],
  "recommended_difficulty": "medium",
  "recommended_domain": "..."
}
```

No `overall_assessment` key. `doc_search` converted to `doc_url` post-parse.

## Prompt constraints (MUST be in the AI prompt)

- `exam_readiness.ready = true` if avg_score >= PASS_THRESHOLD; `gap_pct = abs(avg_score - PASS_THRESHOLD)`
- `exam_readiness.message`: max 1 sentence, under 80 chars
- `weak_domains[].recommendation` and `weak_topics[].recommendation`: max 1 sentence, under 80 chars each
- `study_plan`: exactly 3 items, each under 80 chars, action-oriented verbs
- `weak_topics`: up to 5 UNIQUE entries (deduplicate by topic name). `doc_search`: exactly 2-3 words, no URLs, no commas, max 3 words
- Do NOT include `overall_assessment` key

## Display

1. Readiness: `:green-badge[EXAM READY]` or `:orange-badge[NOT READY YET]` — badge on its own line. Message as `st.caption()` on next line (NOT concatenated with badge).
2. Focus Areas: cards in columns with `:blue-badge[DOMAIN]` + `:red-badge[N errors]` + recommendation
3. Topics to Review: containers with domain badge + topic name + recommendation + `📖 [Snowflake Documentation]({doc_url})`; deduplicate by topic (use `seen_topics` set)
4. Next Steps: numbered list in container
5. Action buttons: "Start Focused Session: {recommended_domain} ({recommended_difficulty})" (include domain name in label) + "Regenerate"

## Start Focused Session

Sets domain_filter, difficulty, round_size=10, `_redirect_to_quiz=True`, then calls `st.rerun()`.

> **CRITICAL**: `render_ai_recommendations()` must NEVER set `nav_pills` directly — the `st.pills()` widget with `key="nav_pills"` has already been instantiated by the time render functions run. Setting it causes `StreamlitAPIException`. Only the entry point (`main()`) may set `nav_pills`, BEFORE the sidebar widget is created. The `_redirect_to_quiz` handler in `main()` must set `nav_pills = "QUIZ"` along with screen/state resets.

## Session state keys

`_ai_recommendations` (dict|None), `_rec_cache_key` (str|None), `_redirect_to_quiz` (bool)

---

## Output

Only the requested optional features implemented and integrated into quiz.py, following the UI patterns from `$quiz/style` and state management from `$quiz/screens`.
