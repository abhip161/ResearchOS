# Architecture Decisions & Trade-offs

## Decision 1: Monolithic Service with Sidecar vs. Microservices

**Decision:** Single FastAPI application with a TensorZero sidecar, not microservices.

**Why chosen:** The four-agent pipeline is inherently sequential (Search→Summarize→Write→Critic). Splitting each agent into its own microservice would add network overhead between every agent call without providing meaningful benefits. The pipeline state is a single TypedDict passed between functions — no complex inter-service coordination is needed.

**Alternatives considered:**
- **Microservices per agent** — each agent as its own ECS service. Rejected because it adds 3 extra network hops per job, requires service discovery, and increases operational complexity for a pipeline that's naturally sequential.
- **Serverless (Lambda)** — each agent as a Lambda function. Rejected because Lambda has a 15-minute timeout, cold starts would add latency, and the sentence-transformers model (~100 MB) would bloat Lambda packages.

**Benefits:** Simple deployment, shared memory (connection pools, embedding model), low latency between agents, single log stream per job.

**Drawbacks:** `main.py` mixes API routes, worker logic, and job processing. Scaling the API independently from the worker is not possible.

**When to reconsider:** If agents need independent scaling (e.g., SearchAgent becomes I/O-bound while WriterAgent is CPU-bound), or if the team grows and different teams own different agents.

---

## Decision 2: TensorZero as LLM Gateway vs. Direct API Calls

**Decision:** All LLM calls go through TensorZero, a sidecar proxy that handles provider routing.

**Why chosen:** TensorZero provides multi-provider routing (OpenAI→Groq fallback), prompt template management, and a single point of configuration for model parameters. The application code never imports OpenAI or Groq SDKs — it just makes HTTP POST calls to localhost:3000.

**Alternatives considered:**
- **Direct OpenAI SDK calls** — simpler setup, but hardcodes provider. Rejected because switching providers would require code changes.
- **LiteLLM** — Python library for multi-provider routing. Rejected in favor of TensorZero's sidecar model because TZ also manages prompt templates and can be configured without code changes.
- **Custom LLM router** — building routing logic in the application. Rejected because TensorZero already solves this well.

**Benefits:** Provider-agnostic code, easy fallback configuration, prompt templates versioned separately from code, easy A/B testing via weight-based routing.

**Drawbacks:** Extra container (sidecar), additional network hop (localhost), dependency on TensorZero project maintenance. If TZ sidecar is down, all LLM calls fail.

**When to reconsider:** If TensorZero becomes unmaintained, or if the application needs features TZ doesn't support (e.g., streaming responses, custom middleware).

---

## Decision 3: PostgreSQL + pgvector vs. Dedicated Vector Database

**Decision:** Use PostgreSQL with pgvector extension for vector similarity search instead of a standalone vector database.

**Why chosen:** The application needs both relational queries (topic lookup, date filtering, report diffs) and vector similarity search. pgvector provides both in one database, eliminating the need to maintain and pay for a separate vector store.

**Alternatives considered:**
- **Pinecone** — managed vector database. Rejected because it adds another external dependency, another bill, and the application only stores a few thousand vectors. Pinecone's scale advantages aren't needed.
- **ChromaDB** — open-source, embeddable. Rejected because it doesn't persist across container restarts without external storage, and RDS already provides managed persistence.
- **FAISS** — Facebook's vector library. Rejected because it's in-process (no persistence), requires manual index management, and doesn't support the relational queries needed for diffs and topic lookups.
- **Weaviate/Qdrant** — full vector databases. Rejected for the same reasons as Pinecone — operational overhead for a small dataset.

**Benefits:** Single database for everything, managed by RDS, standard SQL with vector operators, no additional infrastructure cost.

**Drawbacks:** IVFFlat index is approximate (not exact), pgvector performance degrades with millions of vectors, less feature-rich than dedicated vector databases (no hybrid search, no metadata filtering in vector query).

**When to reconsider:** When the reports table exceeds 100K+ entries, or when hybrid search (text + vector) is needed, or when vector query latency becomes a bottleneck.

---

## Decision 4: Redis Streams for Job Queue vs. SQS/Celery

**Decision:** Use Redis Streams with consumer groups for async job processing.

**Why chosen:** Redis is already required for caching, sessions, and rate limiting. Using Redis Streams for the job queue avoids introducing another infrastructure component. Consumer groups provide exactly-once processing semantics and support horizontal scaling via unique consumer names.

**Alternatives considered:**
- **AWS SQS** — managed queue service. Rejected because Redis is already in the stack and SQS adds another AWS service dependency + cost.
- **Celery + Redis** — task queue framework. Rejected because it adds significant complexity (Celery worker configuration, beat scheduler) for what amounts to simple FIFO processing.
- **AWS Step Functions** — orchestrate multi-step workflows. Rejected because LangGraph already handles workflow orchestration internally.

**Benefits:** No additional infrastructure, built-in consumer groups, ACK semantics, Redis knowledge reused, hostname-based consumer names enable horizontal scaling.

**Drawbacks:** Redis queue is not as durable as SQS (no dead-letter queue, no message replay). If Redis restarts, pending unACK'd messages are lost.

**When to reconsider:** If job durability becomes critical (e.g., jobs that take hours), or if the application needs dead-letter queues, delayed messages, or FIFO ordering guarantees.

---

## Decision 5: Async Job Processing vs. Synchronous API

**Decision:** API returns a `job_id` immediately; the worker processes the job asynchronously. The client polls for results.

**Why chosen:** A research job takes 15–45 seconds (4+ LLM calls). A synchronous API would require the client to hold an HTTP connection for that duration, causing timeouts with ALB (default 60s), proxy servers, and browser fetch APIs.

**Alternatives considered:**
- **Synchronous API** — wait for pipeline to complete, return result. Rejected due to timeout issues.
- **WebSockets** — push result to client when ready. Considered but adds complexity (WebSocket server, connection management, reconnection logic).
- **Server-Sent Events (SSE)** — stream progress updates. Would provide a better UX but adds complexity to both server and frontend.

**Benefits:** API responds in <100ms, no timeout issues, natural horizontal scaling, failed jobs don't block other requests.

**Drawbacks:** Client must poll `/result/{job_id}` repeatedly, no real-time progress updates, polling interval is a UX tradeoff (fast polling = responsive but wasteful, slow polling = delayed feedback).

**When to reconsider:** When real-time progress updates become important for UX (e.g., showing "Searching... Summarizing... Writing...").

---

## Decision 6: VPC Endpoints vs. NAT Gateway

**Decision:** Use 5 VPC Interface Endpoints instead of a NAT Gateway for AWS service access from private subnets.

**Why chosen:** A NAT Gateway costs ~$32/month (fixed) plus data transfer charges. VPC Endpoints cost less for the specific AWS services the application uses (ECR, Secrets Manager, Bedrock, CloudWatch, S3).

**Benefits:** Lower monthly cost (~$36 for 5 endpoints vs. $32+ for NAT + data transfer), better security (traffic stays within AWS network), lower latency.

**Drawbacks:** Each service needs its own endpoint configuration. If the app needs to access a new AWS service, a new endpoint must be added. General internet access from private subnets is not possible (only specific AWS services).

**When to reconsider:** If the application needs outbound internet access from private subnets (e.g., calling external APIs from backend tasks), a NAT Gateway becomes necessary.

---

## Decision 7: Local Embedding Model vs. Embedding API

**Decision:** Use `all-MiniLM-L6-v2` locally instead of OpenAI Embeddings API.

**Why chosen:** Embedding is used for cache lookup, LTM storage, and LTM search — high-frequency operations that would be expensive and slow via API. Local inference eliminates per-embedding cost and external latency.

**Alternatives considered:**
- **OpenAI text-embedding-3-small** — higher quality embeddings but $0.02/1M tokens cost and 50–200ms API latency per call.
- **Cohere Embed** — similar quality, similar cost.

**Benefits:** Zero per-embedding cost, no API latency, no external dependency, works offline.

**Drawbacks:** Model loaded in memory (~100 MB per instance), lower embedding quality than OpenAI's models (384 dims vs. 1536), model duplicated in cache.py and memory.py (~200 MB total).

**When to reconsider:** If embedding quality significantly impacts search/cache accuracy, or if the RAM overhead becomes a problem.

---

## Decision 8: LLM-as-Judge Evaluation vs. Rule-Based / Human Evaluation

**Decision:** Use GPT-4o as an LLM judge to evaluate every research report across 4 dimensions.

**Why chosen:** LLM-as-judge provides nuanced evaluation (relevance, quality, hallucination detection) that rule-based methods can't match. It runs automatically on every job without human intervention.

**Alternatives considered:**
- **Rule-based evaluation** — check for section headers, word count, etc. Partially viable (completeness could be rule-based) but can't assess quality, relevance, or hallucination.
- **Human evaluation** — most accurate but doesn't scale to every request.
- **No evaluation** — reduces cost but provides no quality visibility.

**Benefits:** Automated quality monitoring, no human involvement, nuanced assessment, scores logged to LangSmith for trend analysis.

**Drawbacks:** 4 extra LLM calls per job (50% more cost), uses the same model that generated the report (circular evaluation), runs on cached results too (waste).

**When to reconsider:** If evaluation costs become prohibitive, or if a cheaper model (GPT-4o-mini) provides adequate judge quality, or if dedicated evaluation frameworks (RAGAS, DeepEval) are adopted.

---

## Decision 9: ECS Fargate vs. EC2 / Lambda / EKS

**Decision:** AWS ECS Fargate for container orchestration.

**Why chosen:** Fargate eliminates server management (no EC2 instances to patch/maintain). The application is containerized, long-running, and needs persistent connections to Redis and PostgreSQL — good fit for Fargate.

**Alternatives considered:**
- **EC2** — lower cost for sustained workloads but requires server management.
- **Lambda** — serverless, pay-per-invocation. Rejected because cold starts, 15-minute timeout, and ~100 MB embedding model make it impractical.
- **EKS** — Kubernetes, more powerful but significantly more complex to operate for a small team.

**Benefits:** No servers to manage, auto-scaling built-in, pay for what you use, simple deployment via task definitions.

**Drawbacks:** Higher cost than EC2 for sustained workloads, less flexibility than EKS, no spot instance support (Fargate Spot is available but less reliable).

**When to reconsider:** If cost optimization is critical (switch to EC2), or if the team adopts Kubernetes for other services (switch to EKS).
