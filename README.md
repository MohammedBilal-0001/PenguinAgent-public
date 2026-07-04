# Penguin Agent
### Multi-Tenant AI Customer Service Platform (SaaS)

Production-deployed, multi-tenant SaaS platform delivering AI-powered conversational assistants for training centers, certification bodies, and educational institutions — with full tenant isolation, multi-provider LLM orchestration, and agentic workflows.

![Python](https://img.shields.io/badge/TypeScript-3178C6?style=for-the-badge&logo=typescript&logoColor=white)
![Next.js](https://img.shields.io/badge/Next.js-000000?style=for-the-badge&logo=next.js&logoColor=white)
![Convex](https://img.shields.io/badge/Convex-EE342F?style=for-the-badge)
![LangGraph](https://img.shields.io/badge/LangGraph-1C3C3C?style=for-the-badge)
![Gemini](https://img.shields.io/badge/Gemini-8E75B2?style=for-the-badge&logo=googlegemini&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=for-the-badge&logo=openai&logoColor=white)
![Vercel](https://img.shields.io/badge/Vercel-000000?style=for-the-badge&logo=vercel&logoColor=white)
![Telegram](https://img.shields.io/badge/Telegram-26A5E4?style=for-the-badge&logo=telegram&logoColor=white)

**[Live Demo](https://t.me/MOHESA_Beta_1_Bot)** · Code is private (client/production system) — this repo documents the architecture and engineering decisions.

> 📸 *Screenshots coming soon — placeholders below mark where product/dashboard visuals will go.*

---

## Table of Contents
1. [Project Overview](#project-overview)
2. [Key Engineering Challenges](#key-engineering-challenges)
3. [System Architecture](#system-architecture)
4. [Agentic AI Core](#agentic-ai-core)
5. [Multi-Tenant Architecture](#multi-tenant-architecture)
6. [RAG Pipeline](#rag-pipeline)
7. [Failover & Resilience](#failover--resilience)
8. [Design Patterns](#design-patterns)
9. [Engineering Tradeoffs](#engineering-tradeoffs)
10. [Known Gaps & Roadmap](#known-gaps--roadmap)
11. [What This Project Strengthened](#what-this-project-strengthened)

---

## Project Overview

Penguin Agent is a production-deployed, multi-tenant SaaS platform that delivers AI-powered conversational **enrollment assistants** for training centers and educational institutions.

Each tenant runs in complete isolation, with its own:

- Bot
- Prompts
- RAG knowledge base
- API keys
- Admin dashboard
- Tool configuration

The platform is built to support:

- Certification centers
- Training institutes
- Ministry deployments
- Educational organizations

A **Ministry of Education deployment** currently operates as a reference client, using a bring-your-own-key (BYOK) model.

> 📸 ![Architecture](https://github.com/MohammedBilal-0001/PenguinAgent-public/blob/main/Images/photo_2026-04-08_16-52-08.jpg "Inference example")

---

## Key Engineering Challenges

Building Penguin Agent centered on solving:

- Preventing cross-tenant data leakage across both relational and vector search systems
- Managing long-running conversations without unbounded token growth
- Supporting multiple LLM providers dynamically, without changing agent logic
- Reducing embedding costs while keeping RAG content fresh
- Handling webhook retries and duplicate events safely
- Building scalable AI workflows without dedicated always-on infrastructure
- Keeping inference costs low while preserving reasoning quality

---

## System Architecture

The platform is organized into five layers:

1. **Channel Layer** — external messaging integrations (e.g., Telegram)
2. **Event Processing Pipeline** — ingests, deduplicates, and routes events
3. **Agentic AI Core** — LangGraph-based reasoning and tool orchestration
4. **Persistence Layer** — Convex (real-time, ACID-safe storage)
5. **Frontend / Admin Layer** — Next.js dashboards

This separation keeps the following independent and swappable:

- External APIs
- AI orchestration
- Persistence
- Delivery
- Billing
- Tenant isolation

New providers, tools, and tenants can be added without modifying the core workflow. Each tenant maintains its own **separate billing configuration**.

> 📸 ![Architecture](https://github.com/MohammedBilal-0001/PenguinAgent-public/blob/main/Images/Screenshot%202026-05-16%20211246.png "Dynamic tool picker")

---

## Agentic AI Core

The AI workflow is built on **LangGraph**, using a three-node graph:

```
agent_reasoning → tool_invocation → final_communication
```

- **`agent_reasoning`** — uses a stronger model (Gemini/OpenAI) for tool selection and decision-making
- **`tool_invocation`** — executes the selected tool(s) against tenant-scoped data
- **`final_communication`** — uses a lighter model to generate the final response, reducing inference cost while maintaining response quality

This **two-model architecture** reduced operational costs while preserving reasoning performance.

> 📸 ![Architecture](https://github.com/MohammedBilal-0001/PenguinAgent-public/blob/main/Images/Screenshot%202026-05-16%20205032.png "Dynamic tool picker")

---

## Multi-Tenant Architecture

Tenant isolation is enforced at multiple independent layers:

- Webhook authentication
- Database query isolation
- RBAC enforcement
- Tenant-scoped vector search

Every tenant has fully isolated:

- Prompts
- Embeddings
- Conversations
- API keys
- Admin dashboards

Cross-tenant access is prevented at **both** the application layer and the database query layer.

> 📸 ![Architecture](https://github.com/MohammedBilal-0001/PenguinAgent-public/blob/main/Images/Screenshot%202026-05-16%20201708.png "no cross tenant")

---

## RAG Pipeline

**Retrieval approach:** the platform supports both semantic and syntactic retrieval.

**Pipeline flow:**

```
Content extraction → Chunking → Embedding generation → Vector indexing → Hybrid retrieval → Context reconstruction
```

To reduce unnecessary embedding costs:

- SHA-256 hashes detect content changes
- Unchanged documents skip re-embedding entirely

The system also reconstructs incomplete chunk groups before answer generation, reducing hallucinations caused by partial retrieval.



---

## Failover & Resilience

The platform includes:

- Provider failover
- Webhook deduplication
- Retry delivery queues
- Summarization locks
- Scheduled maintenance jobs

If the primary LLM provider fails, the workflow automatically retries using a fallback provider — users continue receiving responses without interruption. Delivery retries use **exponential backoff** to prevent duplicate AI execution.

---

## Design Patterns

### Factory Pattern — Two Implementations
Used in two independent subsystems to solve the same core problem: selecting the right concrete implementation at runtime from a configuration value, without if/else chains scattered across the codebase.

- **AgentFactory** — takes an `industryAiConfig` and returns the appropriate `IAgent` subclass (`CourseBookingAgent`, `AdvancedCourseBookingAgent`, `AppointmentAgent`, `BaseAgent`). The calling pipeline never knows which concrete type it receives.
- **LLM Provider Factory** — takes an `LLMProviderConfig` and returns a `BaseLLMProvider` instance for Gemini, OpenAI, or Local LLM Studio. All three implement `createModel()`, `bindTools()`, and `formatToolResults()`. Adding a new provider means one new class and one new case in the factory — zero changes to the agent graph or pipeline.

### Registry Pattern — Content Handlers
`ContentHandlerRegistry` holds a map keyed by `industryId_contentType`. At agent startup for a tenant, the registry resolves only the handlers for that tenant's enabled content types. At RAG processing time, the correct handler is retrieved by key — no branching, no `instanceof` checks. A new industry can register custom content-extraction logic for an existing content type without affecting any other tenant.

### Strategy Pattern — Tool Resolution
`buildAllTools()` assembles the full platform tool set (one RAG search tool per enabled content type, plus the escalation tool). `resolveToolAllowlist()` filters it down to only the tools configured for the current tenant. Two tenants can run identical LangGraph code with completely different capability sets — the strategy is a runtime decision, not a hardcoded branch.

### Repository Pattern — Convex Queries and Mutations
All data access is encapsulated in typed Convex query and mutation functions. The AI pipeline never constructs raw queries — it calls named internal functions that own their own indexing and isolation logic, keeping data-access concerns out of the agent graph.

---

## Engineering Tradeoffs

| Decision | Benefit | Tradeoff |
|---|---|---|
| Convex over Firebase | Unified real-time + vector + cron | Smaller ecosystem |
| Two-model graph | Lower inference cost | More config complexity |
| Rolling summarization | Bounded token growth | Possible context loss |
| Factory / Registry patterns | Extensible architecture | More indirection |

---

## Known Gaps & Roadmap

- KMS-based API key encryption *(highest-priority security improvement)*
- WhatsApp integration
- Unit testing expansion
- HTTP rate limiting
- Frontend component modularization

---

## What This Project Strengthened

Penguin Agent was designed as a real-world AI platform rather than a prototype chatbot, with a focus on:

- Scalable architecture
- Operational reliability
- Tenant isolation
- AI cost optimization
- Production deployment
- Maintainable system design

This project significantly deepened my experience in **AI infrastructure, backend architecture, RAG systems, agentic workflows, and distributed SaaS design.**

---
### More ScreenShots

https://github.com/MohammedBilal-0001/PenguinAgent-public/tree/main/Images
📫 **More projects & contact:** · [LinkedIn](https://www.linkedin.com/in/mohammed-bilal-7a1a2a35b) · [mohammed.bilalgr@gmail.com](mailto:mohammed.bilalgr@gmail.com)
