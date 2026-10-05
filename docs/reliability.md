# Error Handling & Reliability

## Error Handling Architecture

The application handles errors at four levels:

```
Level 1: Input validation (guardrails, rate limiting)
Level 2: Retry with backoff (LLM calls, Bedrock calls)
Level 3: Job-level error handling (try/except in worker)
Level 4: Infrastructure (ALB health checks, ECS auto-restart)
```

---

## Retry Mechanism

### Implementation ([retry.py](file:///d:/ai-resarch-agent/app/retry.py))

```python
async def with_retry(coro_fn, max_retries=3, delay=1.0, backoff=2.0):
    # Attempts: 1s → 2s → 4s delays between retries
    # Raises last exception if all retries fail
```

| Setting | Default | Configurable |
|---------|---------|-------------|
| Max retries | 3 | `LLM_MAX_RETRIES` in Secrets Manager |
| Initial delay | 1.0s | `LLM_RETRY_DELAY` in Secrets Manager |
| Backoff multiplier | 2.0 | Hardcoded |
| Retry pattern | 1s → 2s → 4s | Exponential |

### What Uses Retry

| Component | Function | What is retried |
|-----------|----------|----------------|
| Agents | `_tz_call()` | LLM inference via TensorZero |
| Evaluation | `_judge()` | LLM-as-judge calls |
| Guardrails | `validate_input()` / `validate_output()` | Bedrock API calls |

### Known Issue: No Error Classification

The retry mechanism catches **all exceptions** indiscriminately:
- Retries authentication errors (401) — will never succeed
- Retries rate limit errors (429) — without respecting `Retry-After` headers
- Retries invalid request errors (400) — wastes quota on permanent failures
- Retries transient network errors — correct behavior

**Impact:** In the worst case, 3 retries × 8 LLM calls per job = 24 extra wasted calls if the provider is returning persistent errors.

---

## Failure Scenarios

### Failure: LLM Provider (OpenAI) Unavailable

```
Current Behavior:
  1. _tz_call() hits TensorZero /inference
  2. TensorZero tries openai_gpt4o → fails
  3. TensorZero falls back to groq_fallback → succeeds (if Groq is available)
  4. If both fail → _tz_call_once raises exception
  5. with_retry retries up to 3 times (1s → 2s → 4s)
  6. If all retries fail → exception propagates to _process_job
  7. Job status set to {status: "error", error: "..."}

User Impact:
  Job fails with error message. User can retry later.

Fallback:
  TensorZero automatically routes to Groq when OpenAI is down.

Risk:
  If OpenAI is down for extended period, all traffic shifts to Groq.
  With 8K/month free tier, a burst of jobs could exhaust Groq quota in hours.

Improvement:
  - Add circuit breaker to stop retrying after N consecutive failures
  - Add usage tracking to monitor quota consumption
  - Queue jobs for later processing instead of failing immediately
```

### Failure: Redis (ElastiCache) Down

```
Current Behavior:
  1. Health check returns {status: "degraded", redis: "error"}
  2. ALB marks tasks as unhealthy
  3. ECS replaces tasks (but new tasks will also fail)
  4. push_job() fails → HTTP 500
  5. cache_get() fails → job processing fails
  6. Rate limiting fails → _rate_limit() raises exception

User Impact:
  All API calls fail. System is completely unavailable.

Fallback:
  None — Redis is a hard dependency for queue, cache, sessions, and rate limiting.

Improvement:
  - Graceful degradation: skip cache on Redis failure, use in-memory queue fallback
  - ElastiCache Multi-AZ replication for automatic failover
```

### Failure: PostgreSQL (RDS) Down

```
Current Behavior:
  1. Health check does NOT verify PostgreSQL (only checks Redis)
  2. API accepts jobs normally
  3. ltm_search() / ltm_store() fail with connection error
  4. If pipeline runs (no cache/LTM), it completes but fails on ltm_store()
  5. Job status set to "error"

User Impact:
  Jobs may succeed if cache hits, but any pipeline execution fails when trying
  to store results in LTM. The system appears healthy (health check passes)
  but jobs fail.

Fallback:
  None — long-term memory is a hard dependency in the job processing path.

Improvement:
  - Add PostgreSQL connectivity to health check
  - Make LTM storage optional (succeed even if DB write fails)
  - Enable RDS backups (backup_retention_period > 0)
```

### Failure: Bedrock Guardrails Down

```
Current Behavior:
  1. validate_input() retries 3 times
  2. If all retries fail → exception → HTTP 500
  3. Jobs can't be submitted

User Impact:
  All new job submissions fail.

Fallback:
  None — guardrails are in the critical path for input validation.

Improvement:
  - Make guardrails optional (allow bypass with logging when service is unavailable)
  - Add circuit breaker to disable guardrails after N consecutive failures
```

### Failure: TensorZero Sidecar Not Ready

```
Current Behavior:
  1. App container starts and connects to TZ at localhost:3000
  2. If TZ sidecar is still starting, _tz_call_once gets connection refused
  3. with_retry retries 3 times
  4. If TZ comes up within ~7 seconds (1+2+4), calls succeed
  5. If TZ never starts, all LLM calls fail

User Impact:
  All pipeline jobs fail. Cached results still served.

Fallback:
  The retry mechanism provides a short grace period for the sidecar to start.

Improvement:
  - Add TZ health check to application startup
  - Add readiness probe in ECS task definition
```

### Failure: Worker Unbounded Concurrency

```
Current Behavior:
  1. Worker loop calls asyncio.create_task for EVERY job
  2. If 100 jobs arrive simultaneously, 100 concurrent _process_job tasks run
  3. Each makes 8+ LLM calls → 800+ concurrent LLM API calls
  4. LLM provider rate limits all calls → mass retries → quota storm

User Impact:
  Temporarily all jobs slow down or fail. Quota may be consumed rapidly.

Improvement:
  - asyncio.Semaphore to limit concurrent job processing
  - Example: semaphore = asyncio.Semaphore(3)  # max 3 concurrent jobs
```

---

## Reliability Summary

| Component | Availability Pattern | SPOF Risk |
|-----------|---------------------|-----------|
| ALB | Multi-AZ, managed | Low |
| ECS App | Auto-scaling 1–5, auto-restart | Low |
| Redis | Single node, no replication | **High** |
| RDS | Single AZ, no backups | **High** |
| TensorZero | Sidecar (co-located) | Medium |
| OpenAI API | External, uncontrolled | Medium (Groq fallback) |
| Groq API | External, 8K/month quota | High (under load) |
| Bedrock | AWS managed service | Low |
| LangSmith | External SaaS, fire-and-forget | Low (non-critical) |

---

## Observability

### Implemented

| Aspect | Implementation | Location |
|--------|---------------|----------|
| Application logs | Structured JSON format | stdout → CloudWatch Logs |
| Agent tracing | LangSmith `@traceable` decorators | All agents + orchestrator |
| Eval scores | LangSmith dataset | `research-agent-reports` |
| Health endpoint | `/health` (Redis check only) | ALB health probe |
| Container metrics | ECS Container Insights | CPU, memory, network |
| Redis stats | `/stats` endpoint | Cache entries, sessions, memory |

### Logging Format

```json
{
  "time": "2025-10-04T12:00:00",
  "level": "INFO",
  "logger": "job.a1b2c3d4",
  "message": "Starting job for topic: AI chips"
}
```

Each job gets a unique logger name (`job.{job_id[:8]}`) for easy filtering.

### Observability Gaps

| Gap | Impact |
|-----|--------|
| No PostgreSQL health in `/health` | DB failures go undetected by ALB |
| No TensorZero health monitoring | Sidecar failures not detected until LLM calls fail |
| No LLM cost/token tracking | No visibility into API spend |
| No latency metrics | No data on response times or bottleneck identification |
| No alerting | No notifications when errors spike or services degrade |
| No distributed tracing (OpenTelemetry) | Can't trace requests across services |
| No model quality monitoring | No trend analysis on eval scores |
| No Groq quota monitoring | No warning before quota exhaustion |
