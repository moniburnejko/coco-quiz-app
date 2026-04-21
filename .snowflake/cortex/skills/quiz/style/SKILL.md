---
name: quiz-style
description: "UI styling conventions for the quiz app — badge colors, section labels, chart colors, CSS rules, button style. Use for any visual or layout work in quiz.py. Triggers: badge, styling, color, CSS, chart, section label, button style, card, UI, layout"
parent_skill: quiz
---

# When to Load

Parent skill `$quiz` routes here for STYLE intent.

- Any UI work — styling, layout, new components
- Reviewing visual consistency across screens
- Adding new badges, cards, or sections

# When NOT to Use

- Platform constraints (session, rerun) -> use `$sis/patterns`
- Pre-deploy scan -> use `$sis/pre-deploy`
- Screen behavior contracts -> use `$quiz/screens`

---

# Badges

`:color-badge[TEXT]` markdown syntax is the standard for all metadata display.

| Element | Syntax | Example |
|---------|--------|---------|
| Domain | `:blue-badge[DOMAIN_NAME]` | `:blue-badge[PERFORMANCE & OPTIMIZATION]` |
| Easy difficulty | `:green-badge[EASY]` | |
| Medium difficulty | `:orange-badge[MEDIUM]` | |
| Hard difficulty | `:red-badge[HARD]` | |
| Date | `:gray-badge[YYYY-MM-DD]` | `:gray-badge[2026-04-03]` |
| Passed | `:green-badge[PASSED]` | `:green-badge[PASSED] above 75% threshold` |
| Not yet | `:orange-badge[NOT YET]` | `:orange-badge[NOT YET] 12.5% to go` |
| Perfect score | `:green-badge[PERFECT SCORE]` | |
| Correct | `:green-badge[✅ CORRECT]` | |
| Incorrect | `:red-badge[❌ INCORRECT]` | |
| Error count | `:red-badge[N errors]` | `:red-badge[5 errors]` |
| Exam ready | `:green-badge[EXAM READY]` | |
| Not ready | `:orange-badge[NOT READY YET]` | |

Domain badges always UPPERCASE. Difficulty badges always UPPERCASE.

---

# Titles

Define `EXAM_NAME` constant at module level (e.g., `EXAM_NAME = "SnowPro Core"`).

| Location | Format | Example |
|----------|--------|---------|
| App title (render_home) | `{EXAM_NAME} Quiz` | "SnowPro Core Quiz" |
| Sidebar title | `{EXAM_NAME} Quiz` | "SnowPro Core Quiz" |
| Summary (pass) | `🎉 Round Complete!` | emoji conditional |
| Summary (fail) | `📋 Round Complete` | emoji conditional |

No exam code caption under titles. Sidebar shows only `{EXAM_NAME} Quiz` and navigation pills.

---

# Section Headers

Use `st.markdown("**LABEL**")` for all section headers. Labels are UPPERCASE.

Do NOT use `st.subheader()` — it's too visually heavy for section labels.

Examples: `**QUESTIONS**`, `**DOMAINS**`, `**DIFFICULTY**`, `**SOURCE**`, `**WRONG ANSWERS**`, `**SCORE PER SESSION**`, `**ERRORS BY DOMAIN**`, `**FOCUS AREAS**`, `**TOPICS TO REVIEW**`, `**NEXT STEPS**`

---

# Question Text

Use `st.markdown(f"#### {text}")` (h4 heading) for question text display. NOT `st.subheader()` (too large) or `st.markdown(f"**{text}**")` (too small).

---

# CSS Injection

`unsafe_allow_html=True` is used ONLY ONCE — for sidebar pill font size, immediately after `st.set_page_config()`:

```python
st.set_page_config(layout="centered", page_icon="...")
st.markdown("""<style>
    [data-testid="stSidebar"] [data-testid="stPills"] button { font-size: 1.1rem; }
</style>""", unsafe_allow_html=True)
```

No other CSS injection anywhere. `layout="centered"` always — never `"wide"`.

---

# Color Scheme

**Charts:**
- Score line: `#29b5e8` (Snowflake blue), `mark_line(point=True)`
- Error bars: `#F1914C` (orange), `mark_bar()`
- Threshold rules: gray dashed (`strokeDash=[4, 4]`)
- Integer axes (e.g., error counts): use `axis=alt.Axis(tickMinStep=1)` — never show fractional ticks for count data
- Suppress axis titles with `title=None` when meaning is obvious from the section header and data labels (e.g., domain names on Y-axis, error counts on X-axis)
- Y-axis with long domain names: use `axis=alt.Axis(labelLimit=500)` to prevent truncation

**Callouts:**
- `st.info()` ONLY for mnemonic box (`🧠`). Never for pass/fail, readiness, or error counts.
- **NEVER** use `st.success()`, `st.warning()`, or `st.error()`. All status and validation messages use badges (e.g., `:orange-badge[Please select an answer]` instead of `st.warning()`).

---

# Docs Link

Always use `st.markdown(f"📖 [Snowflake Documentation]({url})")` for Snowflake docs links. Do NOT use `st.caption` (too subtle) or abbreviated "Docs" (unclear). The 📖 emoji makes it scannable.

Applies in: quiz screen (correct + incorrect), review cards, AI recommendations topics.

---

# Cards

`st.container(border=True)` for grouped content:
- Wrong answer cards (summary + review)
- Focus area cards (AI recommendations)
- Topics to review cards
- Next steps container
- Explanation containers (why correct, why wrong)

---

# Buttons

- No emoji in button labels: `"Start Round"` not `"▶️ Start Round"`
- Action buttons: `type="primary"`, `use_container_width=True`
- All pills: `label_visibility="collapsed"` (bold markdown label above instead)

---

## Output

Visually consistent UI using the badge, color, and component conventions defined here.
