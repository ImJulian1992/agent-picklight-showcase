# agent-picklight — Case Study
 
> **Production warehouse operations platform built on SAP Business One, with a Claude-powered conversational agent on Microsoft Teams. Handles the full release-to-delivery lifecycle — from pick-list generation with FEFO batch/bin allocation to delivery note creation and route dispatch.**
 
This repository is a **public case study** of a production system. The source code remains private as internal intellectual property of the operating company. This document describes the architecture, design decisions, and technical highlights of the system as it runs in production today.
 
![Python](https://img.shields.io/badge/python-3.11-blue) ![FastAPI](https://img.shields.io/badge/FastAPI-009688) ![SAP B1](https://img.shields.io/badge/SAP-Business%20One-0FAAFF) ![Claude](https://img.shields.io/badge/Claude-Sonnet%204.6-D97706) ![Status](https://img.shields.io/badge/status-production-brightgreen) ![Code](https://img.shields.io/badge/source-private-lightgrey)
 
---
 
## Context
 
A food distribution company operating daily out of a central warehouse needed to replace a manual, spreadsheet-driven release process with a system that could:
 
- Interpret natural-language operational requests from the logistics team
- Integrate bidirectionally with SAP Business One as the source of truth
- Enforce business rules (route zones, batch expiry, stock commitments) that previously lived in people's heads
- Produce all downstream artefacts automatically (warehouse export files, delivery notes, last-mile route dispatch)
- Do all of the above without ever making an irreversible change without explicit human confirmation
The result is a FastAPI service with a Microsoft Teams bot frontend, backed by Claude Sonnet 4.6, running daily in production.
 
---
 
## What the operator experiences
 
A logistics manager opens Microsoft Teams and writes *"libera los pedidos de Madrid para mañana"*. The agent:
 
1. Identifies candidate orders from SAP, filtered by the route zones configured for the target weekday
2. Applies FEFO (First Expired First Out) batch selection using real-time bin-level stock
3. Checks existing active picklists to avoid double-committing stock already reserved for orders pending pickup
4. Presents a preview with a full breakdown of included and omitted orders and their reasons
5. Waits for explicit confirmation
6. On confirmation, creates the picklist in SAP, generates four warehouse export files (Pick-to-Light, locations, distribution, manual template), uploads them to SharePoint, and notifies the operations channel
The same agent continues through Pick-to-Light confirmation processing, delivery note generation with proportional batch allocation for incidences, and route dispatch to Circuit with per-route driver assignment.
 
---
 
## Architecture
 
```
                     ┌──────────────────────┐
                     │  Microsoft Teams     │
                     │  (Operations channel)│
                     └──────────┬───────────┘
                                │ Bot Framework
                     ┌──────────▼───────────┐
                     │   FastAPI service    │
                     │   (Dockerised)       │
                     └──────────┬───────────┘
                                │
        ┌────────────┬──────────┼──────────┬────────────┬─────────────┐
        │            │          │          │            │             │
        ▼            ▼          ▼          ▼            ▼             ▼
  ┌──────────┐ ┌──────────┐ ┌────────┐ ┌────────┐ ┌──────────┐ ┌────────────┐
  │  Claude  │ │   SAP    │ │  SQL   │ │Postgres│ │  Circuit │ │ MS Graph   │
  │Sonnet4.6 │ │ Service  │ │ Server │ │Managed │ │   API    │ │ (SharePoint│
  │          │ │  Layer   │ │        │ │        │ │          │ │  + Teams)  │
  └──────────┘ └──────────┘ └────────┘ └────────┘ └──────────┘ └────────────┘
```
 
**Stack:** FastAPI · Python 3.11 · Docker · SAP Business One Service Layer · SQL Server · PostgreSQL (managed) · Anthropic Claude Sonnet 4.6 with tool_use · Microsoft Bot Framework · Microsoft Graph API · Circuit/Spoke API · MiniLM for semantic intent classification
 
---
 
## Conversational agent
 
The Teams bot is built on Claude Sonnet 4.6 with structured tool use, exposing 22 tools grouped in five families:
 
| Family | Responsibility |
|---|---|
| **Release & revert** | Query candidate orders, release picklists, revert failed or incorrect operations |
| **Circuit routing** | Retrieve drivers, compute routes for a date, create Circuit routes with driver assignment |
| **File exports** | Generate and expose Pick-to-Light, locations, distribution and manual-template CSVs |
| **Pick-to-Light processing** | Read PxL confirmations from SharePoint, process per document number, confirm lines, generate delivery notes |
| **Knowledge & audit** | Semantic search over operational knowledge base, persistent audit log |
 
Every destructive tool requires two-step confirmation before any write to SAP or external system.
 
---
 
## Design highlights
 
### Safety-first write model
 
Every destructive tool supports an `execute` parameter that defaults to `false`. The first invocation produces a preview and issues a confirmation token with a 5-minute time-to-live. A second invocation with `execute=true` must consume that token. No write to SAP happens without an explicit, scoped human confirmation. This mirrors patterns used in infrastructure-as-code tools (`terraform plan` / `terraform apply`) and brings the same discipline to warehouse operations.
 
### Security enforced in code, not in prompts
 
A common mistake in LLM agent design is to rely on the model's instruction-following to prevent dangerous actions. This system does not:
 
- **Rate limiting** is enforced at the FastAPI layer (1 req/sec and 20 req/min per user), returning HTTP 429 regardless of what the LLM intends
- **The confirmation token lifecycle** is managed outside the LLM context — tokens are issued, stored and validated by the application, not by the model
- **A static allowlist in code** defines which tools are destructive, independent of anything the model proposes
- **Pending-confirm bypass:** when a user has an open confirmation token, all subsequent messages skip the intent fast path and go directly to the LLM. This prevents a casual *"ok"* from triggering an unintended operation confirmation
### Intent fast path (zero-token responses)
 
Frequent messages like greetings (*"hola"*, *"buenos días"*) and acknowledgements (*"ok"*, *"gracias"*, *"perfecto"*) are matched against a curated vocabulary and answered from a predefined list, bypassing Claude entirely. This reduces latency and cost on trivial messages while preserving full LLM reasoning for operational intent. The fast path is deliberately bypassed when the user has a pending confirmation, because *"ok"* in that context must go through the LLM to avoid ambiguous confirmations.
 
### FEFO with cross-database integrity
 
Stock allocation uses strict FEFO on batch expiry, filtered to bins with physical stock greater than zero. A pre-flight check queries active picklists from prior days to cap released quantities, preventing the common bug of over-committing stock already reserved for orders awaiting agency pickup. Zone filtering uses a PostgreSQL-first pattern: zone configuration is fetched from PostgreSQL first, then passed as `IN(...)` to SQL Server, avoiding expensive cross-database joins and keeping query plans predictable.
 
### Semantic intent classifier as a secondary router
 
A local MiniLM-based sentence-transformer classifier is used as a second-tier router after the fast path. It ranks candidate tools before invoking Claude. Below a confidence threshold, control passes entirely to the LLM with the full toolset. This keeps most intents resolved deterministically while still allowing open-ended interaction.
 
### Short-term memory per user
 
Conversation history is maintained per user (maximum 10 messages, 24-hour TTL, in-memory) to give the agent short-term memory without accumulating stale context across sessions or leaking one user's state to another.
 
### Structured operational observability
 
Every bot interaction is logged to PostgreSQL with user, intent classification, tool used, and success status. This provides the raw data for feedback analysis, intent pattern mining, and operational auditing — and enables continuous improvement of the classifier based on real production usage.
 
---
 
## Four-phase operational workflow
 
**Phase 1 — Release.** Validate no active picklist exists for the target date; filter candidate orders by configured route zones for the weekday; respect any pre-assigned lots; apply FEFO at bin level; pre-flight cap quantities against active picklists to avoid SAP rejecting the release; insert picklist lines; generate and upload four warehouse CSVs to SharePoint with a unique picklist identifier suffix to prevent file collision on multiple daily releases.
 
**Phase 2 — Pick-to-Light confirmation.** Logistics manager uploads confirmed PxL CSVs to SharePoint. The agent processes them per document number using a REPLACE-never-accumulate pattern. Lines not covered by PxL offer three resolution paths: confirm at original quantity, upload a manual confirmation file, or defer to the warehouse management scanner.
 
**Phase 3 — Delivery notes.** Confirmed quantities are used to generate delivery notes in SAP, with batch allocations scaled proportionally for incidence cases. Lines deferred to SGA/PDA remain open and are excluded from the delivery note. A route summary is uploaded to SharePoint on completion.
 
**Phase 4 — Circuit dispatch.** Routes are pulled from the configured zone mapping joined against delivered orders; the agent prompts for a driver per route and creates the routes in Circuit, notifying the operations channel with zone breakdown, driver name, and a SharePoint link to the route summary.
 
---
 
## What this case study demonstrates
 
- Designing and shipping **LLM agents into production** with code-enforced safety, not prompt-enforced safety
- **Deep enterprise integration** with SAP Business One Service Layer — including write operations with real business rule enforcement
- **Multi-database orchestration** (SAP SQL Server + managed PostgreSQL) with patterns that scale and remain debuggable
- **Operational UX design** — meeting warehouse operators where they already work (Microsoft Teams) instead of forcing them to learn a new interface
- **Two-step confirmation patterns** for any destructive action, borrowed from infrastructure-as-code philosophy and applied to business operations
- **Cost-conscious LLM architecture** — fast path for trivial intent, semantic classifier for mid-confidence routing, Claude only when reasoning is required
---
 
## Repository note
 
This is a case-study repository. The source code is private and belongs to the operating company. For related public work, see my [other repositories](https://github.com/ImJulian1992?tab=repositories) and [profile](https://github.com/ImJulian1992).
 
---
 
## Author
 
[@ImJulian1992](https://github.com/ImJulian1992) — AI & Data Lead working between Spain and Mexico in the food distribution sector.
 
Connect on [LinkedIn](https://www.linkedin.com/in/julian-piedrahita/).
