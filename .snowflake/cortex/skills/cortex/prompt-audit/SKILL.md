---
name: cortex-prompt-audit
description: "7-item audit checklist for AI_COMPLETE prompts — structured-output usage, content completeness, doc_search pattern, injection safety. Use when prompts produce wrong keys, shallow content, or unsafe interpolation. Triggers: audit prompts, prompt quality, wrong keys, shallow explanation, prompt review, KeyError. Do NOT use for Cortex connectivity/calling errors (cortex-patterns) or SiS code patterns (sis-patterns)."
---

# When to Load

Parent skill `$cortex` routes here for AUDIT intent.

- `call_cortex_json` returns None or a dict missing expected content
- Explanations are too short, generic, or missing per-option reasoning
- Question generation produces questions missing required fields (difficulty, domain, etc.)
- A prompt change was made and needs review before deploying
- Suspecting that a prompt is not dollar-quoted or lacks `$$` sanitization

# When NOT to Use

- Cortex connectivity/calling issues -> use `$cortex/patterns`
- SiS rendering patterns -> use `$sis/patterns`
- Pre-deploy scan -> use `$sis/pre-deploy`

# Instructions

Read the app's AI call sites in full — `_cortex.py`, `_questions.py`, and every page that builds a prompt (`pages/quiz.py`, `pages/recommendations.py` if present). Find every string passed to `call_cortex()` / `call_cortex_json()`. For each prompt, check all 7 items below. Report PASS or FAIL per item. On FAIL: show the function name, line number, and the offending text.

## Scan Items

**1. Structured output requested**
Every JSON-expecting call must go through `call_cortex_json` with a `response_format` schema from `RESPONSE_FORMATS` (see `$cortex/patterns`) — not free-text "return JSON" instructions.
- PASS: `call_cortex_json(prompt, fmt_key)` used; schema covers all keys the rendering code reads
- FAIL: prose-only "Return ONLY valid JSON" / "no markdown fences" instructions used instead of a schema, or a schema is missing keys the code reads (causes `.get(...)` returning None)

**2. No ambiguous key descriptions**
Each key description in the prompt must be specific and actionable. Flag these patterns as FAIL:
- `"Brief explanation of..."` - "brief" causes the model to skip detail
- `"Explanation of why..."` - too generic; model produces one sentence
- `"doc_url"` key with instruction to provide a URL - model hallucinates URLs
- `"URL to documentation"` or `"link to docs"` - model will guess or hallucinate

Good patterns (PASS):
- `["First key reason...", "Second reason..."]` - JSON array for why_correct (rendered as bullet list)
- `"for each wrong option (A, C, D), one sentence explaining why it is incorrect"`
- `"exactly 2-3 words for Snowflake docs search (e.g. 'Cortex Search'). No URLs. No commas. Max 3 words."` - doc_search pattern

**3. Question generation prompt - required fields**
Applies to prompts that generate quiz questions. Must include all of:
- difficulty level with full `DIFFICULTY_GUIDE` description (not just the word "easy"/"medium"/"hard")
- domain name
- topic list
- a "DO NOT generate any of these questions" block built from `_get_shown_texts()` (round_history + current question), truncated to 80 chars, last 10
- length guidance alongside the keys (`question_text` max 500 chars, options max 200 chars each) — the schema guarantees shape, not length
- PASS: all are present, difficulty description is the full DIFFICULTY_GUIDE text (multi-sentence with CONSTRAINT and STYLE)
- FAIL: any are missing - especially flag if difficulty is just a bare word ("easy"/"medium"/"hard") without the full guide description including CONSTRAINT and STYLE
- N/A: prompt is not a question generation prompt

**4. Explanation prompt - required context**
Applies to prompts that generate explanations. Must include ALL of: full question text, all answer options with letter labels (e.g. `A) text  B) text`), correct answer letter(s), what the student selected, and explicit list of wrong option letters (e.g. `wrong options: A, B, D`).
- PASS: all items present, `why_correct` described as a JSON array (list), `doc_search` key required (not `doc_url`)
- FAIL: if `why_wrong` description does not mention specific option letters - model will write one generic sentence instead of per-option explanations
- FAIL: if `why_correct` is described as a string instead of a JSON array - code expects list for bullet rendering
- FAIL: if prompt asks for `doc_url` instead of `doc_search` - model hallucinates URLs
- N/A: prompt is not an explanation prompt

**5. Dollar-quoting**
Prompt must be passed via `$$...$$` quoting, not single-quote `'...'`.
- PASS: `$${safe_prompt}$$` pattern used
- FAIL: `'{prompt}'` pattern used - single quotes break on any apostrophe in question text

**6. `$$` sanitization**
Before interpolating the prompt into `$$...$$`, the code must call `.replace("$$", "$ $")`.
- PASS: `safe_prompt = prompt.replace("$$", "$ $")` (or equivalent) is present
- FAIL: missing - a `$$` in any question text will break the SQL query

**7. doc_search pattern (not doc_url)**
All prompts that need documentation references must use `doc_search` (2-3 keyword search terms), NOT `doc_url` or "URL" or "link".
Code converts to URL post-parse: `https://docs.snowflake.com/en/search?q={query}`
- PASS: prompt says `"doc_search"` with word limit instruction (e.g. "exactly 2-3 words for Snowflake docs search. No URLs. No commas. Max 3 words.")
- FAIL: prompt asks for `"doc_url"`, `"URL"`, `"link"`, or `"documentation URL"` - model will hallucinate URLs

---

## Output

For each prompt found, a summary table:

| # | Check | Status | Notes |
|---|-------|--------|-------|
| 1 | Structured output requested | PASS/FAIL | missing keys: ... |
| 2 | No ambiguous key descriptions | PASS/FAIL | line X: `"Brief explanation..."` |
| 3 | Question prompt required fields | PASS/FAIL/N/A | missing: ... |
| 4 | Explanation prompt required context | PASS/FAIL/N/A | missing: ... |
| 5 | Dollar-quoting | PASS/FAIL | |
| 6 | `$$` sanitization | PASS/FAIL | |
| 7 | doc_search pattern | PASS/FAIL | |

**Final verdict per prompt:**
- All 7 PASS -> "Prompt is reliable and safe."
- Any FAIL -> "Rewrite required." - show the corrected prompt in full.
