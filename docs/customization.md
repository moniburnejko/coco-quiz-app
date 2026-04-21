# Customization

The quiz app has four layers you can tweak, from "one-line change" to "fork the pipeline":

1. **Model and runtime defaults** (edit `AGENTS.md`).
2. **Visual styling** (edit `quiz.py` or tell the agent what to change via `$quiz/style`).
3. **Functional features** (opt-in via `$quiz/features`: exam simulation, flashcards, AI study recommendations, ...).
4. **Target exam** (swap the PDF for a different SnowPro cert, or with a little bit more work, any non-Snowflake exam like AWS / GCP / Azure).

Everything below is safe to iterate on: change, redeploy, keep going. Nothing is a one-way door.

---

## 1. Model and runtime defaults

### 1a - switch the Cortex LLM

`AGENTS.md` > `cortex llm` section pins `claude-sonnet-4-6`. To change:

```markdown
preferred model: <llm_model_name>. store as constant `CORTEX_MODEL`.
```

Trade-off: `claude-sonnet-4-6` is the best quality for question generation and explanations but is also the slowest. `mistral-large2` or `llama3.1-405b` are noticeably faster but hallucinate more on Snowflake-specific minutiae. After changing, regenerate `quiz.py` so the constant value propagates, or just find-replace `CORTEX_MODEL = "..."` at the top of the file and redeploy.

### 1b - round size defaults / default question source / any other default

Ask Cortex Code to change it. :) 

Or:
- For round size: Open `quiz.py`, find the `render_home` function, look for the `st.selectbox("Questions", options=[...])` block. Default options are typically `[5, 10, 20, 50]`. Change to taste and redeploy (+ re-run `$sis/pre-deploy`).
- For question source: In `render_home` find the `question_source` selectbox. Options are `mix`, `db`, `ai`. If the user chose "No CSV/JSON" during `$setup-exam`, the default is already `ai`; otherwise `mix`. Change `index=` to change the default.
- and so on.

### 1c - AI explanation format

Explanations render `why_correct`, `why_wrong` (per option), `mnemonic`, `doc_url`. To change the shape (e.g. add a `related_docs` list), edit two places:
- the AI_COMPLETE prompt template in `generate_explanation` (ask for the new field);
- the `render_quiz` explanation block (display the new field).

Run `$cortex/prompt-audit` after the prompt change to catch JSON key mismatches.

---

## 2. Visual styling

Invoke `$quiz/style` and describe what you want. The skill documents:

- **Badge colours** - pass / fail / warning / info palette.
- **Section labels** - uppercase mini-headers above card groups.
- **Chart colours** - currently `#29b5e8` (Snowflake blue) for the score line and `#F1914C` (orange) for domain errors. Swap to any hex.
- **Card layout** - wrong-answer cards on Summary and Review tabs.
- **Button style** - primary vs secondary vs ghost.

Example prompt:

```
run $quiz/style. I want the score line chart in green (#36B37E) and the readiness score metric in the same green. Also change the review-tab "section labels" to be teal, not blue.
```

The agent updates the relevant constants in `quiz.py`, runs `$sis/pre-deploy`, and asks you to re-upload.

### Branding (title, exam name, logo)

- `EXAM_NAME` constant at the top of `quiz.py`: affects the page title and the Exam Simulation results screen.
- `st.set_page_config(page_title=..., page_icon=...)` in the header: change the browser-tab title and favicon. `page_icon` accepts an emoji, a URL to a hosted image, or (for SiS) a path in the app stage.
- For a custom logo on the Home screen: `st.image(url_or_stage_path)` at the top of `render_home`.

---

## 3. Functional features

`$quiz/features` lists 6 opt-in features. They are **not** enabled by default. Add any combination to your setup prompt, or ask the agent to bolt them onto an already-deployed app.

| Feature | What it adds | New session-state / data |
|---|---|---|
| **Exam Simulation Mode** | Timed mock exam: fixed question count per the real exam blueprint, countdown timer, weighted by domain, pass/fail vs real threshold, per-domain breakdown | `_quiz_mode`, `_sim_start_time`, `_sim_time_limit`, `_sim_questions`, `_sim_screen` |
| **Flashcard Review** | Flip-card view of the Review tab wrong-answer history: front = question, back = correct answer + explanation. Swipe / "Got it" / "Still wrong" controls | `_flashcard_idx`, `_flashcard_side` |
| **Quick Stats Sidebar** | Persistent sidebar widget showing today's streak, questions answered this week, avg score last 5 rounds | reads from `QUIZ_SESSION_LOG` |
| **Spaced Repetition (Smart Review)** | "Next-due" queue picks wrong answers from the Review log based on time since last seen (Leitner-style intervals). Integrates with the Review tab | new table: `QUIZ_REVIEW_SCHEDULE` (question_id, next_due_at, interval_days) |
| **Achievement Badges** | Cumulative badges (5 sessions, 100 questions, 7-day streak, first perfect round, ...). Shown on Dashboard | `compute_badges()` reads from `QUIZ_SESSION_LOG` and `QUIZ_REVIEW_LOG` |
| **AI Study Recommendation** | New Review sub-tab: AI reads your error history + domain weights and produces a study plan: "You are weak on Performance Tuning (2/8 correct). Focus on: ..." grounded on `key_facts` | reads from `QUIZ_REVIEW_LOG` + `EXAM_DOMAINS.key_facts`, writes nothing |

### How to request features

Include them in your setup prompt:

```
i am setting up: SnowProAdvanced: Architect (ARA-C01). add exam simulation mode and AI study recommendation.
```

Or bolt them on later:

```
the quiz app is deployed. add flashcards and spaced repetition. treat the existing schema and tables as fixed except where the skill says to add QUIZ_REVIEW_SCHEDULE.
```

The agent reads `$quiz/features`, implements only the ones you name, re-runs `$sis/pre-deploy`, and redeploys.

---

## 4. Target exam

### 4a - swap to another SnowPro certification (easy)

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
- All 4 table schemas.
- App UI and flow.
- `quiz.py` structure (only the `EXAM_NAME` and `EXAM_CODE` constants change, plus the generated question bank).

### 4b - swap to a non-Snowflake cert (AWS, GCP, Azure, ...)

You can absolutely use this scaffolding for AWS Certified AI Practitioner, Google Cloud Professional Data Engineer, Microsoft Certified: DevOps Engineer Expert, or anything else that has a published study guide PDF. The app itself still runs on Streamlit-in-Snowflake (because that is the runtime platform), but the content is fully exam-agnostic.

#### what works out of the box

- `AI_PARSE_DOCUMENT` handles any PDF, not just Snowflake study guides.
- `AI_COMPLETE` can extract domains, weights, topics, key facts from any structured cert blueprint.
- The app UI (home, quiz, summary, review, dashboard) is exam-neutral.
- The 4-table schema is exam-neutral.

#### what you should review

- **Domain extraction prompt** (`$setup-exam` step 5b): the prompt says "extract ALL exam domains from this certification study guide" which is generic, but AWS / Azure study guides sometimes mix "domains" with "subject areas" or "task statements". Spot-check the extracted `EXAM_DOMAINS` rows against the official blueprint.
- **Key facts grounding** (`$setup-exam` step 5c): for Snowflake, facts are SQL-heavy (DDL, function names, limits). For AWS, they are service-heavy (API names, quotas, pricing tiers). The extraction prompt is generic enough that both work, but the agent will extract whatever is in the PDF. Snowflake-specific hints in the current prompt (e.g. "feature names, SQL syntax") are suggestive examples - not exclusive filters. Edit the prompt in the skill if the output leans Snowflake-ish on a non-Snowflake PDF.
- **Question difficulty guide** (`$quiz/questions`): `DIFFICULTY_GUIDE` is Snowflake-flavoured ("easy = surface feature recognition; hard = cross-feature architecture trade-offs"). Tweak the wording for AWS / Azure but keep the 3-tier structure.
- **Explanation doc_url**: the explanation contract asks for a `doc_url`. For Snowflake it points at docs.snowflake.com. For AWS, the AI agent will happily produce `docs.aws.amazon.com/...` URLs - verify it is actually reaching the web-search tool (Snowsight > AI & ML > Agents > Settings > Tools and connectors > Web search).

#### what needs a touch-up

- `EXAM_NAME` and `EXAM_CODE` constants in `quiz.py`: obvious.
- `AGENTS.md` title line (`> snowpro core certification quiz`): update.
- `AGENTS.md` section `## what this is`: the bullet "extract from SnowProCoreStudyGuide.pdf" becomes generic "extract from the study guide PDF". `$setup-exam` step 7 does this automatically if you change the filename in step 1b.
- Exam-simulation time/question defaults (`$quiz/features` Feature 1): Snowflake-specific numbers (COF-C03 = 100q/115min). Ask the agent to look them up for your target cert during setup.
- Any hard-coded "Snowflake" strings in helper text, placeholders, disclaimers. `grep -n "Snowflake" quiz.py` after generation, review each hit. Note: the **platform** runs on Snowflake so some references (e.g. `get_active_session`) are correct; only change exam-content references.

#### minimum viable non-Snowflake run

For a first pass on, say, "AWS Solutions Architect Associate":

1. Upload `AWS-Certified-Solutions-Architect-Associate-Exam-Guide.pdf` to a new workspace.
2. Run the setup prompt with: `i am setting up: AWS Certified Solutions Architect Associate (SAA-C03). use the exam simulation feature with 65 questions, 130 minutes, pass threshold 72%.`
3. The agent extracts AWS domains (4 domains), generates ~30 questions per domain grounded on AWS key facts, deploys.
4. Verify: `EXAM_DOMAINS` has 4 rows, weights sum to 100, a couple of sample questions mention AWS services correctly.

Expect 1-2 rounds of refinement on the extraction prompt (`$cortex/prompt-audit`) before the question quality is where you want it. The Snowflake pipeline had this too - the difference is that for Snowflake it was pre-tuned over several iterations.

### 4c - maintain multiple cert providers in one database

Every exam is isolated in its own schema. So a single `<your_database>` can happily hold:

- `QUIZ_COF_C03` (SnowPro Core)
- `QUIZ_ARA_C01` (SnowPro Advanced: Architect)
- `QUIZ_SAA_C03` (AWS Solutions Architect Associate)
- `QUIZ_AZ_900`  (Azure Fundamentals)

Each has its own `SNOWPRO_QUIZ` Streamlit app (rename to `AWS_SAA_QUIZ`, `AZURE_900_QUIZ`, etc. via `ALTER STREAMLIT ... RENAME TO ...` if you want them to look distinct in the Streamlit list).

`AGENTS.md` tracks whichever schema is "active" for the next chat session. To switch: edit the `snowflake environment` table (schema + exam_code lines), save, start a new chat. Everything the agent does afterwards targets the switched schema.

#### optional: branch per exam (Git-backed workspace only)

If you loaded the asset via Git integration (recommended path in step 2 of [instructions.md](instructions.md)), you can isolate each exam on its own git branch in addition to the schema. This mirrors the CLI variant's setup and keeps each exam's `AGENTS.md` + generated `quiz.py` pinned to a separate commit history.

The agent does **not** run `git checkout` — you create the branch yourself:

- **via the workspace Git panel**: click the branch name in the bottom bar » **Create new branch from `main`** > name it e.g. `exam/aws-saa-c03`. Workspace switches to the new branch.
- **via GitHub**: create the branch on GitHub, then in Snowsight workspace Git panel click **Pull** / **Switch branch**.

Once on the new branch, run the setup prompt. The agent edits `AGENTS.md` and generates `quiz.py` on that branch. Commit when you are happy. To switch back to a previous exam: change branch in the Git panel — the matching `AGENTS.md` snapshot comes with it.

Cost: one manual branch-switch step per exam. 
Benefit: clean history, easier diffs between exams, one fork can hold many exam configurations without any file churn on `main`.

---

## Anti-patterns to avoid

- **Don't hard-code domain names, weights, or topic lists anywhere in `quiz.py`.** They come from `EXAM_DOMAINS` at runtime. Hard-coding breaks the multi-exam design.
- **Don't bypass `$sis/pre-deploy` even for "tiny" UI tweaks.** The scan catches regressions that only surface at runtime in SiS - a 5-minute scan is cheaper than a production redeploy loop.
- **Don't add features by editing `quiz.py` without consulting `$quiz/features`.** The skill documents session-state key conventions and write-back contracts - ad-hoc additions will collide with future features.
- **Don't change the 4-table schema casually.** Every screen and every skill assumes those exact columns. Add columns via `ALTER TABLE` if needed.
