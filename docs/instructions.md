# Instructions - concise

> first time? use [instructions-detailed.md](instructions-detailed.md) instead.

---

## step 0 - one-time account prerequisites

As `ACCOUNTADMIN` (EU accounts only, once per account):

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'AWS_US';
```

In Snowsight: **AI & ML > Agents > Settings > Tools and connectors > Web search → enable**.

---

## step 1 - load context into the workspace

Two paths. Both end with `.snowflake/cortex/skills/` + `AGENTS.md` visible in the workspace file tree.

### via Git integration (recommended)

Fork this repo on GitHub. Then, as a role with `CREATE INTEGRATION`:

```sql
-- 1. API INTEGRATION: authorises Snowflake to reach GitHub's API.
CREATE OR REPLACE API INTEGRATION gih_integration
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

Benefits: `docs/` accessible inside Snowsight, commit and branch from the workspace, skill updates come via `git pull`, generated `quiz.py` can be committed back.

### via manual upload (fallback)

Clone the repo locally. In Snowsight: **Projects > Workspaces > + Workspace** (empty). Then:

1. **custom skills**: in Cortex Code chat, click **+** > **Upload Folder(s)** > select `.snowflake/cortex/skills/` from the clone. All 7 skills (`$setup-exam`, `$adapt-questions`, `$cortex`, `$sis`, `$quiz`, + sub-skills) become available as slash commands.
2. **AGENTS.md**: drag-and-drop to the workspace root (or use **+** > **Upload File(s)**).

`docs/` is not uploaded - reference it from your local clone or from GitHub.

### what you do NOT load either way

- `quiz.py` and `environment.yml` - the agent generates these into the workspace on each `$setup-exam` run.
- the PDF study guide and optional CSV - these go straight to a Snowflake stage in step 5, not the workspace.

---

## step 2 - edit the environment table in AGENTS.md

Open `AGENTS.md` in the workspace, find the `snowflake environment` table. Replace the three `<your_...>` placeholders (`<your_database>`, `<your_warehouse>`, `<your_role>`) with the actual object names. Leave `schema` and `exam_code` as is — `$setup-exam` fills those once the exam code is known. `$setup-exam` halts if it finds unfilled `<...>` placeholders, so make sure all three are replaced before running the setup prompt.

---

## step 3 - run the setup prompt

Paste the **setup prompt** from [prompts.md](prompts.md) into the Cortex Code chat. The agent will run `$setup-exam` end-to-end and stop at three checkpoints:

- after asking you to upload the study guide PDF (and optionally a CSV) → you upload via Snowsight UI (step 5).
- after extracting domains → approves / re-extracts / aborts.
- before deploy → asks you to upload `quiz.py` + `environment.yml` to `STAGE_SIS_APP` (step 6).

---

## step 4 - upload the study guide PDF (and optional CSV)

When the agent stops and asks for the PDF:

1. Snowsight > **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_QUIZ_DATA**.
2. **+ Files** (top-right) > drag-drop the PDF > **Upload**.
3. (optional) repeat for a CSV question bank if you have one.
4. Reply to the agent: "uploaded".

The agent verifies with `LIST @STAGE_QUIZ_DATA` and continues.

---

## step 5 - upload generated `quiz.py` and `environment.yml` to the app stage

After the agent finishes generation + passes its 22-item pre-deploy scan:

1. In your workspace, confirm both files exist (left-hand file tree).
2. Snowsight > **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_SIS_APP**.
3. **+ Files** > drag-drop both files > **Upload**.
4. Reply "uploaded" - agent runs `CREATE OR REPLACE STREAMLIT ... FROM @STAGE_SIS_APP`.

Alternative (unverified, faster if it works): right-click `quiz.py` in the workspace file tree → **Deploy as Streamlit App**.

---

## step 6 - open the app and verify

Snowsight > **Projects > Streamlit > SNOWPRO_QUIZ** (or whatever `app_name` is set in `AGENTS.md`).

Walk through the 4 screens:

- **Home** - round size, difficulty, domain, question source, explanations toggle.
- **Quiz** - answer questions, submit, check feedback + optional AI explanation.
- **Summary** - score, pass/fail vs 75% threshold, wrong-answer cards.
- **Review** - wrong-answer history with filters + Learning Dashboard tab.

Complete at least one round so `QUIZ_SESSION_LOG` and `QUIZ_REVIEW_LOG` get data for dashboard charts.

---

## adding another exam

Keep the same workspace, run the **setup prompt** again with a different PDF. The agent will ask for the exam code, create a new `QUIZ_<NEW_CODE>` schema, and deploy a second Streamlit app. The previous exam is untouched.

---

## something broke?

Paste the **fix prompt** from [prompts.md](prompts.md) with a description of what happened. The agent will run the relevant diagnostic skill, fix the issue, and ask you to re-upload.

See also [troubleshooting.md](troubleshooting.md) for common Snowsight-specific pitfalls.
