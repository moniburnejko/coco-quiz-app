# Snowflake CoCo Quiz App - Snowsight edition

A browser-only asset for building a Streamlit-in-Snowflake certification quiz app with **Snowflake CoCo in Snowsight** (GA since 2026-03-09).

> **Naming:** Snowflake renamed *Cortex Code* → *Snowflake CoCo* at Summit 26 (2026-06-02). Some Snowflake doc pages and URLs still use `cortex-code`; the on-disk skills path stays `.snowflake/cortex/skills/`. Both are expected.

Load a study guide PDF, and the agent creates the schema, extracts exam domains, generates questions, builds a decomposed multipage `app/` project, and deploys it on the warehouse runtime - copying the files to a stage and running `CREATE STREAMLIT` for you. All from a single chat session, zero local tooling.

> Baseline exam: **SnowPro Core COF-C03**. Swappable to any Snowflake certification by re-running `/setup-exam` with a different study guide PDF.

---

## What you get

- **A multipage Streamlit app** (decomposed `app/` project, native `st.navigation`):
  - **Quiz** - configure round, answer single-/multi-select questions, see score + pass/fail (75% threshold) with wrong-answer cards.
  - **Review** - filterable history of every mistake + learning dashboard (score trend, per-domain error distribution, session history).
  - **Admin** (4 tabs) - app config (feature toggles + AI model per call-group), Questions manager (bank KPIs + a filterable/editable/deletable table + one-click AI batch generation), Cortex spend dashboard, and Logs.
- **A learning loop, not just a quiz**: Socratic hints before answering (never spoil), an on-demand AI explanation after (for correct answers too) with a 🔬 deep dive into the question's topic, and an AI debrief of round patterns.
- **AI-grounded content**: domains and key facts come from the official study guide PDF (`AI_PARSE_DOCUMENT`); questions are AI-generated at runtime (`AI_COMPLETE`), with an optional question bank you can seed from CSV/JSON, the Admin panel, or a scheduled recipe (resilience + speed when AI calls are unavailable).
- **Real documentation grounding** (required for Snowflake exams): questions and explanations are generated ONLY from real Snowflake docs - the free **Snowflake Documentation** Cortex Knowledge Extension (install from Marketplace), citing the **exact `SOURCE_URL`** with a readable excerpt, never the model's built-in knowledge. The grounding mode is fixed once at setup (`cke` by default); `$setup-exam` treats the CKE as a hard gate and stops until it's reachable. Only a non-Snowflake exam can opt into the ungrounded `none` mode.
- **Persistent progress** across sessions via five tables (`EXAM_DOMAINS`, `QUIZ_QUESTIONS`, `QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`, `QUIZ_CONFIG`) - the app remembers your weak domains between logins.
- **Schema-per-exam isolation** (`QUIZ_<EXAM_CODE>`) so multiple certifications coexist in one database.
- **Custom CoCo skills** in `.snowflake/cortex/skills/` drive the whole pipeline end to end.
- **Tested features coming soon**: a CoCo skill with verified, ready-to-enable features that extend the quiz app is on the way. For now the asset ships in its core form (Quiz · Review · Admin).
- **Advanced mode (opt-in)**: quality model profile (`claude-opus-4-7`), agent self-verify via Cloud Agents, scheduled maintenance via Automations (**Preview**) - all off by default ([docs/customization.md](docs/customization.md) section 5).

## What is different from the CLI version

| | CLI variant | Snowsight variant (this branch) |
|---|---|---|
| environment | terminal + `snow` CLI + bash | browser workspace, zero local tooling |
| input file uploads | `snow stage copy` | user drops PDF/CSV into the workspace; agent stages them with `COPY FILES` onto `STAGE_QUIZ_DATA` (no manual upload) |
| app deploy | `snow streamlit deploy` | agent copies `app/` to a stage with `COPY FILES` + runs `CREATE STREAMLIT` |
| isolation | Git branch per exam + schema (agent-automated) | schema per exam always; optional branch per exam if workspace is Git-backed (user creates the branch manually) |
| custom skills path | `.cortex/skills/` | `.snowflake/cortex/skills/` |
| global skills | `~/.snowflake/cortex/skills/` | built into CoCo |
| app code | tracked in repo | generated `app/` project (multi-file), not committed |
| input data (`data/`) | tracked in repo | user drops PDF/CSV into the workspace file tree; agent copies them to the stage via `COPY FILES` |

---

## Prerequisites

- Snowflake account with a role that has `USAGE`+`CREATE SCHEMA` on a database, `USAGE` on a warehouse, and access to Cortex AI functions.
- `pandas`/`altair` install from the Snowflake Anaconda channel, so the app **works on trial accounts**.
- One-time as `ACCOUNTADMIN`, only if the model is not reachable in-region: `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = 'ANY_REGION';` (accounts created after 2026-03-09 default to it).
- Snowsight > AI & ML > Agents > Settings > Tools and connectors > Web search > enable.
- A study guide PDF for your target exam - [Snowflake certifications catalog](https://learn.snowflake.com/en/certifications/).

---

## Disclaimer

> AI-generated questions and explanations may be inaccurate - LLMs hallucinate, especially on niche Snowflake features. Verify against official [Snowflake docs](https://docs.snowflake.com/) when in doubt.
>
> When AI question generation or AI explanations are enabled, the app runs slower a little bit: each question is a live `AI_COMPLETE` round-trip. You can switch to a faster model (see `docs/customization.md` if present, or edit `cortex llm` in `AGENTS.md`) at the cost of quality and more hallucinations.

---

## Getting started

**First time in Snowsight or with CoCo?** → [docs/instructions-detailed.md](docs/instructions-detailed.md)

**Already comfortable?** → [docs/instructions.md](docs/instructions.md)

Then paste the prompt from [docs/prompts.md](docs/prompts.md) into the CoCo chat.

**Working with CoCo is iterative, not one-shot.** The first deploy is the start: use the app, report what's off, let the agent fix it, redeploy - a few rounds is normal and expected. The one habit that keeps it reliable: **always ask for the `$sis` pre-deploy scan + `$quiz/screens` UX gate (re-read from disk, not from memory) before every redeploy.** See [docs/instructions.md](docs/instructions.md) "Iterating" and [docs/troubleshooting.md](docs/troubleshooting.md).

### Two ways to load this asset into a workspace

- **Via Git integration** (recommended): fork this repo on GitHub, register it in Snowflake (`CREATE API INTEGRATION` + `CREATE GIT REPOSITORY`), then **Projects > Workspaces > + Workspace > From Git repository**. Workspace opens with the full tree in place - skills, `AGENTS.md`, and `docs/` all accessible from the Snowsight file browser. You also get version control: edits to skills / `AGENTS.md` / the generated `app/` project can be committed back to your fork from inside Snowsight.
- **Via manual upload** (fallback, fully offline-capable): clone the repo locally, then in Snowsight create an empty workspace and drag-drop `.snowflake/cortex/skills/` and `AGENTS.md` into it via the CoCo UI. `docs/` stays on your local clone - no in-browser access.

See [docs/instructions-detailed.md](docs/instructions-detailed.md) step 2 for the DDL snippets and walk-through. Canonical Snowflake docs: [Integrate workspaces with a Git repository](https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-git) and [Using a Git repository in Snowflake](https://docs.snowflake.com/en/developer-guide/git/git-overview).

---

## Repo structure

```
coco-quiz-app
├── README.md                         
├── AGENTS.md                         -- CoCo context (read, but do not restructure)
├── .snowflake/cortex/skills/         -- custom project skills (upload to workspace)
│   ├── setup-exam                    -- 10-step end-to-end pipeline
│   ├── adapt-questions               -- optional CSV/JSON question-bank import
│   ├── cortex/                       -- Cortex AI patterns + prompt audit
│   ├── sis/                          -- Streamlit-in-Snowflake patterns + pre-deploy scan
│   └── quiz/                         -- screens, questions, style
└── docs/
    ├── instructions.md               -- concise step-by-step
    ├── instructions-detailed.md      -- detailed walk-through with Snowsight UI paths
    ├── prompts.md                    -- the prompt to paste into CoCo chat
    ├── skills.md                     -- one-paragraph description per skill
    ├── customization.md              -- model, UI, cross-provider exam swap
    ├── architecture.md               -- data flow, table relationships, skill graph
    └── troubleshooting.md            -- common issues and fixes
```

Outputs (the generated `app/` project - `main.py`, `_*.py` modules, `pages/`, configs) live in your workspace once the agent generates them. They are intentionally not tracked in this repo.
