# Future feature ideas

Optional features that are **designed but not yet implemented** in the skill pack. `$quiz/features` currently implements **Exam Simulation, Flashcards, AI Study Recommendation, Comparison**. To bring one back, re-add its spec to `$quiz/features` (and re-wire the cross-references noted below).

---

## Quick Stats Sidebar

Live mini-stats during an active quiz, shown in the sidebar below "End Round" (only when `screen == "quiz"`): current streak, round accuracy, current domain. Computed live from `st.session_state["round_history"]` + the current question — no DB query, no new table.

## Spaced Repetition (Smart Review)

A new Home **SOURCE** mode ("SMART REVIEW") that re-serves whole MCQs the user got wrong, prioritized by a recency × error-count score (`error_count * (1 / max(days_since_last_error, 1))`), ignoring the domain/difficulty filters. Source: `QUIZ_REVIEW_LOG` → match back to `QUIZ_QUESTIONS`, or re-ask via AI. No new table. **Note:** overlaps with the Flashcards Leitner schedule — if both are wanted, consider having Smart Review read `FLASHCARD_PROGRESS` due/box so the MCQ re-asking and card spacing share one schedule.

## Achievement Badges

Gamification badges computed from logs on each render (not stored), shown in the sidebar: First Perfect (100% round), Century (100+ questions), Week Streak (sessions on 5+ days in last 7), Domain Master (90%+ in a domain, min 10 q), Speed Demon (20-q round under 5 min). Data: `QUIZ_SESSION_LOG`, `QUIZ_REVIEW_LOG`. No new table.

## Misconception Analysis (Review page)

Per wrong-answer card: a "🧠 Diagnose Error" button → AI diagnoses the thinking error behind the user's specific pick (CKE-grounded), written once back to the row. An aggregate "Your Error Patterns" section surfaces recurring patterns when ≥3 diagnoses exist.
- **Requires a DDL change:** `ALTER TABLE {db}.QUIZ_<CODE>.QUIZ_REVIEW_LOG ADD COLUMN selected_answer VARCHAR, misconception VARCHAR;` (re-add them with this `ALTER`). The write-back must then populate `selected_answer` (resolved like `correct_answer`: `"{letter}) {full text}"`).
- New `RESPONSE_FORMATS["misconception"]` {misconception, contrast_with_correct, how_to_avoid} and `["misconception_patterns"]` {recurring_patterns[], advice}; add both to the `$cortex` grounded-paths + delimiting lists.

## Flag a Question

A "🚩 Flag Question" button on the quiz post-answer area to report bad questions; flags surface on the Admin page + a maintenance Automation.
- **Requires a new table:** `QUIZ_FLAGS (flag_id AUTOINCREMENT, flagged_at, question_id, question_text, reason, comment VARCHAR(500), status DEFAULT 'OPEN')`, generated when the feature is enabled.
- Admin question-manager shows OPEN flag count per question + lists flagged rows; the `customization.md` Automations recipe regenerates flagged bank questions (`status='OPEN' AND question_id IS NOT NULL`) and sets `status='REGENERATED'`.
