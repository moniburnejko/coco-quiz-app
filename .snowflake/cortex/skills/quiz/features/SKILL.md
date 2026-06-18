---
name: quiz-features
description: "Optional features for the quiz app — exam simulation, flashcards, AI study recommendation, comparison, remedial round. Implement ONLY features explicitly requested by the user, never by default. Triggers: exam simulation, timed exam, mock exam, flashcard, study card, study recommendation, exam readiness, comparison, compare options, A vs B, remedial round, retry wrong answers. Do NOT use for the core quiz flow (quiz-screens)."
---

# When to Load

Parent skill `$quiz` routes here for FEATURES intent.

- User explicitly requests an optional feature in their prompt
- Adding new functionality beyond the core quiz flow

# When NOT to Use

- Core quiz screens (home, quiz, summary) -> use `$quiz/screens`
- Question generation logic -> use `$quiz/questions`
- UI styling/badges -> use `$quiz/design`

---

# IMPORTANT

This skill contains **OPTIONAL** features. Do NOT implement any feature unless the user explicitly requests it in their prompt. Each feature is self-contained — implement only the requested ones. Core quiz functionality (home, quiz, summary, wrong answers, learning dashboard) does NOT require this skill.

All **visual rendering** (badges, cards, callouts, buttons, charts) follows `$quiz/design`, the single source for styling — including the **`md()` `$`-escaping of every dynamic string** (question text, mnemonics, AI output) before `st.markdown`/`st.info`/`st.write`. Each feature below specifies *what* it shows and *where* in the flow — it never defines colors, theme keys, or chart formatting.

Five features are specced here: **Exam Simulation, Flashcards, AI Study Recommendation, Comparison, Remedial Round.** Further ideas not yet implemented (Quick Stats, Smart Review, Achievement Badges, Misconception Analysis, Flag a Question) live in `docs/future-features.md` — re-add a spec here when one is requested.

---

# Feature 1: Exam Simulation Mode

**OPTIONAL** — implement only if user requests exam simulation, timed quiz, mock exam, practice exam, or exam mode.

## What

New mode alongside Practice. Fixed question count matching real exam, countdown timer, all domains weighted by exam blueprint, no explanations during simulation, final report with pass/fail vs real threshold.

## Navigation

Generated as its own page: `pages/exam_simulation.py`, appended to the `st.navigation` list in `main.py` (title "Exam Simulation"). The Quiz page (practice flow) stays unchanged.

```
pages/exam_simulation.py (own page):
  sim_config -> sim_quiz -> sim_results   (internal state machine via _sim_screen)
```

## UI spec

**sim_config screen**: Shows exam params as read-only info, read from `_config.py` (`EXAM_QUESTION_COUNT`, `EXAM_TIME_LIMIT_MIN` — captured from the study guide at setup, `$setup-exam` Step 5):
- Exam: {EXAM_NAME} ({EXAM_CODE})
- Questions: {EXAM_QUESTION_COUNT} (weighted by domain)
- Time limit: {EXAM_TIME_LIMIT_MIN} minutes
- Pass threshold: {PASS_THRESHOLD}% (a study proxy — the official SnowPro score is scaled 0–1000, pass 750)
- "Start Exam" button (primary, full-width)
- If either value is null (the guide didn't state it), prompt the user to confirm the count/time before enabling Start.

**sim_quiz screen**: Same as regular quiz but:
- Prominent timer: `st.metric("Time Remaining", f"{minutes}:{seconds:02d}")` at top, inside an `st.fragment(run_every="10s")` that recomputes remaining time from `_sim_end_time` and auto-submits the round when it hits 0. Streamlit does NOT rerun on its own, so a plain "check on render" timer would neither tick nor expire while the candidate sits on a question — the fragment is what actually enforces the limit. (Coarse `run_every` like 10s bounds warehouse reruns; if the SiS Streamlit version lacks `st.fragment(run_every=…)`, degrade to checking elapsed on each interaction and say so in a caption.)
- Progress bar below timer
- No AI-explanation button during the timed simulation (explanations suppressed in sim mode)
- NO source selection — questions are sourced bank-first then AI, weighted by domain (see Implementation notes)
- End conditions: all questions answered OR `_sim_end_time` reached (auto-submit)

**sim_results screen**: Detailed breakdown:
- Overall: pass/fail badge (vs `PASS_THRESHOLD`), score, time used
- Domain table: domain name, questions asked, correct, %, exam weight
- Caption: the % is a study proxy for the official scaled 750/1000 — the real exam publishes no raw passing %
- **Write-back** (below) runs once when the round ends, before this screen renders

## Session state keys

`_quiz_mode` (str: "PRACTICE"/"EXAM SIMULATION"), `_sim_start_time` (datetime), `_sim_end_time` (datetime — `start + limit`; the single source of truth for the timer), `_sim_time_limit` (int, seconds), `_sim_questions` (int), `_sim_screen` (str: "config"/"quiz"/"results"). Initialized in the feature page — NOT in the core `init_session_state` contract (`$quiz/screens`).

## Implementation notes

- **Question count + time limit** come from `_config.py` (`EXAM_QUESTION_COUNT`, `EXAM_TIME_LIMIT_MIN`), captured from the user's study guide at setup (`$setup-exam` Step 5) — NEVER hardcode per-code guesses. Exam structure is revised across versions and recalled values go stale, the same risk as the exam code itself (`$setup-exam` Step 1b). If a value is null, confirm it with the user against their guide before starting.
- **Domain distribution must sum to exactly N** — use largest-remainder: `floor(weight_pct/100 * N)` per domain, then hand the leftover questions one each to the domains with the largest fractional remainders until the counts total N. (Independent `round()` per domain does NOT sum to N.)
- **Question sourcing is bank-first** — a live mock needs 65–100 questions and generating them all is slow + costly: per domain, draw from `QUIZ_QUESTIONS` (`ORDER BY RANDOM()`, deduped — `$quiz/questions`) up to its quota, then generate only the shortfall via the grounded `get_question()` path. If a domain still can't be filled, shrink its quota and show a visible caption ("Only M of N for {domain}") — never pad from built-in knowledge. Spinner/progress during the fill.
- **Timer**: set `_sim_end_time = _sim_start_time + _sim_time_limit` once at Start; the `st.fragment(run_every=…)` countdown reads it and auto-submits on expiry (see sim_quiz).
- Reuse the existing `get_question()` patterns for generation — just wrap with the timer and the weighted, bank-first config.

## Write-back (when the round ends)

A finished simulation persists like a practice round so it feeds Review + Flashcards + history:
- Each wrong answer → a `QUIZ_REVIEW_LOG` row (the same `write_review_log` path as practice).
- One `QUIZ_SESSION_LOG` row, tagged so mock ≠ practice: when this feature is enabled, add the column once — `ALTER TABLE {db}.QUIZ_<CODE>.QUIZ_SESSION_LOG ADD COLUMN IF NOT EXISTS session_type VARCHAR DEFAULT 'PRACTICE';` — and write `'SIMULATION'` for sim rounds (practice rounds keep the default). `clear_caches()` after the writes.

---

# Feature 2: Flashcards

**OPTIONAL** — implement only if user requests flashcards, study cards, or review cards.

## What

A **FLASHCARDS tab on the Review page** (NOT a separate page — `$quiz/screens`). It turns the user's **wrong answers** into **atomic, recall-forcing** study cards — never the verbatim quiz question. Each wrong answer is AI-decomposed into a small set of one-fact cards, CKE-grounded, and reviewed with lightweight **Leitner spaced repetition**.

## Card model (atomic — one fact per card)

`card_type` ∈ `qa` | `cloze` | `compare`:
- **qa** — short question → short answer ("At what level does a masking policy apply? → column").
- **cloze** — one sentence with exactly ONE blank ("A ___ filters which rows a role sees.").
- **compare** — directional discrimination for an interference pair ("Masking vs row access: which removes whole rows? → row access").

Fields: `card_front`, `card_back` (≤ ~120 chars, as short as possible — never a paragraph or list), `card_type`, `domain_name`/`topic` (from the source row — badge + context cue on the front), `doc_url` (the CKE chunk's real `SOURCE_URL`; empty only in `none` mode). `card_id` = a **deterministic** key `<source_log_id>:<ordinal>` (NOT a hash of the AI-generated text — so it's stable across rebuilds; the Leitner key). HARD rules: exactly one fact per card; NO "name the N …"/list cards (split into atomic cards or overlapping clozes); a cloze has exactly one blank; **the MCQ option text is NEVER copied verbatim** (that's the recognition-not-recall anti-pattern this redesign removes). All dynamic text via the `md()` `$`-escaper (`$quiz/design`).

## AI transform (grounded — `$cortex`)

Source = the user's `QUIZ_REVIEW_LOG` rows (each carries `question_text`, the resolved `correct_answer`, `domain_name`, `difficulty`, `doc_url`). Per row, ONE grounded `call_cortex_json(prompt, "flashcards")` mines **2–4 atomic cards**: (a) the load-bearing fact behind the correct answer (qa/cloze); (b) the distinction the user missed (a `compare` card for the interference pair). The prompt instructs: split enumerations, strip all MCQ scaffolding, make each card answerable cold, keep backs minimal, add a domain/topic context cue. **Grounding (mandatory, cke/custom):** reuse/retrieve `chunks = search_docs(question_text)`, embed as `<doc_context>` ("build cards ONLY from this documentation + the row's correct answer; never prior knowledge"); on empty retrieval broaden once then **skip that row visibly** ("couldn't ground — retry"), never fabricate; `doc_url = chunks[0]["SOURCE_URL"]`. `none` mode = ungrounded, empty `doc_url`. New `RESPONSE_FORMATS["flashcards"]` = `{cards: [{card_type, card_front, card_back, topic}]}` (see `$cortex`); `domain_name`/`doc_url` are attached by Python from the source row + chunk, NEVER asked of the model (same rule as never asking for a URL). A post-call validator drops any card with >1 blank, an empty back, or list-shaped content (if a row's cards all fail, skip that row visibly). Generation is **idempotent per wrong-answer row**: build + persist cards ONLY for rows with no `FLASHCARD_PROGRESS` entry yet (`source_log_id` absent); already-carded rows are skipped, so existing cards and their Leitner boxes are never regenerated or disturbed.

## Spacing — Leitner boxes (persisted)

DDL (generated ONLY when this feature is enabled; add to `$setup-exam` Step 3) — the table is the **durable card store + Leitner state**, so a due card always has content to render:
```sql
CREATE TABLE IF NOT EXISTS {database}.QUIZ_<CODE>.FLASHCARD_PROGRESS (
    card_id        VARCHAR PRIMARY KEY,   -- deterministic: <source_log_id>:<ordinal>
    source_log_id  NUMBER,
    card_type      VARCHAR,
    card_front     VARCHAR,
    card_back      VARCHAR,
    domain_name    VARCHAR,
    topic          VARCHAR,
    doc_url        VARCHAR,
    box            NUMBER DEFAULT 1,
    due_date       DATE,
    updated_at     TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);
```
Boxes 1–5 → cadences **1 / 2 / 4 / 7 / 14 days**. On reveal, three buttons: **"Again"** → box 1; **"Good"** → box+1; **"Easy"** → box+2 (cap 5). Each click `MERGE`s the row by `card_id` (`box`, `due_date = CURRENT_DATE + cadence`, bind params) + `clear_caches()`. The cached `load_flashcard_progress()` loader is **registered in `clear_caches()`** in `_data.py` (no `ttl`, like every other loader — `$sis`), so the MERGE actually refreshes the deck. The tab's **deck = `FLASHCARD_PROGRESS` rows due today** (`due_date <= CURRENT_DATE`), read straight from the table (content lives there — no dependency on regenerating this session).

## UI (Review FLASHCARDS tab)

**On-demand:** a **"Build cards from my wrong answers"** button generates + **persists** cards for any new (un-carded) wrong answers (spinner; never auto-run on page load; the CKE grounding guard applies). The deck shown is `FLASHCARD_PROGRESS` rows **due today** (content + box from the table). Then per card: `st.container(border=True)` + domain/difficulty/type badges + `card_front` → **"Show Answer"** → `card_back`, the `📖` docs link (qa/compare), and the three Leitner buttons. `st.caption("Due today: N")`. Empty state: `:green-badge[All caught up!] Nothing due — build more from new wrong answers.`

**Data source**: the cached `load_flashcard_progress()` (the due-today deck — content + box from the table); the build step reads `load_review_log()` only to find un-carded rows. **Session state keys**: `_flashcard_cards` (the loaded due deck), `_flashcard_index` (int), `_flashcard_revealed` (bool). Because cards persist and `card_id` is deterministic per `(source_log_id, ordinal)`, a rebuild adds cards only for new wrong answers and never disturbs existing Leitner boxes.

**Scope:** flashcards are atomic recall-card study (Review tab) built from wrong answers — distinct from the core quiz (full MCQs) and from any future spaced-repetition *quiz* mode (`docs/future-features.md`).

---

# Feature 3: AI Study Recommendation

**OPTIONAL** — implement only if user requests study recommendation, AI recommendation, readiness analysis, or exam readiness.

## What

AI-powered exam readiness analysis. Own page `pages/recommendations.py`.

## Navigation

Added to the `st.navigation` list in `main.py` (title "AI Study Recommendation") when this feature is enabled.

## Condition

- < 2 sessions → caption: "Complete at least 2 quiz sessions to see personalized recommendations."
- ≥ 2 sessions AND no errors recorded → all-clear state: `:green-badge[EXAM READY]` + caption "No weak areas detected — you're scoring above threshold across the board." (do NOT show the "complete 2 sessions" caption — it's misleading here).
- ≥ 2 sessions AND errors present → generate recommendations.

## Trigger

"Generate AI Study Recommendation" button. Spinner during call. After saving results to session state, call `st.rerun()` so the button is hidden on the next render cycle. The button only shows when `has_cache` is False.

## Prompt inputs

Session count, avg score, total questions, domain error counts (`load_domain_errors()`), wrong-question samples (last 30 per domain), domain topic lists. The per-domain sampling needs a dedicated cached loader — **add `load_wrong_question_samples()` to `_data.py`** (a `ROW_NUMBER() OVER (PARTITION BY domain_id ORDER BY logged_at DESC) <= 30` query over `QUIZ_REVIEW_LOG`) **and register it in `clear_caches()`** (no `ttl`, like every loader — `$sis`). It does not exist by default.

## JSON output schema

```json
{
  "exam_readiness": {"ready": false, "gap_pct": 12.5, "message": "..."},
  "weak_domains": [{"domain_name": "...", "error_count": 5, "recommendation": "..."}],
  "weak_topics": [{"domain_name": "...", "topic": "...", "recommendation": "...", "doc_search": "..."}],
  "study_plan": ["step 1", "step 2", "step 3"],
  "recommended_difficulty": "medium",
  "recommended_domain": "..."
}
```

No `overall_assessment` key. **Grounding (defer to `$cortex`):** in `cke`/`custom` mode retrieve `<doc_context>` for each weak topic, ground the topic recommendations in it ("answer ONLY from the provided documentation"), and set each topic's `doc_url` from the chunk's real `SOURCE_URL` — the `doc_search` → `https://docs.snowflake.com/en/search?q=` conversion is for **`none` mode only**. (The readiness/weak-domain analysis over the user's own error history is meta-analysis, like the debrief.)

## Computed in Python, NOT by the model

Deterministic values are computed in Python from the user's own stats and **override** whatever the model returns (an LLM does arithmetic unreliably — same rule as never asking it for a URL):
- `exam_readiness.ready` = `avg_score >= PASS_THRESHOLD`; `gap_pct` = `abs(avg_score - PASS_THRESHOLD)` — from the same `avg_score` shown on screen.
- `recommended_domain` = the domain with the most errors (`load_domain_errors()`).

The model authors only qualitative text: `exam_readiness.message`, the `recommendation` strings, `study_plan`, `recommended_difficulty`, and `weak_topics[].doc_search`. The schema still includes the computed keys (Python overwrites them after the call).

## Prompt constraints (MUST be in the AI prompt)

- Respond in **English** (project rule — all generated text is English).
- `exam_readiness.message`: max 1 sentence, under 80 chars
- `weak_domains[].recommendation` and `weak_topics[].recommendation`: max 1 sentence, under 80 chars each
- `study_plan`: exactly 3 items, each under 80 chars, action-oriented verbs
- `weak_topics`: up to 5 UNIQUE entries (deduplicate by topic name). `doc_search`: exactly 2-3 words, no URLs, no commas, max 3 words
- Do NOT include `overall_assessment` key

## Display

1. Readiness: `:green-badge[EXAM READY]` or `:orange-badge[NOT READY YET]` — badge on its own line. Message as `st.caption()` on next line (NOT concatenated with badge).
2. Focus Areas: cards in columns with `:blue-badge[DOMAIN]` + `:red-badge[N errors]` + recommendation
3. Topics to Review: containers with domain badge + topic name + recommendation + `📖 [Snowflake Documentation]({doc_url})`; deduplicate by topic (use `seen_topics` set)
4. Next Steps: numbered list in container
5. Action buttons: "Start Focused Session: {recommended_domain} ({recommended_difficulty})" (include domain name in label) + "Regenerate"

## Start Focused Session

Sets `domain_filter`, `difficulty`, `round_size` (=10), `screen="home"`, then `st.switch_page("pages/quiz.py")`. For the pre-fill to actually take, the **Home screen widgets must seed their initial value from these session keys** (`$quiz/screens` Home contract) — otherwise the redirect lands on a blank Home. (`st.navigation` owns the page, so there is no nav-widget key to mutate — the old `nav_pills` redirect machinery does not exist in the multipage app.)

## Session state keys

`_ai_recommendations` (dict|None), `_rec_cache_key` (str|None)

---

# Feature 4: Comparison

**OPTIONAL** — implement only if user requests comparison, compare options, "A vs B", or concept contrast.

## What

Adds a **"⚖️ Compare two"** control to the **AI-explanation expander** on the quiz screen (alongside the core Deep dive — `$quiz/screens`). Pick **two** options and get a side-by-side discrimination of the underlying concepts. SnowPro questions are mostly "which similar feature fits?", so training that exact distinction is high-value. (Comparing two options is NOT in core — the core Deep dive explains ONE option; this feature adds the compare-two control.)

## UI spec

Inside the explanation expander, below the explanation + Deep dive: an option picker (`st.multiselect`, validated to **exactly 2**) + a **"⚖️ Compare two"** button → `call_cortex_json(prompt, "contrast")` → render a compact `aspect | A | B` table + the `exam_trap` as an `st.caption`, in `st.container(border=True)`. Default the picker to the user's wrong choice + the correct option (any two selectable). All dynamic text via the `md()` `$`-escaper.

## Grounding (mandatory — `$cortex`)

In `cke`/`custom` mode embed the two option texts + the question as `<doc_context>` ("compare ONLY from the provided documentation; never prior knowledge"); reuse the explanation's retrieved chunks, broaden once then fail visibly on empty — never built-in. `none` mode is the only ungrounded path. Schema `RESPONSE_FORMATS["contrast"]`: `concept_a`, `concept_b`, `differences[]` (max 4), `exam_trap`.

## Session state keys

`comparison` (None/{}/dict), reset on Next.

---

# Feature 5: Remedial Round

**OPTIONAL** — implement only if user requests remedial round, remedial quiz, retry wrong answers, or a re-test of missed questions.

## What

When enabled, a **failed practice round** gets a **"Remedial Round"** button on the summary screen (`$quiz/screens`) — a focused re-test of just the questions the user got wrong, reshuffled. It is NOT part of the core quiz: the basic summary on a failed round offers only Round Brief + Configure New Round; this feature inserts the Remedial Round button into that outcome.

## UI hook

On the summary of a **failed** practice round (`score < threshold`, ≥1 wrong) the feature adds a **"Remedial Round"** button (primary) alongside the core "Round Brief" + "Configure New Round". No new page, no Admin toggle — the feature's presence is the enable (a failed round always offers it). A perfect or passed round never shows it.

## Remedial pass contract

- **Queue**: the wrong items from `round_history`, **order shuffled**; `_shuffle_options` re-applied to every question so the option letters move (recall, not position memory).
- **Setup**: set `_round_type="remedial"`, `_remedial_queue`; reset counters / `q_index` / `round_history` for the pass.
- **During**: hints + the on-demand explanation behave normally.
- **Writes NOTHING**: the pass must **bypass `_write_back_results()`, the Round Brief, and all logging** — re-testing just-seen questions would inflate readiness stats and duplicate `QUIZ_REVIEW_LOG` rows. The feature adds the `if _round_type == "remedial"` guard to the summary write-back; core write-back is otherwise unconditional.
- **Remedial summary**: score + **"Configure New Round"** only (no chained remedials).
- **Reset**: `_round_type` returns to `"practice"` on any new round.

## Session state keys

`_round_type` (str: "practice"/"remedial", default "practice"), `_remedial_queue` (list, the wrong items queued). Feature-local — NOT in the core `init_session_state` contract.

---

## Output

Only the requested optional features implemented and integrated into the app, following the UI patterns from `$quiz/design` and state management from `$quiz/screens`.
