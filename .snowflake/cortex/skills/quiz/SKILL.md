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
| STYLE | badge, color, CSS, chart, section label, button, card, layout, styling | `style/SKILL.md` |
| FEATURES | exam simulation, timer, flashcard, spaced repetition, smart review, achievement, badge, streak, study recommendation, AI recommendation | `features/SKILL.md` |

## Workflow

```
User request
  ↓
Intent Detection
  ├─→ SCREENS   → Load screens/SKILL.md
  ├─→ QUESTIONS → Load questions/SKILL.md
  ├─→ STYLE     → Load style/SKILL.md
  └─→ FEATURES  → Load features/SKILL.md (OPTIONAL — only when user requests a feature)
```

## Routing

This parent does no work itself. Match the request against the intent table, then **open the sub-skill file(s) and follow them**:

- SCREENS intent → load `.snowflake/cortex/skills/quiz/screens/SKILL.md` and follow it.
- QUESTIONS intent → load `.snowflake/cortex/skills/quiz/questions/SKILL.md` and follow it.
- STYLE intent → load `.snowflake/cortex/skills/quiz/style/SKILL.md` and follow it.
- FEATURES intent → load `.snowflake/cortex/skills/quiz/features/SKILL.md` and follow it — ONLY when the user explicitly requests an optional feature. Do NOT load for regular quiz work.

Multiple sub-skills may apply to a single task (e.g., adding a new page needs both `screens/` for contracts and `style/` for UI conventions).

## Capabilities

- **Screens**: Screen flow (home/quiz/summary + review tabs), session state contract (28 keys), history item schema, write-back, explanation state machine
- **Questions**: DIFFICULTY_GUIDE (3-tier with CONSTRAINT/STYLE), topic scheduling, deduplication, source logic (db/ai/mix), fallback chain, validation, answer shuffling
- **Style**: Badge colors, section labels (`st.markdown("**LABEL**")`), chart colors, CSS injection rules, button conventions, card patterns, title conventions
- **Features**: OPTIONAL add-ons — exam simulation mode, flashcard review, quick stats sidebar, spaced repetition, achievement badges, AI study recommendations

## Output

Quiz app code conforming to screen contracts, question generation rules, and UI styling conventions.
