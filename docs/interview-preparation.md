# Interview Preparation — Research Agent

## "Tell Me About Your Project"

### 30-Second Answer

> I built an autonomous research agent deployed on AWS. You give it a topic, and a multi-agent pipeline — four specialized AI agents orchestrated with LangGraph — researches, summarizes, writes, and fact-checks a structured report. All LLM calls go through a TensorZero gateway for multi-provider routing with automatic failback to Groq if OpenAI goes down. I added semantic caching with sentence-transformers, long-term memory with pgvector for cross-session context, content safety guardrails with AWS Bedrock, automated LLM-as-judge evaluation on every report, and a PyRIT red team dashboard that attacks the system weekly to test guardrail effectiveness. Everything is deployed on ECS Fargate with Terraform and CI/CD via GitHub Actions with automatic rollback.

### 1-Minute Answer

> I designed and built an autonomous research agent that produces professional-quality research reports. The core is a multi-agent pipeline built with LangGraph: a SearchAgent finds key facts, a SummarizeAgent condenses them, a WriterAgent produces a structured report, and a CriticAgent verifies quality — if the critic rejects the report, the pipeline loops back for another pass.
>
> All LLM calls route through a TensorZero gateway sidecar that handles multi-provider routing — GPT-4o primary with Groq as an automatic fallback. This means the application code never directly calls any LLM provider; switching providers is a config change, not a code change.
>
> For performance, I built a semantic cache using sentence-transformers embeddings in Redis — if a similar topic was researched recently, it skips the entire pipeline. For long-term context, I use PostgreSQL with pgvector: the WriterAgent receives related previous reports so it builds on existing research instead of starting from scratch.
>
> Every report is automatically evaluated by four LLM-as-judge evaluators that score relevance, completeness, hallucination risk, and overall quality, with scores logged to LangSmith. For content safety, I use AWS Bedrock Guardrails on both input and output, and a PyRIT red team dashboard runs jailbreak, XPIA, crescendo, and skeleton key attacks weekly to validate the guardrails.
>
> The infrastructure is fully IaC with Terraform — VPC, ECS Fargate, RDS PostgreSQL, ElastiCache Redis, ALB, Bedrock Guardrails, ECR, and VPC Endpoints — with CI/CD through GitHub Actions that builds three Docker images and deploys with automatic rollback on failure.

### 2-Minute Detailed Answer

> The Research Agent is an autonomous multi-agent system I designed to solve the problem of producing comprehensive, fact-checked research reports on any topic. Instead of a single LLM prompt, I built a four-agent pipeline where each agent specializes in one stage of the research process.
>
> **Architecture:** The core is a LangGraph StateGraph with four agents — SearchAgent, SummarizeAgent, WriterAgent, and CriticAgent. They communicate via a shared state object, not direct calls. The CriticAgent is the quality gate: if it rejects the report for factual inconsistency, the pipeline loops back to SearchAgent for another iteration, up to a configurable maximum.
>
> **LLM Routing:** I deliberately avoided hard-coding any LLM provider into the application. Every LLM call goes through a TensorZero gateway running as a sidecar container. TensorZero handles provider routing — OpenAI GPT-4o as primary, Groq as fallback. I chose this architecture because it means I can swap models, add providers, or change routing without changing a single line of application code.
>
> **Memory & Caching:** The system has three memory tiers. First, a semantic cache in Redis using local sentence-transformers embeddings — if a semantically similar query was processed recently, it returns the cached result, saving 4+ LLM calls. Second, long-term memory in PostgreSQL with pgvector — reports are stored with 384-dimensional vector embeddings. The WriterAgent receives a related (but not identical) previous report as context, so it builds on prior research. Third, session memory in Redis maintains conversation context across follow-up queries.
>
> **Quality & Safety:** Every report goes through automated quality assessment — four LLM-as-judge evaluators score relevance, completeness, hallucination risk, and quality, with all scores logged to LangSmith for trend analysis. For content safety, I use AWS Bedrock Guardrails on both input and output, configured with content filters, topic denials, PII blocking, and prompt attack detection. I also built a PyRIT red team dashboard — a separate ECS service that runs jailbreak, cross-prompt injection, crescendo, and skeleton key attacks against the production API weekly via EventBridge.
>
> **Infrastructure:** Everything is defined in Terraform — VPC with public and private subnets, ECS Fargate with auto-scaling (1–5 instances based on CPU), RDS PostgreSQL with pgvector, ElastiCache Redis, Application Load Balancer, VPC Endpoints instead of NAT Gateway to save costs. The CI/CD pipeline in GitHub Actions builds three Docker images (app, TensorZero, PyRIT), pushes to ECR, deploys to ECS, and automatically rolls back to the previous task definition if deployment fails.
>
> **Key engineering decisions:** I used VPC Endpoints instead of NAT Gateway, saving ~$32/month. I chose pgvector over a standalone vector database because the application needs both relational queries and vector search on the same data. The async job queue uses Redis Streams with consumer groups, which supports horizontal scaling via unique consumer names per ECS task. I pre-download the embedding model during Docker build to eliminate runtime downloads and set HF_HUB_OFFLINE=1 for deterministic builds.

---

## Architecture Whiteboard Walkthrough

When explaining the architecture, draw from left to right:

### Step 1: Start with the user

> "The user interacts through either the web frontend or the REST API. They submit a research topic — let's say 'AI chip market 2025'."

### Step 2: The API gateway layer

> "The request hits an Application Load Balancer, then reaches the FastAPI app. Here three things happen immediately: API key authentication, per-IP rate limiting using Redis, and input validation through AWS Bedrock Guardrails — which checks for harmful content, PII, and prompt injection attacks."

**Likely follow-up:** "How do Bedrock Guardrails work?" → "They're a managed rule-based service, not an LLM call. I configured content filters for hate, violence, and misconduct at HIGH threshold, topic denials for weapons and self-harm, and PII detection for SSNs and credit cards. The key point is this doesn't consume LLM quota."

### Step 3: The async job queue

> "Once validated, the topic is pushed to a Redis Streams job queue and the API immediately returns a job_id. The client polls for results. I chose async processing because the pipeline takes 15–45 seconds — holding an HTTP connection that long would cause timeouts."

**Likely follow-up:** "Why Redis Streams over SQS?" → "Redis is already in the stack for caching and sessions. Redis Streams give me consumer groups with ACK semantics, so horizontal scaling works — each ECS instance uses its hostname as a unique consumer name."

### Step 4: Cache and memory lookup

> "The worker first checks two memory layers. The semantic cache in Redis compares the query embedding against cached entries using cosine similarity. If a similar topic was researched within the last hour, it returns the cached result — zero LLM calls. If no cache hit, it checks long-term memory in PostgreSQL with pgvector for a report on a near-identical topic from the last 7 days."

### Step 5: The multi-agent pipeline

> "If nothing is cached, the four-agent pipeline runs. SearchAgent finds 5 key facts — it also receives the last 4 conversation turns for context awareness. SummarizeAgent condenses the results into structured bullet points. WriterAgent produces the final report — critically, it also receives a related previous report from long-term memory so it builds on existing knowledge. CriticAgent fact-checks the report. If the critic rejects it, the pipeline loops back to SearchAgent. Maximum 2 retries."

**Likely follow-up:** "How do agents communicate?" → "Through a shared state object — a TypedDict with fields for topic, search_results, summaries, report, verified, and iterations. No direct agent-to-agent calls. LangGraph manages the execution order through a StateGraph with conditional edges."

### Step 6: The LLM gateway

> "Every LLM call goes through TensorZero, a sidecar container running on the same ECS task. The application just makes HTTP POST calls to localhost:3000. TensorZero handles provider selection — GPT-4o as primary, Groq as automatic fallback. This is deliberate: I separated LLM routing from application logic so I can change providers, models, or routing without code changes."

### Step 7: Output and evaluation

> "After the report passes the output guardrail, it's stored in Redis cache and PostgreSQL long-term memory, then returned to the user in their chosen format — text, PDF via ReportLab, or structured JSON. In parallel, four LLM-as-judge evaluators score the report on relevance, completeness, hallucination risk, and quality. Results go to LangSmith."

### Step 8: Infrastructure and deployment

> "Everything runs on ECS Fargate — no servers to manage. The app task has the main container and TensorZero sidecar, auto-scales 1–5 based on CPU. Data lives in private subnets: RDS PostgreSQL with pgvector for permanent storage, ElastiCache Redis for transient data. I use VPC Endpoints instead of NAT Gateway — saves about $32/month. CI/CD is GitHub Actions: push to main builds 3 Docker images, pushes to ECR, deploys to ECS with automatic rollback."

---

## Interview Questions & Answers

### Beginner Level

**Q: What problem does this project solve?**

> It automates the process of researching a topic and producing a professional report. Instead of spending hours reading and synthesizing information, a user submits a topic and gets a structured report with executive summary, key findings, analysis, and conclusion — fact-checked and quality-scored automatically.

**Q: What technologies did you use?**

> Python 3.12 with FastAPI for the API, LangGraph for multi-agent orchestration, TensorZero for LLM routing, OpenAI GPT-4o as the primary model with Groq as fallback, PostgreSQL with pgvector for long-term memory, Redis for caching and job queue, AWS Bedrock for content safety, PyRIT for red team testing, Terraform for infrastructure, and GitHub Actions for CI/CD.

**Q: What is your role?**

> I designed the architecture, built the entire application and infrastructure, and deployed it to AWS. Every component — from the agent pipeline to the Terraform infrastructure to the CI/CD pipeline — was built by me.

---

### Intermediate Level

**Q: Why did you choose LangGraph over other frameworks?**

> LangGraph gives me explicit control over the agent execution graph. I needed a specific sequential pipeline with a conditional retry loop — SearchAgent → SummarizeAgent → WriterAgent → CriticAgent with a conditional edge back to SearchAgent if the critic rejects. LangGraph's StateGraph maps directly to this pattern. Alternatives like CrewAI or AutoGen are more opinionated about how agents communicate, and I wanted the agents to share state through a simple TypedDict rather than complex messaging.

**Q: Why did you choose PostgreSQL with pgvector over a dedicated vector database?**

> I need both relational queries and vector search on the same data. For example, I query reports by exact topic match for diffs, by creation date for recency, and by vector similarity for semantic search — all on the same table. A dedicated vector database like Pinecone would handle the similarity search well, but I'd need a separate relational database for everything else. At my current scale — hundreds to thousands of reports — pgvector is more than sufficient, and one managed RDS instance is simpler and cheaper than two databases.

**Q: How does your caching work?**

> I have a semantic cache that uses sentence-transformers embeddings. When a new query comes in, I encode it with all-MiniLM-L6-v2, then compare it against cached embedding vectors using cosine similarity. If the similarity exceeds 0.85, I return the cached result and skip the entire agent pipeline — saving 4 to 12 LLM calls. The cache has a 1-hour TTL. The important thing is that this is a semantic cache, not an exact-match cache — "AI chip market 2025" and "2025 artificial intelligence chip industry" would match because their embeddings are similar.

**Q: How is authentication implemented?**

> Simple API key authentication. The API key is stored in AWS Secrets Manager and loaded at application startup. Every protected endpoint checks the `X-API-Key` header. If the key isn't set in Secrets Manager, authentication is disabled — this is intentional for development but should always be set in production. It's straightforward but fits the current needs. If I needed per-user access, I'd move to JWT-based authentication.

---

### Advanced Level

**Q: What is the biggest scalability bottleneck?**

> The semantic cache has an O(n) lookup. It scans every cached embedding in Redis and computes cosine similarity against each one. At 10 entries it's negligible, but at 1,000+ entries it takes seconds. The fix is to use a proper vector index — either Redis Vector Search module or move cache lookups to pgvector. The second bottleneck is unbounded worker concurrency — there's no semaphore limiting how many jobs run simultaneously, so a burst of requests can trigger hundreds of concurrent LLM calls and hit rate limits.

**Q: How would you handle 100x traffic?**

> Several things would break. First, the O(n) cache scan — I'd replace it with a proper vector index. Second, worker concurrency — I'd add an asyncio.Semaphore to cap concurrent jobs at maybe 5–10 per instance. Third, I'd hit OpenAI rate limits — I'd either upgrade my API tier or add model routing to use GPT-4o-mini for simpler tasks like search and summarize. Fourth, Redis would need scaling — either upgrade to a larger ElastiCache instance or add replication. Fifth, RDS might need an upgrade for the increased write throughput. Sixth, I'd stop running evaluation on every job and instead sample — maybe evaluate every 10th job. The infrastructure would actually handle it with auto-scaling, but the cost would become the real issue at 100x.

**Q: What happens if the LLM provider goes down?**

> TensorZero automatically fails over to Groq. The application doesn't even know this happened — it still makes the same HTTP call to localhost:3000. The risk is that Groq has a monthly quota limit on the free tier (8K requests). If OpenAI goes down for an extended period and jobs keep arriving, Groq quota gets consumed fast because each job makes 8–16 LLM calls. I'd mitigate this by adding circuit breaking — after N consecutive failures, stop accepting new jobs and queue them for later — and by adding usage tracking to monitor quota consumption.

**Q: How would you reduce hallucinations?**

> Currently I address hallucination at four levels: low temperature (0.3) for factual tasks, system prompts that explicitly instruct "do not hallucinate", the CriticAgent that checks factual consistency and triggers retries, and the eval_hallucination judge that scores hallucination risk. The biggest improvement would be adding external search — right now the agents use only the LLM's internal knowledge. Integrating a search API like Serper or Tavily would ground the search results in real-time data. I'd also consider using a different model for the critic than for generation, so it brings independent knowledge to fact-checking.

**Q: How would you reduce LLM costs?**

> The easiest win is to stop running evaluation on cache and LTM hits — that's 4 unnecessary LLM calls per cached job. Next, I'd use task-specific model routing through TensorZero: GPT-4o-mini is 15x cheaper than GPT-4o and good enough for SearchAgent, SummarizeAgent, CriticAgent, and evaluation judges. I'd reserve GPT-4o only for the WriterAgent where report quality matters most. I'd also make the completeness evaluator rule-based — it just checks for section headers, which doesn't need an LLM. Extending the cache TTL from 1 hour to 24 hours would increase cache hit rates. In total, these changes could reduce LLM costs by 60–70%.

---

### Deep-Dive / Cross-Questioning

**Q: You mentioned TensorZero routes to Groq on OpenAI failure. What happens if both providers fail?**

> The `_tz_call` function retries 3 times with exponential backoff (1s → 2s → 4s). If all retries fail, the exception propagates to the job processor, which catches it and sets the job status to "error" with the error message. The user sees a failed job. This is the correct behavior — there's no way to generate a research report without an LLM. The improvement I'd make is adding a circuit breaker: after, say, 5 consecutive failures, the system should stop accepting new jobs and return a "service temporarily unavailable" response rather than queueing jobs that will fail.

**Q: Your retry mechanism retries all exceptions. Isn't that wasteful?**

> Yes, that's a known issue. The retry catches everything — including 401 authentication errors and 400 bad request errors that will never succeed no matter how many times you retry. Each retry is another LLM call that consumes quota. The fix is error classification: only retry 500-level errors and timeouts, immediately fail on 4xx errors, and for 429 rate limits, respect the `Retry-After` header instead of using fixed exponential backoff. This is a low-effort, high-impact improvement.

**Q: You said the CriticAgent uses the same model as the WriterAgent. Isn't that circular?**

> That's a valid criticism. When GPT-4o writes a report and GPT-4o evaluates it, the evaluator shares the same knowledge gaps and biases. It's like grading your own homework. The CriticAgent is still useful because it catches structural issues — missing sections, logical contradictions, obvious fabrications — but it won't catch factual errors that are consistent with the model's training data. To improve this, I'd either use a different model for the critic (e.g., Claude), or better yet, add external verification — cross-reference claims against a search API.

**Q: Why don't you have tests?**

> That's the biggest gap in the project, and I'm honest about it. The focus was on building a working system with production deployment, and I prioritized getting the agent pipeline, caching, memory, guardrails, evaluation, and infrastructure right. The test framework isn't even installed — no pytest in requirements. If I were continuing this project, the first thing I'd add is unit tests for the critical path: score parsing, cache operations, auth validation, retry behavior, and orchestrator routing logic. Then integration tests for the full job lifecycle. The LLM-as-judge evaluation provides some regression detection, but it's not a substitute for proper automated tests.

---

## Questions That Could Expose Weaknesses

**Q: Isn't this over-engineered for a research tool?**

> You could argue that a single LLM prompt would produce a report. The multi-agent architecture provides two concrete benefits: specialization (different temperatures and system prompts per task) and the quality gate (CriticAgent that triggers retries). The caching and LTM layers are what make it production-viable — without them, every identical query burns 4+ LLM calls. The TensorZero gateway and PyRIT dashboard are arguably the most "extra" parts, but TensorZero saved me from hard-coding provider logic, and PyRIT demonstrates security awareness. I'd say the core pipeline justifies the complexity; the surrounding infrastructure is what makes it production-grade rather than a prototype.

**Q: Why not microservices?**

> The pipeline is sequential — SearchAgent output feeds SummarizeAgent, which feeds WriterAgent, which feeds CriticAgent. Splitting each agent into a microservice adds 3 network hops per job with no scaling benefit because they can't run in parallel anyway. The monolithic approach is correct here. If I needed to scale the API independently from the pipeline — say, heavy read traffic on `/result` while pipeline processing is slow — I'd split the API from the worker, not the agents from each other.

**Q: How do you prevent duplicate job processing?**

> Redis Streams consumer groups with ACK semantics handle this. Each consumer group ensures each message is delivered to exactly one consumer. The consumer uses the hostname as its unique name, so when multiple ECS task instances consume from the same stream, each job goes to exactly one instance. Jobs are only removed from the pending list after explicit `XACK` in the `finally` block of `_process_job`.

**Q: How do you handle concurrency and race conditions?**

> The main concurrency concern is the unbounded `asyncio.create_task` in the worker loop. If 100 jobs arrive simultaneously, 100 concurrent tasks run, each making 8+ LLM calls. This could overwhelm the LLM provider with rate limits. The fix is straightforward — an `asyncio.Semaphore` to cap concurrent jobs. For data races, the architecture naturally avoids most issues: each job has a unique `job_id` key in Redis, each report gets a unique UUID in PostgreSQL, and Redis operations are atomic.

**Q: What's your most important architecture trade-off?**

> Using the same GPT-4o model for everything — pipeline, evaluation, and critic. The benefit is simplicity: one model configuration in TensorZero, consistent quality. The cost is literally cost — evaluation adds 50% more LLM calls, and the critic uses the same expensive model for a YES/NO decision that a cheaper model could handle. The platform is already designed to fix this — I just need to add more TensorZero function definitions and model configs, which is a configuration change, not a code change. That's the payoff of the TensorZero abstraction.

---

## Key Talking Points

When discussing this project, always hit these points:

1. **Multi-agent pipeline with quality gate** — not a single prompt, but specialized agents with CriticAgent as a quality checkpoint
2. **Provider-agnostic LLM routing** — TensorZero sidecar abstracts providers; switching models is config, not code
3. **Three-tier memory** — semantic cache (short-term), session memory (conversation), long-term memory (permanent + vector search)
4. **Production safety** — Bedrock Guardrails (input + output), PyRIT red team testing (automated weekly)
5. **Automated quality measurement** — LLM-as-judge on every report, scores in LangSmith
6. **Full IaC** — Terraform for all infrastructure, GitHub Actions CI/CD with rollback
7. **Cost-conscious decisions** — VPC Endpoints over NAT Gateway, local embedding model, semantic caching to avoid redundant LLM calls

---

## What I Learned Building This Project

### Architecture
- A multi-agent pipeline with shared state (TypedDict) is simpler and more debuggable than agents passing messages to each other
- The sidecar pattern for LLM routing is extremely clean — separates concerns without adding network latency
- Async job processing with Redis Streams is the right pattern for long-running LLM jobs

### AI Engineering
- Every LLM call in an AI system should go through a single function — this gives you one place to add logging, retrying, circuit breaking, and cost tracking
- LLM-as-judge evaluation is useful but circular when using the same model for generation and evaluation
- Semantic caching with local embedding models is a highly effective cost-saving technique
- System prompt design significantly impacts output quality — low temperature + explicit anti-hallucination instructions measurably reduce fabrication

### Cloud & Infrastructure
- VPC Endpoints are a significant cost saver over NAT Gateways for AWS-only outbound traffic
- Pre-downloading ML models in Docker builds (`HF_HUB_OFFLINE=1`) eliminates flaky runtime downloads
- ECS Fargate with auto-scaling provides a good balance of simplicity and scalability
- A single Terraform file works for a project this size, but it gets unwieldy past ~500 lines

### Production Engineering
- Content guardrails should be fail-closed (block when uncertain), not fail-open
- Red team testing should be automated, not manual — the PyRIT weekly schedule ensures continuous guardrail validation
- Structured JSON logging is essential for CloudWatch log filtering
- The retry mechanism needs error classification — retrying 401s is pure waste

### Hard Problems
- Balancing LLM quality (higher temperature, better model) against cost and latency
- Making the pipeline context-aware without overloading the LLM context window (truncation at 2000–3000 chars)
- The O(n) semantic cache scan — a tradeoff between implementation simplicity and scalability that works at current scale but needs replacing before growth
