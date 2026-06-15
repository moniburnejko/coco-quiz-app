# Instructions - concise

> First time? Use [instructions-detailed.md](instructions-detailed.md) instead.

---

## Step 0 - one-time account prerequisites

As `ACCOUNTADMIN`, once per account — only if your account cannot reach the model in-region (accounts created after 2026-03-09 already default to `ANY_REGION`):

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

The default container runtime needs TWO things (the warehouse fallback needs neither):

1. A **compute pool** (your role needs `USAGE`):
   ```sql
   SHOW COMPUTE POOLS;
   ```
2. A **PyPI external access integration** — the container installs pandas/altair from PyPI (they aren't in the base image), so as `ACCOUNTADMIN`:
   ```sql
   CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION pypi_access_integration
     ALLOWED_NETWORK_RULES = (snowflake.external_access.pypi_rule)   -- Snowflake's managed rule; no custom rule needed
     ENABLED = TRUE;
   GRANT USAGE ON INTEGRATION pypi_access_integration TO ROLE <your_role>;
   ```
   Attach it at deploy (Deploy dialog → Network, or `EXTERNAL_ACCESS_INTEGRATIONS = (pypi_access_integration)`). **Without it, the deploy fails with "Failed to retrieve package… Have you enabled External Access Integration?"**

No compute pool / no admin for the EAI? The warehouse fallback (deploy Path C) works without either.

In Snowsight: **AI & ML > Agents > Settings > Tools and connectors > Web search → enable**.

---

## Step 1 - load context into the workspace

Two paths. Both end with `.snowflake/cortex/skills/` + `AGENTS.md` visible in the workspace file tree.

### Via Git integration (recommended)

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

Benefits: `docs/` accessible inside Snowsight, commit and branch from the workspace, skill updates come via `git pull`, the generated `app/` project can be committed back.

### Via manual upload (fallback)

Clone the repo locally. In Snowsight: **Projects > Workspaces > + Workspace** (empty). Then:

1. **Custom skills**: in Snowflake CoCo chat, click **+** > **Upload Folder(s)** > select `.snowflake/cortex/skills/` from the clone. All skills (5 invocable — `$setup-exam`, `$adapt-questions`, `$cortex`, `$sis`, `$quiz` — across 13 files incl. sub-skills) become available as slash commands.
2. **AGENTS.md**: drag-and-drop to the workspace root (or use **+** > **Upload File(s)**).

`docs/` is not uploaded - reference it from your local clone or from GitHub.

### What you do NOT load either way

- The `app/` project (`main.py`, `_*.py` modules, `pages/`, configs) - the agent generates it into the workspace on each `$setup-exam` run.
- The PDF study guide and optional CSV - these go straight to a Snowflake stage in step 5, not the workspace.

---

## Step 2 - edit the environment table in AGENTS.md

Open `AGENTS.md` in the workspace, find the `snowflake environment` table. Replace the `<your_...>` placeholders (`<your_database>`, `<your_warehouse>`, `<your_role>`, and `<your_compute_pool>` — the last one is skippable only if you plan the warehouse fallback) with the actual object names. Leave `schema` and `exam_code` as is — `$setup-exam` fills those once the exam code is known. `$setup-exam` halts if it finds unfilled required placeholders, so replace them before running the setup prompt.

---

## Step 3 - run the setup prompt

Paste the **setup prompt** from [prompts.md](prompts.md) into the CoCo chat. The agent will run `$setup-exam` end-to-end and stop at three checkpoints:

- After asking you to upload the study guide PDF (and optionally a CSV) → you upload via Snowsight UI (step 4).
- After extracting domains → approve / re-extract / abort.
- Before deploy → asks you to **Run + Deploy** the generated `app/` from the workspace (or, on the fallback path, to upload `app/` to `STAGE_SIS_APP`) (step 5).

---

## Step 4 - upload the study guide PDF (and optional CSV)

When the agent stops and asks for the PDF:

1. Snowsight > **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_QUIZ_DATA**.
2. **+ Files** (top-right) > drag-drop the PDF > **Upload**.
3. (Optional) repeat for a CSV question bank if you have one.
4. Reply to the agent: "uploaded".

The agent verifies with `LIST @STAGE_QUIZ_DATA` and continues.

---

## Step 5 - preview and deploy the generated app

After the agent finishes generation + passes the pre-deploy scan, the `app/` project sits in your workspace file tree.

**Path A — Workspaces (default):**

1. Open `app/main.py`, click **Run** (or Cmd/Ctrl+Enter). A private **dev app** preview opens in the browser — only you see it. Iterate with the agent until it looks right.
2. Click **Deploy** (project toolbar). Set: app title `SNOWPRO_QUIZ`, database + schema `QUIZ_<CODE>`, **compute pool**, query warehouse.
3. Reply "deployed" — the agent verifies with `SHOW STREAMLITS`.

**Path B — scripted via stage (reproducible fallback):** Snowsight > **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_SIS_APP** > **+ Files** > upload the `app/` files (keep the `pages/` and `.streamlit/` folder layout) > reply "uploaded" — the agent runs `CREATE OR REPLACE STREAMLIT ... RUNTIME_NAME = 'SYSTEM$ST_CONTAINER_RUNTIME_PY3_11' COMPUTE_POOL = ...`.

**Path C — no compute pool:** the agent generates `environment.yml` instead of `pyproject.toml` and deploys on the warehouse runtime (Streamlit 1.52.2) via the Path B stage flow, without the container parameters.

---

## Step 6 - open the app and verify

Snowsight > **Projects > Streamlit > SNOWPRO_QUIZ** (or whatever `app_name` is set in `AGENTS.md`).

Walk through the pages (navigation is native multipage):

- **Quiz page** - home (round size, difficulty, domain, source, explanations toggle) → quiz (try the 💡 hint *before* answering; after submitting: feedback, AI explanation, ⚖️ contrast between two options) → summary (score, pass/fail vs threshold, wrong-answer cards, AI debrief; fail a round to see the **Runda poprawkowa** button — your wrong answers, reshuffled).
- **Review page** - wrong-answer history with filters + Learning Dashboard tab.
- **Admin page** - flip a config toggle (e.g. round-size default), check bank stats, try **Generate batch (AI)** to seed the question bank.

Complete at least one round so `QUIZ_SESSION_LOG` and `QUIZ_REVIEW_LOG` get data for dashboard charts.

---

## Adding another exam

Keep the same workspace, run the **setup prompt** again with a different PDF. The agent will ask for the exam code, create a new `QUIZ_<NEW_CODE>` schema, and deploy a second Streamlit app. The previous exam is untouched.

---

## Something broke?

Paste the **fix prompt** from [prompts.md](prompts.md) with a description of what happened. The agent will run the relevant diagnostic skill, fix the issue, and ask you to re-upload.

See also [troubleshooting.md](troubleshooting.md) for common Snowsight-specific pitfalls.
