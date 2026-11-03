# Sky Custom Reports: Handover Guide

> **Sources:** `handover-en.pdf` (Draft v1, written by Sergei Privalov, Backend Engineer and project owner, when he left) plus running notes from the client call in `notes.md`.
> **Audience:** the next owner of the `sky-custom-reports` service. It assumes you do **not** already know the tech stack. Read §2 (Client Call) to see *what the client cares about*, and §4 (Tech Stack) and §16 (Vocabulary) if names like *Genkit*, *BullMQ* or *Prisma* are new to you.
> **Note:** some parts are *my reading* of the system rather than statements from the handover. They are marked as such.

---

## Table of Contents

1. [TL;DR: What you are inheriting](#1-tldr-what-you-are-inheriting)
2. [Client Call Summary (plain English)](#2-client-call-summary-plain-english)
3. [People and Roles](#3-people-and-roles)
4. [Tech Stack: what each piece is and why it is here](#4-tech-stack-what-each-piece-is-and-why-it-is-here)
5. [System Overview](#5-system-overview)
6. [Queues and Workers](#6-queues-and-workers)
7. [Section Generation Pipeline](#7-section-generation-pipeline)
8. [LLM Layer: Genkit, Models and Prompts](#8-llm-layer-genkit-models-and-prompts)
9. [Retries and Idempotency](#9-retries-and-idempotency)
10. [Redis as a Data Bus](#10-redis-as-a-data-bus)
11. [Infrastructure and Deployment](#11-infrastructure-and-deployment)
12. [Observability: metrics, logs and debugging failures](#12-observability-metrics-logs-and-debugging-failures)
13. [Testing and Evaluation](#13-testing-and-evaluation)
14. [Incident History and Known Problems](#14-incident-history-and-known-problems)
15. [Pending Work, Recommendations and Routine Ops](#15-pending-work-recommendations-and-routine-ops)
16. [Vocabulary / Glossary](#16-vocabulary--glossary)
17. [Access Checklist](#17-access-checklist)
18. [Appendix: Key Paths and Links](#18-appendix-key-paths-and-links)

---

## 1. TL;DR: What you are inheriting

| Item | Summary |
|---|---|
| **What it does** | Generates long, template-based reports (e.g. medical case reports) from a case's documents using LLMs, with **page-level source citations**. |
| **Who calls it** | Only `usesky-api` (the main Sky backend). It has **no authentication of its own**; trust is at the network level. |
| **Main LLM** | Google **Gemini**, with **Claude** (Anthropic) as fallback and **Vertex AI** for the Canada (NA) instance. |
| **Shape** | A NestJS **API** puts jobs on **BullMQ/Redis** queues. **Workers** run **Genkit** LLM pipelines, then save results to **PostgreSQL**. |
| **Team** | None. You become the **sole owner**. |
| **Deploys** | Manual, on **Tuesdays**: staging first, then prod, in sync with prompt releases. |
| **Hardest open problem** | Getting exhaustive fact extraction, readable prose and correct citations all at once. |
| **Biggest operational risks** | Retry multiplication (cost spikes), stuck "processing" rows, Genkit memory growth, the 4h Redis TTL, scaling past ~1000 pages. |
| **Client's top concerns** | Critical facts left out, duplicated facts, events in the wrong order, hard-to-measure quality (QA), template effort per report type, and getting from 75% to 95% without reruns. See §2. |

### Product context (from the client call)

The business problem behind the service, in short (full summary in §2):

- **Real users** are the "human in the loop" (HITL) companies that serve doctors: IME/medical-assessment firms, consultancies, admin staff and large enterprises. Doctors mainly **review and sign off** final reports.
- **What Sky-Tech delivers:** medical summaries and **chronologies** built from case documents, with every fact traceable to its source page.
- **The pain:** building a new template for each report type is heavy work, and customers don't want to spend that time.
- **The quality gap:** reports often reach about **75–80%** of what's needed. The client worries most about **missing critical facts**, **duplicates** and **wrong event order**. Rerunning the whole report to move from 80% to 95% **burns tokens**.
- **The goal:** give customers **iterative, self-serve control** (for example AI-edit of sections, template tuning and evals) so they can get from 75% to 95% *without* heavy technical support from the Sky team.

```mermaid
flowchart LR
    A[First generation<br/>~75%] --> B{Acceptable?}
    B -- no --> C[Targeted edit<br/>AI-edit a section /<br/>tweak template]
    C --> B
    B -- yes --> D[Final report<br/>90–95%]
```

---

## 2. Client Call Summary (plain English)

> **Source:** running notes from a call with the client (`notes.md`). The notes were typed live and are rough, so this section paraphrases them. Where the notes are unclear, it says so.

### 2.1 Why the call happened

Sky-Tech is choosing **who will take over this system** now that Sergei is leaving. The client said the next ("second") round should be based on **Sergei's handover**. Each candidate reads it and proposes **how they would tackle the problems below**, so Sky-Tech can compare approaches. Sky-Tech offered to share more technical documentation for this.

### 2.2 Who actually uses the product

```mermaid
flowchart LR
    D[Case documents] --> S[Sky-Tech<br/>extracts + drafts]
    S --> H[HITL company<br/>IME firm / admin<br/>reviews + fixes]
    H --> P[Doctor<br/>reads + signs off]
```

- **Doctors** have very little time and won't spend it learning a tool. They review and sign off final reports.
- **The real day-to-day users** are the people between the documents and the doctor: IME companies, medical-assessment firms, consultancies and administrators. The client calls them the **human in the loop (HITL)**. Large enterprises would use it the same way.
- **Sky-Tech's job:** take the documents, **extract the information**, and present **medical summaries and chronologies** in an acceptable, correct order.

### 2.3 What the client is worried about

| # | Concern (plain English) | What the client said, paraphrased |
|---|---|---|
| 1 | **Critical facts get left out** | The template is followed, but important facts are missing. A doctor reading the report notices the gaps. |
| 2 | **How much can the system handle?** | "How large is our context capability?" How many facts can it extract and fit into chronologies and summaries? Context windows have limits, and they are **not getting cheaper** as fast as hoped. Long medical reports (the notes mention the 10–20 page range) are becoming very hard. |
| 3 | **Duplicates and event order** | The same information appears more than once, or events come out in the wrong order. The client wants output that is **succinct and defensible**. This is the client's personal top concern. |
| 4 | **Chronologies matter as much as summaries** | Medical summaries are being worked on, but chronologies (timelines) need equal focus. |
| 5 | **QA is hard** | It's hard to tell whether the system did what it was prompted to do: did it follow the template, and is it accurate? |
| 6 | **Templates don't scale** | Every report type needs its own template, and clients don't want to spend time building them. |
| 7 | **75% → 95% without reruns** | Rerunning the whole report to gain 5% burns tokens. Clients need a way to improve a report themselves, **without heavy support** from Sky-Tech. |
| 8 | **The "lazy answer" failure** | A familiar failure in tools like ChatGPT: the model reads only part of the input (say 50%), but writes as if it read everything. The unread half silently disappears. |

### 2.4 What the client says already works

- **Hallucination is limited.** Every fact must be traceable back to a source page. If the system can't find something in the documents, it doesn't include it. This is the citation part of the pipeline (§7.3).

### 2.5 Ideas the client raised

- **Look beyond the LLM.** Ask what the **user should provide as input** so that the variables are **traceable** and the output becomes **predictable**, even with "org-wide context". Then, when something goes wrong, you can tell **where** it failed, for example "this is a context-window issue".
  - The notes don't define "org-wide context". It probably means settings or instructions shared across a client organisation. Ask the client to confirm.

### 2.6 The four questions candidates should answer

1. **What is the available context?** How much can the system process and hold today?
2. **What is the user-facing solution?**
3. **What would the best possible build look like** with today's technology?
4. **How can users get the best output** from the system as it exists now?

### 2.7 Client concerns mapped to the system

*My mapping, using what the handover says exists.*

| Client concern | Where it lives in the system | What exists today | Gap |
|---|---|---|---|
| Missing critical facts | Relevance step, sky-rag page filter, fact merge, prose writing (§7) | Fail-open relevance; page-content pipeline; metrics `fact_attribution`, `source_coverage` | No metric for "facts in the documents that never reached the report". Report-gold eval is the closest (§13) |
| Context capacity | Chunking and fan-out (§6, §7.5) | Case split into page chunks; >400 pages fanned out; validated to **~1000 pages** | 5–10k pages unvalidated; no published "max facts per section" number |
| Hallucination | Citation steps `mergeSourcesStep`, `buildSpansStep` (§7.3) | Every fact linked to a page; `source_coverage` metric | Citation corruption bugs (SKY-3944) |
| Duplicates | `mergeFactsTreeStep` (in a section); cross-section dedup (between sections) | In-section dedup is live | Cross-section dedup is **off in prod**; no whole-report pass (§6) |
| Event order | `generateChronologySectionFlow` | Chronology sections + chronology eval (SKY-2948) | Order across *different* sections isn't checked |
| QA | Eval tracks (§13) | LLM judge; gold set of 3 cases | Report-gold work is **uncommitted**; small gold set; Devin can't run evals alone |
| Template scaling | `generateTemplateFlow` (template from a sample PDF), previews | Exists | Still heavy per report type, according to the client |
| 75% → 95% iteration | AI-edit, `humanInput` sections, previews | Exists | Facts are re-extracted on every run; template-section identity waits on the frontend (§15) |
| Traceable inputs | Per-prompt config and metrics (§8, §12) | Per-prompt token and cost metrics | No user-facing view of *why* a fact was dropped |
| "Lazy answer" failure | Chunking (§7.5) | Each LLM call reads only one chunk, so no call is asked to read more than it can hold | Facts can still be dropped later, at merge or write (§7.6) |

### 2.8 What this means for you

*My suggestions, not from the call.*

- **Answer "how much context?" with numbers.** Run cases of growing size and record pages, facts extracted, facts that made it into the report, and cost. The metrics `docListPageCount`, `factsBeforeDispatch`/`factsAfterDispatch`, `source_coverage` and `usdCost` already exist (§12).
- **Measure missing facts directly.** Compare the facts in a gold report with the facts in the generated report (recall). This is the client's #1 concern, and today it's measured only indirectly.
- **Treat duplicates and ordering as a whole-report problem.** Turning on cross-section dedup is a cheap first step. A whole-report check is the next one.
- **Prepare a proposal for the second round** that answers the four questions in §2.6, using this handover as evidence.

---

## 3. People and Roles

| Person | Role | Ask them about |
|---|---|---|
| **Sergei Privalov** (departing) | Sole owner since first commit (Aug 23, 2025): architecture, API, workers, DB, CI/CD, deploys, Linear (`SKY-*`) planning | Everything (until he leaves) |
| **Victor Karpov** | Early project supervisor | *Why* early decisions were made (Node.js, Redis, separate workers) |
| **Stas Kulesh** | Second-largest contributor (~40 commits), worked mostly through **Devin** (AI coding agent) | Ad-hoc extractions, AI-edit, chronology sections |
| Vlad, Viktor, Gleb | Occasional commits | — |
| **Devin AI** | AI coding bot, ~14 commits | — |

**Your responsibilities as successor:** full development lifecycle, ticket estimation in Linear, code review of all contributions, staging/prod deploys, and occasional stakeholder communication.

---

## 4. Tech Stack: what each piece is and why it is here

### 4.1 Application and language

| Tech | What it is (plain English) | What it's used for here |
|---|---|---|
| **Node.js 22** | Runtime that runs JavaScript/TypeScript on a server. | Runs both the API and the workers. |
| **TypeScript** | JavaScript with types, which catches mistakes before runtime. | All service code. |
| **tsx** | Tool that runs TypeScript files directly without a separate build step. | Starts the workers process. |
| **NestJS 11** | Structured backend framework for Node (controllers, modules, dependency injection). | The **API** service (`api/`) on port **3003**. |
| **Zod** | Library for describing data shapes and validating them at runtime. | Validates queue job payloads and defines LLM input/output schemas. |
| **Swagger / OpenAPI** | Auto-generated, browsable API documentation. | Served at `/api` on the API host. |

### 4.2 AI / LLM

| Tech | What it is | What it's used for here |
|---|---|---|
| **Genkit** (by Google) | Open-source framework for building AI features. It gives **one interface for many LLM providers**, plus *flows* (traceable multi-step AI functions), structured output (via Zod) and a local developer UI that shows a trace of each model call. | The whole LLM pipeline in `workers/src/genkit/`: every prompt call, model fallback and section "flow". |
| **Google Gemini** | Google's LLM family. | **Primary** model (`gemini-3.5-flash(-lite)`, `gemini-3.7/3.8-flash`, `gemini-3.1-pro-preview`). |
| **Anthropic Claude** | Anthropic's LLM family. | **Fallback** model when Gemini fails (`haiku-4-5`, `sonnet-4-6`, `opus-5`). |
| **Vertex AI** | Google Cloud's managed AI platform. It serves Gemini from a chosen region. | The **only** LLM route for the Canada (NA) instance, to meet data-residency rules. |
| **OpenRouter** | LLM marketplace/router that publishes current model prices. | Used **only for price lookup** at worker startup, to calculate cost per call. |
| **LiquidJS** | Templating language (`{{ variable }}`, loops, conditionals). | Prompt templates stored in the `sky-prompts` repo. |
| **sky-rag + pgvector** | Internal Sky service doing vector (semantic) search over document pages; pgvector is the PostgreSQL extension that stores the vectors. | **Picks which pages to use** when a case is very large. |

### 4.3 Data, queues and storage

| Tech | What it is | What it's used for here |
|---|---|---|
| **PostgreSQL** | Relational database. | Stores reports, sections, templates, evaluations and so on. |
| **Prisma 6** | ORM: you write a schema file, and it generates type-safe DB code and migrations. | `prisma/schema.prisma` (~38 migrations), shared by API and workers via `packages/database`. |
| **Redis** | Very fast in-memory key-value store. | Queue backend **and** a temporary data bus (snapshots, caches, results). See §10. |
| **BullMQ** | Job-queue library built on Redis, with producers, consumers, retries and parent/child jobs. | The API enqueues jobs and the workers consume them. Contract lives in `packages/queues`. |
| **Bull Board** | Web dashboard for BullMQ queues. | `/admin/queues`, for inspecting stuck and failed jobs. |
| **AWS S3** | Object (file) storage. | Document pages (doclist) and the settings bucket that holds prompts and `config.json`. |

### 4.4 Infrastructure, CI/CD and monitoring

| Tech | What it is | What it's used for here |
|---|---|---|
| **Docker / docker-compose** | Packages apps into containers; compose starts several containers together. | Local dev stack and integration tests. |
| **AWS ECR** | AWS's private Docker image registry. | Stores built images. |
| **AWS ECS** | AWS's container runner. | Runs API and worker containers (`sky-prod-cluster`, region `ca-central-1`). |
| **GitHub Actions** | CI/CD built into GitHub. | Lint, tests and deploy workflows on self-hosted `[sky-ops]` runners. |
| **GCP Workload Identity Federation (WIF)** | Lets AWS workloads authenticate to Google Cloud without long-lived keys. | Vertex AI auth for the NA instance (`infra/vertexAuth.ts`). |
| **AWS CloudWatch** | AWS log storage and search. | Receives all JSON logs and metrics. |
| **Grafana** | Dashboards on top of data sources such as CloudWatch. | Quality, performance and cost dashboards. |
| **Jest / Vitest** | JavaScript test runners. | Jest for API tests, Vitest for worker tests. |
| **Linear** | Issue tracker. | Tickets with the `SKY-*` prefix. |
| **Devin** | Autonomous AI software-engineer agent. | Used by Stas and for some eval work. It has no AWS/DB access. |

### 4.5 Where each tech sits

```mermaid
flowchart LR
    subgraph API["api/ (NestJS)"]
      N[Controllers<br/>Swagger] --> P1[Prisma]
      N --> Q1[BullMQ producer]
    end
    subgraph WK["workers/ (Node + tsx)"]
      Q2[BullMQ consumers] --> G[Genkit flows]
      G --> L[Gemini / Claude / Vertex]
      Q2 --> P2[Prisma]
    end
    Q1 --> R[(Redis)]
    R --> Q2
    P1 --> DB[(PostgreSQL)]
    P2 --> DB
```

---

## 5. System Overview

### 5.1 Request flow (who calls whom)

```mermaid
flowchart LR
    U[User in app] --> A[app<br/>frontend]
    A --> C[usesky-api<br/>core backend]
    C --> S[sky-custom-reports<br/>API :3003]
```

### 5.2 Generation flow (what happens after a request)

```mermaid
flowchart LR
    S[API] -->|enqueue| Q[reports-queue]
    Q --> F[Fan-out:<br/>1 job per section]
    F --> W[Section workers<br/>LLM pipeline]
    W --> DB[(PostgreSQL)]
```

### 5.3 Read-back flow

```mermaid
flowchart RL
    DB[(PostgreSQL)] --> S[API v3]
    S --> C[usesky-api]
    C --> A[app]
```

### 5.4 Repository layout

| Folder | Contents |
|---|---|
| `api/` | NestJS 11 + Prisma 6, BullMQ **producer**, Bull Board (`/admin/queues`), Swagger (`/api`) |
| `workers/` | Node 22 + tsx, BullMQ **consumers** + Genkit pipelines |
| `packages/database` | Shared `PrismaClient` |
| `packages/queues` | Shared queue contract: names, Zod job schemas, TTLs, `na_` namespace |
| `prisma/` | Single `schema.prisma` + ~38 migrations |

### 5.5 Key features and owners

| Feature | Author | Notes |
|---|---|---|
| Multi-stage section pipeline (chunk → relevance → facts → merge → prose + citations) | Sergei | 2nd iteration; the 1st used summaries |
| Template generation from a sample PDF | Sergei | `generateTemplateFlow` |
| Cross-section fact dispatch/dedup | Sergei | Flag `CROSS_SECTION_DEDUP_ENABLED`, **should be ON in prod** |
| Eval infrastructure | Sergei | §13 |
| RAG sources (page citations) | Sergei | §7 |
| Ad-hoc extractions | Stas | `POST /v1/extractions/ad-hoc`, synchronous via Redis result-store |
| AI-edit for sections/reports | Stas | `POST /v3/cases/:caseId/reports/:id/ai-edit` + related |
| Chronology sections | Stas | `generateChronologySectionFlow` (`contextType=chronology`) |

---

## 6. Queues and Workers

Entry point: `workers/src/index.ts`. Startup steps:

```mermaid
flowchart LR
    A[Check S3] --> B[Vertex auth<br/>if NA]
    B --> C[registerPrompts]
    C --> D[Warm OpenRouter<br/>prices]
    D --> E[Start 5 workers]
```

| Queue | Worker | Purpose |
|---|---|---|
| `reports-queue` | `reportsGenerationWorker` | Report **fan-out**: prepares doclist and section snapshots in Redis (TTL 4h), enqueues one job per section |
| `sections-queue` | `reportSectionGenerationWorker` | Generates one section; also handles ad-hoc and preview jobs |
| `section-page-chunks-queue` | `sectionChunkWorker` | Page chunks for big sections (**>400 pages**) |
| `template-generation-queue` | `templateGenerationWorker` | Builds templates from a sample PDF; also runs report-evaluation jobs |
| — | `templateOrphanReconciler` | Marks "stuck" processing templates as failed, using `heartbeatAt` |

### Fan-out shape

```mermaid
flowchart TD
    R[Report job] --> S1[Section 1]
    R --> S2[Section 2]
    R --> Sn[... up to ~50 sections]
    S2 -->|>400 pages| C1[Chunk A]
    S2 --> C2[Chunk B]
```

**Why fan-out?** Sections are independent, so they run in **parallel**, which cuts generation time a lot.
**What's missing: a final "whole-report pass".** Each section is written without seeing the others, and nothing reads the finished report as a whole.

```mermaid
flowchart LR
    S[All sections<br/>written separately] --> J[Simply joined] --> R[Report]
    S -.->|missing step| W[Whole-report check<br/>dedup · contradictions ·<br/>terms · summary last]
```

Without that check, a report can contain *(my examples; the handover only names the missing pass)*:
- **Repetition:** two sections describe the same MRI in full.
- **Contradictions:** different dates for the same event, because two sections read different pages.
- **Inconsistent terms:** "the patient", "the claimant" and "Mr. Smith" in different sections.
- **A "Summary" section that can't summarize,** because it's written at the same time as the sections it should summarize.

Cross-section dedup (§5.5) covers part of the repetition, but it is **off in prod**. Adding the pass is not trivial: ~50 sections is a lot of text for one call, rewriting can break citations, and it adds cost and time. This gap ties directly to the client's concern about duplicates and event order (§2.3).

---

## 7. Section Generation Pipeline

Applies to sections with `contextType = snapshot`. Code: `workers/src/genkit/flows/report-section/`.

Sections are generated from **actual page content**, not summaries. The first version used summaries, and they could not support page citations.

### 7.1 Stages, part 1: find the right pages

```mermaid
flowchart LR
    A[1. Chunking<br/>split doclist<br/>into page chunks] --> B[2. detectRelevanceStep<br/>LLM marks<br/>relevant pages]
    B --> C[3. fetchPagesStep<br/>load page markdown<br/>S3 and/or sky-rag]
```

- **Chunking:** above 400 pages, chunks are fanned out to `section-page-chunks-queue`.
- **Relevance:** runs in batches, uses `relevanceCache`, and **fails open**: if a batch errors, its pages are kept.
- **Fetch:** reads S3 directly. For large volumes, sky-rag (`POST /v1/tenants/{org}/cases/{case}/context`) acts as a **page-selection filter**. This is a compromise, not full RAG.

### 7.2 Stages, part 2: facts

```mermaid
flowchart LR
    C[Pages] --> D[4. extractChunkFactsFlow<br/>atomic facts<br/>per chunk]
    D --> E[5. mergeFactsTreeStep<br/>hierarchical<br/>merge + dedup]
```

### 7.3 Stages, part 3: write and cite

```mermaid
flowchart LR
    E[Merged facts] --> F[6. finishSnapshotSectionFlow<br/>prose]
    F --> G[buildMdastStep<br/>→ contentMdast]
    G --> H[buildSpansStep]
    H --> I[mergeSourcesStep<br/>citations]
    I --> J[resolveSectionLinksStep]
```

### 7.4 Other section types (`contextType`)

| Flow | Use |
|---|---|
| `generateDocListSectionFlow` | Section listing the case documents |
| `generateChronologySectionFlow` | Timeline/chronology of events |
| `humanInput` | Filled in manually by the report author |

> ⚠️ **The key unsolved problem:** getting **exhaustive facts**, **readable prose** and **correct citations** all at once. Quality is constantly being rebalanced between these three. The client's top concern, "critical facts left behind", is the *exhaustive facts* part of this problem.

### 7.5 How the pipeline deals with the context window

A **context window** is the maximum amount of text an LLM can read in one call. A large case doesn't fit, and when a model is overloaded it tends to give the "lazy answer" the client described (§2.3, #8). So the pipeline **never sends the whole case at once**:

```mermaid
flowchart LR
    A[Whole case<br/>too big for<br/>one call] --> B[Split into<br/>page chunks]
    B --> C[One LLM call<br/>per chunk:<br/>extract facts]
    C --> D[Merge facts<br/>step by step]
    D --> E[One call<br/>writes the<br/>section]
```

Each "reading" call sees only one chunk, so no single call has to hold the whole case. *My reading:* the limit doesn't disappear, it moves. All the merged facts for a section must still fit into the final writing call. The handover gives no number for how many facts that is, and that is exactly what the client is asking ("how large is our context capability?").

### 7.6 Where a critical fact can get lost

*My reading of the pipeline; the handover doesn't list these.*

```mermaid
flowchart LR
    P[Fact on a page] --> R{Page marked<br/>relevant?}
    R -- no --> X1[Lost]
    R -- yes --> M{Survives<br/>merge/condense?}
    M -- no --> X2[Lost]
    M -- yes --> W{Used in<br/>the prose?}
    W -- no --> X3[Lost]
    W -- yes --> OK[In report<br/>with citation]
```

| Stage | How a fact can be lost |
|---|---|
| Relevance (stage 2) | The LLM marks a page as not relevant to this section. Batch *errors* keep pages (fail-open), but a wrong "not relevant" decision drops the page. |
| Fetch (stage 3) | For very large cases, sky-rag selects pages. A page it doesn't pick is never read. |
| Merge (stage 5) | Two different facts are treated as duplicates, or a fact is condensed away. |
| Write (stage 6) | The prose step leaves a fact out to keep the text readable. |
| Cross-section dispatch | When enabled, a fact is assigned to another section. This is correct, unless that section drops it. |

No current metric directly counts facts lost at each stage. `factsBeforeDispatch`/`factsAfterDispatch` and `source_coverage` (§12) are partial signals.

---

## 8. LLM Layer: Genkit, Models and Prompts

### 8.1 Why Genkit

It was chosen for a **single interface across LLM providers**, convenient tracing, and **structured input/output** with Zod schemas.
Wrapper: `workers/src/genkit/prompts/prompt.ts`. Each prompt has a **main** model and a **fallback** model.

> ⚠️ **Known issue:** Genkit appears to keep registered schemas in memory. Workers show slow memory growth and **need periodic restarts**. This is unconfirmed; verify it and consider replacing Genkit or using newer agentic approaches.

### 8.2 Model call with fallback

```mermaid
flowchart LR
    A[Prompt call] --> B[Main model<br/>Gemini<br/>× mainAttempts]
    B -- all failed --> C[Fallback<br/>Claude<br/>× fallbackAttempts]
    C -- all failed --> D[AllAttemptsExhaustedError<br/>permanent]
    B -- ok --> E[Result]
    C -- ok --> E
```

- Backoff: exponential, 2–30s.
- **NA instance:** uses the `vertex-ca` model config, so all calls go via Vertex AI.
- **Pricing:** `infra/modelPricingTable.ts`, plus live OpenRouter prices loaded into Redis at startup.

### 8.3 Prompt release flow

Prompts are **LiquidJS** templates in the `sky-prompts` repo (`prompts/custom-reports/`), together with a single `config.json`. For each prompt, `config.json` sets the main, fallback and vertex-ca models, generation parameters, attempt count and timeout.

```mermaid
flowchart LR
    A[PR in sky-prompts] --> B[Merge to develop]
    B --> C[CI publishes<br/>to S3]
    C --> D[⚠️ Restart workers<br/>to apply]
```

> ⚠️ **Gotcha:** config and templates are loaded **once at startup** and cached (`s3ConfigCache` in `workers/src/infra/promptLoader.ts`). Hot-updating was the original intent, but in practice changes **only apply after a restart**. Recommendation: move prompt version/config selection into code or env vars.

**Local dev without S3:**
```bash
USE_LOCAL_PROMPTS=1
LOCAL_PROMPTS_PATH=<path to sky-prompts clone>/prompts/custom-reports
```

---

## 9. Retries and Idempotency

> This is **a major source of complexity**. Unbalanced retries have already caused an expensive incident.

### 9.1 Retry layers (they multiply)

```mermaid
flowchart TD
    J[BullMQ job retry<br/>~3–4 attempts] --> L[In-process LLM retry<br/>main + fallback attempts]
    L --> X[LLM provider]
    J --> S3R[S3 read retry<br/>exp. backoff]
```

| Layer | Where | Detail |
|---|---|---|
| 1. LLM call | `prompt.ts` | `mainAttempts` then `fallbackAttempts`, backoff 2–30s; per-prompt values from `config.json` |
| 2. Queue job | `packages/queues/src/job-options.ts` | ~3–4 attempts per job type. Parent and child retry **independently**: without `failParentOnFailure`, a parent can hang |
| 3. S3 | `infra/s3.ts` | Exponential backoff on reads |
| 4. Idempotency | DB | `Report.idempotencyKey` (unique) + a **partial unique index** `(reportId, idempotencyKey)` on `ReportSection`. It was created by hand in a migration, not in the Prisma DSL, which makes it **fragile** |

### 9.2 Rules

- `AllAttemptsExhaustedError` is a **permanent** error, so the queue must not start another retry cycle.
- **Before changing any retry count, multiply them out.** Example: 4 job attempts × (3 main + 2 fallback) LLM attempts = **20 LLM calls** for one failing step, repeated across ~50 sections.

---

## 10. Redis as a Data Bus

Redis is more than a queue. It holds:

| Key / data | Purpose | TTL |
|---|---|---|
| BullMQ queues | Jobs | — |
| `report:prepared-doclist:*` | Prepared document list | **4h** |
| `report:section-snapshot:*` | Per-section input snapshot | **4h** |
| `report:section-chunk-results:*` | Results of page-chunk jobs | — |
| Ad-hoc / preview result-store | Synchronous request results | — |
| Relevance cache | Page-relevance decisions | — |
| OpenRouter price cache | Model prices | — |

```mermaid
flowchart LR
    A[Report enqueued<br/>snapshot saved] --> B{Processed<br/>within 4h?}
    B -- yes --> C[OK]
    B -- no --> D[⚠️ Snapshot expired<br/>section fails]
```

---

## 11. Infrastructure and Deployment

### 11.1 CI pipeline (on every PR)

```mermaid
flowchart LR
    A[Lint + typecheck<br/>+ format] --> B[Unit tests<br/>Jest api / Vitest workers]
    B --> C[Integration<br/>full docker-compose<br/>e2e + test:integration]
```

Files: `pr.yml`, `_tests.yml`. Runners: self-hosted `[sky-ops]`, Node 22.

### 11.2 Deploy pipeline

```mermaid
flowchart LR
    A[docker build] --> B[push to ECR]
    B --> C[aws ecs update-service<br/>--force-new-deployment]
    C --> D[api container start<br/>runs Prisma migrations]
```

- Workflows: `deploy-development.yml`, `deploy-staging.yml`, `deploy-production.yml`.
- **Prod:** cluster `sky-prod-cluster`, services `sky-prod-custom-reports-api` / `-worker`, region `ca-central-1`.
- **Prod deploys are manual:** `workflow_dispatch` + type `DEPLOY` to confirm. Tag: `deployed/production`.
- **Schedule:** **Tuesdays**: staging, then prod, synced with the `sky-prompts` release.
- Migrations run in `docker-entrypoint.sh` when the API container starts.

### 11.3 NA (Canada) instance

| Aspect | Value |
|---|---|
| Flag | `IS_NA_INSTANCE=1` (**API and workers must match**) |
| Why | Data-residency requirements |
| Queues / Redis keys | Prefixed `na_` |
| LLM | **Only** Vertex AI (`VERTEX_AI_BASE_URL`, WIF auth, `infra/vertexAuth.ts`) |
| Deploy | Service `<svc>-na`, deployed in parallel with the main one; live in prod |

### 11.4 External dependencies

```mermaid
flowchart TD
    S[sky-custom-reports] --> C[usesky-api<br/>only client]
    S --> R[sky-rag<br/>needs VPN]
    S --> S3[(S3: doclist +<br/>settings/prompts)]
    S --> L[Gemini / Anthropic /<br/>Vertex AI]
    S --> M[CloudWatch → Grafana]
```

---

## 12. Observability: metrics, logs and debugging failures

All metrics are **structured JSON log lines** (`workers/src/infra/metrics.ts`, `component: "metrics"`, plus a `metric` field). They flow to **CloudWatch** and are shown in **Grafana**.

```mermaid
flowchart LR
    W[Workers<br/>JSON logs] --> CW[CloudWatch]
    CW --> G[Grafana<br/>prod-cloudwatch]
```

**Grafana panels:** Evaluation Quality · Performance · Report Generation Insights (avg generation time, pages/case, sections/report, cost/report). Some panels were built for the **old** pipeline.

### 12.1 Metric types

| `metric` | Records | Key fields |
|---|---|---|
| `job_completion` | End of each job (report / section / chunk / template / ad_hoc / preview / media_preview) | `status`, `duration_ms`, `reportId`, `reportSectionId`, `usdCost`, `docListPageCount`, tokens, `factsBefore/AfterDispatch` |
| `report_completion` | Report status recompute | `reportId`, `status`, `totalSections`, `duration_ms`, `usdCost`, `templateName`, `organizationId` |
| `llm_request` | Every LLM call | `provider`, `model`, tokens, `usdCost`, `attempt`, `isRetry`, `contextId` |
| `llm_token_usage` | Per-prompt tokens at flow end | `flowName`, `promptName`, `modelName`, tokens |
| `llm_fallback` | Main → fallback switch | `promptName`, `mainModel`, `fallbackModel`, `mainAttemptsMade` |
| `retry` | Each retry | `operation`, `attempt`, `maxAttempts`, `delay_ms`, `reason` |
| `error` | Classified error | `errorType` (LLM/Network/Validation/Database/Unknown), `reportSectionId`, `jobId` |
| `job_processing` | Start/finish of a section job | `jobId`, `reportSectionId`, `status`, `attempt` |
| `cost_aggregation` | Periodic cost rollup | `provider`, `model`, `totalUsdCost`, `period_minutes` |
| `fact_attribution` | Facts from pages vs summaries | `totalFacts`, `pagePercentage`, `summaryPercentage` |
| `source_coverage` | Share of facts with citations | `coverageRate`, `coveredFacts/totalFacts` |
| `cross_section_dedup` | Cross-section dedup | `reportId`, `runId` |
| `orphan_reconciled` | Row moved to failed by reconciler | `entityType`, `entityId`, `jobState` |
| `evaluation_result` | Eval run result | `evaluatorName`, `score`, `passed` |

### 12.2 How to trace a failed generation

```mermaid
flowchart LR
    A[Get reportId<br/>app URL / Bull Board /<br/>API response] --> B[Filter CloudWatch<br/>by reportId]
    B --> C[Find failed sections<br/>job_completion<br/>status=failed]
    C --> D[Pull error / retry /<br/>llm_request for that<br/>reportSectionId]
```

### 12.3 Pattern → diagnosis cheat-sheet

| You see | It likely means |
|---|---|
| Many `llm_fallback` for one `promptName` | Gemini (main model) is degraded |
| Spike in `retry` with `operation=llm_request` | LLM incident: **check the retry product** (§9) |
| `orphan_reconciled` | A job died without recording a result (worker crash/restart) |
| `report_completion status=processing` for > **15 min** | Stuck report; see `checkAndUpdateStaleReport` |
| Cost spike | `llm_request` by `contextId`/job, `usdCost` on completions, `cost_aggregation`, `/admin/analytics/token-usage` |

---

## 13. Testing and Evaluation

### 13.1 Tests

| Scope | Tool | Command |
|---|---|---|
| API unit (`*.spec.ts`) + e2e (`api/test/*.e2e-spec.ts`) | Jest | `npm run api:test`, `test:e2e` |
| Workers unit | Vitest | `npm run workers:test` |
| Workers against **real** LLMs (`__realtests__/*.realllm.test.ts`) | Vitest | `test:llm` (needs API keys) |
| Integration (`tests/summary-generation-test.integration.ts`) | — | `npm run test:integration` (full stack + LLM keys) |

### 13.2 Eval infrastructure (in progress)

Goal: **measure generation quality objectively**, which was missing until recently.

```mermaid
flowchart LR
    A[Generated report] --> J[LLM judge]
    G[Gold / expected<br/>report or facts] --> J
    J --> S[Score + passed]
    S --> M[evaluation_result<br/>→ Grafana]
```

| Track | Where | Status / notes |
|---|---|---|
| **Chronology eval** | `workers/src/eval/chronology/`, `workers/scripts/eval/chronology/` (`eval:chronology`, SKY-2948) | Matcher / scorer / grounding / reliability |
| **Report-fidelity eval** | `workers/src/report-eval/`, v3 API `report-evaluations`, CLI `scripts/eval/report-fidelity/runner.ts` | LLM judge `reportFidelityJudgePrompt`. ⚠️ Spec `specs/005-full-report-eval/spec.md` describes an **outdated** design |
| **Report-gold eval** | Branch `devin/sky-3944-report-gold` (SKY-3944), `workers/src/eval/report/`, `workers/scripts/eval/report-gold/` | ⚠️ **Uncommitted.** Gold set: cindy, john-doe, stolerman-karp. Judge: `gemini-3.8-flash`, with fact batching, judge caching and cost tracking |

> **Limitation:** fully autonomous eval is blocked because runs need AWS S3 + dev DB access, which Devin doesn't have. Either grant access or keep the semi-manual process.

> **Client view:** QA is one of the client's main worries (§2.3, #5): "has the system done what we prompted it to do?" These eval tracks are the current answer. *My view:* a gold set of 3 cases is small for proving quality across many report types and case sizes, and no eval yet reports **recall** (how many of the expected facts appear) as its own number.

---

## 14. Incident History and Known Problems

| # | Problem | Status / resolution |
|---|---|---|
| 1 | **Retry multiplication → cost spikes.** LLM retries × job retries caused request avalanches; large cases never finished. | Fixed: permanent-error classification, attempt caps, heartbeats |
| 2 | **Stuck "processing" rows.** No reliable way to tell whether a job is dead. | Partial: template reconciler + `heartbeatAt`; lazy `checkAndUpdateStaleReport` (15 min); report heartbeat (SKY-3400). **Stage 6 (section jobs) not done** |
| 3 | **Summary-first approach** couldn't cite pages. | Migrated to page-content pipeline. Validated to **~1000 pages**; **5–10k pages unvalidated** |
| 4 | **504 timeouts** on reports endpoint | Branch `SKY-1205/fix-...-timeout-504` |
| 5 | **Citation corruption** | SKY-3944 (cindy citation), related to `mergeSourcesStep` / `buildSpansStep` |
| 6 | **Genkit memory growth** | Unconfirmed; periodic worker restarts |
| 7 | **No objective quality measure** | Led to eval infrastructure (§13) |

---

## 15. Pending Work, Recommendations and Routine Ops

### 15.1 Active / WIP (do these first)

- [ ] **SKY-3944**: commit/push the uncommitted report-gold eval files on `devin/sky-3944-report-gold`; make it reproducible.
- [ ] **Stage 6 stuck-processing**: heartbeat + orphan reconciler for **section** jobs (copy the template reconciler pattern). See `docs/stuck-processing-remediation-plan.md`.
- [ ] **Enable `CROSS_SECTION_DEDUP_ENABLED` in prod**: add it to infra config (default-off in code).
- [ ] **Template-section identity round-trip**: backend ready (`docs/preserve-templatesection-identity-plan.md`), waiting on the frontend editor.
- [ ] **Dead script**: `workers/package.json → eval:report-fidelity` points to a missing file. The real runner is in root `scripts/`.
- [ ] **README outdated**: old queue scheme, missing chronology/humanInput, current APIs are **v3/v4** (v1/v2 deprecated but live).
- [ ] **Grafana dashboard** for the current pipeline.
- [ ] Possible duplicate: `api/src/reports/constants.ts` vs `constants/reports.constants.ts`.

### 15.2 Strategic recommendations (Sergei's opinion)

| Recommendation | Why |
|---|---|
| Migrate **BullMQ → AWS SQS** | Other Sky services already use SQS; BullMQ parent/child is a poor fit; it would simplify reliability and stuck jobs |
| **Cache/warm shared LLM context** | Avoid re-sending identical context, which saves cost |
| **Revisit Genkit** | Memory growth; newer agentic approaches exist |
| Move prompt config from S3 to **code/env** | The current scheme implies hot updates that don't happen |
| Add **final whole-report pass** | Coherence across parallel-generated sections |
| Validate **5–10k page** scaling | Pipeline and cost unproven |
| Roll out **cross-section dedup** | Already built |
| **Facts as a persistent data layer** | See below |

**Long-term direction: facts as a persistent layer**

```mermaid
flowchart LR
    A[Extract facts<br/>with metadata] --> B[(Store in DB<br/>source, page, date, type)]
    B --> C[Dedup across<br/>runs + sections]
    C --> D[Agent retrieves<br/>fact clusters]
    D --> E[Compose report]
```

Today, facts are re-extracted on every run and thrown away afterwards. Persisting them would make repeat generations cheaper, fit agentic report composition, and help with the core quality problem in §7. It also maps onto the product goal of iterating cheaply from 75% to 95% (§1).

### 15.3 Routine operations

| Task | How |
|---|---|
| **Deploy** | Tuesdays: staging, then prod (`workflow_dispatch` + `DEPLOY`), synced with `sky-prompts` |
| **Update prompts** | PR in `sky-prompts`, merge to `develop`, auto-publish to S3, **restart workers** |
| **Stuck/failed jobs** | Bull Board `/admin/queues` |
| **Stuck processing rows** | DB + logs, `checkAndUpdateStaleReport` |
| **DB migration** | Add to `prisma/migrations`; it auto-runs at API startup |
| **Cost monitoring** | `/admin/analytics/token-usage`, `usdCost` on `ReportSection` |

### 15.4 Priorities suggested by the client call

*My mapping of client concerns (§2.3) to work already listed in this handover.*

| Client concern | Most relevant work items |
|---|---|
| Missing critical facts / QA | Finish report-gold eval (SKY-3944); add a recall-style measure; grow the gold set |
| Duplicates | Enable `CROSS_SECTION_DEDUP_ENABLED`; build the whole-report pass |
| Event order | Chronology flow + chronology eval; check ordering across sections |
| Context capacity | Validate 5–10k pages; publish measured limits (pages, facts, cost) |
| 75% → 95% iteration | Persistent facts layer; template-section identity (frontend); AI-edit |
| Template effort | Improve `generateTemplateFlow` (template from sample PDF) |

---

## 16. Vocabulary / Glossary

### Domain / product terms

| Term | Meaning |
|---|---|
| **Case** | A set of documents about one subject (e.g. a patient/claim). Reports are generated per case. |
| **Report** | The final generated document for a case, built from a template. |
| **Template** | Blueprint of a report: its list of sections and instructions for each. Can be generated from a sample PDF. |
| **Section** | One part of a report (a template may have ~50). Generated independently. |
| **Doclist** | The list of documents (and their pages) in a case. |
| **Snapshot (section)** | A frozen copy of the inputs a section job needs, stored in Redis for 4h. Also the default `contextType`. |
| **contextType** | What a section is built from: `snapshot` (page content), `chronology`, doclist, `humanInput`. |
| **Chronology** | A timeline section listing events in date order. |
| **humanInput** | A section the report author writes by hand. |
| **Fact** | A small, atomic statement extracted from a page (e.g. "MRI on 2024-03-02 showed…"). |
| **Citation / RAG source** | Link from a statement in the report back to the document page it came from. |
| **Ad-hoc extraction** | One-off, synchronous question/extraction against a case, outside a report. |
| **AI-edit** | Endpoint that lets users ask an LLM to revise a section or report. |
| **Preview** | Quick generation of a single section, e.g. while editing a template. |
| **IME** | Independent Medical Examination, a medical assessment done by an independent doctor, often for insurance/legal cases. A key customer segment. |
| **Gold report** | A hand-verified "correct" report used as the reference answer in evals. |
| **NA instance** | Separate deployment in Canada for data residency. |
| **HITL (Human In The Loop)** | A person who reviews and corrects AI output before it's final. Here, IME firms and admin staff who prepare reports for doctors. |
| **Medical summary** | A condensed overview of a patient's medical history and findings. |
| **Sign-off** | The doctor's final approval of a report. |
| **Org-wide context** | Term from the client call; probably settings or instructions shared across a client organisation. Not defined in the notes. |
| **Second round** | The next step in choosing who takes over the system: candidates propose approaches based on this handover. |

### AI / LLM terms

| Term | Meaning |
|---|---|
| **LLM** | Large Language Model (Gemini, Claude, etc.). |
| **Prompt** | The instruction text sent to an LLM. Here, LiquidJS templates. |
| **Flow (Genkit)** | A named, traceable function made of steps (often several LLM calls). |
| **Fallback model** | Second model tried when the main model keeps failing. |
| **Token** | The unit LLMs count text in; cost is per token. |
| **RAG** | Retrieval-Augmented Generation: fetch relevant content first, then let the LLM write from it. |
| **Vector / embedding / pgvector** | Numeric representation of text meaning, used for semantic search; pgvector stores these in Postgres. |
| **Relevance detection** | LLM step that decides which pages matter to a section. |
| **Fail-open** | On error, keep the data (here, keep pages) rather than drop it. |
| **Dedup** | Removing duplicate facts, within a section or across sections. |
| **Tournament / condensedFacts** | Hierarchical merge where fact groups are merged pairwise in rounds until one condensed list remains. |
| **MDAST / contentMdast** | Markdown Abstract Syntax Tree: a structured tree form of Markdown text. Section content is stored this way. |
| **Spans** | Pieces of text in the output that citations are attached to. |
| **LLM judge** | Using an LLM to grade another LLM's output against a reference. |
| **Eval** | An automated quality test of generated output. |
| **Agentic approach** | An LLM that decides its own steps and tool calls, instead of a fixed pipeline. |
| **Context window** | The maximum amount of text an LLM can read in one call. Larger windows cost more per call. |
| **Hallucination** | When an LLM states something that isn't in its source material. |
| **Grounding** | Tying every statement to a source (here, a document page). This is how the system limits hallucination. |
| **"Lazy answer" / truncation** | The model processes only part of its input but answers as if it had read everything. |
| **Recall** | The share of the facts that *should* be in the report that actually are. Low recall means missing facts. |
| **Precision** | The share of facts in the report that are correct and supported. |

### Engineering / infra terms

| Term | Meaning |
|---|---|
| **Microservice** | Small, separately deployed service with one job. |
| **Queue / job** | Work items waiting to be processed; a job is one item. |
| **Producer / consumer** | The one who puts jobs on a queue (API) / the one who processes them (workers). |
| **Worker** | Background process that consumes jobs. |
| **Fan-out** | Splitting one job into many parallel child jobs. |
| **Parent / child job** | BullMQ relationship: a parent waits for its children. |
| **`failParentOnFailure`** | BullMQ option: if a child permanently fails, fail the parent too. |
| **Retry / backoff** | Trying again after failure, waiting longer each time (exponential backoff). |
| **Permanent error** | Error that should **not** be retried (e.g. `AllAttemptsExhaustedError`). |
| **Idempotency key** | Unique key so the same request processed twice doesn't create duplicates. |
| **Partial unique index** | DB uniqueness rule applying only to rows matching a condition. |
| **Heartbeat / `heartbeatAt`** | Timestamp a running job updates regularly to show it's still alive. |
| **Orphan / reconciler** | A job that died silently / a background process that finds such jobs and marks them failed. |
| **Stale threshold** | Time (15 min) after which a "processing" report is considered stuck. |
| **TTL** | Time-To-Live: how long a Redis key exists before it is deleted automatically. |
| **Hot update / hot reload** | Applying config changes without restarting. This does **not** work here. |
| **Migration** | Versioned script that changes the DB schema. |
| **ORM** | Library mapping DB tables to code objects (Prisma). |
| **CI / CD** | Continuous Integration (auto test on PR) / Continuous Deployment (auto build and deploy). |
| **`workflow_dispatch`** | GitHub Actions trigger you start manually from the UI. |
| **Self-hosted runner** | Company-owned machine that runs GitHub Actions jobs. |
| **Data residency** | Legal requirement that data stays in a given country/region. |
| **WIF** | Workload Identity Federation: keyless cross-cloud authentication. |
| **e2e test** | End-to-end test through the real API. |
| **VPN** | Private network tunnel; needed to reach sky-rag. |

### Code identifiers you'll see often

| Identifier | What it is |
|---|---|
| `generateTemplateFlow` | Builds a template from a sample PDF |
| `extractChunkFactsFlow` | Extracts facts from a page chunk |
| `mergeFactsTreeStep` | Merges/dedups facts hierarchically |
| `finishSnapshotSectionFlow` | Writes the final prose + citations |
| `checkAndUpdateStaleReport` | Marks reports stuck > 15 min |
| `templateOrphanReconciler` | Fails stuck templates |
| `AllAttemptsExhaustedError` | Permanent "give up" error |
| `s3ConfigCache` | In-memory cache of prompts/config (needs a restart to refresh) |
| `CROSS_SECTION_DEDUP_ENABLED` | Feature flag for cross-section dedup |
| `IS_NA_INSTANCE` | Turns on Canada mode |
| `USE_LOCAL_PROMPTS` / `LOCAL_PROMPTS_PATH` | Load prompts from disk instead of S3 |

---

## 17. Access Checklist

| ✔ | Resource | What's needed |
|---|---|---|
| [ ] | AWS (ECR/ECS/S3/CloudWatch): dev `418272754107`, prod `717279713537` | Cluster access, doclist/settings/prompts buckets |
| [ ] | GitHub: `sky-custom-reports`, `sky-prompts`, `infra`, `usesky-api` | Write + `workflow_dispatch` |
| [ ] | Google Gemini API key, Anthropic API key | Local dev and `test:llm` |
| [ ] | GCP / Vertex (WIF) | NA instance |
| [ ] | Redis / Postgres | Local via docker-compose; dev env via DevOps |
| [ ] | VPN | sky-rag access |
| [ ] | Linear `SKY-*` | Project board |
| [ ] | Grafana | Dashboards |

---

## 18. Appendix: Key Paths and Links

### Paths

| Area | Path |
|---|---|
| API controllers (v3/v4 current) | `api/src/{templates,reports,report-evaluations,previews,extractions,analytics,health}/` |
| Core API client | `api/src/core-api/core-api.client.ts` |
| Section pipeline | `workers/src/genkit/flows/report-section/` |
| Prompt wrapper / loader | `workers/src/genkit/prompts/prompt.ts`, `workers/src/infra/promptLoader.ts` |
| Metrics | `workers/src/infra/metrics.ts` |
| Pricing | `workers/src/infra/modelPricingTable.ts` |
| Vertex auth | `workers/src/infra/vertexAuth.ts` |
| Queue contract (retries) | `packages/queues/src/job-options.ts` |
| Eval | `workers/src/eval/{chronology,report}/`, `workers/scripts/eval/`, `scripts/eval/report-fidelity/` |
| Design docs | `docs/stuck-processing-remediation-plan.md`, `docs/preserve-templatesection-identity-plan.md`, `workers/TEMPLATE_GENERATION.md`, `VERIFICATION.md` |
| CI/CD | `.github/workflows/` |

### Links

- **Grafana (prod):** https://grafana.sky-tech.ai:8843/dashboards/f/ff3a7eddosjy8f/?orgId=1
- **Bull Board:** `https://<api-host>/admin/queues`
- **Swagger:** `https://<api-host>/api`
- **Token analytics:** `GET /admin/analytics/token-usage`
- **Prompts:** `sky-prompts` → `prompts/custom-reports/` (+ `config.json`)
- **Genkit docs:** https://genkit.dev/docs/js
