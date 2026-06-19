# LetsDemo Submission: Snowflake CoCo Quiz App

Three options for every box on the "Add new demo" form. Pick one per field, mix and match, or tell me to merge two. Written in your voice, normal capitalization.

A quick note on framing: this is a software/AI asset, not a physical demonstrator. So the "Technical requirements" boxes (equipment, space, logistics) lean digital. I have given you options that handle that honestly instead of pretending there is a booth to ship.

---

## Name of demo *

**Option A (descriptive, what it is)**
Snowflake CoCo Quiz App: AI-Built Certification Trainer on Streamlit in Snowflake

**Option B (outcome-led, shorter)**
CoCo Quiz Builder: Certification Prep Apps Built by Snowflake's AI Agent

**Option C (learning angle)**
Learn Snowflake CoCo by Building It: A Self-Updating SnowPro Quiz App

---

## Short description *

**Option A (what + who it is for)**
A browser-only asset that turns any certification study guide PDF into a deployed Streamlit quiz app, built end to end by Snowflake CoCo. Used in Capgemini's Snowflake upskilling for SnowPro Core prep, and swappable to any other exam.

**Option B (value-first, tighter)**
Build a full certification trainer from a study guide PDF in one CoCo chat session, no local tooling. Already used across Capgemini upskilling, with employees passing SnowPro Core off the back of it.

**Option C (dual-purpose hook)**
Two assets in one: a hands-on way to learn Snowflake CoCo in practice, and a working quiz app you can adapt to any exam (SnowPro, AWS, Azure, GCP) by swapping the PDF.

---

## Detailed description *
*(form hint: explain how the solution addresses industry challenges, focusing on practical implementation and functionality)*

**Option A (full, functionality-rich)**
The asset is a context package for Snowflake CoCo, Snowflake's AI agent in Snowsight (renamed from Cortex Code at Summit 26). You drop a certification study guide PDF into a Snowsight workspace, paste one prompt, and CoCo runs the whole pipeline: it creates the schema and stages, parses the PDF with AI_PARSE_DOCUMENT, extracts exam domains and weights with AI_COMPLETE, builds a decomposed multipage Streamlit-in-Snowflake app, and deploys it on the warehouse runtime by copying files to a stage and running CREATE STREAMLIT. No local tooling, no manual upload, all from one chat session.

What makes it reliable instead of a one-shot gamble is the layer I built on top of CoCo: a set of custom skills and an AGENTS.md template that encode every decision, convention, and gotcha. The skills cover the setup pipeline, Cortex AI patterns, Streamlit-in-Snowflake runtime rules with a mandatory pre-deploy scan, and the quiz app's screen contracts and visual rules. That structure is what lets CoCo customize the app at every step (model, UI, exam, features) while staying inside guardrails that keep the output correct.

The app itself is a learning loop, not just a quiz. It gives Socratic hints before you answer, an on-demand AI explanation after (for correct answers too) with a deep dive into the topic, a Review page of every mistake, and a learning dashboard that tracks your weak domains across sessions against the 75% pass threshold. All AI content is grounded on real Snowflake documentation through the free Documentation Cortex Knowledge Extension, citing the exact source URL, never the model's own guesses.

**Option B (medium, emphasizes the CoCo learning value)**
This is a study guide PDF in, deployed quiz app out, built entirely by Snowflake CoCo (Snowflake's AI agent in Snowsight, formerly Cortex Code). One prompt kicks off the full pipeline: schema and stages, AI document parsing, domain extraction, a multipage Streamlit-in-Snowflake app, and deployment on the warehouse runtime, with zero local tooling.

The real depth is in how it teaches CoCo. I wrote custom CoCo skills and an AGENTS.md template that drive the build, so anyone using the asset sees in practice how skills work, what CoCo can do, where its limits are, and how to work with it well. You learn the tool by watching it build something real, then you keep customizing: change the model, restyle the UI, swap the exam, add features, all through chat.

The deployed app is a proper learning tool: Socratic hints, on-demand AI explanations with topic deep dives, a mistake review page, and a dashboard tracking weak domains across sessions. Every AI answer is grounded on real Snowflake docs via the Documentation Cortex Knowledge Extension and cites its source, so the content stays trustworthy.

**Option C (concise, scannable)**
Drop a certification study guide PDF into a Snowsight workspace, paste one prompt, and Snowflake CoCo (the AI agent in Snowsight, formerly Cortex Code) builds and deploys a full Streamlit quiz app for you: schema, AI-parsed domains, multipage app, and a CREATE STREAMLIT deploy, no local tooling.

It works because of the custom CoCo skills and AGENTS.md template I built on top. They encode the pipeline, the Cortex AI patterns, the Streamlit-in-Snowflake rules, and the app contracts, so the build is repeatable and customizable at every step instead of a one-shot guess. That same structure doubles as a hands-on lesson in how CoCo skills work and where the tool's limits are.

The app is a learning loop: Socratic hints before answering, on-demand AI explanations with topic deep dives after, a mistake review page, and a dashboard that tracks your weak domains across sessions. All AI content is grounded on the real Snowflake documentation (via the free Documentation Cortex Knowledge Extension) and cites its exact source.

---

## Benefits *
*(form hint: highlight the concrete business value and measurable improvements the solution brings)*

**Option A (full, proof-led)**
Concrete, proven results first: I built this asset, learned from it myself, passed my certification, and then donated it to the firm. Since then many colleagues have studied with it and passed their certs. SnowPro Core is a mandatory, company-paid step after Snowflake upskilling, so a faster, more effective path to passing it has direct value across every Snowflake hire.

Beyond exam prep, the asset is a practical way into Snowflake CoCo, which is exactly the kind of AI capability clients are asking about. Instead of reading about an AI agent, people use one to build and deploy something real, and come away understanding how skills work, what CoCo can and cannot do, and how to drive it well. That hands-on fluency transfers straight to client work.

It is also reusable, not single-use. The same scaffolding builds a trainer for any exam with a published study guide PDF (SnowPro Advanced, AWS, Azure, GCP, and others), and multiple certifications coexist in one database with schema-per-exam isolation. I keep it updated as CoCo ships new releases, so it tracks the latest Snowflake features rather than aging out.

**Option B (medium, business value)**
The asset shortens the path to SnowPro Core, a mandatory and company-funded certification after Snowflake upskilling. It is already in active use across Capgemini, and colleagues have passed their exams studying with it. The same tool then adapts to any other exam by swapping the study guide PDF, so one asset covers the whole certification roadmap, from SnowPro Advanced tracks to AWS, Azure, and GCP.

Just as valuable is the exposure to Snowflake CoCo. Clients have strong interest in Snowflake's AI agent, and this asset lets anyone learn it by doing, building and deploying a real app, seeing how skills work, and understanding its strengths and limits. That practical knowledge is directly reusable on client engagements.

I maintain the asset against CoCo's frequent releases, so teams are always learning on current features, not a stale snapshot.

**Option C (concise, punchy)**
Real outcomes, not projected ones: I built it, certified with it, gave it to the firm, and colleagues have since passed their exams using it. It speeds up SnowPro Core, which is mandatory and company-paid, and the same scaffolding adapts to any exam by swapping the PDF, so it covers the full certification roadmap.

It is also the easiest way to get hands-on with Snowflake CoCo, the AI agent clients keep asking about. People learn the tool by building something real with it, including how skills work and where its limits are, and that knowledge transfers to client work. I keep it current with each CoCo release.

---

## Drivers and Challenges *
*(form hint: focus on market challenges and industry pain points that led to the solution's development)*

**Option A (full)**
Two pressures drove this. First, certification is a real bottleneck in Snowflake upskilling. SnowPro Core is mandatory after the program, and generic question banks do not match the exam's grounding in actual Snowflake behavior, so people study from material that is either too shallow or quietly wrong. I wanted a trainer grounded in real Snowflake documentation, that tracks my weak domains and tells me when I am actually ready, not just a static quiz.

Second, Snowflake CoCo is one of the AI capabilities clients are most curious about, but curiosity is not skill. There was no low-stakes, hands-on way to learn how to work with it: how skills shape its behavior, what it does well, where it breaks. Reading release notes does not build that fluency.

This asset solves both at once. It is a genuinely useful study tool built on grounded AI, and the act of building it with CoCo is itself the lesson in how the agent works. Because CoCo releases new features often, a static guide would go stale fast, so I update the asset continuously to keep it aligned with the current product.

**Option B (medium)**
Snowflake upskilling ends in a mandatory SnowPro Core exam, and prep material is a weak point: generic banks do not reflect how Snowflake actually behaves, so they teach the wrong things or skim the surface. People needed a trainer grounded in real documentation that adapts to their weak areas and signals genuine readiness.

At the same time, clients are increasingly interested in Snowflake CoCo, but there was no practical, low-risk way for people to actually learn the agent: how skills steer it, what it handles well, where its limits are. This asset answers both. It is a grounded, adaptive study tool, and building it with CoCo is the hands-on lesson in using the agent. I keep it current because CoCo ships features frequently and a static asset would not keep up.

**Option C (concise)**
SnowPro Core is mandatory after Snowflake upskilling, but typical prep banks are not grounded in real Snowflake behavior, so they teach shallow or incorrect material. People needed a trainer grounded in actual documentation that tracks weak domains and shows real readiness.

Separately, clients are keen on Snowflake CoCo, yet there was no hands-on, low-stakes way to actually learn the agent: how skills shape it, what it does well, where it breaks. This asset covers both, since building the study tool with CoCo is itself the lesson. I update it continuously because CoCo releases features often.

---

## Industries *
*(dropdown, multi-select. Recommended picks below; choose whatever the platform offers closest to these)*

**Option A (broadest)**
Cross-Industry / All Industries. The asset is exam-agnostic and the skill is transferable, so it is not tied to one vertical.

**Option B (tech-led)**
Technology, Software, and Platforms (or the closest "High Tech" / "TMT" category), since it is a data-platform and AI-tooling asset.

**Option C (internal/enablement framing)**
Pick the category that maps to internal capability building or professional services, if the platform separates internal enablement assets from client-vertical ones.

---

## Group Offer Priorities *
*(dropdown. I cannot see the exact options, so these are the angles to map to whatever is listed)*

**Option A**
Data and AI (the primary fit: this is a Snowflake plus Cortex AI asset).

**Option B**
Cloud and Platforms, if Data/AI is not a distinct option and Snowflake sits under a cloud-platform offer.

**Option C**
Intelligent Industry / Generative AI, if the platform surfaces a Gen AI or AI-agent priority, given the CoCo angle.

---

## Sustainability *
*(dropdown/required. Pick the honest one; do not overclaim)*

**Option A (honest, recommended)**
Not directly applicable / No specific sustainability impact. It is a learning asset, so I would not overclaim here unless the form requires a positive selection.

**Option B (if a positive selection is mandatory)**
Indirect: runs serverless on Snowflake's shared infrastructure with no dedicated hardware, no physical materials to ship, and a browser-only footprint, which avoids the resource cost of a physical demonstrator.

**Option C (enablement angle)**
Digital, paperless training: replaces printed study material and physical lab setups with a fully digital, on-demand tool.

---

## Key point of contact *

**Option A**
You (Monika Burnejko), Capgemini Snowflake CoE. Asset owner, builder, and maintainer.

**Option B**
You as primary, with your CoE lead or manager named as secondary point of contact (see below).

**Option C**
You, with a note that the asset lives on the Capgemini Snowflake CoE GitLab and you maintain it directly.

*(I do not have your team's exact names. Tell me your CoE lead and I will slot them into the secondary contact field.)*

---

## Secondary point of contact
*(optional. Suggestions, fill in real names)*

**Option A**
Your Snowflake CoE lead or manager.

**Option B**
A colleague who has co-presented or contributed (for example a meetup co-organizer).

**Option C**
Leave blank if there is no clear second owner. Better empty than a name who cannot answer questions about it.

---

## Country / Unit(s) / Demo Location(s)
*(dropdowns. Recommendations)*

**Country:** Poland (your base, and where the meetup happened).
**Unit(s):** your Capgemini unit / the Snowflake CoE's unit. Pick the one that owns the asset.
**Demo Location(s):** Poznań (the Snowflake x Capgemini meetup location), plus any other office where upskilling runs. Add "digital / remote" if available, since it is browser-only.

---

## Upload demo image *

Three options for what to use, since the form requires an image:

**Option A (recommended)**
A clean screenshot of the deployed app: the Quiz screen mid-round, or the learning dashboard with the score trend and per-domain error chart. Shows the product immediately.

**Option B**
A screenshot of the Snowflake CoCo chat building or deploying the app, to emphasize the AI-agent angle that makes this distinctive.

**Option C**
A simple title card: app name, "Built with Snowflake CoCo", and the Snowflake plus Streamlit logos. Use this only if a screenshot is too busy at thumbnail size.

*(I can help you frame or annotate whichever screenshot you choose.)*

---

## Media
*(optional, but a strong differentiator. You have real assets here)*

**Option A (recommended, lead with the talk)**
Attach your Snowflake x Capgemini Poznań meetup presentation (you were a speaker with the Cortex Code / CoCo demo panel). It is direct evidence of the asset being presented publicly.

**Option B**
Attach the developer-focused slide deck from your Snowflake CoE newsletter, the one that walks through this asset and links the repo. It explains the asset in your own framing.

**Option C**
Attach both decks (enterprise and developer) plus a short screen recording of a CoCo build-and-deploy run, if the form allows multiple files. The recording is the most convincing single artifact for a "demo" platform.

---

## Source Code
*(dropdown/link field)*

**Option A (recommended)**
Link the Capgemini Snowflake CoE GitLab repo where the asset currently lives.

**Option B**
GitLab as primary, and mention that it was also distributed via the CoE newsletter (every reader could pull it from the linked GitHub repo).

**Option C**
If the field only accepts a yes/no or a type, select the option indicating internal source is available, then put the GitLab link in Additional info.

---

## Technical requirements

This asset is software running on Streamlit-in-Snowflake. There is nothing physical to ship, so these boxes describe the digital setup. The form marks Equipment as required and accepts "NA" where things do not apply.

### Equipments needed *

**Option A (recommended, accurate)**
A laptop with a browser and internet access. A Snowflake account with a role that has CREATE SCHEMA on a database, USAGE on a warehouse, and access to Cortex AI functions. Snowflake CoCo enabled in Snowsight, with Web search turned on under AI and ML > Agents > Settings. For doc-grounded mode, the free Snowflake Documentation listing installed from Marketplace. For a live presentation: a screen or projector and an HDMI adapter. No dedicated hardware, the app runs serverless on the warehouse runtime.

**Option B (minimal)**
Laptop, browser, internet, and a Snowflake account with Cortex AI access and CoCo enabled. For an in-room demo: a screen and adapter. NA on anything physical, it is browser-only.

**Option C (presentation-focused)**
For a digital event: a browser and a shared screen, nothing else. For a physical event: a laptop, a TV or projector with HDMI adapter, and reliable internet (the app calls Cortex AI live, so connectivity matters). No stand, no kit, no shipping.

### Space required

**Option A (recommended)**
None physical. It is a browser-based app, so it needs only screen space at a booth or a shared screen on a call. A standard demo station or a single monitor is enough.

**Option B**
NA. Software only, no physical footprint.

**Option C**
Minimal: any space that fits a laptop and a screen. Works equally in a meeting room, at a booth, or fully remote.

### Logistics involved

**Option A (recommended)**
None. Nothing ships. Access is via a Snowsight URL on a Snowflake account, so the only "logistics" is making sure the presenting account has Cortex AI, CoCo, and (for grounded mode) the Documentation extension set up ahead of time. Allow a few minutes before a live build demo to confirm the workspace and account settings.

**Option B**
NA, no physical logistics. Pre-event: confirm Snowflake account access and that CoCo plus Web search are enabled.

**Option C**
Fully digital, no shipping or setup transport. The only prep is account readiness (role grants, Cortex AI, CoCo enabled) and, if you plan a live build, having the study guide PDF on hand.

### Additional info

**Option A (recommended, credibility-rich)**
This asset is in active use in Capgemini's Snowflake upskilling for SnowPro Core prep, and colleagues have passed their certifications using it. It was presented at the Snowflake x Capgemini meetup in Poznań (Cortex Code / CoCo demo panel) and featured in the Snowflake CoE newsletter with two slide decks and a public repo link. It currently lives on the Capgemini Snowflake CoE GitLab. I maintain it continuously against CoCo's releases. It swaps to any exam with a study guide PDF (SnowPro Advanced, AWS, Azure, GCP), and I am currently using it to study for SnowPro Advanced Data Engineer.

**Option B (concise)**
In active use across Snowflake upskilling, presented at the Poznań Snowflake x Capgemini meetup, and shared via the CoE newsletter. Hosted on the CoE GitLab, maintained continuously, and adaptable to any certification by swapping the study guide PDF.

**Option C (setup-tip angle)**
For the best live demo, pre-enable CoCo and Web search and install the Snowflake Documentation extension, then either show the deployed app or do a live build from a PDF. The build-from-PDF run is the most compelling, since it shows the AI agent doing the work end to end. Asset is on the CoE GitLab and kept current with CoCo releases.

---

## What I would pick (my quick take)

If you want a coherent, standout submission, I would go: Name **A**, Short description **C** (the dual-purpose hook is your real differentiator), Detailed description **B**, Benefits **A** (lead with the proof that people certified with it), Drivers **A**, and lean hard on the Media section, your meetup talk and newsletter decks are what most submissions will not have. Want me to assemble those into a single clean fill-in sheet?
