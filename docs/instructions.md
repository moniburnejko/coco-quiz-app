# Instructions - concise

> First time? Use [instructions-detailed.md](instructions-detailed.md) instead.

---

## Step 0 - one-time account prerequisites

As `ACCOUNTADMIN`, once per account - only if your account cannot reach the model in-region (accounts created after 2026-03-09 already default to `ANY_REGION`):

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

`pandas`/`altair` come from the Snowflake Anaconda channel, so the warehouse runtime works on **trial accounts**.

In Snowsight: **AI & ML > Agents > Settings > Tools and connectors > Web search → enable**.

**Required for Snowflake exams** (recommended for all): install the free **Snowflake Documentation** listing in Snowsight » Data Products » Marketplace. Snowflake-exam apps default to CKE doc-grounding (`grounding_mode = cke`) - questions and explanations are generated ONLY from real docs, never built-in knowledge - so `$setup-exam` treats the CKE as a hard gate and stops until it's reachable (free, works on trial). Grounding mode is fixed once at setup; only a non-Snowflake exam can opt into the ungrounded `none` mode.

---

## Step 1 - load context into the workspace

Two paths. Both end with `.snowflake/cortex/skills/` + `AGENTS.md` visible in the workspace file tree.

### Via Git integration (recommended)

Fork this repo on GitHub. Then, as a role with `CREATE INTEGRATION`:

```sql
-- 1. API INTEGRATION: authorises Snowflake to reach GitHub's API.
CREATE OR REPLACE API INTEGRATION gh_integration
  API_PROVIDER = GIT_HTTPS_API
  API_ALLOWED_PREFIXES = ('https://github.com/<your_github_user>')
  ENABLED = TRUE;

GRANT USAGE ON INTEGRATION gh_integration TO ROLE <your_role>;

-- 2. SECRET: only needed for private forks (OAuth2 and public repos skip this).
--    Generate a GitHub PAT at github.com/settings/tokens with 'repo' scope.
CREATE OR REPLACE SECRET <your_database>.<schema>.git_pat
  TYPE = PASSWORD
  USERNAME = '<your_github_user>'
  PASSWORD = '<personal_access_token>';

GRANT USAGE ON SECRET <your_database>.<schema>.git_pat TO ROLE <your_role>;

-- 3. GIT REPOSITORY: registers your fork as a first-class Snowflake object.
CREATE OR REPLACE GIT REPOSITORY <your_database>.<schema>.coco_quiz_fork
  API_INTEGRATION = gh_integration
  GIT_CREDENTIALS = <your_database>.<schema>.git_pat   -- omit for public fork
  ORIGIN = 'https://github.com/<your_github_user>/coco-quiz-app.git';
```

See [Integrate workspaces with a Git repository](https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-git).

Then in Snowsight: **Projects > Workspaces > + Workspace > From Git repository** > select `coco_quiz_fork`. Workspace opens with skills, `AGENTS.md`, and `docs/` ready to use. 

Benefits: `docs/` accessible inside Snowsight, commit and branch from the workspace, skill updates come via `git pull`, the generated `app/` project can be committed back.

### Via manual upload (fallback)

Clone the repo locally. In Snowsight: **Projects > Workspaces > + Workspace** (empty). Then:

1. **Custom skills**: in Snowflake CoCo chat, click **+** > **Upload Folder(s)** > select `.snowflake/cortex/skills/` from the clone. All skills (5 invocable - `$setup-exam`, `$adapt-questions`, `$cortex`, `$sis`, `$quiz` - across 9 `SKILL.md` files incl. the 4 `$quiz` sub-skills) become available as slash commands.
2. **AGENTS.md**: drag-and-drop to the workspace root (or use **+** > **Upload File(s)**).

`docs/` is not uploaded - reference it from your local clone or from GitHub.

### What you do NOT load either way

- The `app/` project (`main.py`, `_*.py` modules, `pages/`, configs) - the agent generates it into the workspace on each `$setup-exam` run.
- The PDF study guide and optional CSV - you add these to the workspace file tree later, when the agent asks (step 4) - not now.

---

## Step 2 - edit the environment table in AGENTS.md

Open `AGENTS.md` in the workspace, find the `snowflake environment` table. Replace the `<your_...>` placeholders (`<your_database>`, `<your_warehouse>`, `<your_role>`) with the actual object names. Leave `schema` and `exam_code` as is - `$setup-exam` fills `schema`/`exam_code` once the exam code is known. `$setup-exam` halts if it finds unfilled required placeholders, so replace them before running the setup prompt.

---

## Step 3 - run the setup prompt

Paste the **setup prompt** from [prompts.md](prompts.md) into the CoCo chat - it attaches `@AGENTS.md` and invokes `/setup-exam`. The agent runs the skill end-to-end and stops at three checkpoints:

- After asking you to add the study guide PDF (and optionally a CSV) → you drop it into the workspace file tree and the agent stages it via `COPY FILES` (step 4).
- After extracting domains → approve / re-extract / abort.
- Before deploy → the agent copies the generated `app/` onto `STAGE_SIS_APP` and deploys it on the warehouse runtime - nothing for you to upload (step 5).

---

## Step 4 - add the study guide PDF (and optional CSV) to the workspace

When the agent stops and asks for the PDF:

1. Drop the study-guide PDF into the **workspace file tree** (same place as the `app/` project) - not a stage.
2. (Optional) drop a CSV/JSON question bank in alongside it.
3. Reply to the agent: "added".

The agent then copies the file onto `STAGE_QUIZ_DATA` with `COPY FILES` and verifies with `LIST @STAGE_QUIZ_DATA` before continuing - you never upload to a stage by hand.

---

## Step 5 - preview and deploy the generated app

After the agent finishes generation + passes the pre-deploy scan, the `app/` project sits in your workspace file tree.

**The agent deploys it for you:** your workspace files already live on an internal stage, so the agent copies them onto `STAGE_SIS_APP` with `COPY FILES` (preserving `pages/` and `.streamlit/`), then runs `CREATE OR REPLACE STREAMLIT … MAIN_FILE = 'main.py' QUERY_WAREHOUSE = …` and verifies with `SHOW STREAMLITS`. After later edits it re-copies the changed files and re-creates the app.

---

## Step 6 - open the app and verify

Snowsight > **Projects > Streamlit > SNOWPRO_QUIZ** (or whatever `app_name` is set in `AGENTS.md`).

Walk through the pages (navigation is native multipage):

- **Quiz page** - home (round size, difficulty, domain, source - no explanations toggle; the explanation is on-demand) → quiz (try the 💡 Hint *before* answering; after submitting: instant feedback, then an on-demand **💡 AI explanation** button - for correct answers too - opening an expander with the explanation and a 🔬 Deep dive into the question's topic, with **Next** pinned at the very bottom under the expander) → summary (score, pass/fail vs threshold, a TO REMEMBER expander with the correct answers in full, an on-demand Round Summary).
- **Review page** - sub-tabs (`st.pills`): **Wrong answers** (history with domain + date-range filters) and **Learning Dashboard** (charts).
- **Admin page** (4 tabs: App config, Questions manager, Cortex spend, Logs) - flip a feature toggle (hints / Round Summary) or pick a model per call-group in App config, check the bank KPIs, and in Questions manager use the filter/edit/delete table or **Generate batch (AI)** to seed the question bank; Logs shows the review + session tables with a guarded reset.

Complete at least one round so `QUIZ_SESSION_LOG` and `QUIZ_REVIEW_LOG` get data for dashboard charts.

---

## Adding another exam

Keep the same workspace, run the **setup prompt** again with a different PDF. The agent will ask for the exam code, create a new `QUIZ_<NEW_CODE>` schema, and deploy a second Streamlit app. The previous exam is untouched.

---

## Something broke?

Paste the **fix prompt** from [prompts.md](prompts.md) with a description of what happened. The agent will run the relevant diagnostic skill, fix the issue, and redeploy.

**Seeing errors while you test:** the app ships with `[client] showErrorDetails = "none"` in `app/.streamlit/config.toml` - viewers get a generic message, never a traceback. While debugging your own setup, set it to `"full"` and redeploy to see the real traceback; set it back to `"none"` before sharing the app.

See also [troubleshooting.md](troubleshooting.md) for common Snowsight-specific pitfalls.
