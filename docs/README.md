# Autonomous Research Agent — Documentation Index

Welcome to the technical documentation for the **Autonomous Research Agent**. This repository houses an enterprise-grade, multi-agent AI research pipeline deployed on AWS ECS Fargate with automated LangGraph orchestration, multi-provider LLM routing via TensorZero, semantic caching, vector-backed long-term memory, Bedrock content safety guardrails, and adversarial red-team testing.

---

## 📚 Documentation Map

The documentation is organized into focused, modular specifications:

| Document | Primary Audience | Description |
|----------|------------------|-------------|
| [Architecture](file:///d:/ai-resarch-agent/docs/architecture.md) | Architects, All Engineers | Complete system architecture, high-level Mermaid diagrams, request lifecycle, deployment flow, and data flow. |
| [Components](file:///d:/ai-resarch-agent/docs/components.md) | Backend & AI Engineers | Deep dive into each component (`main.py`, `agents.py`, `cache.py`, `memory.py`, `eval.py`, `guardrails.py`, etc.), inputs, outputs, failure handling. |
| [AI & RAG Pipeline](file:///d:/ai-resarch-agent/docs/ai-rag.md) | AI/ML Engineers, Architects | Agent state machines, LangGraph graph flow, TensorZero LLM gateway routing, embeddings (`all-MiniLM-L6-v2`), pgvector retrieval, and LLM-as-judge evaluators. |
| [API Reference](file:///d:/ai-resarch-agent/docs/api.md) | Frontend & Integration Eng | REST API endpoints, request/response schemas, error handling, rate limiting, and curl invocation examples. |
| [Database & Storage](file:///d:/ai-resarch-agent/docs/database.md) | Backend & Data Engineers | PostgreSQL 15 schema, `pgvector` indexing (IVFFlat), Redis Streams data structures, TTL strategies, and connection pooling. |
| [Security & Guardrails](file:///d:/ai-resarch-agent/docs/security.md) | Security & SecOps Eng | AWS Bedrock Guardrails, PII masking, topic denial, prompt injection defense, API key middleware, and PyRIT red team attacks. |
| [Infrastructure & Cloud](file:///d:/ai-resarch-agent/docs/infrastructure.md) | DevOps & SRE | Terraform configuration, AWS VPC architecture, ECS Fargate services, VPC Endpoints vs. NAT Gateway cost analysis, and IAM roles. |
| [CI/CD & Deployment](file:///d:/ai-resarch-agent/docs/ci-cd.md) | DevOps & Release Eng | GitHub Actions automated workflows, multi-container Docker build matrix, ECR tagging, ECS deployment with automatic rollback. |
| [Reliability & Error Handling](file:///d:/ai-resarch-agent/docs/reliability.md) | SRE & Backend Eng | Resilience patterns, LLM fallback routing, exponential backoffs, circuit breaking, and failure scenarios. |
| [Performance & Cost](file:///d:/ai-resarch-agent/docs/performance.md) | Architects, Eng Leadership | Latency profiles, bottleneck analysis, 10x/100x traffic scaling paths, monthly AWS & LLM cost breakdown, and optimization levers. |
| [Testing & Evaluation](file:///d:/ai-resarch-agent/docs/testing.md) | QA & Test Engineers | Current testing status, unit/integration test roadmap, LLM-as-judge automated scoring, and red team adversarial testing. |
| [Design Decisions & Trade-offs](file:///d:/ai-resarch-agent/docs/design-decisions.md) | Architects, Interviewers | Architectural decision records (ADRs): why LangGraph, why pgvector, why TensorZero, why Redis Streams, alternatives evaluated. |
| [Limitations & Roadmap](file:///d:/ai-resarch-agent/docs/limitations-and-roadmap.md) | Product & Eng Leadership | Honest technical limitations, architectural debt, and phased short/medium/long-term roadmap. |
| [Local Development Guide](file:///d:/ai-resarch-agent/docs/local-development.md) | New Developers | Local workstation setup, Docker Compose environment, virtualenv, schema initialization, and running apps locally. |
| [Production Operations Guide](file:///d:/ai-resarch-agent/docs/operations.md) | SRE & On-Call Engineers | Day-to-day operations, manual rollbacks, CloudWatch/LangSmith monitoring, incident runbooks, and disaster recovery. |
| [Interview Preparation Guide](file:///d:/ai-resarch-agent/docs/interview-preparation.md) | Candidates & Interviewers | 30s/1m/2m elevator pitches, whiteboard speaking walkthrough, beginner/intermediate/advanced Q&A, and cross-questioning defenses. |

---

## 🎯 Role-Based Reading Paths

Depending on your objective, follow these curated reading paths:

### 1. New Software Engineer Joining the Project
1. Start with the root [README.md](file:///d:/ai-resarch-agent/README.md) for a project overview.
2. Read [Architecture](file:///d:/ai-resarch-agent/docs/architecture.md) to understand how requests flow through the system.
3. Follow the [Local Development Guide](file:///d:/ai-resarch-agent/docs/local-development.md) to spin up the services on your machine.
4. Dive into [Components](file:///d:/ai-resarch-agent/docs/components.md) and [API Reference](file:///d:/ai-resarch-agent/docs/api.md) before writing code.

### 2. DevOps & Infrastructure Engineer
1. Read [Infrastructure & Cloud](file:///d:/ai-resarch-agent/docs/infrastructure.md) for the complete AWS architecture and Terraform specs.
2. Review [CI/CD & Deployment](file:///d:/ai-resarch-agent/docs/ci-cd.md) to understand container builds and automated rollbacks.
3. Keep [Production Operations Guide](file:///d:/ai-resarch-agent/docs/operations.md) handy for incident runbooks and monitoring.
4. Consult [Security & Guardrails](file:///d:/ai-resarch-agent/docs/security.md) for IAM permissions and network security groups.

### 3. AI / Machine Learning Engineer
1. Read [AI & RAG Pipeline](file:///d:/ai-resarch-agent/docs/ai-rag.md) for LangGraph state machine mechanics and prompt routing.
2. Study [Testing & Evaluation](file:///d:/ai-resarch-agent/docs/testing.md) to understand LangSmith LLM-as-judge datasets and scoring metrics.
3. Review [Reliability & Error Handling](file:///d:/ai-resarch-agent/docs/reliability.md) for LLM fallback behavior and retry limits.
4. Inspect [Performance & Cost](file:///d:/ai-resarch-agent/docs/performance.md) for token usage analysis and cost reduction levers.

### 4. Technical Interviewer / Architect Reviewer
1. Review [Architecture](file:///d:/ai-resarch-agent/docs/architecture.md) and [Design Decisions & Trade-offs](file:///d:/ai-resarch-agent/docs/design-decisions.md) to assess engineering maturity.
2. Review [Limitations & Roadmap](file:///d:/ai-resarch-agent/docs/limitations-and-roadmap.md) to examine self-awareness of technical debt.
3. Study [Interview Preparation Guide](file:///d:/ai-resarch-agent/docs/interview-preparation.md) for whiteboard explanations and hard cross-questions.

---

## ⚡ System Quick Reference

```
+--------------------------------------------------------------------------------------------------+
|                                    AUTONOMOUS RESEARCH AGENT                                     |
+--------------------------------------------------------------------------------------------------+
| Client Layer        | Web UI (index.html) | REST API (FastAPI) | HTTP Client                     |
| Ingestion & Safety  | AWS Bedrock Guardrails (Content Filters + Topic Denial + PII Masking)      |
| In-Memory Caching   | Redis 7.1 Semantic Cache (Cosine similarity >= 0.92 on sentence embeddings)      |
| Job Queue           | Redis Streams (research:jobs) + Consumer Groups (research-agent-group)            |
| Agent Pipeline      | LangGraph 4-Node Directed Graph: Search -> Summarize -> Write -> Critic-Verify   |
| LLM Gateway Sidecar | TensorZero (OpenAI GPT-4o primary -> Groq Llama-3.3-70b-versatile fallback)       |
| Long-Term Memory    | PostgreSQL 15 + pgvector (384-d embeddings, IVFFlat index, similarity >= 0.88)   |
| Evaluation Tier     | LangSmith LLM-as-Judge (Relevance, Completeness, Hallucination Risk, Quality)     |
| Adversarial Testing | PyRIT Red Team Dashboard (Jailbreak, XPIA, Crescendo, Skeleton Key)              |
| Compute Platform    | AWS ECS Fargate (2048 CPU / 4096 MB memory) + Auto-scaling (1 to 5 tasks)        |
| Networking          | VPC (Public/Private subnets across 2 AZs) + 5 VPC Endpoints (No NAT Gateway)    |
| Infrastructure-Code | Terraform (~944 lines in main.tf) with S3 + DynamoDB state locking               |
| Deployment Pipeline | GitHub Actions deploy.yml (3 Docker images built, tagged with SHA, auto-rollback) |
+--------------------------------------------------------------------------------------------------+
```
