---
name: quiz-design
description: "THE single source for every visual rule in the quiz app — theme/config.toml keys, badge palette, chart colors + axis formatting, cards, buttons, titles, docs-link style. Use for any visual, color, theme, or chart work. Triggers: theme, color, badge, chart, config.toml, styling, layout, card, button, section label, docs link. Do NOT use for screen behavior/state (quiz-screens), question generation (quiz-questions), or platform constraints (sis)."
---

# When to Load

Parent skill `$quiz` routes here for DESIGN intent.

- Any UI work — styling, layout, new components
- Reviewing visual consistency across screens
- Adding new badges, cards, or sections

# When NOT to Use

- Platform constraints (session, rerun) -> use `$sis`
- Pre-deploy scan -> use `$sis`
- Screen behavior contracts -> use `$quiz/screens`

---

# Language

**All generated UI text is English** — every label, button, heading, badge, toast, caption, and section header. No other language, regardless of the conversation or prompt language. (Exam *content* follows the study guide.)

# Badges

`:color-badge[TEXT]` Markdown syntax is the standard for all metadata display.

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

Examples: `**QUESTIONS**`, `**DOMAINS**`, `**DIFFICULTY**`, `**SOURCE**`, `**SCORE PER SESSION**`, `**ERRORS BY DOMAIN**`, `**FOCUS AREAS**`, `**TOPICS TO REVIEW**`, `**NEXT STEPS**` (the summary's `WRONG ANSWERS` is an `st.expander` title now, not a bold label)

---

# Escaping dynamic text

Streamlit Markdown renders text between two `$` as LaTeX math, so any model- or DB-sourced string containing `$` (e.g. `SYSTEM$CLASSIFY`) renders garbled (everything between the two `$` turns into italic math). Route **every** rendered dynamic string — question text, options, explanation / deep-dive fields, mnemonics, summary + review + flashcard cards — through a `md()` helper in `_ui.py` that **backslash-escapes `$`** so it renders literally. Never pass raw question/option/AI text straight to `st.markdown` / `st.write`. (Enforced by the `$sis` pre-deploy scan.)

# Question Text

Use `st.markdown(f"#### {text}")` (h4 heading) for question text display. NOT `st.subheader()` (too large) or `st.markdown(f"**{text}**")` (too small).

---

# Theming Contract (config.toml — the ONLY styling mechanism)

NO `unsafe_allow_html` anywhere in the app (enforced by `$sis`). ALL visual styling = native Streamlit `[theme]` / `[theme.sidebar]` keys in `.streamlit/config.toml`; any key the pinned Streamlit doesn't recognize is ignored gracefully.

## Canonical default theme

```toml
[theme]
base = "light"
primaryColor = "#29b5e8"
linkColor = "#1572a1"
baseRadius = "0.5rem"
borderColor = "#d6e4ec"
showWidgetBorder = true
chartCategoricalColors = ["#29b5e8", "#F1914C", "#36B37E", "#7C5CFC"]

[theme.sidebar]
secondaryBackgroundColor = "#eef6fa"
```

## Available knobs (for the Step 1e custom-look dialog)

| User intent | Theme key(s) |
|---|---|
| Light/dark mode | `base` |
| Brand/accent color | `primaryColor` (+ `linkColor` to match) |
| Background tones | `backgroundColor`, `secondaryBackgroundColor` |
| Corner roundness (sharp/soft/round) | `baseRadius`, `buttonRadius` ("none"/"small"/"medium"/"large"/"full"/rem) |
| Borders on/off + color | `showWidgetBorder`, `borderColor`, `showSidebarBorder`, `dataframeBorderColor` |
| Font stack | `font`, `headingFont`, `codeFont` — **built-in stacks only** ("sans-serif"/"serif"/"monospace"); NO `fontFaces`/external font URLs (CSP) |
| Text sizing/weight | `baseFontSize`, `baseFontWeight`, `headingFontSizes`, `headingFontWeights` |
| Badge palette (`:green-badge[]` etc.) | `greenColor`/`redColor`/`orangeColor`/`blueColor`/`grayColor` + their `*BackgroundColor`/`*TextColor` variants |
| Chart series colors | `chartCategoricalColors` (array) |
| Sidebar distinct look | any of the above under `[theme.sidebar]` |

Rules:
- Map dialog answers ONLY to these keys; if the user asks for something theming cannot do (animations, per-element CSS, custom layout), say so and offer the nearest theme-level effect.
- **Charts ↔ theme alignment**: `chartCategoricalColors[0]` must equal the score-line constant and `[1]` the error-bar constant in `_config.py` (the Altair specs reference the constants explicitly).
- SiS caveats: `st.set_page_config` `page_title`/`page_icon`/`menu_items` are NOT supported in SiS — do not set them; `layout="centered"` always, never `"wide"` (set once, in `main.py`).
- Shared visual helpers (badges, cards, doc links) live in `_ui.py` as plain Streamlit components — no raw HTML.

---

# Color Scheme

**Charts** (the single source — `$quiz/screens` and `$quiz/features` reference this, never restate chart rules):

*Score per Session* (line): X = `LABEL:N` (NOMINAL, `sort=None` to keep chronological order; label `#{session_id} · {date}`) — never `:Q`, which interpolates floats (1.0, 1.1, …); Y = `SCORE_PCT:Q`, `scale=alt.Scale(domain=[0, 100])`; `mark_line(point=True, color="#29b5e8")` (Snowflake blue); threshold = dashed gray rule (`strokeDash=[4, 4]`) at `PASS_THRESHOLD`.

*Errors by Domain* (bar): X = `ERROR_COUNT:Q`, `axis=alt.Axis(tickMinStep=1, title=None)` (integer ticks, no fractional counts); Y = `DOMAIN_NAME:N`, `sort="-x"`, `axis=alt.Axis(labelLimit=500, title=None)` (full names, no truncation); `mark_bar(color="#F1914C")` (orange, NOT red).

*General*: suppress axis titles (`title=None`) when the section header + labels make meaning obvious; chart colors must match the `_config.py` constants (see the theming contract above).

**Callouts:**
- `st.info()` ONLY for mnemonic box (`🧠`). Never for pass/fail, readiness, or error counts.
- **NEVER** use `st.success()`, `st.warning()`, or `st.error()`. All status and validation messages use badges (e.g., `:orange-badge[Please select an answer]` instead of `st.warning()`).

---

# Docs Link

Always use `st.markdown(f"📖 [Snowflake Documentation]({url})")` for Snowflake docs links. Do NOT use `st.caption` (too subtle) or abbreviated "Docs" (unclear). The 📖 emoji makes it scannable.

Applies in: inside the on-demand AI-explanation expander (after the **💡 AI explanation** click — correct + incorrect alike; never auto-shown on the quiz screen), review cards, AI recommendations topics.

---

# Cards

`st.container(border=True)` for grouped content:
- Wrong answer cards (summary + review)
- Focus area cards (AI recommendations)
- Topics to review cards
- Next steps container
- Explanation containers (why correct, why wrong)
- Round Brief containers (summary): the PATTERNS + PRIORITY ACTIONS block, and the 🎯 FOCUS one-thing line (its own bordered container — NOT `st.info`; `st.info` is mnemonic-only)

---

# Buttons

- No emoji in button labels: `"Start Round"` not `"▶️ Start Round"`
- Action buttons: `type="primary"`, `use_container_width=True`
- All pills: `label_visibility="collapsed"` (bold Markdown label above instead)

---

## Output

Visually consistent UI using the badge, color, and component conventions defined here.
