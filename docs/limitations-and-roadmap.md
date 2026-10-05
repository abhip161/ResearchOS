# Known Limitations

## Architectural Limitations

| Limitation | Impact | Mitigation |
|-----------|--------|------------|
| **Monolithic `main.py`** | Mixes API routes, worker logic, and job processing in ~260 lines. Hard to test or scale independently. | Split into `routes.py`, `worker.py`, `processor.py` |
| **Single worker loop** | One worker per ECS task instance; scaling requires adding more instances. | Horizontal scaling via ECS auto-scaling (1–5 instances) |
| **Unbounded job concurrency** | All jobs processed simultaneously via `asyncio.create_task` with no semaphore. | Add `asyncio.Semaphore` to limit concurrent processing |
| **Sequential agent pipeline** | 4 LLM calls must execute in order (Search→Summarize→Write→Critic). Minimum ~15 seconds per job. | Inherent to pipeline design; can't parallelize agent steps |

## AI / LLM Limitations

| Limitation | Impact |
|-----------|--------|
| **No external search** | Agents produce text from LLM knowledge only. No web search, no document retrieval, no RAG from external sources. Facts are limited to model's training data cutoff. |
| **Same model for generation and evaluation** | CriticAgent and eval judges use the same GPT-4o model that generates reports. The evaluator shares the same knowledge gaps and biases as the generator. |
| **Evaluation on every job** | 4 extra LLM calls per job, including cache/LTM hits where the report was already evaluated. Wastes ~50% more API calls. |
| **Single model for all tasks** | Only 1 TensorZero model configuration shared across search, summarize, write, critic, and eval. Cheaper models could handle simpler tasks. |
| **No streaming** | Responses are fully generated before returning. No real-time progress updates during the 15–45 second pipeline execution. |
| **Critic decision is binary** | CriticAgent returns YES/NO. A rejected report triggers a full re-search, even if only minor issues exist. |

## Performance Limitations

| Limitation | Impact |
|-----------|--------|
| **O(n) cache scan** | `cache_get()` iterates through all cached embeddings in Redis. Performance degrades linearly with cache size. |
| **Duplicate embedding model** | `cache.py` and `memory.py` each load `all-MiniLM-L6-v2` (~100 MB each), wasting ~100 MB RAM. |
| **No httpx connection pooling for TZ** | Each `_tz_call_once` creates a new `httpx.AsyncClient`. Wastes connection setup time. |
| **Retry doesn't classify errors** | Retries 401s, 400s, and 429s with the same exponential backoff. Wastes retries on permanent failures. |

## Security Limitations

| Limitation | Impact |
|-----------|--------|
| **PyRIT publicly accessible** | Red team dashboard on port 8001 is accessible to anyone. No authentication. |
| **Auth disabled by default** | If `API_KEY` is not set in Secrets Manager, all endpoints are unauthenticated. |
| **Docker containers run as root** | No `USER` directive in Dockerfiles. |
| **Single shared API key** | All clients share one key. No per-client tracking or revocation. |
| **No WAF** | No AWS WAF on ALB. No protection against common web attacks. |

## Infrastructure Limitations

| Limitation | Impact |
|-----------|--------|
| **No database backups** | `backup_retention_period = 0` on RDS. Data loss on instance failure. |
| **Single Redis node** | No replication, no automatic failover. Redis failure = total system outage. |
| **No staging environment** | Code deploys directly to production. No pre-production validation. |
| **No tests in CI/CD** | Pipeline builds and deploys without any automated verification. |
| **ECS tasks in public subnet** | Tasks have public IPs. Less secure than private subnet + NAT/VPC Endpoints. |

## Observability Limitations

| Limitation | Impact |
|-----------|--------|
| **Health check only verifies Redis** | PostgreSQL and TensorZero failures go undetected by ALB. |
| **No LLM cost tracking** | No visibility into API spend until the monthly bill arrives. |
| **No alerting** | No notifications when error rates spike, services degrade, or costs increase. |
| **No latency metrics** | No data on per-stage timing or overall response time distributions. |

---

# Future Roadmap

## Short-term (1–2 weeks, high-value low-complexity)

| Improvement | Impact | Effort |
|------------|--------|--------|
| **Skip eval on cache/LTM hits** | Saves 4 LLM calls per cached job | 1 line of code |
| **Add asyncio.Semaphore to worker** | Prevents quota storms from concurrent jobs | 5 lines of code |
| **Classify retry errors** | Don't retry 401/400; respect 429 Retry-After | Small refactor of `retry.py` |
| **Enable RDS backups** | Prevent data loss | Terraform: `backup_retention_period = 7` |
| **Restrict PyRIT access** | Prevent unauthorized attack triggering | ALB listener rule or security group |
| **Add non-root Docker user** | Container security hardening | 1 line per Dockerfile |
| **Deduplicate embedding model** | Save ~100 MB RAM | Shared module singleton |

## Medium-term (1–2 months, important architecture improvements)

| Improvement | Impact | Effort |
|------------|--------|--------|
| **Task-specific TZ model configs** | Route eval/critic to cheaper models (GPT-4o-mini) | TZ config change |
| **Split main.py** | Separate routes, worker, processor for testability | Medium refactor |
| **Replace O(n) cache scan** | Use Redis Vector module or pgvector for cache | Medium |
| **Add baseline tests** | Unit tests for critical functions, integration tests for pipeline | Medium |
| **Add CI test step** | Run tests before deployment | Small pipeline change |
| **HTTPS on ALB** | TLS termination with ACM certificate | Terraform + DNS |
| **OIDC for GitHub Actions** | Replace static AWS credentials | IAM + workflow change |
| **Rule-based completeness check** | Eliminate 1 of 4 eval LLM calls | Small code change |

## Long-term (3–6 months, major platform improvements)

| Improvement | Impact | Effort |
|------------|--------|--------|
| **External search integration** | Web search API (Serper, Tavily) for real-time facts | Agent refactor |
| **WebSocket/SSE for progress** | Real-time pipeline status updates | Frontend + backend change |
| **Multi-model routing** | Different models per agent based on task complexity | TZ config + possible refactor |
| **Redis Sentinel/Cluster** | HA for Redis with automatic failover | Infrastructure change |
| **Staging environment** | Pre-production validation before deploying | Terraform modules |
| **Comprehensive evaluation** | RAGAS/DeepEval integration for systematic quality testing | New eval pipeline |
| **Observability platform** | OpenTelemetry, Prometheus, Grafana for full observability | Infrastructure + instrumentation |
| **Event-driven architecture** | Replace polling with event push for better UX | Architecture change |
| **Multi-tenant support** | Per-user API keys, usage tracking, billing | Auth + data model refactor |
