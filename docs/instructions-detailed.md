# Instructions - detailed

A full walk-through for someone who has never used Snowflake CoCo in Snowsight. If you are already comfortable, use [instructions.md](instructions.md).

---

## What this project does

You give CoCo a Snowflake certification **study guide PDF**. The agent:

1. Creates a dedicated schema `QUIZ_<EXAM_CODE>` inside your database.
2. Creates 2 stages (one for input data, one for the Streamlit app) and 5 tables (`EXAM_DOMAINS`, `QUIZ_QUESTIONS`, `QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`, `QUIZ_CONFIG`) - plus a transient `_DOC_CONTENT` that holds the parsed PDF during setup and is kept across the run and reloads to avoid re-parsing, not auto-dropped (and a CSV file format only if you load a bank CSV).
3. Extracts domain list, weights, topics, and testable facts from the PDF using `AI_PARSE_DOCUMENT` + `AI_COMPLETE`.
4. Loads a question bank if you provide one (CSV/JSON); otherwise the bank stays empty and questions are AI-generated at runtime (you can seed the bank later from the Admin page, a worksheet recipe, or a scheduled task).
5. Generates the multipage `app/` Streamlit project in the workspace (`main.py`, `_*.py` modules, `pages/`, configs - with `environment.yml` from the Snowflake Anaconda channel).
6. Runs two mandatory pre-deploy checks - the `$sis` scan (Streamlit-in-Snowflake footguns) and the `$quiz/screens` UX-conformance gate (the screens match the UX contracts).
7. Deploys the app - the agent copies `app/` from the workspace stage onto `STAGE_SIS_APP` and runs `CREATE STREAMLIT` on the warehouse runtime.

You never leave the browser. You never run `bash`, `git`, `snow`, or `PUT`. You only:
- Load this asset into a workspace - fork the repo and open a **Git-backed workspace**, or upload the `.snowflake/cortex/skills/` folder + `AGENTS.md` (Step 1);
- Drop the study-guide PDF (and any optional CSV) into the workspace file tree when asked - the agent stages it via `COPY FILES`;
- Read what the agent proposes and say "go" or "no, do X differently".

The deploy itself is **scripted on the warehouse runtime** (the agent runs `COPY FILES` + `CREATE STREAMLIT`).

---

## Prerequisites

### Account-level (one time, `ACCOUNTADMIN`)

If your account cannot reach `claude-sonnet-4-6` in-region (typically EU regions like `AWS_EU_CENTRAL_1`), enable cross-region inference:

```sql
ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';
```

Without this, every `AI_COMPLETE` call will fail with "not allowed to access this endpoint". Accounts created after 2026-03-09 already default to `ANY_REGION`; `'AWS_GLOBAL'` is a narrower alternative, the legacy `'AWS_US'` still works but is narrowest.

`pandas`/`altair` come from the Snowflake Anaconda channel.

### Doc grounding (required for Snowflake exams)

Get the free **Snowflake Documentation** listing (Snowsight » Data Products » Marketplace; `IMPORT SHARE`/ACCOUNTADMIN). It gives the app a Cortex Search service over real Snowflake docs. `$setup-exam` Step 1g probes it and sets `grounding_mode = cke` for a Snowflake exam. For a Snowflake exam this is a **hard gate**: if the CKE listing is absent, setup stops and asks you to install it - the app never generates from built-in knowledge. The mode is fixed once at setup (stored in `QUIZ_CONFIG`) - there is no runtime toggle (the Admin page doesn't surface it; it only warns if the doc service is unreachable). (`none`, an explicitly ungrounded mode, is only for non-Snowflake exams.)

### Role-level

The role you will use needs, on the target database:
- `USAGE`, `CREATE SCHEMA`;
- On a warehouse - `USAGE`, `OPERATE`;
- Cortex AI functions are usable by any role with `USAGE` on the `SNOWFLAKE.CORTEX_USER` database role.

### Snowsight feature flags

1. **CoCo** - a white star icon bottom-right of any workspace; click to open the chat panel. GA since 2026-03-09, no enablement needed beyond being in a supported region.
2. **Web search for AI agents** - Snowsight > **AI & ML > Agents > Settings > Tools and connectors > Web search > enable**. 

### Study guide PDF

You need a PDF of the target exam's study guide. https://learn.snowflake.com/en/certifications/

Typical source:
- SnowPro Core: [SnowProCoreStudyGuide.pdf](https://learn.snowflake.com/) (the baseline of this repo is COF-C03).
- SnowPro Advanced / Specialty tracks: each has its own study guide.

Save it locally with a clean filename. You will drop it into the workspace file tree in step 4; the agent stages it for you.

### Question bank CSV/JSON (optional)

If you happen to have a ready-made question bank, you can feed it to the agent and skip AI generation. Schema requirements are documented in `AGENTS.md` > `table schemas` > `QUIZ_QUESTIONS`. If columns don't match, `$adapt-questions` maps them for you.

---

## Step 1 - load project context into the workspace

CoCo in Snowsight supports two kinds of skills:

- **Global / built-in** - like `cortex-ai-function-studio`. These are part of CoCo itself and are always on. **You do not upload these.**
- **Custom / project-scoped** - live in `.snowflake/cortex/skills/` inside the workspace. You bring them into the workspace yourself.

Two paths:
- **Git integration** is the recommended one - you get all repo files (skills + `AGENTS.md` + `docs/`) in the workspace, plus commit/branch from inside Snowsight. 
- **Manual upload** is the fallback when you cannot enable Git integration on your account.

### Via Git integration (recommended)

Snowsight workspaces can be backed by a `GIT REPOSITORY` object. Editing inside the workspace is editing a local checkout. You commit and push from inside Snowsight. 

Benefits:
- `docs/troubleshooting.md`, `docs/architecture.md`, etc. are visible inside Snowsight - no need to jump back to GitHub while working.
- Skill updates come via `git pull` - no re-upload.
- The generated `app/` project can be committed back to your fork for reproducibility.
- Branch for experiments (a different exam, your own tweaks).
- Teammate with access to the same fork can open the same workspace against the same branch.

#### 1a - fork the repo

Fork `coco-quiz-app` on GitHub. The fork is your own copy. Upstream pulls are optional later.

#### 1b - create the Snowflake-side integration objects

As a role with `CREATE INTEGRATION` - run these in a Snowflake worksheet:

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

For a public fork you intend to only pull from: skip the `SECRET` and omit `GIT_CREDENTIALS` from `CREATE GIT REPOSITORY`.

Auth method summary:
- **OAuth2** - browser flow, cleanest, but needs admin to approve the Snowflake GitHub App in your GitHub org.
- **PAT** (shown above) - works without admin approval; you rotate the token yourself.
- **Public read-only** - no auth; you can `pull` but not `push`. Fine for consuming upstream only.

See [Integrate workspaces with a Git repository](https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-git).

#### 1c - create the workspace

In Snowsight: **Projects > Workspaces > + Workspace > From Git repository** > select `coco_quiz_fork` > pick the branch (`main` or your own).

The workspace mounts the full repo: `.snowflake/cortex/skills/`, `AGENTS.md`, `docs/`, `README.md`, etc. all appear in the file tree. CoCo picks up the skills as slash commands automatically (`/setup-exam`, `/adapt-questions`, and routers for `/cortex`, `/sis`, `/quiz` - see [skills.md](skills.md)).

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

- `docs/` stays on your local clone - no in-browser access; reference from local or GitHub.
- No version control inside Snowsight - skill edits live in the workspace only.
- Skill updates require re-uploading the folder.
- No branching for experiments.

### 1d - what you do NOT load, either way

- The `app/` project (`main.py`, `_*.py` modules, `pages/`, configs) - the agent generates it into the workspace on each `$setup-exam` run.
- Your study guide PDF and optional CSV/JSON - you add these to the **workspace file tree** later, when the agent asks (step 4); it stages them for you via `COPY FILES`. Not now.

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
- `<your_role>` - the role you will be using (must be active in your Snowsight session, and must have `CREATE SCHEMA` on the database above).

Leave `schema` and `exam_code` as is - `$setup-exam` will fill those in once you tell it which exam you want. `stage`, `app stage`, `app_name`, `main_file`, `runtime`, `deps_file` have working defaults - change them only if you need different names.

Save. CoCo re-reads `AGENTS.md` on the next message. If you forget to fill any required placeholder, `$setup-exam` halts in Step 1a and prompts you to finish the edit.

---

## Step 3 - run the setup prompt

Copy the **setup prompt** from [prompts.md](prompts.md) and paste it into CoCo. The prompt is intentionally short - it attaches `@AGENTS.md` (always-on context) and invokes `/setup-exam`; the skill itself carries every step and stop. You don't re-describe the procedure in the prompt.

The agent will:

1. Ask you for the exam name and exam code (e.g. "SnowPro Core" / "COF-C03").
2. Ask for the PDF filename (required) and optionally the CSV filename.
3. Ask about additional customisations (e.g. a specific extra you want, or different scoring).
4. Ask whether you want the **default look or a custom one** - custom means a short style dialog (light/dark, accent color, roundness, fonts), applied via Streamlit theming only.
5. Create the schema, stages, and tables (and a CSV file format only if you're loading a bank CSV) - SQL visible in the chat, approve or reject each step.
6. Stop and ask you to add the PDF to the workspace (it stages it via `COPY FILES`).

Do **not** try to pre-empt the agent by creating objects manually. Let it drive.

---

## Step 4 - add the study guide PDF (and optional CSV/JSON) to the workspace

When the agent says something like *"drop SnowProCoreStudyGuide.pdf into the workspace"*:

1. In the workspace file tree (the same place the `app/` project lives), add the PDF - drag-drop it, or use the file-browser **+** / upload control.
2. (If you also have a CSV/JSON question bank) add it alongside.
3. Return to the CoCo chat and reply: **"added"** (or just "done").

You never open the Stages UI; the agent stages the file for you:

```sql
COPY FILES INTO @...STAGE_QUIZ_DATA FROM @<workspace_stage>/...;
ALTER STAGE ... STAGE_QUIZ_DATA REFRESH;
LIST @...STAGE_QUIZ_DATA;
```

and confirms the file(s) are visible before extraction.

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

**No CSV/JSON** > the bank deliberately stays **empty** - the agent does NOT generate questions during setup. The app works fully on runtime AI questions. The agent explains how to seed it later: CSV upload, the Admin page **Generate batch** button, the worksheet recipe in [customization.md](customization.md) (section 6), or a scheduled task / Automation.

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

The agent reads the updated `AGENTS.md` plus all `$quiz/*` skills (screens, questions, style) and writes the decomposed multipage project into the workspace:

- `app/main.py` - entry point: `st.set_page_config`, session-state init, `st.navigation`;
- `app/_config.py`, `app/_cortex.py`, `app/_data.py`, `app/_questions.py`, `app/_ui.py`, `app/_search.py` - constants, Cortex calls, cached loaders, question engine, shared UI helpers, docs-CKE retrieval;
- `app/pages/quiz.py` + `app/pages/review.py` + `app/pages/admin.py` - the three pages;
- `app/.streamlit/config.toml` + `app/environment.yml` + `app/snowflake.yml` - app config, dependencies (Snowflake Anaconda channel), deploy descriptor.

Everything appears in the workspace file tree under `app/`.

Then it runs the **pre-deploy scan** from `$sis` across all app files. It catches things like:

- `AI_COMPLETE` prompts not dollar-quoted;
- SQL injection risks (f-string interpolation instead of bind params);
- Column-name case mismatches between SQL and pandas;
- `get_active_session()` called outside the right cache scope.

Then it runs a second, separate check: the **UX-conformance gate** from `$quiz/screens`. The scan proves the app *runs* and is *SQL-safe*; the gate proves the screens *match the UX contracts* - round size is a slider, the hint state machine and sidebar End Round are present, missed answers show full text, the Admin page has its four tabs with the editable Questions table, the config saves without crashing, and so on. (A clean scan alone can still ship a broken UI - that gap is exactly what the gate closes.)

If either the scan or the gate fails, the agent fixes it and re-runs until both are clean. Do not proceed to deploy on a failed scan or a failed gate.

---

## Step 9 - deploy

When the scan is clean, the agent deploys. **The deploy is fully scripted.**

Your workspace files already live on an internal stage, so the agent:

1. Copies `app/` from the workspace stage onto `STAGE_SIS_APP` with `COPY FILES`, preserving the `pages/` and `.streamlit/` subfolders.
2. Creates the app and verifies:

```sql
CREATE OR REPLACE STREAMLIT <your_database>.QUIZ_<CODE>.SNOWPRO_QUIZ
  FROM '@<your_database>.QUIZ_<CODE>.STAGE_SIS_APP'
  MAIN_FILE = 'main.py'
  QUERY_WAREHOUSE = <your_warehouse>;

SHOW STREAMLITS LIKE 'SNOWPRO_QUIZ' IN SCHEMA <your_database>.QUIZ_<CODE>;
```

3. After later edits, it re-copies the changed files (`COPY FILES` overwrites same-named files) and re-runs `CREATE OR REPLACE STREAMLIT`.

Packages come from the Snowflake Anaconda channel.

---

## Step 10 - open the app, complete one round

Snowsight > **Projects > Streamlit > SNOWPRO_QUIZ**.

- On **Home**: pick 5 questions, medium difficulty, any domain, "AI Generated" source (there is no explanations toggle - the explanation is on-demand). Click **Start Round**.
- On **Quiz**: wait a couple seconds for the first AI-generated question to load. Try the **💡 Hint** button *before* answering (two levels, never spoils). Answer, submit, then click **💡 AI explanation** to load it on demand (works for correct answers too); the **Next** button sits at the very bottom, under the explanation. Inside the expander, try **🔬 Deep dive** for an in-depth breakdown of the question's topic.
- Click through all 5, then **Finish Round**.
- **Summary**: score, pass/fail vs threshold, a **TO REMEMBER** expander (the missed questions, correct answer in full text), and an on-demand **Round Summary**.
- **Review** page (tabs: Learning Dashboard · Wrong Answers, Dashboard first): the **Learning Dashboard** opens by default - you should see your first session plotted; switch to **Wrong Answers** and filter by domain and date.
- **Admin** page (4 tabs: App config · Questions manager · Cortex spend · Logs): flip an App-config toggle (e.g. hints off/on) or set a per-call-group model, check the bank KPIs in Questions manager, use its filter/edit/delete table, and optionally **Generate batch (AI)** to start seeding the bank; Logs shows the review + session tables with a guarded "reset all logs".

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

If you followed the **Git integration** path in step 2, you can additionally isolate each exam on its own branch. The agent does not switch branches - you do, before running the setup prompt:

- **Workspace Git panel** (bottom bar): click the branch name > **Create new branch from `main`** > e.g. `exam/ARA-C01`. Workspace switches automatically.
- **GitHub**: create the branch on github.com, then in the workspace Git panel click **Switch branch**.

Run the setup prompt on the new branch. `AGENTS.md` edits and the `app/` generation happen on that branch. Commit when ready.

To switch back to a previous exam later: change branch in the Git panel - the matching `AGENTS.md` snapshot comes with it, so you don't even need to re-edit the schema / exam_code lines. Branch creation is manual (via the workspace Git panel or GitHub), an `exam/<code>` branch per exam.

Skip this step if you only plan one or two exams - the schema-per-exam + single-`AGENTS.md`-that-you-edit approach is simpler and already isolates runtime data.

---

## Iterating, and when something breaks

**The first deploy is the start, not the end.** A good quiz app comes from a loop - **deploy → open the app and actually use it → report what's off → the agent fixes it → redeploy** - run a few times. Reporting things to improve (a misbehaving screen, a wrong label, an awkward flow, a weak question) is a **normal, core part of the work**, not a sign something failed. To make each round reliable:

- **Report concretely:** *what* you saw, on *which* screen, and *what you expected* (a screenshot helps). Vague feedback forces the agent to guess.
- **Always ask for the pre-deploy checks before each redeploy:** end your message with *"…then re-read the app files from disk (not from memory) and re-run the `$sis` scan + the `$quiz/screens` UX gate; redeploy only if both are clean."* Both checks must ground on a **fresh read of the files** - without that ask, the agent can redeploy a change it only *thinks* it made (a from-memory "pass"). This single habit is the most important one for trustworthy iteration.
- **After a chat reload, page refresh, or a "session stopped" message:** the chat can lose its working context, or the Snowflake session can silently re-bind to a read-only connection. First tell the agent to **re-read every app file from disk and re-run the scan + gate - explicitly "do NOT use your memory"** - before it changes anything. If it reports it cannot read the files or run SQL, re-open the setup in the interactive Snowsight Workspace as your `ACCOUNTADMIN` role (configured warehouse active) and resume; never let it certify or redeploy from memory.

Paste the **fix prompt** from [prompts.md](prompts.md), describe the symptom. The agent triages:

- `AI_COMPLETE` / `AI_PARSE_DOCUMENT` errors > runs `$cortex` diagnostics;
- App crashes in Streamlit > re-runs `$sis`;
- Wrong content / shallow explanations > runs `$cortex`;
- Screen flow glitches > reads `$quiz/screens`.

After a fix - and after the `$sis` scan + UX gate pass on a fresh re-read - it re-copies the changed files onto `STAGE_SIS_APP` and re-runs `CREATE OR REPLACE STREAMLIT` to redeploy.

**Seeing errors while you test:** the app ships with `[client] showErrorDetails = "none"` in `app/.streamlit/config.toml` - viewers get a generic message, never a traceback (the production setting). While debugging your own setup, set it to `"full"` and redeploy (`COPY FILES` the `.streamlit/config.toml` + `CREATE OR REPLACE STREAMLIT`) to see the real traceback in the app; set it back to `"none"` before sharing.

See [troubleshooting.md](troubleshooting.md) for a curated list of the most common Snowsight-specific issues.

---

## What NOT to do

- Do **not** manually create schemas, stages, or tables before running `$setup-exam`. The skill expects to own the full lifecycle and will skip or collide.
- Do **not** edit the environment table placeholders to values that don't exist - the agent will try to `USE WAREHOUSE <name>` and fail with a clear error, but it wastes a cycle.
- Do **not** drop the previous exam's schema unless you genuinely want to. Each schema is self-contained and coexistence is the default.
