---
name: cortex-prompt-audit
description: "8-item audit checklist for AI_COMPLETE prompts — JSON reliability, content completeness, doc_search pattern, injection safety. Use when prompts produce parse errors, wrong keys, or shallow content. Triggers: audit prompts, prompt quality, parse error, wrong keys, shallow explanation, prompt review, KeyError"
parent_skill: cortex
---

# When to Load

Parent skill `$cortex` routes here for AUDIT intent.

- `parse_cortex_json` raises a KeyError or returns None for expected fields
- Explanations are too short, generic, or missing per-option reasoning
- Question generation produces questions missing required fields (difficulty, domain, etc.)
- A prompt change was made and needs review before deploying
- Suspecting that a prompt is not dollar-quoted or lacks `$$` sanitization

# When NOT to Use

- Cortex connectivity/calling issues -> use `$cortex/patterns`
- SiS rendering patterns -> use `$sis/patterns`
- Pre-deploy scan -> use `$sis/pre-deploy`

# Instructions

Read `quiz.py` in full. Find every string passed to `call_cortex()`. For each prompt, check all 8 items below. Report PASS or FAIL per item. On FAIL: show the function name, line number, and the offending text.

## Scan Items

**1. Explicit output instruction**
The prompt must contain `Return ONLY valid JSON` (or equivalent with "ONLY").
- PASS: phrase "ONLY" is present before or alongside "JSON"
- FAIL: says "return JSON" without "ONLY" - model may add explanation text before/after, breaking `parse_cortex_json`

**2. JSON shape shown as example**
The prompt must include the exact JSON shape expected, with key names and example values.
- PASS: full JSON example with all expected keys is present in the prompt
- FAIL: no example shape - model will invent key names, causing `.get("why_correct")` to return None

**2.5. No-markdown-fences instruction**
The prompt must explicitly say "no markdown fences" or "no code fences" or "no backticks".
- PASS: explicit instruction present (e.g., "no markdown fences, no extra text")
- FAIL: missing - the model frequently wraps JSON in ` ```json ... ``` ` without this instruction

**3. No ambiguous key descriptions**
Each key description must be specific and actionable. Flag these patterns as FAIL:
- `"Brief explanation of..."` - "brief" causes the model to skip detail
- `"Explanation of why..."` - too generic; model produces one sentence
- `"doc_url"` key with instruction to provide a URL - model hallucinates URLs
- `"URL to documentation"` or `"link to docs"` - model will guess or hallucinate

Good patterns (PASS):
- `["First key reason...", "Second reason..."]` - JSON array for why_correct (rendered as bullet list)
- `"for each wrong option (A, C, D), one sentence explaining why it is incorrect"`
- `"exactly 2-3 words for Snowflake docs search (e.g. 'Cortex Search'). No URLs. No commas. Max 3 words."` - doc_search pattern

**4. Question generation prompt - required fields**
Applies to prompts that generate quiz questions. Must include all of:
- difficulty level with full `DIFFICULTY_GUIDE` description (not just the word "easy"/"medium"/"hard")
- domain name
- topic list
- a "DO NOT generate any of these questions" block built from `_get_shown_texts()` (round_history + current question), truncated to 80 chars, last 10
- PASS: all are present, difficulty description is the full DIFFICULTY_GUIDE text (multi-sentence with CONSTRAINT and STYLE)
- FAIL: any are missing - especially flag if difficulty is just a bare word ("easy"/"medium"/"hard") without the full guide description including CONSTRAINT and STYLE
- N/A: prompt is not a question generation prompt

**5. Explanation prompt - required context**
Applies to prompts that generate explanations. Must include ALL of: full question text, all answer options with letter labels (e.g. `A) text  B) text`), correct answer letter(s), what the student selected, and explicit list of wrong option letters (e.g. `wrong options: A, B, D`).
- PASS: all items present, `why_correct` described as a JSON array (list), `doc_search` key required (not `doc_url`)
- FAIL: if `why_wrong` description does not mention specific option letters - model will write one generic sentence instead of per-option explanations
- FAIL: if `why_correct` is described as a string instead of a JSON array - code expects list for bullet rendering
- FAIL: if prompt asks for `doc_url` instead of `doc_search` - model hallucinates URLs
- N/A: prompt is not an explanation prompt

**6. Dollar-quoting**
Prompt must be passed via `$$...$$` quoting, not single-quote `'...'`.
- PASS: `$${safe_prompt}$$` pattern used
- FAIL: `'{prompt}'` pattern used - single quotes break on any apostrophe in question text

**7. `$$` sanitization**
Before interpolating the prompt into `$$...$$`, the code must call `.replace("$$", "$ $")`.
- PASS: `safe_prompt = prompt.replace("$$", "$ $")` (or equivalent) is present
- FAIL: missing - a `$$` in any question text will break the SQL query

**8. doc_search pattern (not doc_url)**
All prompts that need documentation references must use `doc_search` (2-3 keyword search terms), NOT `doc_url` or "URL" or "link".
Code converts to URL post-parse: `https://docs.snowflake.com/en/search?q={query}`
- PASS: prompt says `"doc_search"` with word limit instruction (e.g. "exactly 2-3 words for Snowflake docs search. No URLs. No commas. Max 3 words.")
- FAIL: prompt asks for `"doc_url"`, `"URL"`, `"link"`, or `"documentation URL"` - model will hallucinate URLs

---

## Output

For each prompt found, a summary table:

| # | Check | Status | Notes |
|---|-------|--------|-------|
| 1 | Explicit output instruction | PASS/FAIL | |
| 2 | JSON shape example | PASS/FAIL | missing keys: ... |
| 3 | No ambiguous key descriptions | PASS/FAIL | line X: `"Brief explanation..."` |
| 4 | Question prompt required fields | PASS/FAIL/N/A | missing: ... |
| 5 | Explanation prompt required context | PASS/FAIL/N/A | missing: ... |
| 6 | Dollar-quoting | PASS/FAIL | |
| 7 | `$$` sanitization | PASS/FAIL | |
| 8 | doc_search pattern | PASS/FAIL | |

**Final verdict per prompt:**
- All 8 PASS -> "Prompt is reliable and safe."
- Any FAIL -> "Rewrite required." - show the corrected prompt in full.
