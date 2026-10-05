# Autonomous Research Agent

[![AWS ECS Fargate](https://img.shields.io/badge/AWS-ECS%20Fargate-orange?logo=amazon-aws)](https://aws.amazon.com/fargate/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.138.1-009688?logo=fastapi)](https://fastapi.tiangolo.com)
[![LangGraph](https://img.shields.io/badge/LangGraph-1.2.6-blue?logo=langchain)](https://github.com/langchain-ai/langgraph)
[![TensorZero](https://img.shields.io/badge/TensorZero-Gateway-purple)](https://tensorzero.com)
[![PostgreSQL pgvector](https://img.shields.io/badge/PostgreSQL-pgvector-336791?logo=postgresql)](https://github.com/pgvector/pgvector)
[![Redis](https://img.shields.io/badge/Redis-7.1-DC382D?logo=redis)](https://redis.io)
[![Terraform](https://img.shields.io/badge/Terraform-1.5+-7B42BC?logo=terraform)](https://www.terraform.io)

An enterprise-ready, multi-agent AI research system deployed on AWS. Give it any topic — it executes an autonomous 4-stage research and verification pipeline, checks content safety, verifies factual accuracy, stores knowledge in vector memory, evaluates quality with LLM judges, and returns structured reports in Text, PDF, or JSON format.

---

## 📑 Table of Contents

- [Overview](#overview)
- [Problem & Solution](#problem--solution)
- [Key Features](#key-features)
- [Architecture](#architecture)
- [System & Agent Flow](#system--agent-flow)
- [Technology Stack](#technology-stack)
- [Project Structure](#project-structure)
- [Local Development Quickstart](#local-development-quickstart)
- [AWS Cloud Setup & Deployment](#aws-cloud-setup--deployment)
- [Configuration & Environment Variables](#configuration--environment-variables)
- [REST API Reference](#rest-api-reference)
- [AI, RAG & Long-Term Memory](#ai-rag--long-term-memory)
- [Quality Evaluation & Red Teaming](#quality-evaluation--red-teaming)
- [Security & Compliance](#security--compliance)
- [Production Operations & Monitoring](#production-operations--monitoring)
- [Known Limitations & Roadmap](#known-limitations--roadmap)
- [Detailed Technical Documentation](#detailed-technical-documentation)

---

## 🌟 Overview

The **Autonomous Research Agent** is a production-grade AI platform that replaces shallow single-prompt research queries with a disciplined multi-agent workflow. Powered by **LangGraph**, the system coordinates specialized agents (Search, Summarize, Write, and Critic-Verify) to produce deep, structured research documents.

All LLM calls route through a **TensorZero** gateway sidecar for multi-provider routing (OpenAI GPT-4o primary with automatic failover to Groq). The platform includes semantic caching via Redis, long-term vector memory via PostgreSQL + `pgvector`, input/output safety enforcement via AWS Bedrock Guardrails, and automated red-team adversarial evaluation via PyRIT.

---

## 🎯 Problem & Solution

### The Problem
- **Hallucinations & Shallows:** Single-prompt LLM generations lack depth, invent citations, and often miss critical context.
- **Provider Vendor Lock-In:** Hardcoded SDK calls to a single LLM provider cause catastrophic outages during rate-limit throttling or provider downtime.
- **Redundant Token Spend:** Repeated queries on related topics waste expensive model context windows.
- **Safety Vulnerabilities:** AI systems deployed without deterministic input/output guardrails are susceptible to jailbreaks, prompt injections (XPIA), and PII leaks.

### The Solution
- **Multi-Agent Quality Gate:** LangGraph orchestrates a 4-agent pipeline where the CriticAgent actively verifies facts and triggers corrective search/write loops if criteria are not met.
- **Decoupled LLM Gateway:** TensorZero runs as an ECS sidecar, providing seamless fallback from OpenAI to Groq without application code modifications.
- **Tiered Memory:** Sentence-transformers semantic caching in Redis provides sub-second responses for cached queries; PostgreSQL with `pgvector` enables cross-session contextual recall.
- **Enterprise Defense-in-Depth:** AWS Bedrock Guardrails filter malicious input/output before and after agent execution, validated weekly by automated PyRIT adversarial attacks.

---

## 🚀 Key Features

- **Autonomous Multi-Agent Pipeline:** 4-node LangGraph state machine (`SearchAgent` → `SummarizeAgent` → `WriterAgent` → `CriticAgent`).
- **Dynamic Critic Feedback Loop:** Automatically re-triggers search and summarization if the Critic finds inconsistencies (up to 2 iterations).
- **Multi-Provider LLM Gateway:** TensorZero routes traffic to OpenAI GPT-4o with automatic failover to Groq (`llama-3.3-70b-versatile`).
- **Two-Tier Semantic Caching & Memory:**
  - *Tier 1 (Redis):* In-memory semantic cache using `all-MiniLM-L6-v2` embeddings (similarity threshold ≥ 0.92).
  - *Tier 2 (PostgreSQL + pgvector):* Persistent vector storage for historical research reports (similarity threshold ≥ 0.88).
- **Asynchronous Processing:** Redis Streams job queue decouples request intake from long-running agent workflows (30–90s processing time).
- **Built-in Content Safety:** AWS Bedrock Guardrails block harmful topics, hate speech, violence, and mask PII (SSN, credit cards).
- **Automated LLM-as-Judge Evaluation:** Every completed report is scored on relevance, completeness, hallucination risk, and overall quality, logged to LangSmith.
- **Adversarial Red Teaming:** Standalone PyRIT dashboard tests the system against jailbreak, XPIA, crescendo, and skeleton key attacks.
- **Multi-Format Export:** Produces clean Markdown, structured JSON, or publication-ready PDF reports via ReportLab.
- **Infrastructure as Code (IaC):** Fully reproducible AWS environment via Terraform with VPC Endpoints (eliminating expensive NAT Gateway charges).

---

## 🏗️ Architecture

```mermaid
graph TB
    subgraph "Clients"
        FE["Web Frontend<br>(index.html)"]
        CLI["API Client<br>(curl / SDK)"]
    end

    subgraph "AWS ECS Fargate Task — App"
        subgraph "App Container :8000"
            API["FastAPI API<br>Auth & Rate Limiting"]
            Worker["Background Worker<br>Redis Streams Consumer"]
            Pipeline["LangGraph Pipeline<br>4-Agent Graph"]
        end
        subgraph "TensorZero Sidecar :3000"
            TZ["TZ Gateway<br>LLM Multi-Provider Routing"]
        end
    end

    subgraph "AWS ECS Fargate Task — Red Team"
        PyRIT["PyRIT Attack Dashboard :8001"]
    end

    subgraph "AWS Managed Infrastructure"
        ALB["Application Load Balancer"]
        Redis["ElastiCache Redis 7.1<br>Queue + Cache + Session"]
        PG[("RDS PostgreSQL 15<br>+ pgvector<br>Long-Term Memory")]
        Bedrock["Bedrock Guardrails<br>Safety Filtering"]
        SM["Secrets Manager<br>Centralized Config"]
        EB["EventBridge<br>Weekly Schedule"]
    end

    subgraph "External Providers"
        OAI["OpenAI GPT-4o<br>(Primary)"]
        GRQ["Groq Llama-3<br>(Fallback)"]
        LS["LangSmith<br>Tracing & Eval"]
    end

    FE --> ALB
    CLI --> ALB
    ALB -->|":80 → :8000"| API
    ALB -->|":8001"| PyRIT

    API -->|"1. Validate Safety"| Bedrock
    API -->|"2. Push Job"| Redis
    Worker -->|"3. Consume Job"| Redis
    Worker -->|"4. Check Cache"| Redis
    Worker -->|"5. Search LTM"| PG
    Worker -->|"6. Run Agents"| Pipeline

    Pipeline -->|"LLM Requests"| TZ
    TZ -->|"Primary"| OAI
    TZ -->|"Fallback"| GRQ

    Worker -->|"7. Log Traces & Eval"| LS
    Worker -->|"8. Save Result"| Redis
    Worker -->|"9. Save Report Vector"| PG

    EB -->|"Trigger Attacks"| PyRIT
    PyRIT -->|"Test Probes"| API
```

---

## 🔄 System & Agent Flow

When a research request is received, it moves through distinct pipeline phases:

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant API as FastAPI
    participant Guard as Bedrock Guardrails
    participant Queue as Redis Streams
    participant Worker as Worker
    participant Cache as Semantic Cache
    participant Agents as LangGraph Agents
    participant TZ as TensorZero
    participant LTM as pgvector (RDS)

    User->>API: POST /research {topic, session_id}
    API->>Guard: Validate topic safety
    Guard-->>API: Status: ALLOWED
    API->>Queue: XADD job to research:jobs
    API-->>User: 202 Accepted {job_id}

    Worker->>Queue: XREADGROUP consume job
    Worker->>Cache: Semantic cache lookup
    alt Cache Hit (Similarity ≥ 0.92)
        Cache-->>Worker: Return cached report
    else Cache Miss
        Worker->>LTM: Query related historical reports
        LTM-->>Worker: Return contextual reports
        Worker->>Agents: Execute LangGraph StateGraph
        Agents->>TZ: SearchAgent: extract facts
        Agents->>TZ: SummarizeAgent: condense evidence
        Agents->>TZ: WriterAgent: draft report with LTM context
        Agents->>TZ: CriticAgent: verify consistency
        alt Critic Rejects
            Agents->>TZ: Re-search & refine (up to 2 iterations)
        end
        Agents-->>Worker: Final validated report
        Worker->>LTM: Store report with 384-d vector embedding
        Worker->>Cache: Save report in Redis cache (TTL 1hr)
    end
    Worker->>Queue: XACK job complete
    User->>API: GET /result/{job_id}
    API-->>User: 200 OK {status: "completed", report: {...}}
```

---

## 💻 Technology Stack

| Layer | Technology | Version | Purpose |
|---|---|---|---|
| **API Framework** | FastAPI | 0.138.1 | High-performance asynchronous REST API |
| **Agent Orchestration** | LangGraph | 1.2.6 | Stateful multi-agent graph with cyclical execution |
| **LLM Gateway** | TensorZero | Latest | Multi-provider routing, fallbacks, and prompt management |
| **Primary LLM** | OpenAI GPT-4o | — | Complex reasoning, drafting, and evaluation judge |
| **Fallback LLM** | Groq (`llama-3.3-70b`) | — | High-speed fallback provider |
| **Vector Embeddings** | `all-MiniLM-L6-v2` | 5.6.0 | Local 384-dimensional dense semantic embeddings |
| **Long-Term Memory** | PostgreSQL + pgvector | 15.8 | Relational storage + IVFFlat vector similarity search |
| **Cache & Queue** | Redis (ElastiCache) | 7.1 | Semantic cache, session history, and Redis Streams |
| **Content Safety** | AWS Bedrock Guardrails | Managed | Real-time input/output moderation & PII protection |
| **Observability** | LangSmith | 0.9.3 | Trace logging & automated LLM-as-judge dataset scoring |
| **Adversarial Testing** | PyRIT | 0.14.0 | Automated red-teaming (jailbreak, XPIA, crescendo) |
| **Cloud Infrastructure** | AWS ECS Fargate | — | Serverless container compute across multi-AZ VPC |
| **Infrastructure as Code**| Terraform | ~5.0 AWS | Complete automated cloud provisioning |
| **CI/CD** | GitHub Actions | — | Multi-image Docker build, push to ECR, and deployment |

---

## 📁 Project Structure

```
research-agent/
├── app/                              # Core application service
│   ├── main.py                       # FastAPI application, worker loop, API endpoints
│   ├── agents.py                     # LangGraph multi-agent graph & agent nodes
│   ├── cache.py                      # Redis semantic cache with vector similarity
│   ├── guardrails.py                 # AWS Bedrock Guardrails integration
│   ├── memory.py                     # Session memory (Redis) & Long-Term Memory (pgvector)
│   ├── queue.py                      # Redis Streams job producer & consumer
│   ├── eval.py                       # LangSmith 4-judge automated evaluation
│   ├── output.py                     # Multi-format exports (PDF, JSON, report diff)
│   ├── config.py                     # Configuration loader (AWS Secrets Manager)
│   ├── auth.py                       # API key authentication middleware
│   ├── retry.py                      # Exponential backoff retry handler
│   ├── pool.py                       # PostgreSQL asyncpg connection pool
│   └── Dockerfile                    # Multi-stage production Dockerfile
├── pyrit_dashboard/                  # Red team adversarial testing service
│   ├── main.py                       # FastAPI dashboard & attack runner
│   ├── requirements.txt              # PyRIT dependencies
│   └── Dockerfile                    # Container definition
├── tensorzero/                       # LLM routing gateway configuration
│   ├── tensorzero.toml               # Gateway config with OpenAI/Groq routes
│   └── Dockerfile                    # Gateway image definition
├── terraform/                        # Infrastructure as Code
│   └── main.tf                       # Complete AWS infrastructure (~944 lines)
├── docs/                             # In-depth technical documentation
│   ├── README.md                     # Documentation index & reading paths
│   ├── architecture.md               # Detailed architecture & sequence diagrams
│   ├── components.md                 # Deep-dive module breakdowns
│   ├── ai-rag.md                     # Agent state machine & RAG mechanics
│   ├── api.md                        # Complete API documentation
│   ├── database.md                   # Relational & vector schemas
│   ├── security.md                   # Security, Bedrock, and PyRIT testing
│   ├── ci-cd.md                      # Pipeline & deployment specs
│   ├── infrastructure.md             # AWS topology & cost analysis
│   ├── reliability.md                # Error recovery & failure modes
│   ├── performance.md                # Bottlenecks & 10x/100x scaling
│   ├── design-decisions.md           # Architecture Decision Records (ADRs)
│   ├── testing.md                    # Test strategy & LLM evaluation
│   ├── limitations-and-roadmap.md    # Known limitations & future roadmap
│   ├── local-development.md          # Local developer setup guide
│   └── operations.md                 # Production runbooks & monitoring
├── .github/workflows/
│   └── deploy.yml                    # Automated GitHub Actions deployment pipeline
├── bootstrap.bat                     # Windows Terraform state backend setup
├── bootstrap.sh                      # Linux/macOS Terraform state backend setup
├── requirements.txt                  # Python production dependencies
├── index.html                        # Web user interface
└── README.md                         # Project overview (this file)
```

---

## 💻 Local Development Quickstart

You can run the entire system locally using Docker Compose without needing AWS resources:

### 1. Clone & Set Up Python Virtual Environment
```bash
git clone https://github.com/YOUR_USERNAME/research-agent.git
cd research-agent
python -m venv venv
# Activate: .\venv\Scripts\activate (Windows) or source venv/bin/activate (Linux/Mac)
pip install -r requirements.txt
```

### 2. Launch Local Backing Services (Redis, Postgres+pgvector, TensorZero)
See the complete [Local Development Guide](file:///d:/ai-resarch-agent/docs/local-development.md) for the ready-to-run `docker-compose.local.yml`.
```bash
docker compose -f docker-compose.local.yml up -d
```

### 3. Run FastAPI Application & Worker
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

Navigate to `http://localhost:8000/health` to verify.

---

## ☁️ AWS Cloud Setup & Deployment

Follow these steps to deploy all infrastructure to AWS:

### Step 1: Configure AWS CLI
```bash
aws configure
# Region: us-east-1, Output format: json
```

### Step 2: Bootstrap S3 & DynamoDB for Terraform State
```bash
# Windows
bootstrap.bat

# Linux / macOS
chmod +x bootstrap.sh && ./bootstrap.sh
```

### Step 3: Deploy AWS Infrastructure with Terraform
```bash
cd terraform
terraform init
terraform apply -var="app_image=placeholder" -var="pyrit_image=placeholder"
```
*This provisions the VPC, Subnets, ALB, ElastiCache Redis, RDS PostgreSQL, Bedrock Guardrails, Secrets Manager, ECR repos, and ECS Fargate services.*

### Step 4: Populate Secrets in AWS Secrets Manager
Navigate to **AWS Console → Secrets Manager → `research-agent/config`** and supply your API keys:
```json
{
  "OPENAI_API_KEY": "sk-...",
  "GROQ_API_KEY": "gsk_...",
  "LANGSMITH_API_KEY": "ls__...",
  "API_KEY": "optional-custom-secret-key"
}
```

### Step 5: Trigger CI/CD Deployment
Push to your GitHub repository `main` branch. GitHub Actions will build all 3 Docker images, push them to ECR, and execute a zero-downtime deployment on ECS Fargate.

---

## ⚙️ Configuration & Environment Variables

All settings are managed via AWS Secrets Manager in production, with local environment fallbacks:

| Variable | Description | Default / Example |
|---|---|---|
| `DATABASE_URL` | PostgreSQL connection string with pgvector | `postgresql://user:pass@host:5432/db` |
| `REDIS_URL` | Redis endpoint for queue, cache, and sessions | `redis://host:6379` |
| `TENSORZERO_GATEWAY_URL`| TensorZero sidecar URL | `http://localhost:3000` |
| `OPENAI_API_KEY` | OpenAI API key for primary LLM calls | `sk-...` |
| `GROQ_API_KEY` | Groq API key for fallback routing | `gsk_...` |
| `LANGSMITH_API_KEY` | LangSmith API key for tracing & judge evals | `ls__...` |
| `API_KEY` | Client authorization key (`X-API-Key`) | Optional secret string |
| `RATE_LIMIT_PER_MINUTE` | Per-client rate limit window | `60` |
| `SEMANTIC_CACHE_TTL` | Cache expiration in seconds | `3600` (1 hour) |
| `SEMANTIC_CACHE_THRESHOLD`| Cosine similarity threshold for cache hit | `0.92` |
| `LTM_SIMILARITY_THRESHOLD`| Cosine similarity threshold for LTM exact hit | `0.88` |
| `LTM_RELATED_THRESHOLD` | Cosine similarity threshold for LTM context | `0.50` |
| `AGENT_MAX_ITERATIONS` | Max CriticAgent retry cycles | `2` |

---

## 📡 REST API Reference

All requests require the `X-API-Key` header if configured.

### 1. Submit a Research Topic
```bash
curl -X POST http://<ALB_DNS>/research \
  -H "Content-Type: application/json" \
  -H "X-API-Key: your-key" \
  -d '{
    "topic": "Gallium nitride semiconductors market 2025",
    "session_id": "sess_123",
    "output_format": "text"
  }'
```
*Returns `{"job_id": "...", "session_id": "..."}` immediately (HTTP 202).*

### 2. Poll for Job Results
```bash
curl http://<ALB_DNS>/result/<JOB_ID> -H "X-API-Key: your-key"
```
*Returns `{"status": "pending"}` while running, or the complete report with evaluation scores when finished.*

### 3. Download Formatted PDF
```bash
curl http://<ALB_DNS>/result/<JOB_ID>/pdf \
  -H "X-API-Key: your-key" \
  -o research_report.pdf
```

### 4. Fetch Report Diff vs Prior Version
```bash
curl "http://<ALB_DNS>/diff/Gallium%20nitride%20semiconductors%20market%202025" \
  -H "X-API-Key: your-key"
```

For full endpoint documentation, see the [API Reference](file:///d:/ai-resarch-agent/docs/api.md).

---

## 🧠 AI, RAG & Long-Term Memory

- **Multi-Agent State Machine:** Implemented using LangGraph. Each node (`search`, `summarize`, `write`, `verify`) transforms an immutable TypedDict state.
- **Fact-Checking Gate:** If `CriticAgent` detects hallucinations or structural deficiencies, it decrements iterations and routes back to `SearchAgent`.
- **Hybrid Retrieval:** Long-term memory performs cosine similarity search via `pgvector` to inject historical findings into `WriterAgent`'s context window.
- **Local Dense Embeddings:** Employs `all-MiniLM-L6-v2` (384 dimensions) running locally within the container, avoiding external embedding API latencies and costs.

For detailed prompts, state transitions, and retrieval algorithms, see [AI & RAG Pipeline](file:///d:/ai-resarch-agent/docs/ai-rag.md).

---

## 📊 Quality Evaluation & Red Teaming

### 1. Automated LLM-as-Judge Evaluation (LangSmith)
Every completed report is evaluated across 4 dimensions:
- **Relevance:** Topic alignment and topical focus.
- **Completeness:** Presence of Executive Summary, Key Findings, Analysis, and Conclusion.
- **Hallucination Risk:** Inverted probability of fabricated facts.
- **Overall Quality:** Depth, analytical rigor, and clarity.

Scores are pushed automatically to the LangSmith dataset `research-agent-reports`.

### 2. PyRIT Adversarial Red Team Dashboard
A dedicated microservice runs automated attacks against the system to validate safety guardrails:
- **Jailbreak Attacks:** Attempt direct safety instruction overrides.
- **XPIA (Cross-Prompt Injection):** Hides malicious instructions inside research topics.
- **Crescendo Attacks:** Multi-turn conversational escalation toward harmful content.
- **Skeleton Key:** Authority-spoofing prompts claiming executive clearance.

Access the dashboard at `http://<ALB_DNS>:8001/` or run via scheduled AWS EventBridge runs every Monday at 2:00 AM UTC.

---

## 🔒 Security & Compliance

- **AWS Bedrock Guardrails:** Synchronously scans input queries and generated reports against content moderation policies (hate, violence, sexual content, misconduct) and masks PII (SSN, credit cards).
- **Private Subnet Isolation:** Databases (RDS PostgreSQL, ElastiCache Redis) reside in private subnets with no public internet ingress.
- **VPC Endpoints:** Inter-service communication with AWS Bedrock, Secrets Manager, ECR, S3, and CloudWatch stays inside the AWS private network backbone.
- **Zero Hardcoded Secrets:** All secrets, keys, and configurations are loaded dynamically via IAM task roles from AWS Secrets Manager.

See [Security & Guardrails](file:///d:/ai-resarch-agent/docs/security.md) for full compliance specifications.

---

## 📈 Known Limitations & Roadmap

### Known Limitations
- **God-File Pattern:** `app/main.py` combines routing, worker scheduling, and rate limiting in a single module.
- **Embedding Model Redundancy:** Memory and Cache modules load separate instances of the sentence-transformers model.
- **Linear Cache Scan:** Semantic cache evaluates similarity via linear scan in Redis instead of Redis VL vector indexing.
- **No Automated Unit Tests:** CI/CD pipeline does not yet execute automated pytest unit test suites prior to deployment.

### Phased Roadmap
- **Short-Term:** Modularize `main.py`, share singleton embedding model, add pytest CI stage.
- **Medium-Term:** Migrate Redis cache to RediSearch Vector Similarity Search (HNSW); add RAGAS evaluation metrics.
- **Long-Term:** Multi-tenant organization support; dynamic tool invocation (live web search APIs); streaming token output via WebSockets.

See [Limitations & Roadmap](file:///d:/ai-resarch-agent/docs/limitations-and-roadmap.md) for detailed analysis.

---

## 📖 Detailed Technical Documentation

Explore the full production documentation in the [docs/](file:///d:/ai-resarch-agent/docs/) directory:

- 🏛️ [Architecture & System Flow](file:///d:/ai-resarch-agent/docs/architecture.md)
- 🧩 [Component Specifications](file:///d:/ai-resarch-agent/docs/components.md)
- 🤖 [AI, RAG & LLM Routing](file:///d:/ai-resarch-agent/docs/ai-rag.md)
- 🔌 [API Reference](file:///d:/ai-resarch-agent/docs/api.md)
- 💾 [Database & Data Models](file:///d:/ai-resarch-agent/docs/database.md)
- 🛡️ [Security, Bedrock & PyRIT](file:///d:/ai-resarch-agent/docs/security.md)
- 🚀 [CI/CD & Deployment](file:///d:/ai-resarch-agent/docs/ci-cd.md)
- ☁️ [Infrastructure & AWS Topology](file:///d:/ai-resarch-agent/docs/infrastructure.md)
- ⚡ [Reliability & Error Handling](file:///d:/ai-resarch-agent/docs/reliability.md)
- ⏱️ [Performance & Cost Optimization](file:///d:/ai-resarch-agent/docs/performance.md)
- ⚖️ [Architecture Decisions & Trade-offs](file:///d:/ai-resarch-agent/docs/design-decisions.md)
- 🧪 [Testing & Evaluation Strategy](file:///d:/ai-resarch-agent/docs/testing.md)
- 🛠️ [Local Development Setup](file:///d:/ai-resarch-agent/docs/local-development.md)
- 📋 [Production Operations Runbook](file:///d:/ai-resarch-agent/docs/operations.md)
- 🎓 [Comprehensive Interview Preparation Guide](file:///d:/ai-resarch-agent/docs/interview-preparation.md)

---

## 📄 License

This project is licensed under the Apache 2.0 License.