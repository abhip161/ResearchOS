# Components

## Component Overview

| Component | File | Purpose | LLM Calls | Dependencies |
|-----------|------|---------|------------|-------------|
| FastAPI API | [main.py](file:///d:/ai-resarch-agent/app/main.py) | REST endpoints, rate limiting, lifespan management | None | All other modules |
| Agent Pipeline | [agents.py](file:///d:/ai-resarch-agent/app/agents.py) | LangGraph multi-agent orchestration | 4–12 per job | config, retry, TensorZero |
| Evaluation | [eval.py](file:///d:/ai-resarch-agent/app/eval.py) | 4 LLM-as-judge evaluators + LangSmith logging | 4 per job | config, retry, LangSmith |
| Guardrails | [guardrails.py](file:///d:/ai-resarch-agent/app/guardrails.py) | Content safety via AWS Bedrock | **None** (managed service) | config, retry, Bedrock |
| Session Memory | [memory.py](file:///d:/ai-resarch-agent/app/memory.py) | Redis session + pgvector long-term memory | **None** (local embeddings) | config, pool, Redis, sentence-transformers |
| Semantic Cache | [cache.py](file:///d:/ai-resarch-agent/app/cache.py) | Embedding-based semantic cache in Redis | **None** (local embeddings) | config, Redis, sentence-transformers |
| Job Queue | [queue.py](file:///d:/ai-resarch-agent/app/queue.py) | Redis Streams producer/consumer | None | config, Redis |
| Output | [output.py](file:///d:/ai-resarch-agent/app/output.py) | PDF generation, JSON reports, report diffs | None | config, pool, memory, ReportLab |
| Auth | [auth.py](file:///d:/ai-resarch-agent/app/auth.py) | API key validation | None | FastAPI |
| Retry | [retry.py](file:///d:/ai-resarch-agent/app/retry.py) | Generic exponential backoff | None | None |
| Connection Pool | [pool.py](file:///d:/ai-resarch-agent/app/pool.py) | asyncpg connection pool | None | config, asyncpg |
| Config | [config.py](file:///d:/ai-resarch-agent/app/config.py) | Loads all config from Secrets Manager | None | boto3 |
| TensorZero Gateway | [tensorzero.toml](file:///d:/ai-resarch-agent/tensorzero/tensorzero.toml) | LLM routing, prompt templates, multi-provider fallback | N/A (proxy) | OpenAI, Groq |
| PyRIT Dashboard | [pyrit_dashboard/main.py](file:///d:/ai-resarch-agent/pyrit_dashboard/main.py) | Adversarial attack testing | Indirect (via API) | httpx, pyrit, Redis |
| Frontend | [index.html](file:///d:/ai-resarch-agent/index.html) | Web UI for submitting research topics | None | ALB |

---

## FastAPI API (`main.py`)

### Purpose
Central application entrypoint. Hosts all REST endpoints, manages the application lifespan (Redis, database pool, LangGraph initialization), runs the background worker loop, and processes research jobs.

### Responsibilities
- Serve REST API endpoints
- Manage application lifespan (startup/shutdown)
- Rate limiting (per-IP using Redis INCR + EXPIRE)
- Run background worker loop consuming from Redis Streams
- Process jobs: cache/LTM lookup → pipeline execution → output generation → evaluation

### Inputs
- HTTP requests from users/clients
- Job messages from Redis Streams

### Outputs
- HTTP responses (JSON, PDF, HTML)
- Job results stored in Redis
- Reports stored in PostgreSQL

### Runtime Flow
```
Application startup (lifespan):
  1. Connect to Redis
  2. Initialize asyncpg connection pool
  3. Run database migration (CREATE TABLE, indexes)
  4. Build LangGraph pipeline
  5. Start worker loop as background task
  
Worker loop:
  1. XREADGROUP from Redis Streams (block 5s)
  2. For each job: asyncio.create_task(_process_job)
  3. _process_job: cache check → LTM check → pipeline → guardrails → store → eval
```

### Failure Handling
- Job processing errors: caught by try/except, job result set to `{status: "error", error: "..."}`, message ACK'd
- Redis connection failure: health endpoint returns `{status: "degraded"}`
- Worker loop errors: caught silently, sleep 1s, continue

### Scaling Considerations
- Worker uses `asyncio.create_task` with **unbounded concurrency** — if 100 jobs arrive simultaneously, all are processed concurrently
- Single worker loop per ECS task instance — horizontal scaling via ECS auto-scaling (1–5 instances)
- Each instance uses hostname as Redis consumer name, enabling safe horizontal scaling with consumer groups

### Known Issues
- **God-file** — mixes API routes, worker logic, and job processing in ~260 lines
- Global mutable state (`redis_client`, `graph`)

---

## Agent Pipeline (`agents.py`)

### Purpose
Implements the four-agent research pipeline using LangGraph StateGraph. Each agent is a Python class that calls TensorZero for LLM inference.

### Responsibilities
- Define `ResearchState` TypedDict (shared state between agents)
- Implement SearchAgent, SummarizeAgent, WriterAgent, CriticAgent
- Build and compile the LangGraph workflow
- Single LLM call point (`_tz_call`) for all agents

### The `_tz_call` Function
Every LLM call in the application flows through this single function:

```python
async def _tz_call(config, function_name, message) -> str:
    # Calls _tz_call_once with retry wrapper
    # _tz_call_once: POST to TensorZero /inference endpoint
    # Returns: response.json()["content"][0]["text"]
```

This is a critical architectural choice — it creates a single point of control for all LLM interactions.

### Agent Details

**SearchAgent** ([agents.py:47–69](file:///d:/ai-resarch-agent/app/agents.py#L47-L69))
- Receives: topic + last 4 session turns (conversation context)
- Produces: 5 key facts about the topic
- TZ Function: `research_summarize` (temp=0.3, max_tokens=2000)

**SummarizeAgent** ([agents.py:72–86](file:///d:/ai-resarch-agent/app/agents.py#L72-L86))
- Receives: raw search results
- Produces: structured bullet points
- TZ Function: `research_summarize`

**WriterAgent** ([agents.py:89–119](file:///d:/ai-resarch-agent/app/agents.py#L89-L119))
- Receives: topic + summaries + LTM context (related previous report, truncated to 2000 chars)
- Produces: structured report with Executive Summary, Key Findings, Analysis, Conclusion
- TZ Function: `report_write` (temp=0.5, max_tokens=4000)

**CriticAgent** ([agents.py:122–138](file:///d:/ai-resarch-agent/app/agents.py#L122-L138))
- Receives: report text (truncated to `agent_report_truncate` chars, default 3000)
- Produces: YES (approved) or NO (rejected)
- TZ Function: `research_summarize`
- Decision logic: `check.strip().upper().startswith("YES")`

**OrchestratorAgent** ([agents.py:141–184](file:///d:/ai-resarch-agent/app/agents.py#L141-L184))
- Coordinates all sub-agents via LangGraph nodes and edges
- Routes retries: if critic rejects and `iterations < agent_max_iterations`, loops back to SearchAgent
- Makes 0 LLM calls itself

### Failure Handling
- LLM call failures: handled by `with_retry` (exponential backoff, max 3 retries)
- If all retries fail: exception propagates to job processor, job status set to "error"
- If critic always rejects: stops after `agent_max_iterations` (default 2), returns last report

---

## Evaluation (`eval.py`)

### Purpose
Runs 4 LLM-as-judge evaluations on every completed research report and logs scores to LangSmith.

### The Four Judges

| Judge | Purpose | Scoring | LLM Call |
|-------|---------|---------|----------|
| `eval_relevance` | Is the report relevant to the topic? | SCORE: X/10 | Yes |
| `eval_completeness` | Does it have all 4 required sections? | SCORE: X/10 | Yes |
| `eval_hallucination` | Any fabricated facts? | SCORE: X/10 (inverted — higher = more hallucinations) | Yes |
| `eval_quality` | Overall quality score | SCORE: X/10 | Yes |

All 4 judges run in parallel via `asyncio.gather`. Scores are parsed with regex (`SCORE: X/10`) and normalized to 0.0–1.0.

### Runtime Flow
```
evaluate_report() called via asyncio.create_task (fire-and-forget)
  ↓
asyncio.gather(eval_relevance, eval_completeness, eval_hallucination, eval_quality)
  ↓
Each judge: _judge() → _judge_once() → POST TZ /inference
  ↓
Parse SCORE: X/10 from response → normalize to 0.0–1.0
  ↓
Log to LangSmith dataset "research-agent-reports"
```

### Critical Issue
Evaluation runs on **every** completed job, including cache and LTM hits, adding 4 unnecessary LLM calls for already-evaluated reports.

---

## Guardrails (`guardrails.py`)

### Purpose
Content safety validation for both input (user topics) and output (generated reports) using AWS Bedrock Guardrails managed service.

### Important
This component does **NOT** make LLM calls. It uses the Bedrock Guardrails API (`bedrock-runtime.apply_guardrail`), which is a rule-based managed service. It does not consume OpenAI or Groq quota.

### Configured Filters (via Terraform)
- **Content filters (all HIGH)**: HATE, VIOLENCE, SEXUAL, INSULTS, MISCONDUCT, PROMPT_ATTACK
- **Topic denials**: weapons, illegal_activities, self_harm
- **PII blocking**: SSN, credit cards, AWS keys
- **PII anonymization**: email, phone
- **Word filtering**: profanity

### Behavior
- `validate_input()`: checks user topic before processing. If blocked → HTTP 400.
- `validate_output()`: checks generated report after pipeline. If blocked → job status "blocked".
- Both use `asyncio.to_thread` because boto3 is synchronous — correct approach.
- **Fail-closed**: blocked content is rejected, never returned to the user.

---

## Semantic Cache (`cache.py`)

### Purpose
Prevents redundant LLM calls by matching new queries against previously processed topics using embedding similarity.

### How It Works
1. **cache_get**: Encode the query with `all-MiniLM-L6-v2` → scan ALL `emb:*` keys in Redis → compute cosine similarity against each → return cached result if similarity ≥ 0.85
2. **cache_set**: Encode query → store result as `semantic:{hash(query)}` and embedding as `emb:{hash(query)}`, both with 1-hour TTL

### Performance Issue
The `cache_get` function uses `SCAN_ITER` to iterate through **all** embedding keys in Redis and computes cosine similarity for each. This is O(n) where n = number of cached entries. As the cache grows, this becomes increasingly CPU-intensive.

### Why Not a Vector Database for Cache?
The semantic cache uses Redis for simplicity and because cache entries have short TTLs (1 hour). The volume is expected to be small enough that O(n) scan is acceptable for the current scale. For higher scale, this should migrate to a proper vector index.

---

## Long-Term Memory (`memory.py`)

### Purpose
Two-tier memory system:
1. **Session Memory** — short-term conversation context in Redis
2. **Long-Term Memory** — permanent report storage with vector search in PostgreSQL + pgvector

### Session Memory
- **Storage**: Redis lists keyed by `session:{session_id}`
- **Max messages**: 5 (trimmed with LTRIM)
- **Content truncation**: 500 chars per message
- **TTL**: 30 minutes
- **Purpose**: Passes the last 4 conversation turns to SearchAgent for context-awareness

### Long-Term Memory
- **Storage**: PostgreSQL `reports` table with `vector(384)` column
- **Embedding model**: `all-MiniLM-L6-v2` (384 dimensions, loaded at module level)
- **Index**: IVFFlat with cosine distance (configurable lists, default 100)

#### Three LTM Operations

1. **`ltm_search`** — Find near-exact match (similarity ≥ 0.88, within last 7 days). Returns full report if found, skipping the entire pipeline.
2. **`ltm_search_related`** — Find related-but-different report (similarity 0.50–0.87). Returns report text as context for WriterAgent to build upon.
3. **`ltm_diff`** — Compare latest 2 reports on the same topic. Returns unified diff showing what changed between versions.

### Database Migration
`db_migrate()` runs at startup, creating:
- `reports` table (id, topic, report, embedding vector(384), created_at)
- IVFFlat index on embedding column
- B-tree indexes on topic and created_at

---

## Job Queue (`queue.py`)

### Purpose
Async job queue using Redis Streams with consumer groups for reliable message processing.

### Why Redis Streams (Not a Simple Queue)
Redis Streams provide consumer groups, which means:
- Multiple ECS task instances can consume from the same stream safely
- Each consumer uses its hostname as a unique consumer name
- Messages are only removed after explicit ACK, preventing data loss on worker crashes
- Built-in blocking read (5-second timeout) avoids busy-waiting

### Operations
- `push_job` — XADD to stream, returns job_id
- `consume_jobs` — XREADGROUP, count=1, block=5000ms
- `ack_job` — XACK after processing complete
- `set_result` / `get_result` — Redis key/value with TTL for results

---

## TensorZero Gateway

### Purpose
LLM routing layer running as a sidecar container in the same ECS task as the application. All LLM calls pass through TensorZero — the application never calls OpenAI or Groq directly.

### Configuration ([tensorzero.toml](file:///d:/ai-resarch-agent/tensorzero/tensorzero.toml))

**1 model definition:**
- `research_model` → routing: `openai_gpt4o` (primary), `groq_fallback` (fallback)

**2 function definitions:**
- `research_summarize` — temp=0.3, max_tokens=2000, system prompt for research analysis
- `report_write` — temp=0.5, max_tokens=4000, system prompt for report writing

### System Prompts

**research_summarize** ([template](file:///d:/ai-resarch-agent/tensorzero/templates/research_summarize_system.minijinja)):
> "You are a precise research analyst. Be factual and specific. Do not hallucinate. Structure output as numbered points. Focus on recent, relevant, verifiable information."

**report_write** ([template](file:///d:/ai-resarch-agent/tensorzero/templates/report_write_system.minijinja)):
> "You are a senior research writer. Structure every report with: Executive Summary, Key Findings, Analysis, Conclusion. Use precise, professional language. Cite specific facts."

### Current Limitation
Only 1 model configuration (`research_model`) shared across all functions. The CriticAgent and evaluation judges use `research_summarize` even though they perform different tasks. Adding task-specific model configurations (e.g., a cheaper model for evaluation) would reduce costs without changing application code.

---

## PyRIT Red Team Dashboard

### Purpose
Separate ECS service that runs adversarial attacks against the main Research Agent API to test guardrail effectiveness.

### Attack Types
| Attack | Description |
|--------|-------------|
| **Jailbreak** | Direct attempts to bypass safety instructions |
| **XPIA** | Cross-Prompt Injection Attack — hides malicious instructions inside research topics |
| **Crescendo** | Gradually escalates from innocent questions toward harmful content |
| **Skeleton Key** | Claims authority (researcher, CISO approval) to bypass restrictions |

### How It Works
1. PyRIT generates adversarial prompts
2. Sends them as legitimate research requests to the main API (`POST /research`)
3. Waits for results
4. Scores whether guardrails blocked the harmful content or not
5. Stores results in Redis

### Scheduling
- **Manual**: User clicks "Run Selected Attacks" on the dashboard UI
- **API**: `GET /run-attacks` with optional `?types=` parameter
- **Automated**: EventBridge triggers a PyRIT ECS task every Monday at 2 AM UTC

### Important
Each PyRIT attack generates real research jobs against the production API. A full red team run (14 adversarial prompts) generates 14 jobs × 8–16 LLM calls each = 112–224 LLM calls.

---

## Configuration (`config.py`)

### Purpose
Single centralized configuration class that loads all settings from AWS Secrets Manager at startup.

### How It Works
1. `_load_secret()` calls Secrets Manager to retrieve `research-agent/config`
2. Decorated with `@lru_cache(maxsize=1)` — loaded once, cached permanently
3. `Config.__init__` extracts and type-casts all values

### Configuration Groups

| Group | Parameters | Example Values |
|-------|-----------|----------------|
| AWS | `aws_region` | `us-east-1` |
| Bedrock | `bedrock_guardrail_id`, `bedrock_guardrail_version` | Auto-set by Terraform |
| Storage | `redis_url`, `database_url`, `tensorzero_url` | Auto-set by Terraform |
| Auth | `api_key` | User-defined |
| LangSmith | `langsmith_api_key`, `langchain_project`, `langsmith_dataset` | User-provided key |
| Semantic Cache | `cache_ttl` (3600), `cache_similarity_threshold` (0.85) | Tunable |
| Session | `session_ttl` (1800), `session_max_messages` (5), `session_content_truncate` (500) | Tunable |
| LTM | `ltm_days` (7), `ltm_threshold` (0.88), `ltm_diff_threshold` (0.7) | Tunable |
| Queue | `stream_key`, `consumer_group`, `consumer_name`, `result_ttl` (3600) | Auto |
| Agent | `agent_report_truncate` (3000), `agent_max_iterations` (2) | Tunable |
| Eval | `eval_report_truncate` (1500), `eval_comment_truncate` (300) | Tunable |
| Retry | `llm_max_retries` (3), `llm_retry_delay` (1.0) | Tunable |
| Rate Limit | `rate_limit_requests` (10), `rate_limit_window` (60) | 10 req/min per IP |
| DB Pool | `db_pool_min` (2), `db_pool_max` (10) | Tunable |

### Important Behavior
- `lru_cache` means secret changes require an application restart to take effect
- LangSmith environment variables are set in `Config.__init__` if the API key is present
- Terraform auto-populates Redis URL, Database URL, Guardrail ID, and other infrastructure values. Users only need to set API keys.
