# Instructions - detailed

A full walk-through for someone who has never used Snowflake CoCo in Snowsight. If you are already comfortable, use [instructions.md](instructions.md).

---

## What this project does

You give CoCo a Snowflake certification **study guide PDF**. The agent:

1. Creates a dedicated schema `QUIZ_<EXAM_CODE>` inside your database.
2. Creates 2 stages (one for input data, one for the Streamlit app) and 4 tables (`EXAM_DOMAINS`, `QUIZ_QUESTIONS`, `QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`).
3. Extracts domain list, weights, topics, and testable facts from the PDF using `AI_PARSE_DOCUMENT` + `AI_COMPLETE`.
4. Loads a question bank if you provide one (CSV/JSON); otherwise the bank stays empty and questions are AI-generated at runtime (you can seed the bank later from the Admin page, a worksheet recipe, or a scheduled task).
5. Generates the multipage `app/` Streamlit project in the workspace (`main.py`, `_*.py` modules, `pages/`, configs — container runtime).
6. Runs a mandatory pre-deploy scan to catch Streamlit-in-Snowflake footguns.
7. Deploys the app — by default via the Workspaces **Run + Deploy** flow (live preview, no stage upload), or via stage + `CREATE STREAMLIT` as the scripted fallback.

You never leave the browser. You never run `bash`, `git`, `snow`, or `PUT`. You only:
- Click around Snowsight UI to upload input files to a stage;
- Read what the agent proposes and say "go" or "no, do X differently";
- Click **Run** to preview and **Deploy** to publish the app at the end.

---

## Prerequisites

### Account-level (one time, `ACCOUNTADMIN`)

If your account cannot reach `claude-sonnet-4-6` in-region (typically EU regions like `AWS_EU_CENTRAL_1`), enable cross-region inference:

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

Without this, every `AI_COMPLETE` call will fail with "not allowed to access this endpoint". Accounts created after 2026-03-09 already default to `ANY_REGION`; `'AWS_GLOBAL'` is a narrower alternative, the legacy `'AWS_US'` still works but is narrowest.

The container runtime (default deploy target) also needs a **compute pool**:

```sql
SHOW COMPUTE POOLS;   -- at least one, with USAGE for your role
```

No pool and no admin to create one? Use the warehouse fallback (step 9, Path C) — no compute pool needed.

### Role-level

The role you will use needs, on the target database:
- `USAGE`, `CREATE SCHEMA`;
- On a warehouse - `USAGE`, `OPERATE`;
- On a compute pool (container runtime) - `USAGE`;
- Cortex AI functions are usable by any role with `USAGE` on the `SNOWFLAKE.CORTEX_USER` database role.

### Snowsight feature flags

1. **CoCo** - a white star icon bottom-right of any workspace; click to open the chat panel. GA since 2026-03-09, no enablement needed beyond being in a supported region.
2. **Web search for AI agents** - Snowsight > **AI & ML > Agents > Settings > Tools and connectors > Web search > enable**. 

### Study guide PDF

You need a PDF of the target exam's study guide. https://learn.snowflake.com/en/certifications/

Typical source:
- SnowPro Core: [SnowProCoreStudyGuide.pdf](https://learn.snowflake.com/) (the baseline of this repo is COF-C03).
- SnowPro Advanced / Specialty tracks: each has its own study guide.

Save it locally with a clean filename. You will upload it to a Snowflake stage via UI in step 4.

### Question bank CSV/JSON (optional)

If you happen to have a ready-made question bank, you can feed it to the agent and skip AI generation. Schema requirements are documented in `AGENTS.md` > `table schemas` > `QUIZ_QUESTIONS`. If columns don't match, `$adapt-questions` maps them for you.

---

## Step 1 - load project context into the workspace

CoCo in Snowsight supports two kinds of skills:

- **Global / built-in** - like `cortex-ai-functions`. These are part of CoCo itself and are always on. **You do not upload these.**
- **Custom / project-scoped** - live in `.snowflake/cortex/skills/` inside the workspace. You bring them into the workspace yourself.

Two paths:
- **Git integration** is the recommended one — you get all repo files (skills + `AGENTS.md` + `docs/`) in the workspace, plus commit/branch from inside Snowsight. 
- **Manual upload** is the fallback when you cannot enable Git integration on your account.

### Via Git integration (recommended)

Snowsight workspaces can be backed by a `GIT REPOSITORY` object. Editing inside the workspace is editing a local checkout. You commit and push from inside Snowsight. 

Benefits:
- `docs/troubleshooting.md`, `docs/architecture.md`, etc. are visible inside Snowsight — no need to jump back to GitHub while working.
- Skill updates come via `git pull` — no re-upload.
- The generated `app/` project can be committed back to your fork for reproducibility.
- Branch for experiments (new optional feature, different exam).
- Teammate with access to the same fork can open the same workspace against the same branch.

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

The workspace mounts the full repo: `.snowflake/cortex/skills/`, `AGENTS.md`, `docs/`, `README.md`, etc. all appear in the file tree. CoCo picks up the skills as slash commands automatically (`/setup-exam`, `/adapt-questions`, and routers for `/cortex`, `/sis`, `/quiz` — see [skills.md](skills.md)).

#### Working with the Git-backed workspace

- **Edit a skill** > workspace shows it as modified > use the **Git** panel to review the diff > commit > push. Pushes go to your fork's branch.
- **Pull upstream improvements**: `git pull` equivalent inside the workspace.
- **Branch** for an experiment: create a new branch from the Git panel.
- **Generated artefacts**: the `app/` project shows up as untracked after `$setup-exam` runs. Choose to commit it (for reproducibility / sharing) or `.gitignore` it (outputs, not sources).

#### Gotchas specific to Git integration

- The Snowflake GitHub App needs admin approval in your GitHub org (OAuth2 path). If locked down, fall back to PAT.
- `SECRET` objects holding PATs require `USAGE` granted to your role. Miss that grant and the workspace shows "repository not accessible".

Canonical docs:
- [Integrate workspaces with a Git repository](https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-git) - primary page for the workspace side.
- [Using a Git repository in Snowflake](https://docs.snowflake.com/en/developer-guide/git/git-overview) - underlying `GIT REPOSITORY` object, supported providers (GitHub, GitLab, Bitbucket, Azure DevOps, AWS CodeCommit).
- [Setting up the Git integration](https://docs.snowflake.com/en/developer-guide/git/git-setting-up) - API integration, secrets, permissions.

### Via manual upload (fallback)

Use this when you cannot enable Git integration (locked-down account, no admin access) or for a quick throw-away run.

#### 1a.M - clone the repo locally

Standard `git clone`. You will drag files out of this clone in the next two substeps.

#### 1b.M - create an empty workspace and upload the skill folder

In Snowsight: **Projects > Workspaces > + Workspace** (no Git backing). Then in the CoCo chat input:

1. Click the **+** (or paperclip) icon.
2. Choose **Upload Folder(s)**.
3. Select `.snowflake/cortex/skills/` from your local clone.

After upload, the skills appear under `.snowflake/cortex/skills/` and become available as slash commands.

#### 1c.M - upload AGENTS.md

Drag-drop `AGENTS.md` from your local clone into the workspace root (or use **+** > **Upload File(s)**).

#### What you give up vs. Git integration

- `docs/` stays on your local clone — no in-browser access; reference from local or GitHub.
- No version control inside Snowsight — skill edits live in the workspace only.
- Skill updates require re-uploading the folder.
- No branching for experiments.

### 1d - what you do NOT load, either way

- The `app/` project (`main.py`, `_*.py` modules, `pages/`, configs) - the agent generates it into the workspace on each `$setup-exam` run.
- Your study guide PDF - uploaded to a **Snowflake stage** in step 4, not the workspace.
- Your optional CSV/JSON - same as the PDF.

Note on Git integration: the generated `app/` files land in the workspace file tree and can optionally be committed to your fork - an explicit choice, not automatic.

---

## Step 2 - edit the environment table in AGENTS.md

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

Replace the `<your_...>` placeholders with actual object names:

- `<your_database>` - the database in which you want `QUIZ_<CODE>` schemas created;
- `<your_warehouse>` - the warehouse that will power the Streamlit app and Cortex AI calls;
- `<your_role>` - the role you will be using (must be active in your Snowsight session, and must have `CREATE SCHEMA` on the database above);
- `<your_compute_pool>` - the compute pool for the container runtime (skippable only if you plan the warehouse fallback, step 9 Path C).

Leave `schema` and `exam_code` as is - `$setup-exam` will fill those in once you tell it which exam you want. `stage`, `app stage`, `app_name`, `main_file`, `runtime`, `deps_file` have working defaults - change them only if you need different names.

Save. CoCo re-reads `AGENTS.md` on the next message. If you forget to fill any required placeholder, `$setup-exam` halts in Step 1a and prompts you to finish the edit.

---

## Step 3 - run the setup prompt

Copy the **setup prompt** from [prompts.md](prompts.md) and paste it into CoCo. The prompt is intentionally short - it just tells the agent to run `$setup-exam` end-to-end and stop at each manual upload.

The agent will:

1. Ask you for the exam name and exam code (e.g. "SnowPro Core" / "COF-C03").
2. Ask for the PDF filename (required) and optionally the CSV filename.
3. Ask about additional customisations (optional features from `$quiz/features` - exam simulation mode, flashcards, AI study recommendations, misconception analysis, question flagging, etc.).
4. Ask whether you want the **default look or a custom one** - custom means a short style dialog (light/dark, accent color, roundness, fonts), applied via Streamlit theming only.
5. Create the schema, stages, tables, file format (SQL visible in the chat - approve or reject each step).
6. Stop and ask you to upload the PDF.

Do **not** try to pre-empt the agent by creating objects manually. Let it drive.

---

## Step 4 - upload the study guide PDF (and optional CSV/JSON)

When the agent says something like *"please upload SnowProCoreStudyGuide.pdf to STAGE_QUIZ_DATA"*:

1. Open a new browser tab to Snowsight (keep the CoCo tab open - you'll come back).
2. Navigate: **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_QUIZ_DATA**.
3. Top-right: click **+ Files**.
4. Drag-drop the PDF (or browse). Single files up to **250 MB**.
5. Click **Upload**. Wait for the progress bar to complete.
6. (If you also have a CSV/JSON) click **+ Files** again, repeat.

Alternative UI path: **Ingestion > Add Data > Load files into a Stage > STAGE_QUIZ_DATA**. Same outcome.

Return to the CoCo tab and reply: **"uploaded"** (or just "done"). The agent will run:

```sql
ALTER STAGE ... STAGE_QUIZ_DATA REFRESH;
LIST @...STAGE_QUIZ_DATA;
```

and confirm the file(s) are visible.

### Stage encryption - why it matters

`STAGE_QUIZ_DATA` is created with `ENCRYPTION = (TYPE = 'SNOWFLAKE_SSE')` and `DIRECTORY = (ENABLE = TRUE)`. Both are **required** for `AI_PARSE_DOCUMENT` to read the PDF. If you see "file not accessible" later, the most common cause is a stage that was created without `SNOWFLAKE_SSE`. `$setup-exam` always uses the right DDL; but if you recycle an old stage, drop and recreate it.

---

## Step 5 - domain extraction + key_facts

After you confirm upload, the agent runs:

```sql
SELECT AI_PARSE_DOCUMENT(
    TO_FILE('@...STAGE_QUIZ_DATA', 'SnowProCoreStudyGuide.pdf'),
    {'mode': 'LAYOUT'}
):content::VARCHAR;
```

then feeds the parsed text into `AI_COMPLETE` to extract:

- Each domain's `domain_id`, `domain_name`, `weight_pct`, `topics` - inserted into `EXAM_DOMAINS`.
- Each domain's `key_facts` - a plain-text list of testable facts per domain - stored in `EXAM_DOMAINS.key_facts`, reused later to ground AI question generation.

### Date-based disambiguation

Study guides sometimes cover a transition: "old blueprint effective until date X, new blueprint effective from date Y". If the agent finds both, it will ask you which one to use. If in doubt: pick the one **effective as of today**.

### Verification checkpoint

The agent will report: *"N domains found, weights sum to 100.0, all key_facts populated. Approve / Re-extract / Abort?"*

If numbers look wrong (e.g. weights sum to 97 - AI missed a domain), choose **Re-extract** and the agent will wipe `EXAM_DOMAINS` and try again.

---

## Step 6 - load the question bank (optional)

Two paths depending on what you said in step 4:

**CSV/JSON available** > `COPY INTO QUIZ_QUESTIONS FROM @STAGE_QUIZ_DATA/<filename>.csv FILE_FORMAT = FF_*;` then backfills `domain_name` from `EXAM_DOMAINS`. If the CSV columns don't match the target schema, the agent invokes `$adapt-questions` which maps columns. The agent reports row count, distinct domain count, and null-domain count.

**No CSV/JSON** > the bank deliberately stays **empty** — the agent does NOT generate questions during setup (it's slow and would burn your token budget before you ever see the app). The app works fully on runtime AI questions. The agent explains why a populated bank is still worth having (resilience when AI calls fail, instant load, curated consistency) and how to seed it later: CSV upload, the Admin page **Generate batch** button, the worksheet recipe in [customization.md](customization.md) (section 6), or a scheduled task / Automation.

---

## Step 7 - update AGENTS.md

`$setup-exam` edits `AGENTS.md` in place:

- Schema name: `QUIZ_<NEW_CODE>`;
- Exam code: the value you provided;
- Title line: exam name;
- Source files section: PDF filename, CSV presence.

Other sections (table schemas, platform constraints, Cortex LLM patterns, app structure) are **not** touched - they are generic.

---

## Step 8 - generate the `app/` project and scan

The agent reads the updated `AGENTS.md` plus all `$quiz/*` skills (screens, questions, style, optionally features) and writes the decomposed multipage project into the workspace:

- `app/main.py` - entry point: `st.set_page_config`, session-state init, `st.navigation`;
- `app/_config.py`, `app/_cortex.py`, `app/_data.py`, `app/_questions.py`, `app/_ui.py` - constants, Cortex calls, cached loaders, question engine, shared UI helpers;
- `app/pages/quiz.py` + `app/pages/review.py` - the two pages (plus one page per requested optional feature);
- `app/.streamlit/config.toml` + `app/pyproject.toml` + `app/snowflake.yml` - app config, container-runtime dependencies (PyPI), deploy descriptor.

Everything appears in the workspace file tree under `app/`.

Then it runs the **pre-deploy scan** from `$sis/pre-deploy` across all app files. It catches things like:

- `AI_COMPLETE` prompts not dollar-quoted;
- SQL injection risks (f-string interpolation instead of bind params);
- Column-name case mismatches between SQL and pandas;
- `get_active_session()` called outside the right cache scope.

If anything fails, the agent fixes it and re-scans until clean. Do not proceed to deploy on a failed scan.

---

## Step 9 - preview and deploy

When the scan is clean, pick a deploy path. **Path A is the default.**

### Path A - Workspaces live preview + Deploy (default)

Streamlit-in-Workspaces (Public Preview) runs the app straight from the workspace - no stage upload:

1. Open `app/main.py` in the workspace and click **Run** (or press Cmd/Ctrl+Enter).
2. A private **dev app** preview opens in the browser - only you can see it. Iterate with the agent until it looks right (agent edits, you Run again).
3. Click **Deploy** in the project toolbar. In the dialog set: app title `SNOWPRO_QUIZ`, database `<your_db>`, schema `QUIZ_<CODE>`, **compute pool**, query warehouse.
4. Reply "deployed" - the agent verifies with `SHOW STREAMLITS`.

Remember: after later edits, the published app updates only when you **Deploy** again - **Run** refreshes only your private dev app.

### Path B - scripted: stage + CREATE STREAMLIT (container runtime)

For a fully scripted, reproducible flow:

1. Snowsight > **Data > Databases > `<your_db>` > `QUIZ_<CODE>` > Stages > STAGE_SIS_APP**.
2. **+ Files** > upload the `app/` files, preserving the folder layout (`pages/`, `.streamlit/`).
3. Reply "uploaded".

The agent runs:

```sql
CREATE OR REPLACE STREAMLIT <your_database>.QUIZ_<CODE>.SNOWPRO_QUIZ
  FROM '@<your_database>.QUIZ_<CODE>.STAGE_SIS_APP'
  MAIN_FILE = 'main.py'
  RUNTIME_NAME = 'SYSTEM$ST_CONTAINER_RUNTIME_PY3_11'
  COMPUTE_POOL = <your_compute_pool>
  QUERY_WAREHOUSE = <your_warehouse>;

SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA <your_database>.QUIZ_<CODE>;
```

(From a machine with the Snowflake CLI, `snow streamlit deploy` does the same, driven by the generated `snowflake.yml`.)

### Path C - warehouse fallback (no compute pool)

If the account has no usable compute pool, the agent generates `environment.yml` instead of `pyproject.toml` and deploys via the Path B stage flow **without** `RUNTIME_NAME`/`COMPUTE_POOL`. The warehouse runtime caps Streamlit at 1.52.2.

---

## Step 10 - open the app, complete one round

Snowsight > **Projects > Streamlit > SNOWPRO_QUIZ**.

- On **Home**: pick 5 questions, medium difficulty, any domain, "AI Generated" source, explanations ON. Click **Start Round**.
- On **Quiz**: wait a couple seconds for the first AI-generated question to load. Try the **💡 Podpowiedź** button *before* answering (two levels, never spoils). Answer, submit, check the explanation — and try **⚖️ Porównaj** on two options.
- Click through all 5, then **Finish**.
- **Summary**: score, pass/fail vs threshold, wrong-answer cards, the AI debrief. If you failed, you'll also see **Runda poprawkowa** — your wrong answers back, reshuffled (it doesn't write to stats).
- **Review** page: filter by domain; click **Learning Dashboard** - you should see your first session plotted (remedial rounds excluded by design).
- **Admin** page: check bank stats, flip a toggle (e.g. hints off/on), optionally **Generate batch (AI)** to start seeding the bank.

Confirm:

```sql
SELECT COUNT(*) FROM <your_database>.QUIZ_<CODE>.QUIZ_SESSION_LOG;  -- >= 1
SELECT COUNT(*) FROM <your_database>.QUIZ_<CODE>.QUIZ_REVIEW_LOG;   -- >= 0 (only non-zero if you got one wrong)
```

---

## Adding a second exam in the same workspace

Keep everything as is, open a new CoCo chat (or the same one), paste the **setup prompt** again, give a different exam code. The agent:

- Creates a new schema `QUIZ_<NEW_CODE>`;
- Leaves the previous schema completely untouched;
- Edits `AGENTS.md` to point at the new schema (so future chats target the new exam by default; you can flip between them by re-editing `AGENTS.md`).

Both Streamlit apps coexist at `Projects > Streamlit > SNOWPRO_QUIZ` (new) and whatever app name the old one had. Rename either via `ALTER STREAMLIT ... RENAME TO ...` if you want more descriptive names.

### Optional: branch per exam (Git-backed workspace)

If you followed the **Git integration** path in step 2, you can additionally isolate each exam on its own branch. The agent does not switch branches — you do, before running the setup prompt:

- **Workspace Git panel** (bottom bar): click the branch name > **Create new branch from `main`** > e.g. `exam/ARA-C01`. Workspace switches automatically.
- **GitHub**: create the branch on github.com, then in the workspace Git panel click **Switch branch**.

Run the setup prompt on the new branch. `AGENTS.md` edits and the `app/` generation happen on that branch. Commit when ready.

To switch back to a previous exam later: change branch in the Git panel — the matching `AGENTS.md` snapshot comes with it, so you don't even need to re-edit the schema / exam_code lines. This matches the CLI variant's `exam/<code>` branch pattern, just with the branch creation step being manual rather than `git checkout -b`.

Skip this step if you only plan one or two exams — the schema-per-exam + single-`AGENTS.md`-that-you-edit approach is simpler and already isolates runtime data.

---

## Something broke

Paste the **fix prompt** from [prompts.md](prompts.md), describe the symptom. The agent triages:

- `AI_COMPLETE` / `AI_PARSE_DOCUMENT` errors > runs `$cortex/patterns` 5-step diagnostic;
- App crashes in Streamlit > re-runs `$sis/pre-deploy`;
- Wrong content / shallow explanations > runs `$cortex/prompt-audit`;
- Screen flow glitches > reads `$quiz/screens`.

After a fix it asks you to **Deploy** again from the workspace (Path A) or re-upload the changed files to `STAGE_SIS_APP` (Path B) and redeploys.

See [troubleshooting.md](troubleshooting.md) for a curated list of the most common Snowsight-specific issues.

---

## What NOT to do

- Do **not** manually create schemas, stages, or tables before running `$setup-exam`. The skill expects to own the full lifecycle and will skip or collide.
- Do **not** edit the environment table placeholders to values that don't exist - the agent will try to `USE WAREHOUSE <name>` and fail with a clear error, but it wastes a cycle.
- Do **not** drop the previous exam's schema unless you genuinely want to. Each schema is self-contained and coexistence is the default.
