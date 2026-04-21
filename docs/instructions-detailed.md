# Instructions - detailed

A full walk-through for someone who has never used Cortex Code in Snowsight. If you are already comfortable, use [instructions.md](instructions.md).

---

## what this project does

You give Cortex Code a Snowflake certification **study guide PDF**. The agent:

1. Creates a dedicated schema `QUIZ_<EXAM_CODE>` inside your database.
2. Creates 2 stages (one for input data, one for the Streamlit app) and 4 tables (`EXAM_DOMAINS`, `QUIZ_QUESTIONS`, `QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`).
3. Extracts domain list, weights, topics, and testable facts from the PDF using `AI_PARSE_DOCUMENT` + `AI_COMPLETE`.
4. Either loads a question bank you provide, or generates ~30 questions per domain via AI.
5. Generates `quiz.py` (a full Streamlit-in-Snowflake app) and `environment.yml` in the workspace.
6. Runs a mandatory 22-item pre-deploy scan to catch Streamlit-in-Snowflake footguns.
7. Deploys the app via `CREATE STREAMLIT`.

You never leave the browser. You never run `bash`, `git`, `snow`, or `PUT`. You only:
- click around Snowsight UI to upload files to stages;
- read what the agent proposes and say "go" or "no, do X differently";
- open the app at the end.

---

## prerequisites

### account-level (one time, `ACCOUNTADMIN`)

EU accounts (region `AWS_EU_CENTRAL_1`, `EU_WEST_1`, etc.) must allow cross-region inference so `claude-sonnet-4-6` is reachable:

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'AWS_US';
```

Without this, every `AI_COMPLETE` call will fail with "not allowed to access this endpoint".

### role-level

The role you will use needs, on the target database:
- `USAGE`, `CREATE SCHEMA`;
- on a warehouse - `USAGE`, `OPERATE`;
- Cortex AI functions are usable by any role with `USAGE` on the `SNOWFLAKE.CORTEX_USER` database role.

### Snowsight feature flags

1. **Cortex Code** - a white star icon bottom-right of any workspace; click to open the chat panel. GA since 2026-03-09, no enablement needed beyond being in a supported region.
2. **Web search for AI agents** - Snowsight > **AI & ML > Agents > Settings > Tools and connectors > Web search > enable**. 

### study guide PDF

You need a PDF of the target exam's study guide. https://learn.snowflake.com/en/certifications/

Typical source:
- SnowPro Core: [SnowProCoreStudyGuide.pdf](https://learn.snowflake.com/) (the baseline of this repo is COF-C03).
- SnowPro Advanced / Specialty tracks: each has its own study guide.

Save it locally with a clean filename. You will upload it to a Snowflake stage via UI in step 5.

### question bank CSV/JSON (optional)

If you happen to have a ready-made question bank, you can feed it to the agent and skip AI generation. Schema requirements are documented in `AGENTS.md` > `table schemas` > `QUIZ_QUESTIONS`. If columns don't match, `$adapt-questions` maps them for you.

---

## step 1 - load project context into the workspace

Cortex Code in Snowsight supports two kinds of skills:

- **global / built-in** - like `cortex-ai-functions`. These are part of Cortex Code itself and are always on. **You do not upload these.**
- **custom / project-scoped** - live in `.snowflake/cortex/skills/` inside the workspace. You bring them into the workspace yourself.

Two paths:
- **Git integration** is the recommended one — you get all repo files (skills + `AGENTS.md` + `docs/`) in the workspace, plus commit/branch from inside Snowsight. 
- **Manual upload** is the fallback when you cannot enable Git integration on your account.

### via Git integration (recommended)

Snowsight workspaces can be backed by a `GIT REPOSITORY` object. Editing inside the workspace is editing a local checkout. You commit and push from inside Snowsight. 

Benefits:
- `docs/troubleshooting.md`, `docs/architecture.md`, etc. are visible inside Snowsight — no need to jump back to GitHub while working.
- skill updates come via `git pull` — no re-upload.
- generated `quiz.py` / `environment.yml` can be committed back to your fork for reproducibility.
- branch for experiments (new optional feature, different exam).
- teammate with access to the same fork can open the same workspace against the same branch.

#### 1a - fork the repo

Fork `coco-cli-quiz-app` on GitHub. The fork is your own copy. Upstream pulls are optional later.

#### 1b - create the Snowflake-side integration objects

As a role with `CREATE INTEGRATION` — run these in a Snowflake worksheet:

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

For a public fork you intend to only pull from: skip the `SECRET` and omit `GIT_CREDENTIALS` from `CREATE GIT REPOSITORY`.

Auth method summary:
- **OAuth2** — browser flow, cleanest, but needs admin to approve the Snowflake GitHub App in your GitHub org.
- **PAT** (shown above) — works without admin approval; you rotate the token yourself.
- **Public read-only** — no auth; you can `pull` but not `push`. Fine for consuming upstream only.

See [Integrate workspaces with a Git repository](https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-git).

#### 1c - create the workspace

In Snowsight: **Projects > Workspaces > + Workspace > From Git repository** > select `coco_quiz_fork` > pick the branch (`main` or your own).

The workspace mounts the full repo: `.snowflake/cortex/skills/`, `AGENTS.md`, `docs/`, `README.md`, etc. all appear in the file tree. Cortex Code picks up the skills as slash commands automatically (`/setup-exam`, `/adapt-questions`, and routers for `/cortex`, `/sis`, `/quiz` — see [skills.md](skills.md)).

#### working with the Git-backed workspace

- **Edit a skill** > workspace shows it as modified > use the **Git** panel to review the diff > commit > push. Pushes go to your fork's branch.
- **Pull upstream improvements**: `git pull` equivalent inside the workspace.
- **Branch** for an experiment: create a new branch from the Git panel.
- **Generated artefacts**: `quiz.py` and `environment.yml` show up as untracked after `$setup-exam` runs. Choose to commit them (for reproducibility / sharing) or `.gitignore` them (they are outputs, not sources).

#### gotchas specific to Git integration

- The Snowflake GitHub App needs admin approval in your GitHub org (OAuth2 path). If locked down, fall back to PAT.
- `SECRET` objects holding PATs require `USAGE` granted to your role. Miss that grant and the workspace shows "repository not accessible".

Canonical docs:
- [Integrate workspaces with a Git repository](https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-git) - primary page for the workspace side.
- [Using a Git repository in Snowflake](https://docs.snowflake.com/en/developer-guide/git/git-overview) - underlying `GIT REPOSITORY` object, supported providers (GitHub, GitLab, Bitbucket, Azure DevOps, AWS CodeCommit).
- [Setting up the Git integration](https://docs.snowflake.com/en/developer-guide/git/git-setting-up) - API integration, secrets, permissions.

### via manual upload (fallback)

Use this when you cannot enable Git integration (locked-down account, no admin access) or for a quick throw-away run.

#### 1a.M - clone the repo locally

Standard `git clone`. You will drag files out of this clone in the next two substeps.

#### 1b.M - create an empty workspace and upload the skill folder

In Snowsight: **Projects > Workspaces > + Workspace** (no Git backing). Then in the Cortex Code chat input:

1. Click the **+** (or paperclip) icon.
2. Choose **Upload Folder(s)**.
3. Select `.snowflake/cortex/skills/` from your local clone.

After upload, the skills appear under `.snowflake/cortex/skills/` and become available as slash commands.

#### 1c.M - upload AGENTS.md

Drag-drop `AGENTS.md` from your local clone into the workspace root (or use **+** > **Upload File(s)**).

#### what you give up vs. Git integration

- `docs/` stays on your local clone — no in-browser access; reference from local or GitHub.
- no version control inside Snowsight — skill edits live in the workspace only.
- skill updates require re-uploading the folder.
- no branching for experiments.

### 1d - what you do NOT load, either way

- `quiz.py` - the agent generates it into the workspace on each `$setup-exam` run.
- `environment.yml` - same as above.
- your study guide PDF - uploaded to a **Snowflake stage** in step 5, not the workspace.
- your optional CSV/JSON - same as the PDF.

Note on Git integration: `quiz.py` and `environment.yml` land in the workspace file tree and can optionally be committed to your fork - an explicit choice, not automatic.

---

## step 2 - edit the environment table in AGENTS.md

Open `AGENTS.md` in the workspace (click it in the file tree). Find:

```markdown
## snowflake environment

| setting   | value                                |
|-----------|--------------------------------------|
| database  | `<your_database>`                    |
| schema    | `<your_database>.QUIZ_<EXAM_CODE>`   |
| warehouse | `<your_warehouse>`                   |
| role      | `<your_role>`                        |
...
```

Replace the three `<your_...>` placeholders with actual object names:

- `<your_database>` - the database in which you want `QUIZ_<CODE>` schemas created;
- `<your_warehouse>` - the warehouse that will power the Streamlit app and Cortex AI calls;
- `<your_role>` - the role you will be using (must be active in your Snowsight session, and must have `CREATE SCHEMA` on the database above).

Leave `schema` and `exam_code` as is - `$setup-exam` will fill those in once you tell it which exam you want. `stage`, `app stage`, `app_name`, `main_file` have working defaults - change them only if you need different names.

Save. Cortex Code re-reads `AGENTS.md` on the next message. If you forget to fill any of the three placeholders, `$setup-exam` halts in Step 1a and prompts you to finish the edit.

---

## step 3 - run the setup prompt

Copy the **setup prompt** from [prompts.md](prompts.md) and paste it into the Cortex Code. The prompt is intentionally short - it just tells the agent to run `$setup-exam` end-to-end and stop at each manual upload.

The agent will:

1. Ask you for the exam name and exam code (e.g. "SnowPro Core" / "COF-C03").
2. Ask for the PDF filename (required) and optionally the CSV filename.
3. Ask about additional customisations (optional features from `$quiz/features` - exam simulation mode, flashcards, AI study recommendations, etc.).
4. Create the schema, stages, tables, file format (SQL visible in the chat - approve or reject each step).
5. Stop and ask you to upload the PDF.

Do **not** try to pre-empt the agent by creating objects manually. Let it drive.

---

## step 4 - upload the study guide PDF (and optional CSV/JSON)

When the agent says something like *"please upload SnowProCoreStudyGuide.pdf to STAGE_QUIZ_DATA"*:

1. Open a new browser tab to Snowsight (keep the Cortex Code tab open - you'll come back).
2. Navigate: **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_QUIZ_DATA**.
3. Top-right: click **+ Files**.
4. Drag-drop the PDF (or browse). Single files up to **250 MB**.
5. Click **Upload**. Wait for the progress bar to complete.
6. (If you also have a CSV/JSON) click **+ Files** again, repeat.

Alternative UI path: **Ingestion > Add Data > Load files into a Stage > STAGE_QUIZ_DATA**. Same outcome.

Return to the Cortex Code tab and reply: **"uploaded"** (or just "done"). The agent will run:

```sql
ALTER STAGE ... STAGE_QUIZ_DATA REFRESH;
LIST @...STAGE_QUIZ_DATA;
```

and confirm the file(s) are visible.

### stage encryption - why it matters

`STAGE_QUIZ_DATA` is created with `ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')` and `DIRECTORY = (ENABLE = TRUE)`. Both are **required** for `AI_PARSE_DOCUMENT` to read the PDF. If you see "file not accessible" later, the most common cause is a stage that was created without `SNOWFLAKE_SSE`. `$setup-exam` always uses the right DDL; but if you recycle an old stage, drop and recreate it.

---

## step 5 - domain extraction + key_facts

After you confirm upload, the agent runs:

```sql
SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@...STAGE_QUIZ_DATA', 'SnowProCoreStudyGuide.pdf'),
    {'mode': 'LAYOUT'}
):content::VARCHAR;
```

then feeds the parsed text into `AI_COMPLETE` to extract:

- each domain's `domain_id`, `domain_name`, `weight_pct`, `topics` - inserted into `EXAM_DOMAINS`.
- each domain's `key_facts` - a plain-text list of testable facts per domain - stored in `EXAM_DOMAINS.key_facts`, reused later to ground AI question generation.

### date-based disambiguation

Study guides sometimes cover a transition: "old blueprint effective until date X, new blueprint effective from date Y". If the agent finds both, it will ask you which one to use. If in doubt: pick the one **effective as of today**.

### verification checkpoint

The agent will report: *"N domains found, weights sum to 100.0, all key_facts populated. Approve / Re-extract / Abort?"*

If numbers look wrong (e.g. weights sum to 97 - AI missed a domain), choose **Re-extract** and the agent will wipe `EXAM_DOMAINS` and try again.

---

## step 6 - load or generate the question bank

Two paths depending on what you said in step 4:

**CSV/JSON available** > `COPY INTO QUIZ_QUESTIONS FROM @STAGE_QUIZ_DATA/<filename>.csv FILE_FORMAT = FF_*;` then backfills `domain_name` from `EXAM_DOMAINS`. If the CSV columns don't match the target schema, the agent invokes `$adapt-questions` which maps columns.

**AI generation only** > for each domain, the agent reads `key_facts` + `topics` and runs `AI_COMPLETE` in batches of 10 questions (JSON output). Target: ~30 questions per domain, mix of easy/medium/hard. For 5–6 domains that is 150–180 questions, takes a few minutes.

Either way, the agent reports row count, distinct domain count, and null-domain count. If anything is zero it investigates before continuing.

---

## step 7 - update AGENTS.md

`$setup-exam` edits `AGENTS.md` in place:

- schema name: `QUIZ_<NEW_CODE>`;
- exam code: the value you provided;
- title line: exam name;
- source files section: PDF filename, CSV presence.

Other sections (table schemas, platform constraints, Cortex LLM patterns, app structure) are **not** touched - they are generic.

---

## step 8 - generate quiz.py + environment.yml and scan

The agent reads the updated `AGENTS.md` plus all `$quiz/*` skills (screens, questions, style, optionally features) and writes:

- `quiz.py` - ~30 KB, a full Streamlit app with home / quiz / summary / review + dashboard;
- `environment.yml` - minimal conda manifest (`streamlit=1.52.*`, `pandas`, `altair`).

Both appear in the workspace file tree.

Then it runs the **22-item pre-deploy scan** from `$sis/pre-deploy`. This catches things like:

- `AI_COMPLETE` prompts not dollar-quoted;
- `st.rerun()` in the wrong number of places (must be exactly 6);
- `st.fragment` / `st.connection` / `unsafe_allow_html` used anywhere (not supported in SiS v1.52.*);
- SQL injection risks;
- column-name case mismatches between SQL and pandas;
- `get_active_session()` called outside the right cache scope.

If anything fails, the agent fixes it and re-scans until clean. Do not proceed to deploy on a failed scan.

---

## step 9 - upload to STAGE_SIS_APP and deploy

When the scan is clean, the agent asks you to upload `quiz.py` + `environment.yml` to `STAGE_SIS_APP`:

1. Snowsight > **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_SIS_APP**.
2. **+ Files** > drag-drop both files from your workspace file tree (right-click > download, or open each and save) > **Upload**.
3. Reply "uploaded".

The agent runs:

```sql
CREATE OR REPLACE STREAMLIT <your_database>.QUIZ_<CODE>.SNOWPRO_QUIZ
  FROM '@<your_database>.QUIZ_<CODE>.STAGE_SIS_APP'
  MAIN_FILE = '/quiz.py'
  QUERY_WAREHOUSE = <your_warehouse>;

SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA <your_database>.QUIZ_<CODE>;
```

### alternative - "Deploy as Streamlit App" right-click

If you are on a recent Snowsight build, right-click `quiz.py` in the workspace file tree and choose **Deploy as Streamlit App**. Snowsight handles the stage + `CREATE STREAMLIT` internally and asks you for warehouse / schema / app name.

This is marked **unverified** in our current docs - it works, but we haven't fully validated that `environment.yml` is picked up end-to-end. If you want a fully scripted reproducible flow, stick with Path A (upload to stage).

---

## step 10 - open the app, complete one round

Snowsight > **Projects > Streamlit > SNOWPRO_QUIZ**.

- on **Home**: pick 5 questions, medium difficulty, any domain, "AI Generated" source, explanations ON. Click **Start Round**.
- on **Quiz**: wait a couple seconds for the first AI-generated question to load, answer it, submit, check the explanation.
- click through all 5, then **Finish**.
- **Summary**: score, pass/fail vs 75%, wrong-answer cards.
- **Review** tab: filter by domain; click **Learning Dashboard** - you should see your first session plotted.

Confirm:

```sql
SELECT COUNT(*) FROM <your_database>.QUIZ_<CODE>.QUIZ_SESSION_LOG;  -- >= 1
SELECT COUNT(*) FROM <your_database>.QUIZ_<CODE>.QUIZ_REVIEW_LOG;   -- >= 0 (only non-zero if you got one wrong)
```

---

## adding a second exam in the same workspace

Keep everything as is, open a new Cortex Code chat (or the same one), paste the **setup prompt** again, give a different exam code. The agent:

- creates a new schema `QUIZ_<NEW_CODE>`;
- leaves the previous schema completely untouched;
- edits `AGENTS.md` to point at the new schema (so future chats target the new exam by default; you can flip between them by re-editing `AGENTS.md`).

Both Streamlit apps coexist at `Projects > Streamlit > SNOWPRO_QUIZ` (new) and whatever app name the old one had. Rename either via `ALTER STREAMLIT ... RENAME TO ...` if you want more descriptive names.

### optional: branch per exam (Git-backed workspace)

If you followed the **Git integration** path in step 2, you can additionally isolate each exam on its own branch. The agent does not switch branches — you do, before running the setup prompt:

- **workspace Git panel** (bottom bar): click the branch name > **Create new branch from `main`** > e.g. `exam/ARA-C01`. Workspace switches automatically.
- **GitHub**: create the branch on github.com, then in the workspace Git panel click **Switch branch**.

Run the setup prompt on the new branch. `AGENTS.md` edits and `quiz.py` generation happen on that branch. Commit when ready.

To switch back to a previous exam later: change branch in the Git panel — the matching `AGENTS.md` snapshot comes with it, so you don't even need to re-edit the schema / exam_code lines. This matches the CLI variant's `exam/<code>` branch pattern, just with the branch creation step being manual rather than `git checkout -b`.

Skip this step if you only plan one or two exams — the schema-per-exam + single-`AGENTS.md`-that-you-edit approach is simpler and already isolates runtime data.

---

## something broke

Paste the **fix prompt** from [prompts.md](prompts.md), describe the symptom. The agent triages:

- `AI_COMPLETE` / `AI_PARSE_DOCUMENT` errors > runs `$cortex/patterns` 5-step diagnostic;
- app crashes in Streamlit > re-runs `$sis/pre-deploy`;
- wrong content / shallow explanations > runs `$cortex/prompt-audit`;
- screen flow glitches > reads `$quiz/screens`.

After a fix it asks you to re-upload `quiz.py` (step 10 again) and redeploys.

See [troubleshooting.md](troubleshooting.md) for a curated list of the most common Snowsight-specific issues.

---

## what NOT to do

- Do **not** manually create schemas, stages, or tables before running `$setup-exam`. The skill expects to own the full lifecycle and will skip or collide.
- Do **not** edit the environment table placeholders to values that don't exist - the agent will try to `USE WAREHOUSE <name>` and fail with a clear error, but it wastes a cycle.
- Do **not** drop the previous exam's schema unless you genuinely want to. Each schema is self-contained and coexistence is the default.
