# Coco Quiz App — Refresh & Refactor Plan

**Date:** 2026-06-09 · **Trigger:** Snowflake Summit 26 (2026-06-02) renamed Cortex Code → **Snowflake CoCo** and shipped a wave of changes that make parts of this asset stale, redundant, or working around problems that no longer exist.

**Scope of this document:** analysis + plan only. No code/skill files changed yet.

---

## 0. Executive summary

This asset is the **Snowsight edition** of a CoCo-driven generator that turns a certification study-guide PDF into a deployed Streamlit-in-Snowflake quiz app (baseline: SnowPro Core COF-C03). It was built against **"Cortex Code in Snowsight" (GA 2026-03-09)**, the **warehouse** Streamlit runtime (**SiS 1.52.\***), a **manual stage-upload deploy** flow, and **hand-rolled JSON-parsing + skill-router** conventions.

Since then, four things changed that matter a lot:

1. **Rename.** "Cortex Code" → **Snowflake CoCo** (official 2026-06-02). Every "Cortex Code" reference in this repo is now off-brand. (Docs/URLs still lag — many doc pages still read `cortex-code`.)
2. **Streamlit got a second runtime + Workspaces.** The **container runtime** (GA 2026-03-09, Streamlit ≥1.50) plus **Streamlit-in-Workspaces** (Public Preview: live in-browser preview + one-click **Deploy**, no stage upload) **dissolve ~4 of the 6 SiS limitations this asset hard-codes** and **replace the manual deploy dance**.
3. **AI_COMPLETE got structured outputs (GA).** `response_format` guarantees schema-valid JSON — which **obsoletes the markdown-fence-stripping machinery** in `$cortex/patterns` and `parse_cortex_json`.
4. **CoCo skills are an official, documented system now — and it does NOT work the way this asset assumes.** The `parent_skill:` router frontmatter this repo uses is **not a real CoCo feature** (inert). Snowflake also now **ships bundled skills** (`developing-with-streamlit`, `cortex-ai-functions`, `deploy-to-spcs`, `skill-development`) that **overlap with `$sis/*` and `$cortex/*`**.

Net: the asset still *works*, but it's defending against a 2026-Q1 world. A focused refactor can **delete a large amount of workaround code, modernize the deploy story, fix the skill architecture, and re-brand** — while keeping the genuinely good parts (the `$quiz/*` playbooks, the 4-table model, the setup pipeline shape).

Separately, the audit found **internal inconsistencies that exist regardless of the ecosystem changes** (see §3.6) — worth fixing in the same pass.

---

## 1. What the asset is today (grounding)

- **Type:** a context package (no app source in-repo) = `AGENTS.md` + 13 `SKILL.md` files under `.snowflake/cortex/skills/` + `docs/`. CoCo reads these and generates `quiz.py` + `environment.yml` into the workspace, then deploys.
- **Skills (13 files):** 2 standalone (`setup-exam`, `adapt-questions`); 3 "router" parents (`cortex`, `sis`, `quiz`) each with sub-skills (`cortex/{patterns,prompt-audit}`, `sis/{patterns,pre-deploy}`, `quiz/{screens,questions,style,features}`).
- **Pipeline (`$setup-exam`):** PDF→stage → `AI_PARSE_DOCUMENT(LAYOUT)` → `AI_COMPLETE` extracts domains/weights/topics/key_facts → load or AI-generate questions → write `quiz.py` → **22-item pre-deploy scan** → manual upload to `STAGE_SIS_APP` → `CREATE STREAMLIT`.
- **Runtime app:** 4 tables (`EXAM_DOMAINS`, `QUIZ_QUESTIONS`, `QUIZ_REVIEW_LOG`, `QUIZ_SESSION_LOG`); Quiz/Review pages; live `AI_COMPLETE` question + explanation generation; learning dashboard; 6 optional features.
- **Hard-coded platform anchors:** model `claude-sonnet-4-6`; **SiS v1.52.\***; manual stage deploy; markdown-fence JSON parsing; `st.rerun()` budget; "no `st.fragment` / `st.connection` / `st.container(horizontal=True)`"; `environment.yml`.

The asset is **well-written and thoughtful** — the docs are genuinely good, the security/isolation posture (schema-per-exam, parameterized SQL, no-DROP) is sound, and the `$quiz/*` UX contracts are detailed. The refactor is about *platform drift*, not poor quality.

---

## 2. What changed in the ecosystem (researched 2026-06-09, cited)

> Tags: **[GA]** generally available · **[Preview]** public/private preview · **[Announced]** stated, ship date unclear. Where docs and marketing disagree, both are shown.

### 2.1 CoCo rebrand & the Snowsight agent

- **Cortex Code → Snowflake CoCo**, official at Summit 26. "Snowflake CoCo (formerly known as Cortex Code)." Family-wide brand: **CoCo in Snowsight / Desktop / CLI**, **VS Code & Claude Code extensions**, Excel, mobile/Slack (coming), **Agent SDK + MCP server**. Sibling rebrand: Snowflake Intelligence → **CoWork**. Underlying architecture unchanged. Docs/URLs still say `cortex-code` in many places.
  - Sources: [product/snowflake-coco](https://www.snowflake.com/en/product/snowflake-coco/) · [blog 2026-06-02](https://www.snowflake.com/en/blog/snowflake-coco-ai-coding-agent-modern-data-stack/) · [press release 2026-06-02](https://www.snowflake.com/en/news/press-releases/snowflake-coco-redefines-enterprise-ai-development-as-the-coding-agent-built-for-faster-easier-and-more-powerful-innovation-anywhere/)
- **Cloud Agents** — the Snowsight agent now spins up an isolated Snowflake-managed container per session; can **run shell/Python, install packages, read/write files, run dbt**. **Status ambiguous: blog implies [GA], press release says "GA soon."** Verify in your account/region.
- **Automations** [Preview] — autonomous recurring/event-driven runs (background monitoring/validation).
- **Web search** tool [GA] — unchanged toggle: ACCOUNTADMIN → AI & ML > Agents > Settings.
- **MCP** first-class (server + SDK + Async API). **Skill Catalog** [Preview] — share/reuse skills.
- **Models (agent picker):** default **Claude Opus** — docs say `claude-opus-4-6` "Recommended"; marketing cites Opus 4.7 / Sonnet 4.7 / GPT 5.4; CLI adds an `auto` router. (Docs mid-migration; treat exact version as soft.)
- **Pricing:** token-based; **$40 free credits / 30 days, then $20/mo** CoCo tier; AI consumption billed in **"AI Credits"** (separate currency since 2026-04-01); standard compute/storage extra. Budget-trackable via the `CORTEX_CODE_SNOWSIGHT` workload.
- **Not confirmed (do not assume):** in-panel Streamlit run/preview *from the CoCo chat*; multi-agent "teams"; image input to the Snowsight agent.

### 2.2 Streamlit-in-Snowflake & Workspaces — the biggest structural change

- **Two runtimes now exist:**
  | Runtime | Streamlit | Python | Notes |
  |---|---|---|---|
  | **Warehouse** (classic) | capped at **1.52.2** | 3.9–3.11 | this asset's current target |
  | **Container** (SPCS) | **1.50+ / any, incl. nightly** | 3.11 | **[GA 2026-03-09]**, compute pool, no idle sleep |
- **Streamlit-in-Workspaces** [Preview] — create app in a workspace (`+ Add new » Streamlit app`); scaffolds `streamlit_app.py` + **`pyproject.toml`** + **`snowflake.yml`** + `.streamlit/config.toml`; **live in-browser preview without a stage** (Run / Cmd+Enter → private dev app); **one-click Deploy button** (the old right-click is now real & documented). Requires a compute pool → runs on the **container runtime**.
  - Sources: [streamlit-in-workspaces overview](https://docs.snowflake.com/en/developer-guide/streamlit/streamlit-in-workspaces/streamlit-in-workspaces-overview) · [create-run (Preview banner)](https://docs.snowflake.com/en/developer-guide/streamlit/streamlit-in-workspaces/streamlit-in-workspaces-create-run)
- **Six baseline SiS limitations — current status** (container runtime unless noted):
  | Limitation this asset hard-codes | Status now | Why |
  |---|---|---|
  | `st.fragment` / `@st.fragment` unsupported | **Resolved** | OSS since 1.37; not on SiS limitations page |
  | `st.connection("snowflake")` unsupported | **Resolved** (container) | works; `snowflake-callers-rights` added in 1.53 |
  | `st.container(horizontal=True)` unsupported | **Resolved** | OSS 1.48 |
  | `st.rerun()` "budget" / buggy | **Effectively resolved** | no SiS-specific restriction documented now |
  | `.applymap` removed | **Still avoid** | now a **pandas 3.0** removal — use `.map` |
  | `unsafe_allow_html` narrow | **Unchanged** | platform **CSP** (no external `<script>`, `eval`, external iframes) — runtime-independent |
  - Source: current [SiS limitations page](https://docs.snowflake.com/en/developer-guide/streamlit/limitations) lists none of the first four.
- **Deploy paths (all valid):** (1) **Workspace one-click Deploy** [Preview, new recommended interactive]; (2) **stage + `CREATE STREAMLIT`** [GA, still the underlying primitive — now with `RUNTIME_NAME='SYSTEM$ST_CONTAINER_RUNTIME_PY3_11'` + `COMPUTE_POOL`]; (3) **Git-backed via Workspaces** [GA].
- **Dependencies:** warehouse = `environment.yml` (Anaconda); **container = `pyproject.toml`** (PyPI, e.g. `streamlit[snowflake]==1.56.0`).
- **Workspaces** [GA 2025-09-11]; **legacy Worksheets removed 2026-06-22**; **Git integration [GA]** (branch/commit/push/pull + visual conflict diff).

### 2.3 Cortex AISQL functions (runtime + setup)

- **`AI_COMPLETE`** is current; model-as-string still correct (`AI_COMPLETE('claude-sonnet-4-6', $$…$$)`). `SNOWFLAKE.CORTEX.COMPLETE` is the legacy predecessor.
- **Structured outputs [GA]** — `response_format` (JSON schema **or** `TYPE OBJECT(...)`); output is **token-verified** against the schema. → **the markdown-fence/double-encode handling is no longer needed.**
  - Source: [AI_COMPLETE structured outputs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/complete-structured-outputs) · [AI_COMPLETE ref](https://docs.snowflake.com/en/sql-reference/functions/ai_complete-single-string)
- **Models for AI_COMPLETE (the SQL function):** `claude-sonnet-4-6` **still valid** (no `claude-sonnet-4-7` exists in Cortex). Newer: `claude-opus-4-6`, `-4-7`, **`-4-8` [Preview, 2026-05-28]**; `claude-haiku-4-5`. Authoritative list: [models & regional availability](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-regional-availability). **No urgency to change the default; opus-4-x is an optional quality bump.**
- **`AI_PARSE_DOCUMENT`** — same stage reqs (**`SNOWFLAKE_SSE` + `DIRECTORY`**, unchanged); modes OCR/LAYOUT; new options `page_split` / `page_filter` / `extract_images`; **page limit raised to 2,000 (2026-04-30)**.
- **`AI_EXTRACT`** — better-fit for the domain/weight/topic extraction step (named-field map or schema; can take the file directly). [ref](https://docs.snowflake.com/en/sql-reference/functions/ai_extract)
- **Cross-region:** still `ALTER ACCOUNT SET CORTEX_ENABLED_CROSS_REGION = …`; **prefer `ANY_REGION` or `AWS_GLOBAL`** over `AWS_US` (still valid but narrower); **new accounts after 2026-03-09 default to `ANY_REGION`** (may need no manual ALTER). [cross-region inference](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cross-region-inference)
- **Pricing:** AISQL billed in **AI Credits** (since 2026-04-01), generally cheaper; same call pattern.

### 2.4 CoCo skills & AGENTS.md (the asset's own mechanism)

- **Official `SKILL.md` frontmatter = `name`, `description`, optional `tools`.** Invocation: **auto-load by `description`** or explicit `$name` / `/skill` (Snowsight `/` menu). The asset's `.snowflake/cortex/skills/<name>/SKILL.md` path is valid for Snowsight workspaces (project scope is `.cortex/skills/`; `.claude/skills/` also accepted).
  - Source: [Cortex Code extensibility](https://docs.snowflake.com/en/user-guide/cortex-code/extensibility) · [bundled skills](https://docs.snowflake.com/en/user-guide/cortex-code/bundled-skills)
- **`parent_skill:` is NOT an official field — it is inert.** Verified across 6 official surfaces. "Router skills" are real but route via **markdown-body prose** ("if intent X, load sub-skill Y") + discovery scripts, **never via frontmatter**. Snowflake's own `developing-with-streamlit`, `sharing`, `mlops` routers prove this. **Consequence:** CoCo ignores `parent_skill:` and surfaces each `SKILL.md` independently by its own `description` — so the asset's nested sub-skills can **mis-trigger**, and the intended parent→child wiring **does nothing** unless re-expressed as body prose.
- **Snowflake now ships bundled skills that overlap this asset:** `developing-with-streamlit` (17 sub-skills — Snowflake's official Streamlit playbook), `cortex-ai-functions` (Cortex AI functions reference), `deploy-to-spcs` / `snowflake-apps` (deploy), `skill-development` (author/audit skills). → `$sis/*` and `$cortex/*` largely **reimplement** these.
- **AGENTS.md** — open cross-agent format, workspace root, always-on ambient context, supported in CLI + Snowsight. No documented size/precedence rules. **Used correctly here.**
- **Official Streamlit-skills guide** ([build-streamlit-apps-with-agent-skills, 2026-03-24](https://www.snowflake.com/en/developers/guides/build-streamlit-apps-with-agent-skills/)) stops at local `streamlit run` — it generates correct *code* but **does not deploy to SiS**; deploy is a separate skill. (Same shape as this asset: good generation, deploy is its own concern.)

---

## 3. Gap analysis — asset component → what changed

| Asset component | Status | Driver |
|---|---|---|
| **All "Cortex Code" branding** (README, AGENTS.md, every doc/skill) | **Off-brand** | §2.1 rename → CoCo |
| **GA date "2026-03-09" / "Cortex Code in Snowsight"** framing | Stale label | §2.1 |
| **`$sis/pre-deploy` items 11, 12, 14 (st.fragment / container-horizontal / st.connection)** | **Obsolete on container runtime** | §2.2 |
| **`$sis/patterns` rerun budget + "buggy outside N sites"** | **Obsolete** (and internally contradictory — see §3.6) | §2.2 |
| **`environment.yml` (`streamlit=1.52.*`)** | **Wrong file for container runtime** | §2.2 → `pyproject.toml` + `snowflake.yml` |
| **SiS "v1.52.\*" pinned everywhere** | Warehouse-only ceiling | §2.2 |
| **Manual "upload to STAGE_SIS_APP → CREATE STREAMLIT" deploy** (setup-exam Step 9; instructions; README) | **Superseded** by Workspaces live-preview + Deploy | §2.2 |
| **`$cortex/patterns` fence-stripping + `parse_cortex_json` 4-case decoder** + the matching prompt-audit items | **Obsolete** | §2.3 structured outputs |
| **`$cortex/patterns` diagnostics & general AI_COMPLETE calls** | Still correct (model-as-string) | §2.3 — only the *parsing* changes |
| **`claude-sonnet-4-6` default** | **Still valid** — keep; opus-4-x optional | §2.3 |
| **`AI_PARSE_DOCUMENT(LAYOUT)` + SSE/DIRECTORY stage** | **Unchanged — keep** | §2.3 |
| **Domain extraction via parse→AI_COMPLETE** | Works; `AI_EXTRACT` is a cleaner option | §2.3 |
| **EU cross-region `= 'AWS_US'`** | Works; **prefer `ANY_REGION`/`AWS_GLOBAL`**; new accounts may not need it | §2.3 |
| **`parent_skill:` router architecture (3 parents + sub-skills)** | **Inert / mis-triggering** | §2.4 |
| **`$sis/*` and `$cortex/*` custom skills** | **Duplicate bundled skills** | §2.4 |
| **`$quiz/*`, `$setup-exam`, `$adapt-questions`** | **Genuinely custom — keep** (re-home routing) | §2.4 |
| **4-table model, schema-per-exam, parameterized SQL, no-DROP** | **Sound — keep** | — |

### 3.6 Internal inconsistencies found (independent of ecosystem changes)

These are defects in the asset *as written* and should be fixed in the same pass:

1. **Rerun-budget contradiction.** `architecture.md` + `instructions*.md` say **"exactly 6 `st.rerun()`"**; `$sis/patterns` + `$sis/SKILL.md` say **"exactly 10 locations"**; `$sis/pre-deploy` item 9 defers to `$sis/patterns` (10). A generator can't enforce a count its own docs dispute. (The container-runtime change lets you *drop* the rigid count entirely — but the contradiction should be resolved either way.)
2. **`domain_id` type drift.** DDL creates `EXAM_DOMAINS.domain_id VARCHAR` (and `QUIZ_QUESTIONS.domain_id VARCHAR`), but the `architecture.md` ER diagram + the `$setup-exam` 5b extraction prompt call it **integer**. Works (VARCHAR accepts "1"), but the ER diagram is wrong and the prompt mis-describes the column.
3. **"9-step" vs "10-step."** `setup-exam` frontmatter says **"9-step"**; its body has **Steps 1–10** and every doc says "10-step."
4. **Invalid dollar-quoting in `$cortex/patterns` diagnostics.** The 5-step diagnostic SQL uses single `$…$` (e.g. `$Tell me…$`), not `$$…$$` — i.e. the diagnostics violate the dollar-quoting rule the skill itself mandates.
5. **`$adapt-questions` step reference.** Says "Invoked from `$setup-exam` Step 1"; it's actually invoked at **Step 6**.
6. **Skill count wording.** README says "All 7 skills"; `skills.md` says "13 skill files." (Different things — invocable vs files — but reads as a mismatch.)

---

## 4. Action plan (prioritized, phased)

> Effort = rough relative size. Each phase is independently shippable.

### Phase 0 — Rebrand & correctness *(low effort, high signal, no behavior change)*
- **P0.1** Global **"Cortex Code" → "Snowflake CoCo"** across README, AGENTS.md, all `docs/`, all `SKILL.md`. Keep one note that doc URLs may still read `cortex-code`. *(effort: S)*
- **P0.2** Fix the six internal inconsistencies in §3.6 (rerun count wording, `domain_id` type, 9-vs-10, single-`$` diagnostics, adapt-questions step ref, skill-count wording). *(S)*
- **P0.3** Refresh dated facts: GA framing, model list note (sonnet-4-6 still default; opus-4-6/4-7/4-8 available), cross-region guidance (`ANY_REGION`/`AWS_GLOBAL`, new-account default). *(S)*

### Phase 1 — Deploy & runtime modernization *(the big structural win)*
- **P1.1** Make **Streamlit-in-Workspaces** the **primary** documented flow: live preview (Run/Cmd+Enter) + one-click **Deploy**; demote manual stage-upload to a **fallback** (still valid, and useful while Workspaces-SiS is Preview). Rewrite `$setup-exam` Step 9 + `instructions*.md` + README accordingly. *(M)*
- **P1.2** Target the **container runtime**: emit **`pyproject.toml` + `snowflake.yml`** instead of `environment.yml`; set `RUNTIME_NAME='SYSTEM$ST_CONTAINER_RUNTIME_PY3_11'` + `COMPUTE_POOL` in the stage-deploy fallback; document the compute-pool prerequisite. Un-pin "1.52.\*" → "≥1.50 (container)". *(M)*
- **P1.3** Decide the dependency/runtime default and thread it through AGENTS.md + skills consistently. *(S, gated on P1.2)*

### Phase 2 — Delete the workarounds *(simplify on the new platform)*
- **P2.1** **Slim `$sis/pre-deploy`** from 22 items to what still bites: keep SQL-injection/bind-params, `unsafe_allow_html`/CSP, `.applymap`→`.map`, structured-output usage, column-name casing, `set_page_config` first; **drop/relax** the st.fragment / st.connection / container-horizontal / rerun-count items (or gate them behind "if targeting warehouse runtime"). *(M)*
- **P2.2** **Adopt AI_COMPLETE structured outputs** (`response_format`): rewrite `$cortex/patterns` (remove fence-stripping / 4-case decoder), simplify `parse_cortex_json`, and update the question-gen + explanation prompts and `$cortex/prompt-audit` (the "no markdown fences" items become moot). Big quality + reliability win at runtime. *(M–L)*
- **P2.3** *(optional)* Use **`AI_EXTRACT`** for domain/weight/topic extraction in `$setup-exam` Step 5. *(M)*

### Phase 3 — Skill architecture realignment *(make it a real CoCo skill pack)*
- **P3.1** **Fix routing:** move parent→sub-skill dispatch into the **parent `SKILL.md` body** (Snowflake's actual router pattern) OR flatten to standalone skills; **delete the inert `parent_skill:` frontmatter**; tighten every `description` with specific triggers + "Do NOT use for…" to stop mis-triggering. *(M)*
- **P3.2** **Rebase on bundled skills:** lean on `developing-with-streamlit` + `cortex-ai-functions` (+ `deploy-to-spcs`/`snowflake-apps`) instead of re-implementing `$sis/*` and most of `$cortex/*`; keep only genuinely custom playbooks (`$quiz/*`, `$setup-exam`, `$adapt-questions`). Audit with the bundled `skill-development` skill. *(M–L)*
- **P3.3** **Frontmatter hygiene** → `name` + `description` (+ `tools`); optionally adopt the catalog-contribution fields (`title`, `summary`, `type`, `prompt`, `LICENSE`) for **Skill Catalog** publishability. *(S)*

### Phase 4 — Optional enhancements *(opportunistic)*
- Use **Cloud Agents** (agent can now run code) to have CoCo *run/preview the app itself* during setup, if/when available in-account.
- **Automations** for unattended periodic regeneration/validation of a deployed quiz.
- **`opus-4-x`** as an optional higher-quality model profile; **`extract_images`** for diagram-heavy guides.
- Re-sync the **CLI variant** of this asset (the README references one) so both stay consistent.

---

## 5. Open decisions (need your call before implementation)

1. **Target runtime:** container-first (modern, unlocks everything, but Workspaces-SiS is Preview + needs a compute pool) vs keep warehouse-1.52 as default with container as opt-in? *(Recommend: container-first, warehouse fallback documented.)*
2. **Custom vs bundled skills:** how aggressively to delete `$sis/*` / `$cortex/*` in favor of bundled skills? *(Recommend: keep thin custom wrappers that call into bundled skills; delete the duplicated bulk.)*
3. **Scope of this pass:** Phase 0 now (fast, safe), or 0→3 as one larger refactor? Branch strategy?
4. **CLI + Snowsight variants:** refactor both, or Snowsight edition only for now?
5. **Report language/sharing:** this doc is in English to match the asset + CoE sharing — want a Polish version too?

---

## 6. Key sources

CoCo: [product](https://www.snowflake.com/en/product/snowflake-coco/) · [press 2026-06-02](https://www.snowflake.com/en/news/press-releases/snowflake-coco-redefines-enterprise-ai-development-as-the-coding-agent-built-for-faster-easier-and-more-powerful-innovation-anywhere/) · [extensibility](https://docs.snowflake.com/en/user-guide/cortex-code/extensibility) · [bundled skills](https://docs.snowflake.com/en/user-guide/cortex-code/bundled-skills)
Streamlit/Workspaces: [SiS limitations](https://docs.snowflake.com/en/developer-guide/streamlit/limitations) · [runtime environments](https://docs.snowflake.com/en/developer-guide/streamlit/app-development/runtime-environments) · [Streamlit-in-Workspaces](https://docs.snowflake.com/en/developer-guide/streamlit/streamlit-in-workspaces/streamlit-in-workspaces-overview) · [Workspaces Git](https://docs.snowflake.com/en/user-guide/ui-snowsight/workspaces-git)
AISQL: [AI_COMPLETE](https://docs.snowflake.com/en/sql-reference/functions/ai_complete-single-string) · [structured outputs](https://docs.snowflake.com/en/user-guide/snowflake-cortex/complete-structured-outputs) · [models & regions](https://docs.snowflake.com/en/user-guide/snowflake-cortex/aisql-regional-availability) · [AI_PARSE_DOCUMENT](https://docs.snowflake.com/en/sql-reference/functions/ai_parse_document) · [AI_EXTRACT](https://docs.snowflake.com/en/sql-reference/functions/ai_extract) · [cross-region](https://docs.snowflake.com/en/user-guide/snowflake-cortex/cross-region-inference)
Streamlit-via-skills: [official guide 2026-03-24](https://www.snowflake.com/en/developers/guides/build-streamlit-apps-with-agent-skills/) · [streamlit/agent-skills](https://github.com/streamlit/agent-skills)

*Confidence notes: Cloud Agents GA status (blog vs press release) and exact agent-model defaults (docs vs marketing) are genuinely ambiguous in Snowflake's own sources as of 2026-06-09 — verify in-account. The "container runtime resolves limitations X/Y/Z" conclusion is inferred from OSS version-introduction dates + their removal from the SiS limitations page (solid but indirect). Streamlit-in-Workspaces is Public Preview, not GA.*
