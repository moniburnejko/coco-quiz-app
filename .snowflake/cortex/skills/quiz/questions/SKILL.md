---
name: quiz-questions
description: "Question generation patterns - DIFFICULTY_GUIDE, topic scheduling, deduplication, fallback chain, retry logic, schema validation. Use when loading questions or generating via AI. Triggers: generate questions, question generation, topic schedule, deduplication, difficulty guide, fallback chain, question validation, DIFFICULTY_GUIDE. Do NOT use for page/screen contracts (quiz-screens) or prompt audits (cortex-prompt-audit)."
---

# When to Load

Parent skill `$quiz` routes here for QUESTIONS intent.

- Loading questions from DB or generating via AI
- Building or modifying the topic schedule
- Working on question deduplication logic
- Debugging question quality or coverage issues

# When NOT to Use

- Prompt audit checklist -> use `$cortex`
- Cortex connectivity/errors -> use `$cortex`
- Quiz screen UI -> use `$quiz/screens`

---

# Module Boundaries

All question logic lives in `_questions.py`: `parse_topics`, `_build_topic_schedule`, `generate_ai_question`, `get_question`, `_shuffle_options`, `_get_shown_texts`. AI calls go through `call_cortex_json` from `_cortex.py`; `DIFFICULTY_GUIDE` and `RESPONSE_FORMATS` come from `_config.py`. Pages never call Cortex directly.

**Model routing (MANDATORY):** every question-generation call passes the **generation** group's model - `call_cortex_json(prompt, "question", model=model_for("generation"))` (`model_for` from `_cortex.py`, `$cortex`). Without the `model=` arg the call silently stays on `CORTEX_MODEL` and the Admin model selector does nothing.

---

# DIFFICULTY_GUIDE

**REQUIRED constant** - must be defined at module level alongside `EXAM_CODE`, `CORTEX_MODEL`, `PASS_THRESHOLD`. Pass the full multi-sentence description to AI_COMPLETE prompts - never bare words like "easy" or "medium".

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

Each tier has CONSTRAINT (what the question must do) and STYLE (how the question reads). The prompt validator (`$cortex` item 4) checks for these multi-sentence descriptions.

---

# Topic Scheduling

`_build_topic_schedule(domains, domain_filter, round_size)` builds a shuffled list of `(domain, topic)` pairs for fair coverage.

**Principle**: Generate questions per TOPIC, not per domain. This guarantees even coverage - if 5 topics and 20 questions, each topic gets exactly 4 questions.

**How it works** (no early repeats, no back-to-back repeats):
1. Collect all **distinct** `(domain, topic)` pairs from `EXAM_DOMAINS.topics` (`parse_topics`).
2. Filter by `domain_filter` if set (empty = all domains).
3. Shuffle the distinct pairs - this first pass means **every topic appears once before any topic repeats**.
4. If `round_size` exceeds the pool, append more **whole passes**, each re-shuffled - never a flat `pool * n` slice that would re-hit the same topics early.
5. Trim to `round_size`, then break adjacency: if a pair equals the one before it, swap it forward so the **same topic never lands twice in a row**.
6. Store in `st.session_state["_topic_schedule"]`; `get_question()` pops the next entry.

Built **once per round** (in Start Round handler). Sampling without replacement within each pass is what stops a 20-question round from drilling the same three topics; tiling by whole passes keeps coverage even when the pool is small.

```python
def _build_topic_schedule(domains, domain_filter, round_size):
    sel = [d for d in domains if not domain_filter or d["DOMAIN_NAME"] in domain_filter]
    pairs = [(d["DOMAIN_NAME"], t) for d in sel for t in parse_topics(d.get("TOPICS"))]
    if not pairs:                                        # topic-less exam: schedule by domain
        pairs = [(d["DOMAIN_NAME"], None) for d in sel]
    if not pairs:
        return []
    schedule = []
    while len(schedule) < round_size:                    # tile by WHOLE shuffled passes
        chunk = pairs[:]; random.shuffle(chunk)
        schedule.extend(chunk)
    schedule = schedule[:round_size]
    for i in range(1, len(schedule)):                    # break same-topic adjacency
        if schedule[i] == schedule[i - 1]:
            for j in range(i + 1, len(schedule)):
                if schedule[j] != schedule[i - 1]:
                    schedule[i], schedule[j] = schedule[j], schedule[i]; break
    return schedule
```

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

Reference: see `_build_topic_schedule()` and `parse_topics()` in `_questions.py`

---

# Deduplication

**DB path**: Collect shown question texts from `round_history` + current question via `_get_shown_texts()`. Pass as bind params to `NOT IN` clause in SQL:
```sql
WHERE question_text NOT IN (?, ?, ?, ...)
```

**AI path**: Collect `question_text[:80]` from round_history. Include as "DO NOT repeat these already-asked questions:" block in the prompt.

**Near-duplicate rejection (token-Jaccard).** Exact-text `NOT IN` / `WHERE NOT EXISTS` only catches byte-identical text, but two AI questions can reword the same fact. After a candidate passes structural + correctness validation, compute the token-Jaccard overlap of its `question_text` against each shown text this round; on any pair `>= 0.6`, reject and regenerate. **Cap rejections at 2** per question: on the 3rd near-duplicate, stop regenerating and serve a DB question instead (in pure `ai` mode with no DB, accept the candidate rather than fail the round) - a thin topic must never spin the loop forever.
```python
def _tokens(text):
    return set(re.findall(r"[a-z0-9]+", (text or "").lower()))      # re imported at module level

def _too_similar(text, shown, threshold=0.6):
    a = _tokens(text)
    if not a:
        return False
    return any(len(a & b) / len(a | b) >= threshold
               for b in (_tokens(s) for s in shown) if b)
```

**Key rule**: Do NOT use separate `shown_question_ids` or `shown_question_texts` keys in session_state. Mutable objects in SiS session_state are unreliable. Always derive from `round_history`.

Reference: see `_get_shown_texts()` in `_questions.py`

---

# Source Logic

Three modes controlled by `question_source` in session state:

| Mode | Behavior |
|------|----------|
| `"db"` | Query DB; if DB exhausted (all 3 fallback levels return None), auto-fallback to `generate_ai_question` |
| `"ai"` | Call `generate_ai_question` with topic from schedule, retry up to 5x; return `None` if all fail |
| `"mix"` | 20% chance AI first; if AI fails, fall through to DB; if DB exhausted, try AI again |

**Answer shuffling**: BEFORE returning any question from `get_question()`, apply `_shuffle_options(q)` to randomize answer positions. This is critical - without it, AI-generated questions always have the correct answer at position A. See "Answer Position Shuffling" section below for the function.

**Difficulty distribution** (mixed mode): 30% easy, 50% medium, 20% hard. Implementation pattern:
```python
r = random.random()
difficulty = "easy" if r < 0.30 else ("medium" if r < 0.80 else "hard")
```
Do NOT use `random.choice(["easy", "medium", "hard"])` - that gives 33/33/33.

**Guards**:
- If domain lookup fails (domain_name not in EXAM_DOMAINS), set `last_cortex_error` and return `None`. Do NOT default to `domain_id = "1"` - this silently picks a wrong domain.
- Before calling `random.choice(eligible)`, check that the list is non-empty. If empty, set `last_cortex_error = "No eligible domains for the current filter"` and return `None`.

---

# Fallback Chain (DB Path)

When loading from DB, try in order:
1. `domain + difficulty` (excluding shown questions)
2. `domain only` (excluding shown)
3. `domain only` (full pool, no exclusion)
4. `generate_ai_question()` (auto-fallback when DB exhausted)

**Every level selects with `ORDER BY RANDOM() LIMIT 1` - never sequential / ordered-by-id.** combined with the `NOT IN (shown)` dedup this gives shuffled, non-repeating coverage. (`_shuffle_options` then randomizes option positions - see below.)

Reference: see `_get_db_question()` in `_questions.py`

---

# Validation

Generation goes through `call_cortex_json(prompt, "question")` - the `RESPONSE_FORMATS["question"]` schema (see `$cortex`) **guarantees** the required keys and types (`question_text`, `is_multi`, `option_a`, `option_b`, `correct_answer`), so there is no key-stripping or shape-checking step.

What the code still does:
- **Length - two mechanisms** (the schema guarantees shape, not length):
  1. The AI prompt MUST include length guidance: `question_text (string, max 500 chars)`, `option_a through option_e (string, max 500 chars each)` - so the model targets the right length
  2. After the call, apply safety-net truncation: `data["question_text"][:500]`, `data["option_a"][:500]`, etc. - this should rarely activate if the prompt constraint works, but prevents DB overflow (matches the `VARCHAR(500)` option columns)
- **Structural gate** (`_answers_valid`): `correct_answer` letters reference options that are actually present (no `"E"` when `option_e` is empty), `>=2` options, `>=1` correct. Regenerate on violation.
- **Correctness gate** (`_verify_correct`, the `$cortex` `"verify"` pass): a temperature-0, doc-grounded re-check that the marked answer is right and every distractor is wrong. `is_correct=false` (or a failed call) regenerates - structural validity alone never ships a question.
- **Near-duplicate gate** (`_too_similar`): token-Jaccard vs this round's shown texts, cap 2 rejects then DB fallback (see Deduplication).
- **Retry loop**: structural -> correctness -> near-duplicate, each retry shifting the variety angle:
  ```python
  dupe_rejects = 0
  for attempt in range(5):
      data = call_cortex_json(build_prompt(attempt), "question", model=model_for("generation"))
      if data is None: continue                                  # call failed / NULL / guard
      if not _answers_valid(data): continue                      # structural
      if not _verify_correct(data, chunks, model): continue      # correctness ($cortex "verify")
      if _too_similar(data["question_text"], shown):             # near-duplicate (token-Jaccard)
          dupe_rejects += 1
          if dupe_rejects > 2: break                             # give up -> DB fallback (accept in pure ai)
          continue
      return build_question(data)                                # success
  st.session_state["last_cortex_error"] = f"AI generation failed after 5 attempts for {domain}/{difficulty}"
  return None
  ```
  ```python
  def _verify_correct(data, chunks, model):
      """$cortex 'verify' pass: temperature-0, grounded in the same chunks. True = answer stands."""
      v = call_cortex_json(build_verify_prompt(data, chunks), "verify", model=model)
      return bool(v and v.get("is_correct"))                     # None (call failed) -> reject + regenerate
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

Reference: see `generate_ai_question()` in `_questions.py`

---

# Question variety (angle rotation + nonce)

Repeated `(domain, topic)` pairs (a small bank, a long round) must not yield the same question. Generation already runs at a moderate temperature (`FEATURE_TEMPERATURE["question"]`, `$cortex`); layer two cheap variety controls on top:

- **Angle rotation** - rotate how the same topic is framed, keyed off the attempt index:
  ```python
  QUESTION_ANGLES = ["a direct definition", "a real-world scenario", "a troubleshooting situation",
      "a comparison between two similar features", "a best-practice / when-to-use decision",
      "a cost or performance trade-off"]                       # _config.py
  angle = QUESTION_ANGLES[attempt % len(QUESTION_ANGLES)]      # shifts on every retry
  ```
- **Nonce** - a throwaway token the model uses only to vary phrasing, never as content:
  ```python
  nonce = str(random.randint(1000, 9999))
  ```

Append to the prompt: `Approach this as {angle}. Variety token (do NOT treat as content; use only to vary phrasing): {nonce}`. The angle **varies the framing only** - it never overrides the `DIFFICULTY_GUIDE` STYLE/CONSTRAINT or the topic constraint (a `hard` question stays multi-step analysis whatever the angle).

---

# Doc grounding (MANDATORY in cke/custom mode - see `$cortex`)

In `cke`/`custom` grounding mode the generator grounds in retrieved docs and answers **ONLY from them - never the model's built-in knowledge**. Per `$cortex`:

```python
chunks = search_docs(f"{domain_name}: {topic}")
```
- **Embed `chunks` in `<doc_context>`** as the primary source ("answer only from this; do not use prior knowledge"), with the topic-relevant `key_facts` as supporting scope (keeps coverage when a topic is thin in the docs). Store the top chunk's `SOURCE_URL` on the question as `DOC_URL`.
- **If `chunks == []`:** broaden the query once (`f"{domain_name}"`); if still empty, **fail this generation** (`return None` - the retry loop counts it). Do NOT generate from built-in knowledge.

`none` mode (non-Snowflake exams, explicit opt-in) is the only ungrounded path - `$cortex` owns that branch. Grounding never narrows exam scope: the topic/domain still drive the question.

---

# AI Question Format

The response shape is enforced by `RESPONSE_FORMATS["question"]` (see `$cortex`) - `question_text`, `is_multi`, `option_a..e`, `correct_answer`. The prompt's job is **content**: it MUST still include the max-length guidance alongside the keys (`question_text` max 500 chars, options max 500 chars each), the topic constraint, the full `DIFFICULTY_GUIDE` text, and the dedup block - the schema cannot express any of that.

Some SnowPro questions have 5 options. `option_e` is in the schema as optional. After the call, set `OPTION_E` from `option_e` if present.

The code then adds UPPERCASE keys (`QUESTION_TEXT`, `DOMAIN_ID`, `DOMAIN_NAME`, `DIFFICULTY`, etc.) and returns the dict for in-memory use during the current round.

**Note**: every validated runtime AI question is **persisted to `QUIZ_QUESTIONS`** (`source='AI_GENERATED'`) so the bank grows as the user practices - see **Bank persistence** below. The `source` field distinguishes `'MANUAL'` (CSV) vs `'AI_GENERATED'` (runtime + Admin "Generate batch") rows. Build-time never mass-generates.

---

# Bank persistence

Runtime AI generation **grows the bank**: when `generate_ai_question` returns a validated question, **INSERT it into `QUIZ_QUESTIONS`** (bind params per `$sis`; `source='AI_GENERATED'`; columns `domain_id`, `domain_name`, `difficulty`, `question_text`, `is_multi`, `option_a..e`, `correct_answer` - `question_id`/`created_at` are auto). The persist is **fire-and-forget**: the in-memory question keeps `QUESTION_ID = None` (`history_item.question_id` is `int/None`). **"Question Bank" mode serves persisted rows with no AI call.**

- **Exact-text dedup**: insert only when no row with the same `question_text` already exists - `INSERT … SELECT … WHERE NOT EXISTS (SELECT 1 FROM {FQ}.QUIZ_QUESTIONS WHERE QUESTION_TEXT = ?)` (qmark bind, per `$sis`) - so re-asked concepts don't pile up duplicates.
- **Always on** (no toggle). Persist the question as generated; `_get_db_question` re-shuffles option order on load anyway.
- A failed persist must **never break the round** - wrap the INSERT so an error just skips saving (the question still displays). Bank rows come from rounds + the Admin "Generate batch" button.

## Output

Questions generated or loaded per the DIFFICULTY_GUIDE, topic schedule, and validation rules.
