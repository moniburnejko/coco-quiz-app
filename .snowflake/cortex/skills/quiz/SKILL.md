---
name: quiz
description: "Quiz app implementation — page/screen contracts, question generation, UI styling, optional features. Use when building or modifying any part of the generated app. Triggers: screen, page, quiz, home, summary, review, questions, generate, badge, styling, layout, session state, write-back"
---

# Quiz App

## When to Use

When building or modifying any part of quiz.py — screens, question generation, or UI styling.

## Intent Detection

| Intent | Triggers | Load |
|--------|----------|------|
| SCREENS | screen flow, quiz screen, home screen, summary, review, session state, write-back, explanation, history_item | `screens/SKILL.md` |
| QUESTIONS | generate questions, topic schedule, dedup, difficulty, fallback, DIFFICULTY_GUIDE, question validation | `questions/SKILL.md` |
| DESIGN | theme, color, badge, chart, config.toml, section label, button, card, layout, styling | `design/SKILL.md` |
| FEATURES | exam simulation, timer, flashcard, spaced repetition, smart review, achievement, badge, streak, study recommendation, AI recommendation | `features/SKILL.md` |

## Workflow

```
User request
  ↓
Intent Detection
  ├─→ SCREENS   → Load screens/SKILL.md
  ├─→ QUESTIONS → Load questions/SKILL.md
  ├─→ DESIGN    → Load design/SKILL.md
  └─→ FEATURES  → Load features/SKILL.md (OPTIONAL — only when user requests a feature)
```

## Routing

This parent does no work itself. Match the request against the intent table, then **open the sub-skill file(s) and follow them**:

- SCREENS intent → load `.snowflake/cortex/skills/quiz/screens/SKILL.md` and follow it.
- QUESTIONS intent → load `.snowflake/cortex/skills/quiz/questions/SKILL.md` and follow it.
- DESIGN intent → load `.snowflake/cortex/skills/quiz/design/SKILL.md` and follow it.
- FEATURES intent → load `.snowflake/cortex/skills/quiz/features/SKILL.md` and follow it — ONLY when the user explicitly requests an optional feature. Do NOT load for regular quiz work.

Multiple sub-skills may apply to a single task (e.g., adding a new page needs both `screens/` for contracts and `design/` for visual conventions).

## Capabilities

- **Screens**: Screen flow (home/quiz/summary + review tabs), session state contract (28 keys), history item schema, write-back, explanation state machine
- **Questions**: DIFFICULTY_GUIDE (3-tier with CONSTRAINT/STYLE), topic scheduling, deduplication, source logic (db/ai/mix), fallback chain, validation, answer shuffling
- **Design**: Theme (config.toml keys), badge palette, section labels, chart colors + axis formatting, cards, buttons, titles, docs-link — the single source for all visual rules
- **Features**: OPTIONAL add-ons — exam simulation mode, flashcard review, quick stats sidebar, spaced repetition, achievement badges, AI study recommendations

## App module map

The generated app is a decomposed multipage project under `app/`. What each file owns (read the module for HOW; the contracts are in the sub-skills above):

| File | Responsibility |
|------|----------------|
| `main.py` | Entry: `st.set_page_config` (first `st.` call), `init_session_state()`, sidebar title, `st.navigation([...]).run()` |
| `_config.py` | `EXAM_NAME`, `EXAM_CODE`, `CORTEX_MODEL`, `PASS_THRESHOLD`, `DIFFICULTY_GUIDE`, color + `RESPONSE_FORMATS` constants |
| `_cortex.py` | `call_cortex` + `call_cortex_json` (see `$cortex`) |
| `_data.py` | Cached loaders (`load_domains`, `load_session_stats`, `load_recent_sessions`, `load_domain_errors`, `load_config`) + `clear_caches()` |
| `_questions.py` | Topic schedule, `get_question`, AI generation, answer shuffling, dedup (`questions/`) |
| `_ui.py` | Shared render helpers — badges, cards, explanation expander, docs link (`design/`) |
| `_search.py` | Docs-CKE retrieval (`search_docs`/`docs_available`/`grounding_required`/`grounding_mode`) — MANDATORY doc grounding in cke/custom mode, never built-in knowledge; `none` = the only ungrounded path (see `$cortex`) |
| `pages/quiz.py` | QUIZ: home → quiz → summary state machine (hints, contrast, debrief, remedial) |
| `pages/review.py` | REVIEW: wrong-answer history + learning dashboard |
| `pages/admin.py` | ADMIN: config, question manager + Generate batch, bank stats, Cortex spend, tools |
| `pages/<feature>.py` | ONLY when a feature is requested (`features/`) — e.g. `exam_simulation.py`, `flashcards.py`, `recommendations.py` |

Navigation is native multipage (`st.Page` + `st.navigation` in `main.py`); `st.session_state` is shared across pages; optional features add their own pages, never tabs. Full flow/state/write-back contracts → `screens/`.

## Output

Quiz app code conforming to screen contracts, question generation rules, and UI styling conventions.
