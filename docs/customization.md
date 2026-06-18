# Customization

The quiz app has four layers you can tweak, from "one-line change" to "fork the pipeline":

1. **Model and runtime defaults** (edit `AGENTS.md`).
2. **Visual styling** (edit the app modules or tell the agent what to change via `$quiz/design`).
3. **Functional features** (opt-in via `$quiz/features`: exam simulation, flashcards, AI study recommendation, remedial round).
4. **Target exam** (swap the PDF for a different SnowPro cert, or with a little bit more work, any non-Snowflake exam like AWS / GCP / Azure).

Everything below is safe to iterate on: change, redeploy, keep going. Nothing is a one-way door.

---

## 1. Model and runtime defaults

### 1a - Switch the Cortex LLM

`AGENTS.md` > `cortex llm` section pins `claude-sonnet-4-6`. To change:

```markdown
preferred model: <llm_model_name>. store as constant `CORTEX_MODEL`.
```

Trade-off: `claude-sonnet-4-6` is the best quality for question generation and explanations but is also the slowest. `mistral-large2` or `llama3.1-405b` are noticeably faster but hallucinate more on Snowflake-specific minutiae. After changing, regenerate the app so the constant value propagates, or just edit `CORTEX_MODEL = "..."` in `_config.py` and redeploy.

### 1b - Round size defaults / default question source / any other default

Ask Snowflake CoCo to change it. :) 

Or:
- For round size: Open `pages/quiz.py`, find the home-screen block, look for the `st.selectbox("Questions", options=[...])` widget. Default options are typically `[5, 10, 20, 50]`. Change to taste and redeploy (+ re-run `$sis`).
- For question source: In `render_home` find the `question_source` selectbox. Options are `mix`, `db`, `ai`. If the user chose "No CSV/JSON" during `$setup-exam`, the default is already `ai`; otherwise `mix`. Change `index=` to change the default.
- And so on.

### 1c - AI explanation format

The on-demand AI explanation (a button shown after answering, for correct **and** incorrect) renders `why_correct`, `why_wrong` (per option), `mnemonic`, and a doc link. The model's JSON schema returns `doc_search`; in `cke`/`custom` mode Python attaches the real `doc_url` from the retrieved CKE chunk (the search-link heuristic is `none`-mode only). To change the shape (e.g. add a `related_docs` list), edit two places:
- The `RESPONSE_FORMATS["explanation"]` schema + the explanation prompt in `_generate_explanation` (ask for the new field);
- The quiz-screen explanation expander block (display the new field).

Run `$cortex` after the prompt change to catch JSON key mismatches.

---

## 2. Visual styling

Two moments to style the app:

**At build time** — `$setup-exam` Step 1e asks: default look or custom? Custom focuses on the essentials (base light/dark + accent color); roundness, font stack, and sidebar tint are optional. Anything you state up front (e.g. "dark + violet") is taken as-is, not re-asked. All of it maps to native Streamlit `[theme]` / `[theme.sidebar]` keys in `.streamlit/config.toml` — no CSS is ever used.

**After build** — invoke `$quiz/design` and describe the change. Modern theming covers much more than colors:

- **Badge palette** - `:green-badge[]` / `:red-badge[]` / `:orange-badge[]` colors are themable (`greenColor`, `redColor`, ... + `*BackgroundColor`/`*TextColor`).
- **Chart colours** - `chartCategoricalColors` in the theme, aligned with the `_config.py` constants (`#29b5e8` score line, `#F1914C` error bars by default).
- **Roundness / borders / fonts** - `baseRadius`, `buttonRadius`, `borderColor`, `showWidgetBorder`, built-in font stacks.
- **Sidebar look** - any of the above under `[theme.sidebar]`.

Example prompt:

```
run $quiz/design. I want a dark base, violet accent (#7C5CFC), round corners, and the score line chart in green (#36B37E).
```

The agent updates `.streamlit/config.toml` (+ matching `_config.py` chart constants), runs `$sis`, and asks you to re-deploy.

### Branding (title, exam name, logo)

- `EXAM_NAME` constant in `_config.py`: affects the in-app title and the Exam Simulation results screen.
- Note: `st.set_page_config` `page_title` / `page_icon` / `menu_items` are **not supported in Streamlit-in-Snowflake** — the browser tab is controlled by Snowsight. Don't fight it.
- For a custom logo on the Home screen: `st.image(url_or_stage_path)` at the top of the home screen in `pages/quiz.py`.

---

## 3. Functional features

`$quiz/features` ships **four** opt-in features — **Exam Simulation, Flashcards, AI Study Recommendation, Remedial Round** — reference implementations that have been tested and hold up across different conditions (various exams, grounded and ungrounded modes, with or without a question bank). They are **not** enabled by default; add any combination to your setup prompt, or ask the agent to bolt them onto an already-deployed app.

**These are examples, not a ceiling — you're encouraged to build your own.** Describe the feature you want (in the setup prompt or against a deployed app) and the agent designs it to the same `$quiz/screens` state + write-back contracts and `$quiz/design` visual conventions, so it fits the app cleanly.

| Feature | What it adds | New session-state / data |
|---|---|---|
| **Exam Simulation Mode** | Timed mock exam on its own page (`pages/exam_simulation.py`): question count + time limit read from `_config.py` (`EXAM_QUESTION_COUNT` / `EXAM_TIME_LIMIT_MIN`, captured from the study guide at setup — never hardcoded), domain-weighted (largest-remainder), bank-first sourcing, `st.fragment`-enforced countdown, pass/fail vs `PASS_THRESHOLD`, per-domain breakdown | `_quiz_mode`, `_sim_start_time`, `_sim_end_time`, `_sim_time_limit`, `_sim_questions`, `_sim_screen`; `ALTER QUIZ_SESSION_LOG ADD session_type` |
| **Flashcards** | A **FLASHCARDS tab on the Review page** (not a separate page). Atomic, recall-forcing cards (qa / cloze / compare) AI-decomposed from your wrong answers — never the verbatim MCQ — reviewed with Leitner spaced repetition (boxes 1–5). On-demand "Build cards from my wrong answers"; deck = cards due today | `_flashcard_cards`, `_flashcard_index`, `_flashcard_revealed`; new table `FLASHCARD_PROGRESS` |
| **AI Study Recommendation** | Own page (`pages/recommendations.py`): readiness math computed in Python from your error history (not the LLM), plus an AI-authored qualitative study plan / weak-domain + weak-topic recommendations grounded on the docs CKE | `_ai_recommendations`, `_rec_cache_key`; reads `QUIZ_REVIEW_LOG` + `EXAM_DOMAINS`, writes nothing |
| **Remedial Round** | A "Remedial Round" button on the summary of a *failed* practice round: re-tests just your wrong answers, reshuffled (no Admin toggle — present = enabled). The pass writes nothing — no stats, no review log, no debrief | `_round_type`, `_remedial_queue` |

### How to request features

Include them in your setup prompt:

```
i am setting up: SnowProAdvanced: Architect (ARA-C01). add exam simulation mode and AI study recommendation.
```

Or bolt them on later:

```
the quiz app is deployed. add flashcards and the remedial round feature. treat the existing schema and tables as fixed except where the skill says to add FLASHCARD_PROGRESS.
```

The agent reads `$quiz/features`, implements only the ones you name, re-runs `$sis`, and redeploys.

---

## 4. Target exam

### 4a - Swap to another SnowPro certification (easy)

Every SnowPro cert (Core COF-C03, Advanced: Architect / Administrator / Data Engineer / Data Analyst / Data Scientist / Security Engineer, Specialty: Gen AI / Snowpark) is handled by the same pipeline. Just:

1. Get the study guide PDF.
2. Re-run the setup prompt in the same workspace (or a new one) with the new exam name.
3. The agent creates a new `QUIZ_<CODE>` schema and a second Streamlit app.

What adjusts automatically:
- Domain list and weights (re-extracted from the new PDF).
- Key facts (per-domain, re-extracted).
- Question bank (AI-generated from new key facts).
- `AGENTS.md` environment table entries.

What stays the same:
- All 5 table schemas (`EXAM_DOMAINS`, `QUIZ_QUESTIONS`, `QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`, `QUIZ_CONFIG`).
- App UI and flow.
- The `app/` structure (only the `EXAM_NAME` and `EXAM_CODE` constants change, plus the generated question bank).

### 4b - Swap to a non-Snowflake cert (AWS, GCP, Azure, ...)

This scaffolding works for any cert with a published study guide PDF — AWS Certified AI Practitioner, Google Cloud Professional Data Engineer, Microsoft Certified: DevOps Engineer Expert, and others. The app itself still runs on Streamlit-in-Snowflake (because that is the runtime platform), but the content is fully exam-agnostic.

#### What works unchanged

- `AI_PARSE_DOCUMENT` handles any PDF, not just Snowflake study guides.
- `AI_COMPLETE` can extract domains, weights, topics, key facts from any structured cert blueprint.
- The app UI (home, quiz, summary, review, dashboard) is exam-neutral.
- The 5-table schema is exam-neutral.

#### What you should review

- **Domain extraction prompt** (`$setup-exam` Step 5): the prompt says "extract ALL exam domains from this certification study guide" which is generic, but AWS / Azure study guides sometimes mix "domains" with "subject areas" or "task statements". Spot-check the extracted `EXAM_DOMAINS` rows against the official blueprint.
- **Key facts grounding** (`$setup-exam` Step 5): for Snowflake, facts are SQL-heavy (DDL, function names, limits). For AWS, they are service-heavy (API names, quotas, pricing tiers). The extraction prompt is generic enough that both work, but the agent will extract whatever is in the PDF. Snowflake-specific hints in the current prompt (e.g. "feature names, SQL syntax") are suggestive examples - not exclusive filters. Edit the prompt in the skill if the output leans Snowflake-ish on a non-Snowflake PDF.
- **Question difficulty guide** (`$quiz/questions`): `DIFFICULTY_GUIDE` is Snowflake-flavoured ("easy = surface feature recognition; hard = cross-feature architecture trade-offs"). Tweak the wording for AWS / Azure but keep the 3-tier structure.
- **Explanation doc_url**: the explanation contract asks for a `doc_url`. For Snowflake it points at docs.snowflake.com. For AWS, the AI agent will happily produce `docs.aws.amazon.com/...` URLs - verify it is actually reaching the web-search tool (Snowsight > AI & ML > Agents > Settings > Tools and connectors > Web search).

#### What needs a touch-up

- `EXAM_NAME` and `EXAM_CODE` constants in `_config.py`: obvious.
- `AGENTS.md` title line (`> snowpro core certification quiz`): update.
- `AGENTS.md` section `## what this is`: the bullet "extract from SnowProCoreStudyGuide.pdf" becomes generic "extract from the study guide PDF". `$setup-exam` step 7 does this automatically if you change the filename in step 1b.
- Exam-simulation question count + time limit (`$quiz/features` Feature 1): extracted from your study guide at setup (`$setup-exam` Step 5) into `EXAM_QUESTION_COUNT` / `EXAM_TIME_LIMIT_MIN` in `_config.py` — never hardcoded. If the guide doesn't state them, the feature confirms the values with you before the first round.
- Any hard-coded "Snowflake" strings in helper text, placeholders, disclaimers. Search the generated `app/` files for "Snowflake" after generation, review each hit. Note: the **platform** runs on Snowflake so some references (e.g. `get_active_session`) are correct; only change exam-content references.

#### Minimum viable non-Snowflake run

For a first pass on, say, "AWS Solutions Architect Associate":

1. Upload `AWS-Certified-Solutions-Architect-Associate-Exam-Guide.pdf` to a new workspace.
2. Run the setup prompt with: `i am setting up: AWS Certified Solutions Architect Associate (SAA-C03). use the exam simulation feature with 65 questions, 130 minutes, pass threshold 72%.`
3. The agent extracts AWS domains (4 domains), generates ~30 questions per domain grounded on AWS key facts, deploys.
4. Verify: `EXAM_DOMAINS` has 4 rows, weights sum to 100, a couple of sample questions mention AWS services correctly.

Expect 1-2 rounds of refinement on the extraction prompt (`$cortex`) before the question quality is where you want it.

### 4c - Maintain multiple cert providers in one database

Every exam is isolated in its own schema. So a single `<your_database>` can happily hold:

- `QUIZ_COF_C03` (SnowPro Core)
- `QUIZ_ARA_C01` (SnowPro Advanced: Architect)
- `QUIZ_SAA_C03` (AWS Solutions Architect Associate)
- `QUIZ_AZ_900`  (Azure Fundamentals)

Each has its own `SNOWPRO_QUIZ` Streamlit app (rename to `AWS_SAA_QUIZ`, `AZURE_900_QUIZ`, etc. via `ALTER STREAMLIT ... RENAME TO ...` if you want them to look distinct in the Streamlit list).

`AGENTS.md` tracks whichever schema is "active" for the next chat session. To switch: edit the `snowflake environment` table (schema + exam_code lines), save, start a new chat. Everything the agent does afterwards targets the switched schema.

#### Optional: branch per exam (Git-backed workspace only)

If you loaded the asset via Git integration (recommended path in step 2 of [instructions.md](instructions.md)), you can isolate each exam on its own git branch in addition to the schema. This mirrors the CLI variant's setup and keeps each exam's `AGENTS.md` + generated `app/` project pinned to a separate commit history.

The agent does **not** run `git checkout` — you create the branch yourself:

- **Via the workspace Git panel**: click the branch name in the bottom bar » **Create new branch from `main`** > name it e.g. `exam/aws-saa-c03`. Workspace switches to the new branch.
- **Via GitHub**: create the branch on GitHub, then in Snowsight workspace Git panel click **Pull** / **Switch branch**.

Once on the new branch, run the setup prompt. The agent edits `AGENTS.md` and generates the `app/` project on that branch. Commit when you are happy. To switch back to a previous exam: change branch in the Git panel — the matching `AGENTS.md` snapshot comes with it.

Cost: one manual branch-switch step per exam. 
Benefit: clean history, easier diffs between exams, one fork can hold many exam configurations without any file churn on `main`.

---

## 5. Advanced mode (opt-in)

Three extras for power users. All OFF by default — enable by asking for them in the setup prompt (Step 1d) or in any later chat. Items marked **Preview** depend on account/region availability; verify before relying on them. Summary table: `AGENTS.md` > `Advanced options`.

### 5a - Quality model profile (opus-4-x)

The default `CORTEX_MODEL` is `claude-sonnet-4-6` — the best balance of quality, speed, and cost for question generation. The **quality profile** swaps it for `claude-opus-4-7`: noticeably stronger on hard questions (plausible distractors, multi-concept trade-offs), but slower and markedly more expensive per token.

`claude-opus-4-8` is **Public Preview** — use only on explicit request (preview models aren't production-ready). Check the [models & regional availability page](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-regional-availability) for current status.

Enable: say "use the quality model profile" in the setup prompt — the agent sets `CORTEX_MODEL = "claude-opus-4-7"` in `_config.py`. Cheaper hybrid worth considering: seed the question bank once on opus (Admin "Generate batch" or the worksheet recipe in §6), keep runtime AI questions and explanations on sonnet.

### 5b - Agent self-verify (Cloud Agents)

With self-verify enabled, `$setup-exam` adds Step 8.5: before handing you the app, the agent byte-compiles every generated module (`py_compile` over `main.py`, `_*.py`, `pages/*.py`) and fixes whatever fails. Catches syntax errors before you ever click **Run**.

Requires a CoCo session that can execute code (**Cloud Agents** — rolling out since Summit 26). If the session cannot execute code, the agent says so and skips the step — the Step 8 pre-deploy scan still runs either way.

### 5c - Automations (Preview)

CoCo **Automations** (Preview) run recurring, unattended jobs. A useful report-only recipe for this asset — weekly maintenance:

```
read AGENTS.md. then:
1. run the $sis scan over app/ and report any FAIL.
2. check QUIZ_QUESTIONS: if any domain has fewer than 20 questions, generate one
   batch (10) for that domain per the seeding recipe (docs/customization.md
   section 6) and report counts.
3. if a QUIZ_FLAGS table exists: regenerate flagged bank questions
   (status='OPEN' and question_id is not null), set status='REGENERATED', report.
4. summarize: scan verdict, rows added per domain, flags handled, anything
   needing my attention.
do not deploy anything; report only.
```

Schedule it weekly in CoCo's Automations UI once available in your account. Keep automations **report-only** — deploys stay a human decision (see the commit/deploy hygiene the whole asset follows).

---

## 6. Seeding the question bank

`$setup-exam` deliberately does **not** pre-generate questions (slow, burns your token budget during setup, and runtime AI questions work without it). A populated bank is still worth having:

- **Resilience** - if an AI call fails (service interruption, cross-region issue, exhausted token limits), the app falls back to bank questions instead of erroring;
- **Speed** - bank questions load instantly, no AI round-trip;
- **Consistency** - a curated, repeatable set (and the only thing the "From question bank" source mode uses).

Four ways to seed it, cheapest-effort first:

1. **CSV/JSON at (or after) build** - upload to `STAGE_QUIZ_DATA`, the agent runs `$adapt-questions`.
2. **Admin page → "Generate batch (AI)"** - pick a domain, get 10 grounded questions inserted; repeat as needed.
3. **Worksheet recipe** - run this in a Snowsight worksheet (per domain; adjust names):

   ```sql
   INSERT INTO <db>.QUIZ_<CODE>.QUIZ_QUESTIONS
     (domain_id, domain_name, difficulty, question_text, is_multi,
      option_a, option_b, option_c, option_d, option_e, correct_answer, source)
   SELECT d.domain_id, d.domain_name,
          q.value:difficulty::VARCHAR,
          LEFT(q.value:question_text::VARCHAR, 2000),
          q.value:is_multi::BOOLEAN,
          LEFT(q.value:option_a::VARCHAR, 500), LEFT(q.value:option_b::VARCHAR, 500),
          LEFT(q.value:option_c::VARCHAR, 500), LEFT(q.value:option_d::VARCHAR, 500),
          LEFT(q.value:option_e::VARCHAR, 500),
          q.value:correct_answer::VARCHAR, 'AI_GENERATED'
   FROM <db>.QUIZ_<CODE>.EXAM_DOMAINS d,
        LATERAL FLATTEN(PARSE_JSON(AI_COMPLETE(
          model => 'claude-sonnet-4-6',
          prompt => 'Generate 10 exam questions for the domain "' || d.domain_name ||
                    '" with a 3 easy / 4 medium / 3 hard difficulty mix, grounded ONLY on these facts: ' || d.key_facts,
          model_parameters => {},
          response_format => {'type':'json','schema':{'type':'object','properties':{
            'questions':{'type':'array','items':{'type':'object','properties':{
              'difficulty':{'type':'string'},'question_text':{'type':'string'},
              'is_multi':{'type':'boolean'},'option_a':{'type':'string'},'option_b':{'type':'string'},
              'option_c':{'type':'string'},'option_d':{'type':'string'},'option_e':{'type':'string'},
              'correct_answer':{'type':'string'}},
              'required':['difficulty','question_text','is_multi','option_a','option_b','correct_answer']}}},
            'required':['questions']}})):questions) q
   WHERE d.domain_id = '1';   -- run per domain (or remove to do all at once)
   ```

4. **Scheduled** - wrap recipe 3 in a Snowflake `TASK` (weekly cron, UTC) or let the CoCo **Automation** from section 5c top up thin domains.

Spot-check a few rows after seeding (`SELECT ... ORDER BY RANDOM() LIMIT 5`). For ongoing quality control, the Admin **Question manager** lets you edit or regenerate weak questions.

---

## 7. Doc-grounded mode (Snowflake Documentation CKE)

Install the free **Snowflake Documentation** listing from Marketplace (Snowsight » Data Products » Marketplace; needs `IMPORT SHARE`/ACCOUNTADMIN; creates `SNOWFLAKE_DOCUMENTATION`) and the app grounds its AI in real Snowflake docs — a Cortex Knowledge Extension (a shared Cortex Search service, ~56K chunks). The grounding source is governed by `grounding_mode` (`cke` | `custom` | `none`), **set once at setup** (`$setup-exam` Step 1g) and stored in `QUIZ_CONFIG` — a fixed value, **not a runtime toggle**. The Admin page shows it **read-only**. `cke` (default for Snowflake exams) uses this CKE; `custom` points at a private Cortex Search service over your own corpus; `none` is ungrounded (non-Snowflake exams only).

What changes when on:
- **Questions** are generated from retrieved doc chunks + `key_facts` (hybrid — prefers the docs, keeps the exam scope).
- **Explanations** cite the chunk's exact `SOURCE_URL` and show a "📚 From the docs" excerpt — instead of a guessed search link.
- All retrieval is isolated in `app/_search.py` (Python `snowflake.core` API). In `cke`/`custom` mode grounding is **mandatory — no silent fallback to built-in knowledge**: if retrieval is empty the query broadens once, then the generation **fails visibly** ("couldn't ground — retry"); if the service is unreachable (CKE uninstalled / no grant), the app shows an install/grant message and **disables Start Round** rather than guessing. Only `none` mode (non-Snowflake exams, explicit opt-in) is ungrounded — there `$setup-exam` sets `grounding_mode='none'`.

Cost: each grounded question/explanation adds one Cortex Search query (consumer-billed compute, small; cached per query/session).

**Doc-grounded seeding (worksheet):** the section-6 recipe can ground the bank too — `SEARCH_PREVIEW` is fine in a worksheet (ad-hoc):

```sql
WITH ctx AS (
  SELECT d.domain_name,
         SNOWFLAKE.CORTEX.SEARCH_PREVIEW(
           'SNOWFLAKE_DOCUMENTATION.SHARED.CKE_SNOWFLAKE_DOCS_SERVICE',
           OBJECT_CONSTRUCT('query', d.domain_name, 'columns',
             ARRAY_CONSTRUCT('CHUNK','SOURCE_URL'), 'limit', 5)::STRING) AS r
  FROM <db>.QUIZ_<CODE>.EXAM_DOMAINS d WHERE d.domain_id = '1'
)
-- feed r:results (the doc chunks) into the AI_COMPLETE generation prompt from section 6,
-- instructing the model to prefer the documentation context.
SELECT * FROM ctx;
```
(In the app, Admin "Generate batch" uses the Python `search_docs()` — never `SEARCH_PREVIEW`.)

---

## Anti-patterns to avoid

- **Don't hard-code domain names, weights, or topic lists anywhere in the app code.** They come from `EXAM_DOMAINS` at runtime. Hard-coding breaks the multi-exam design.
- **Don't bypass `$sis` even for "tiny" UI tweaks.** The scan catches regressions that only surface at runtime in SiS - a 5-minute scan is cheaper than a production redeploy loop.
- **Don't add features by editing the app code without consulting `$quiz/features`.** The skill documents session-state key conventions and write-back contracts - ad-hoc additions will collide with future features.
- **Don't change the 5-table schema casually.** Every screen and every skill assumes those exact columns. Add columns via `ALTER TABLE` if needed.
