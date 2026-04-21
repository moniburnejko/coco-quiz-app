---
name: quiz-questions
description: "Question generation patterns — DIFFICULTY_GUIDE, topic scheduling, deduplication, fallback chain, retry logic, validation. Use when loading questions or generating via AI. Triggers: generate questions, question generation, topic schedule, deduplication, difficulty guide, fallback chain, question validation, DIFFICULTY_GUIDE"
parent_skill: quiz
---

# When to Load

Parent skill `$quiz` routes here for QUESTIONS intent.

- Loading questions from DB or generating via AI
- Building or modifying the topic schedule
- Working on question deduplication logic
- Debugging question quality or coverage issues

# When NOT to Use

- Prompt audit checklist -> use `$cortex/prompt-audit`
- Cortex connectivity/errors -> use `$cortex/patterns`
- Quiz screen UI -> use `$quiz/screens`

---

# DIFFICULTY_GUIDE

**REQUIRED constant** — must be defined at module level alongside `EXAM_CODE`, `CORTEX_MODEL`, `PASS_THRESHOLD`. Pass the full multi-sentence description to AI_COMPLETE prompts — never bare words like "easy" or "medium".

```python
DIFFICULTY_GUIDE = {
    "easy": (
        "EASY - Single concept recall. 'What is X?', 'Which feature does Y?' "
        "One correct answer clearly distinguishable. Tests terminology and definitions. "
        "CONSTRAINT: Reference only 1 Snowflake feature per question. "
        "STYLE: Direct factual question, no scenario setup needed."
    ),
    "medium": (
        "MEDIUM - Applied scenario involving 2 concepts. Real-world use case "
        "requiring understanding of relationships, not just recall. "
        "CONSTRAINT: Must include a scenario/context sentence before the question. "
        "STYLE: 'A team wants to... Which approach/function/configuration should they use?'"
    ),
    "hard": (
        "HARD - Multi-step analysis combining 3+ concepts. Troubleshooting, "
        "subtle differences between similar features, cost/performance/security trade-offs. "
        "CONSTRAINT: At least 2 plausible distractors. No obviously wrong options. "
        "STYLE: 'Given this configuration... what is the most likely cause/best approach?'"
    ),
}
```

Each tier has CONSTRAINT (what the question must do) and STYLE (how the question reads). The prompt validator (`$cortex/prompt-audit` item 4) checks for these multi-sentence descriptions.

---

# Topic Scheduling

`_build_topic_schedule(domains, domain_filter, round_size)` builds a shuffled list of `(domain, topic)` pairs for fair coverage.

**Principle**: Generate questions per TOPIC, not per domain. This guarantees even coverage — if 5 topics and 20 questions, each topic gets exactly 4 questions.

**How it works:**
1. Collect all `(domain, topic)` pairs from `EXAM_DOMAINS.topics` JSON array
2. Filter by `domain_filter` if set (empty = all domains)
3. Repeat the pool until we have `round_size` entries
4. Shuffle for random order
5. Store in `st.session_state["_topic_schedule"]`

Built **once per round** (in Start Round handler). `get_question()` pops the next entry.

`parse_topics()` helper safely parses the VARIANT/JSON topics field:
```python
def parse_topics(topics_raw):
    if not topics_raw:
        return []
    try:
        return json.loads(str(topics_raw)) if isinstance(topics_raw, str) else list(topics_raw)
    except Exception:
        return []
```

Reference: see `_build_topic_schedule()` and `parse_topics()` in `quiz.py`

---

# Deduplication

**DB path**: Collect shown question texts from `round_history` + current question via `_get_shown_texts()`. Pass as bind params to `NOT IN` clause in SQL:
```sql
WHERE question_text NOT IN (:3, :4, :5, ...)
```

**AI path**: Collect `question_text[:80]` from round_history. Include as "DO NOT repeat these already-asked questions:" block in the prompt.

**Key rule**: Do NOT use separate `shown_question_ids` or `shown_question_texts` keys in session_state. Mutable objects in SiS session_state are unreliable. Always derive from `round_history`.

Reference: see `_get_shown_texts()` in `quiz.py`

---

# Source Logic

Three modes controlled by `question_source` in session state:

| Mode | Behavior |
|------|----------|
| `"db"` | Query DB; if DB exhausted (all 3 fallback levels return None), auto-fallback to `generate_ai_question` |
| `"ai"` | Call `generate_ai_question` with topic from schedule, retry up to 5x; return `None` if all fail |
| `"mix"` | 20% chance AI first; if AI fails, fall through to DB; if DB exhausted, try AI again |

**Answer shuffling**: BEFORE returning any question from `get_question()`, apply `_shuffle_options(q)` to randomize answer positions. This is critical — without it, AI-generated questions always have the correct answer at position A. See "Answer Position Shuffling" section below for the function.

**Difficulty distribution** (mixed mode): 30% easy, 50% medium, 20% hard. Implementation pattern:
```python
r = random.random()
difficulty = "easy" if r < 0.30 else ("medium" if r < 0.80 else "hard")
```
Do NOT use `random.choice(["easy", "medium", "hard"])` — that gives 33/33/33.

**Guards**:
- If domain lookup fails (domain_name not in EXAM_DOMAINS), set `last_cortex_error` and return `None`. Do NOT default to `domain_id = "1"` — this silently picks a wrong domain.
- Before calling `random.choice(eligible)`, check that the list is non-empty. If empty, set `last_cortex_error = "No eligible domains for the current filter"` and return `None`.

---

# Fallback Chain (DB Path)

When loading from DB, try in order:
1. `domain + difficulty` (excluding shown questions)
2. `domain only` (excluding shown)
3. `domain only` (full pool, no exclusion)
4. `generate_ai_question()` (auto-fallback when DB exhausted)

Reference: see `_get_db_question()` in `quiz.py`

---

# Validation

After parsing AI response, validate:
- Must have: `question_text`, `option_a`, `option_b`, `correct_answer`
- If response contains extra keys (`why_correct`, `why_wrong`), **strip them** before validation — the model sometimes includes explanation keys alongside question keys. Alternatively, using `.get()` for all field access is an acceptable defensive pattern that achieves the same result
- **Length — two mechanisms**:
  1. The AI prompt MUST include length guidance: `question_text (string, max 500 chars)`, `option_a through option_e (string, max 200 chars each)` — so the model targets the right length
  2. After parsing, apply safety-net truncation: `data["question_text"][:500]`, `data["option_a"][:200]`, etc. — this should rarely activate if the prompt constraint works, but prevents DB overflow
- **Retry loop**: Wrap the entire Cortex call → parse → validate sequence in `for _attempt in range(5):`. On any failure (call returns None, parse fails, validation fails), `continue` to next attempt. After all 5 exhausted, set `last_cortex_error` and return `None`:
  ```python
  for _attempt in range(5):
      raw = call_cortex(prompt)
      if not raw: continue
      data = parse_cortex_json(raw)
      if not valid(data): continue
      return build_question(data)  # success
  st.session_state["last_cortex_error"] = f"AI generation failed after 5 attempts for {domain}/{difficulty}"
  return None
  ```

---

# Answer Position Shuffling

AI models tend to place the correct answer at position A. This makes quizzes trivially solvable. **Every question MUST have its options shuffled before display.**

Apply shuffling in `get_question()` after receiving the question from ANY source (AI or DB):

```python
def _shuffle_options(q):
    """Shuffle option positions and remap correct_answer."""
    letters = ["A", "B", "C", "D", "E"]
    items = [(lt, q.get(f"OPTION_{lt}")) for lt in letters if q.get(f"OPTION_{lt}")]
    random.shuffle(items)
    old_to_new = {}
    for i, (old_lt, text) in enumerate(items):
        new_lt = letters[i]
        q[f"OPTION_{new_lt}"] = text
        old_to_new[old_lt] = new_lt
    # clear unused option slots
    for lt in letters[len(items):]:
        q[f"OPTION_{lt}"] = None
    # remap correct_answer
    old_correct = q.get("CORRECT_ANSWER", "")
    new_correct = ",".join(old_to_new.get(c.strip(), c.strip()) for c in old_correct.split(","))
    q["CORRECT_ANSWER"] = new_correct
    return q
```

Call `_shuffle_options(q)` on every question returned by `get_question()` before storing in session state.

---

# Topic-Focused Prompt

When a topic is available from the schedule:

```
TOPIC CONSTRAINT (mandatory): Your question MUST be specifically about "{topic}".
Do not write a generic domain question. The question stem must reference "{topic}" concepts directly.
```

Also filter `key_facts` to topic-relevant lines before including in prompt. Track used topics to avoid generating about topics already covered this round.

Reference: see `generate_ai_question()` in `quiz.py`

---

# AI Question JSON Format

The prompt MUST include max length constraints alongside the JSON keys (so the model respects them):

```json
{
  "question_text": "max 500 chars",
  "is_multi": false,
  "option_a": "max 200 chars",
  "option_b": "max 200 chars",
  "option_c": "max 200 chars (optional)",
  "option_d": "max 200 chars (optional)",
  "option_e": "max 200 chars (optional, for 5-option questions)",
  "correct_answer": "A"
}
```

Some SnowPro questions have 5 options. Include `option_e` in the generation prompt as optional. After parsing, set `OPTION_E` from `option_e` if present.

After validation, the code adds UPPERCASE keys (`QUESTION_TEXT`, `DOMAIN_ID`, `DOMAIN_NAME`, `DIFFICULTY`, etc.) and returns the dict for in-memory use during the current round.

**Note**: Runtime AI questions are **ephemeral** — they are NOT persisted to `QUIZ_QUESTIONS`. Only pre-loaded questions (from CSV/`$setup-exam` generation) exist in the table. The `source = 'AI_GENERATED'` field is for display/tracking, not for insert.

---

## Output

Questions generated or loaded per the DIFFICULTY_GUIDE, topic schedule, and validation rules.
