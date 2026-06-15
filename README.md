# Snowflake CoCo Quiz App - Snowsight edition

A browser-only asset for building a Streamlit-in-Snowflake certification quiz app with **Snowflake CoCo in Snowsight** (GA since 2026-03-09).

> **Naming:** Snowflake renamed *Cortex Code* → *Snowflake CoCo* at Summit 26 (2026-06-02). Some Snowflake doc pages and URLs still use `cortex-code`; the on-disk skills path stays `.snowflake/cortex/skills/`. Both are expected.

Load a study guide PDF, and the agent creates the schema, extracts exam domains, generates questions, builds a decomposed multipage `app/` project (container runtime), and deploys it with a live in-browser preview. All from a single chat session, zero local tooling.

> Baseline exam: **SnowPro Core COF-C03**. Swappable to any Snowflake certification by re-running `/setup-exam` with a different study guide PDF.

---

## What you get

- **A multipage Streamlit app** (decomposed `app/` project, native `st.navigation`, container runtime):
  - **Quiz** - configure round, answer single-/multi-select questions, see score + pass/fail (75% threshold) with wrong-answer cards.
  - **Review** - filterable history of every mistake + learning dashboard (score trend, per-domain error distribution, session history).
  - **Admin** - app configuration (toggles/sliders), question manager with one-click AI batch generation, bank stats, Cortex spend dashboard, maintenance tools.
- **A learning loop, not just a quiz**: Socratic hints before answering (never spoil), AI explanations + concept contrast ("how does A differ from C?") after, an AI debrief of round patterns, and a remedial round when you fail (your wrong answers, reshuffled).
- **AI-grounded content**: domains and key facts come from the official study guide PDF (`AI_PARSE_DOCUMENT`); questions are AI-generated at runtime (`AI_COMPLETE`), with an optional question bank you can seed from CSV/JSON, the Admin panel, or a scheduled recipe (resilience + speed when AI calls are unavailable).
- **Real documentation grounding** (optional, default-on when available): if you install the free **Snowflake Documentation** Cortex Knowledge Extension from Marketplace, questions and explanations are grounded in actual Snowflake docs and cite the **exact `SOURCE_URL`** (with a readable excerpt) — no more guessed search links. Falls back cleanly when absent.
- **Persistent progress** across sessions via `QUIZ_SESSION_LOG` + `QUIZ_REVIEW_LOG` - the app remembers your weak domains between logins.
- **Schema-per-exam isolation** (`QUIZ_<EXAM_CODE>`) so multiple certifications coexist in one database.
- **Custom CoCo skills** in `.snowflake/cortex/skills/` drive the whole pipeline end to end.
- **Optional add-ons**: spaced repetition, flashcards, exam-simulation mode, achievement badges, AI study recommendations (`$quiz/features`).
- **Advanced mode (opt-in)**: quality model profile (`claude-opus-4-7`), agent self-verify via Cloud Agents, scheduled maintenance via Automations (**Preview**) — all off by default ([docs/customization.md](docs/customization.md) section 5).

## What is different from the CLI version

| | CLI variant | Snowsight variant (this branch) |
|---|---|---|
| environment | terminal + `snow` CLI + bash | browser workspace, zero local tooling |
| input file uploads | `snow stage copy` | manual upload via Snowsight UI |
| app deploy | `snow streamlit deploy` | Workspaces **Run** (live dev preview) + **Deploy**; stage + `CREATE STREAMLIT` as scripted fallback |
| isolation | Git branch per exam + schema (agent-automated) | schema per exam always; optional branch per exam if workspace is Git-backed (user creates the branch manually) |
| custom skills path | `.cortex/skills/` | `.snowflake/cortex/skills/` |
| global skills | `~/.snowflake/cortex/skills/` | built into CoCo |
| app code | tracked in repo | generated `app/` project (multi-file, container runtime), not committed |
| input data (`data/`) | tracked in repo | user uploads PDF/CSV directly to stage |

---

## Prerequisites

- Snowflake account with a role that has `USAGE`+`CREATE SCHEMA` on a database, `USAGE` on a warehouse, and access to Cortex AI functions.
- For the default **container runtime**, two things (the warehouse fallback needs neither): a **compute pool** with `USAGE` for your role (`SHOW COMPUTE POOLS;`), and a **PyPI external access integration** so it can install pandas/altair (`CREATE EXTERNAL ACCESS INTEGRATION … ALLOWED_NETWORK_RULES = (snowflake.external_access.pypi_rule)`, ACCOUNTADMIN). `$setup-exam` checks both up front (Step 1f) and gives you the DDL.
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
│   └── quiz/                         -- screens, questions, style, optional features
└── docs/
    ├── instructions.md               -- concise step-by-step
    ├── instructions-detailed.md      -- detailed walk-through with Snowsight UI paths
    ├── prompts.md                    -- the prompt to paste into CoCo chat
    ├── skills.md                     -- one-paragraph description per skill
    ├── customization.md              -- model, UI, features, cross-provider exam swap
    ├── architecture.md               -- data flow, table relationships, skill graph
    └── troubleshooting.md            -- common issues and fixes
```

Outputs (the generated `app/` project - `main.py`, `_*.py` modules, `pages/`, configs) live in your workspace once the agent generates them. They are intentionally not tracked in this repo.
