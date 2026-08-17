# CodeContext AI — Intelligent Knowledge Transfer & Application Context Engine

## Innovation Proposal Submission

---

## Title

CodeContext AI — Intelligent Knowledge Transfer & Application Context Engine

---

## Selected Capability

Software Engineering

## Focus Area & Problem Statement

PS: Effectively integrate AI assistants to enhance coding productivity and quality

---

## Problem Statement

Software development teams maintaining complex, multi-service applications accumulate critical knowledge through daily activities — merge request discussions, code reviews, architectural decisions, and troubleshooting conversations. However, this knowledge remains scattered across GitLab comments, wiki pages, issue boards, and individuals' heads with no unified way to capture or retrieve it. When developers need to understand unfamiliar code, trace why a decision was made, or debug a service they don't own, they spend 1-2 hours daily searching through disconnected tools or waiting for a teammate to respond — assuming that teammate is still on the team. This creates a compounding problem: as teams grow, people rotate, and applications evolve, the gap between what the codebase does and what any individual developer understands keeps widening. The result is slower delivery, repeated mistakes, painful onboarding, and dangerous bus factor risks where entire modules become unmaintainable if one person leaves.

---

## Idea Description

CodeContext AI is an AI-powered web application that automatically captures institutional knowledge from the full GitLab ecosystem — code, merge requests, comments, issues/boards, and wiki — and makes it instantly queryable through natural language. Developers can ask questions like "Why was this module built this way?" or "Who knows about the payment service?" and get cited answers in seconds, with links back to the source MR, issue, or commit.

The system uses Retrieval-Augmented Generation (RAG) to provide accurate, context-aware answers. Data ingestion is fully automatic via GitLab webhooks and scheduled crawlers — zero manual effort from developers. It also detects bus factor risks by analyzing code ownership patterns and alerts teams when critical knowledge is concentrated in too few people.

---

## Business Value

**Persona:** Software developers and tech leads who maintain applications and frequently need to understand unfamiliar code or track down decisions made by others.

**Scale:** Can be used by any team working with GitLab. Starting with one team of 15-20 developers, scalable across BIETC.

**Dollar Value:** Developers typically spend 1-2 hours/day searching for context (reading old MRs, asking teammates, digging through docs). Even saving 30-45 minutes/day per developer across a 20-person team at $40/hr translates to roughly $8,000-10,000/month in recovered productivity. Additionally, reducing onboarding time by 2 weeks per new joiner saves around $6,000-8,000 per hire. For a single 20-person team, estimated annual value is in the range of $100,000-120,000.

---

## Technical Approach

The solution uses a Retrieval-Augmented Generation (RAG) approach with the following components:

1. **Data Ingestion:** GitLab webhooks capture code pushes, merge request events, comments, issue updates, and merge decisions in real-time. A nightly scheduled crawler performs git blame analysis to build contributor/ownership maps and re-indexes any missed data. No manual data entry required.

2. **Indexing & Storage:** Code, MR discussions, issues, and documentation are chunked and converted into vector embeddings using BCAI's embedding model, stored in PostgreSQL with the PGVector extension. A relational layer tracks people-to-code-to-decision relationships for ownership and bus factor queries.

3. **Query & Answer Engine:** When a developer asks a question, the system performs semantic search against the vector store to retrieve relevant context (code snippets, MR discussions, issue descriptions), then passes it to BCAI LLM to generate a natural language answer with source citations.

4. **User Interface:** Internal web portal (React) accessible via corporate SSO — no separate registration needed. Users log in with existing credentials; their GitLab permissions determine what repos they can query.

5. **Tech Stack:** TypeScript, Node.js, React, PostgreSQL + PGVector, BCAI (LLM + embeddings), Kubernetes for deployment. All on existing internal infrastructure — no external vendors or new licenses required.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      User Interfaces                         │
├───────────────┬──────────────────┬──────────────────────────┤
│   Web Portal  │       CLI        │  VS Code Extension (V2)  │
│   (React)     │  (terminal)      │  (sidebar)               │
└───────┬───────┴────────┬─────────┴────────────┬─────────────┘
        │                │                      │
        ▼                ▼                      ▼
┌─────────────────────────────────────────────────────────────┐
│               API Server (Node.js / Express)                 │
├─────────────────────────────────────────────────────────────┤
│  • /ask — Natural language Q&A                               │
│  • /search — Semantic code/doc search                        │
│  • /health — Bus factor & ownership reports                  │
│  • /onboard — Onboarding path generation                     │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
   ┌────────────┐  ┌───────────┐   ┌──────────┐
   │ Vector DB  │  │ Relational│   │   LLM    │
   │ (PGVector) │  │    DB     │   │  (BCAI)  │
   │            │  │(PostgreSQL)│   │          │
   └────────────┘  └───────────┘   └──────────┘
          ▲               ▲
          │               │
┌─────────────────────────────────────────────────────────────┐
│            Ingestion Pipeline (Fully Automatic)               │
├─────────────────────────────────────────────────────────────┤
│  • GitLab Webhooks → MR events, comments, pushes, issues    │
│  • Scheduled Crawlers → Wiki, git blame, bulk re-indexing   │
└─────────────────────────────────────────────────────────────┘
```

### Data Sources (all from GitLab):

| Source | Trigger | What Gets Indexed |
|--------|---------|-------------------|
| Code pushes | Webhook (real-time) | File changes, commit messages, author |
| Merge requests | Webhook (real-time) | Title, description, discussions, reviews, approvals |
| Issues/Boards | Webhook (real-time) | Ticket descriptions, comments, status, linked MRs |
| Wiki | Scheduled crawler (every 4hrs) | Documentation pages |
| Git blame/ownership | Nightly batch job | Line-by-line contributor history |

---

## User Flow

### Setup (one-time, by team admin):
1. Admin opens web portal → "Add a Group"
2. Enters GitLab group name (e.g., `vast`)
3. System generates a webhook URL
4. Admin adds webhook in GitLab group settings
5. Indexing starts automatically

### Registration:
No manual registration. Users authenticate via corporate SSO. Their existing GitLab permissions determine what repos they can query.

### Daily usage:
Developer opens the web portal → types a question → gets an answer with source citations. Like Google, but for your codebase.

---

## Existing Solutions

| Solution | Gap CodeContext Fills |
|----------|---------------------|
| GitLab built-in search | Keyword-only, no semantic understanding, no cross-referencing with decisions |
| Confluence/Wiki | Manually written, always outdated, not code-aware |
| Asking teammates on Slack | Depends on availability, knowledge leaves when they leave |
| GitHub Copilot / AI assistants | Suggest code, but don't explain decisions or team history |

None of these automatically capture and connect knowledge from development workflows.

---

## Success Criteria

1. MVP indexes at least one GitLab group (all repos, MRs, issues, comments) without manual intervention
2. Developers get relevant, cited answers at least 75-80% of the time (validated through user feedback during 2-week trial)
3. System correctly identifies code ownership and bus factor risks
4. At least 5 developers from pilot team use it 3+ times/week and report time savings
5. Simulated onboarding test: a developer unfamiliar with a module answers 5 questions about it using CodeContext without needing to ask a teammate

---

## Infrastructure Requirements

No new hardware or paid software licenses required. The solution runs on existing internal infrastructure:

- BCAI access (LLM + embedding model APIs — already available internally)
- PostgreSQL with PGVector extension (free, open-source extension on existing DB)
- Kubernetes namespace (existing cluster)
- GitLab webhook permissions for target group

If PGVector cannot be enabled on existing instances, alternative is self-hosted Qdrant (open-source, no license cost) on the same cluster.

---

## Feasibility for Product Development

**8-10 person months** to scale MVP into a full product.

The team has strong experience with the core application stack (TypeScript, Node.js, React, PostgreSQL, GitLab APIs), making the API layer, web UI, and data ingestion pipeline low-risk. However, two areas require experimentation early in the MVP phase: (1) Vector DB tuning — finding the right chunking strategy for code vs. MR discussions, validating PGVector query performance at scale with real repo data, and (2) LLM integration — prompt engineering for code-context Q&A, evaluating BCAI model quality for technical queries, and handling token limits effectively. The MVP phase includes a dedicated 1-2 week spike for these experiments before committing to the full pipeline. No external vendor dependencies — everything runs on existing BCAI, Kubernetes, and PostgreSQL infrastructure, with Azure OpenAI as a fallback if needed. Incremental delivery across phases ensures each stage produces working functionality, reducing overall risk even if later phases are descoped.

---

## MVP Development Cost

**Estimated: $20,000**

| Item | Cost |
|------|------|
| Development effort (2 devs × 6 weeks × 40 hrs × $40/hr) | ~$19,200 |
| BCAI usage during development/testing | ~$500 |
| Miscellaneous (testing, docs, demo) | ~$300 |

Infrastructure cost: $0 (existing Kubernetes, PostgreSQL, BCAI)
Software licenses: $0 (all open-source components)

---

## Phased Delivery

| Phase | Duration | Deliverable |
|-------|----------|-------------|
| MVP | 6 weeks | CLI + Web portal, single group indexing, Q&A, bus factor detection |
| V1 | 6 weeks | Multi-group, real-time webhooks, issue board integration, onboarding paths |
| V2 | 8 weeks | VS Code extension, living documentation, sprint summaries, advanced analytics |

---

## ROI Summary

| Metric | Value |
|--------|-------|
| MVP cost | $20,000 |
| Annual infrastructure cost | ~$0-1,200 |
| Annual savings (20-person team) | ~$100,000-120,000 |
| First-year ROI | ~5x return |
| Payback period | ~2-3 months |

---

*Team: CodeContext*
*Capability: Software Engineering*
*Contact: [Your Name]*
*Organization: BIETC*
*Date: July 2026*
