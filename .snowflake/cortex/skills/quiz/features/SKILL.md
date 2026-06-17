---
name: quiz-features
description: "Optional features for the quiz app — exam simulation, flashcards, quick stats, spaced repetition, achievement badges, AI study recommendations. Implement ONLY features explicitly requested by the user, never by default. Triggers: exam simulation, timer, flashcard, spaced repetition, smart review, achievement, badge, streak, study recommendation, AI recommendation. Do NOT use for the core quiz flow (quiz-screens)."
---

# When to Load

Parent skill `$quiz` routes here for FEATURES intent.

- User explicitly requests an optional feature in their prompt
- Adding new functionality beyond the core quiz flow

# When NOT to Use

- Core quiz screens (home, quiz, summary) -> use `$quiz/screens`
- Question generation logic -> use `$quiz/questions`
- UI styling/badges -> use `$quiz/design`

---

# IMPORTANT

This skill contains **OPTIONAL** features. Do NOT implement any feature unless the user explicitly requests it in their prompt. Each feature is self-contained — implement only the requested ones. Core quiz functionality (home, quiz, summary, wrong answers, learning dashboard) does NOT require this skill.

All **visual rendering** (badges, cards, callouts, buttons, charts) follows `$quiz/design`, the single source for styling — including the **`md()` `$`-escaping of every dynamic string** (question text, mnemonics, AI output) before `st.markdown`/`st.info`/`st.write`. Each feature below specifies *what* it shows and *where* in the flow — it never defines colors, theme keys, or chart formatting.

---

# Feature 1: Exam Simulation Mode

**OPTIONAL** — implement only if user requests exam simulation, timed quiz, mock exam, practice exam, or exam mode.

## What

New mode alongside Practice. Fixed question count matching real exam, countdown timer, all domains weighted by exam blueprint, no explanations during simulation, final report with pass/fail vs real threshold.

## Navigation

Generated as its own page: `pages/exam_simulation.py`, appended to the `st.navigation` list in `main.py` (title "Exam Simulation"). The Quiz page (practice flow) stays unchanged.

```
pages/exam_simulation.py (own page):
  sim_config -> sim_quiz -> sim_results   (internal state machine via _sim_screen)
```

## UI spec

**sim_config screen**: Shows exam params as read-only info:
- Exam: {EXAM_NAME} ({EXAM_CODE})
- Questions: {N} (weighted by domain)
- Time limit: {M} minutes
- Pass threshold: {PASS_THRESHOLD}%
- "Start Exam" button (primary, full-width)

**sim_quiz screen**: Same as regular quiz but:
- Prominent timer: `st.metric("Time Remaining", f"{minutes}:{seconds:02d}")` at top
- Progress bar below timer
- No AI-explanation button during the timed simulation (explanations suppressed in sim mode)
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

Own page `pages/flashcards.py` (added to `st.navigation` in `main.py`, title "Flashcards"). Shows wrong answers as flashcards — front = question, click to reveal = correct answer + mnemonic + docs link.

## UI spec

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

AI-powered exam readiness analysis. Own page `pages/recommendations.py`.

## Navigation

Added to the `st.navigation` list in `main.py` (title "AI Study Recommendation") when this feature is enabled.

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

No `overall_assessment` key. **Grounding (defer to `$cortex`):** in `cke`/`custom` mode retrieve `<doc_context>` for each weak topic, ground the topic recommendations in it ("answer ONLY from the provided documentation"), and set each topic's `doc_url` from the chunk's real `SOURCE_URL` — the `doc_search` → `https://docs.snowflake.com/en/search?q=` conversion is for **`none` mode only**. (The readiness/weak-domain analysis over the user's own error history is meta-analysis, like the debrief.)

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

Sets `domain_filter`, `difficulty`, `round_size=10`, `screen="home"` (with the config pre-filled), then calls `st.switch_page("pages/quiz.py")` — `st.navigation` owns the page, so there is no nav-widget key to mutate (the old `nav_pills` redirect machinery does not exist in the multipage app).

## Session state keys

`_ai_recommendations` (dict|None), `_rec_cache_key` (str|None)

---

# Feature 7: Misconception Analysis (Review page)

**OPTIONAL** — implement only if user requests misconception analysis, error diagnosis, "why do I keep getting these wrong", or thinking-pattern analysis.

## What

Extends the **Review page** (no new page): per wrong answer, AI diagnoses the thinking error behind the user's specific selection; an aggregate section surfaces recurring error patterns.

## Requirements

`QUIZ_REVIEW_LOG.selected_answer` + `.misconception` columns (in the Step 3 DDL) — the write-back must populate `selected_answer` (resolved like `correct_answer`: `"{letter}) {full_text}"`).

## Per-card diagnosis

On each wrong-answer card: button "🧠 Diagnose Error" → `call_cortex_json(prompt, "misconception")` with question, all options, the user's selection, the correct answer — wrapped per the untrusted-content delimiting rule AND **grounded per `$cortex`**: in `cke`/`custom` mode embed the question's retrieved `<doc_context>` (reuse the chunks fetched for its explanation) and diagnose ONLY from the docs, never built-in knowledge; fail visibly on empty retrieval. Schema:

```
"misconception": "the likely thinking error behind choosing {selected} (1-2 sentences, names the confused concepts)",
"contrast_with_correct": "the key distinction the user missed (1 sentence)",
"how_to_avoid": "a practical check to apply next time (1 sentence)"
```

Render in a bordered container on the card. **Write-once**: persist `misconception` back to the row (UPDATE by `log_id`, bind params) — later visits read from the DB, no repeat call. Button shows only when the column is NULL; otherwise render the stored text.

## Patterns section ("Your Error Patterns")

At the top of Wrong Answers, when ≥3 rows have `misconception IS NOT NULL`: button → one `call_cortex_json(..., "misconception_patterns")` over the collected diagnoses → `{recurring_patterns (array, max 3), advice (string)}`. Cache in session state keyed by diagnosis count; render as bullets + callout. (Meta-analysis over the already-grounded per-card diagnoses — no new doc retrieval needed, like the debrief.)

## Session state keys

`_misconception_{log_id}` transient render state only; persistence is the DB column.

---

# Feature 8: Flag a Question

**OPTIONAL** — implement only if user requests question flagging, reporting bad questions, or bank quality control.

## What

A small "🚩 Flag Question" button on the quiz screen (post-answer area) that records quality complaints; flags surface on the Admin page and in the Automations maintenance recipe.

## DDL (generated ONLY when this feature is enabled; add to Step 3)

```sql
CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.QUIZ_FLAGS (
    flag_id        NUMBER AUTOINCREMENT PRIMARY KEY,
    flagged_at     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP(),
    question_id    NUMBER,            -- NULL for runtime-AI questions
    question_text  VARCHAR,           -- snapshot (AI questions have no id)
    reason         VARCHAR,
    comment        VARCHAR(500),
    status         VARCHAR DEFAULT 'OPEN'
);
```

## UI spec

Post-answer, next to (not competing with) "Next": small secondary button → popover/expander with reason pills (`wrong key` / `unclear` / `typo` / `other`) + optional comment (`st.text_input`, max 500 chars) → INSERT with bind params (question_id when the question came from the bank, text snapshot always) → toast "Flagged". One flag per question per round (disable after submit).

## Admin + Automations integration

- Admin question manager shows OPEN flag count per question and lists flagged rows.
- The customization.md §5c Automations recipe gains a step: regenerate flagged bank questions (`status='OPEN' AND question_id IS NOT NULL`) via the seeding machinery, set `status='REGENERATED'`, report.

---

## Output

Only the requested optional features implemented and integrated into quiz.py, following the UI patterns from `$quiz/design` and state management from `$quiz/screens`.
