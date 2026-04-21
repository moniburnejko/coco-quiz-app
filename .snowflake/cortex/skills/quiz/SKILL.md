---
name: quiz
description: "Quiz app implementation — screen contracts, question generation, UI styling. Use when building or modifying any part of quiz.py. Triggers: screen, quiz, home, summary, review, questions, generate, badge, styling, layout, session state, write-back"
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

Multiple sub-skills may apply to a single task (e.g., adding a new screen needs both `screens/` for contracts and `style/` for UI conventions).

**FEATURES intent**: Load only when the user explicitly requests an optional feature (exam simulation, flashcards, etc.). Do NOT load for regular quiz work.

## Capabilities

- **Screens**: Screen flow (home/quiz/summary + review tabs), session state contract (28 keys), history item schema, write-back, explanation state machine
- **Questions**: DIFFICULTY_GUIDE (3-tier with CONSTRAINT/STYLE), topic scheduling, deduplication, source logic (db/ai/mix), fallback chain, validation, answer shuffling
- **Style**: Badge colors, section labels (`st.markdown("**LABEL**")`), chart colors, CSS injection rules, button conventions, card patterns, title conventions
- **Features**: OPTIONAL add-ons — exam simulation mode, flashcard review, quick stats sidebar, spaced repetition, achievement badges, AI study recommendations

## Output

Quiz app code conforming to screen contracts, question generation rules, and UI styling conventions.
