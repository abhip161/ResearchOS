# Performance & Scalability

## Current Architecture Boundaries

| Resource | Current Config | Soft Limit | Hard Limit |
|----------|---------------|------------|------------|
| ECS App instances | 1 (auto-scale to 5) | 5 tasks | Fargate service quota |
| ECS CPU/Memory | 2048 / 4096 MB per task | — | Fargate max: 16 vCPU / 120 GB |
| Redis | cache.t3.micro, 1 node | ~500 MB usable | cache.t3.micro memory |
| PostgreSQL | db.t3.micro, 20–100 GB | 100 GB storage | db.t3.micro IOPS |
| OpenAI GPT-4o | Tier-dependent RPM | 500–10,000 RPM | Account tier |
| Groq (fallback) | 8,000 req/month | — | Hard monthly cap |
| Worker concurrency | Unbounded | — | Available memory/CPU |

---

## Current Bottlenecks

### 1. LLM Latency (Most Significant)

Each research job requires 4–12 LLM calls sequentially through the agent pipeline (Search → Summarize → Write → Critic), plus 4 parallel evaluation calls.

| Operation | Estimated Latency | Impact |
|-----------|------------------|--------|
| Single LLM call (GPT-4o via TZ) | 2–8 seconds | — |
| Full pipeline (4 calls, sequential) | 10–30 seconds | Primary latency driver |
| Full pipeline + retry | 20–60 seconds | Worst case |
| Evaluation (4 calls, parallel) | 2–8 seconds | Fire-and-forget, doesn't block response |
| **Total job time** | **15–45 seconds** | **User-visible** |

**Why sequential:** The agents are inherently sequential — SummarizeAgent needs SearchAgent's output, WriterAgent needs SummarizeAgent's output, CriticAgent needs WriterAgent's output. This cannot be parallelized without changing the pipeline structure.

### 2. Semantic Cache O(n) Scan

`cache_get()` iterates through ALL `emb:*` keys in Redis and computes cosine similarity for each. As the cache grows, this becomes increasingly CPU-intensive.

| Cache Size | Estimated Lookup Time | Impact |
|-----------|----------------------|--------|
| 10 entries | <10ms | Negligible |
| 100 entries | ~50ms | Acceptable |
| 1,000 entries | ~500ms | Noticeable |
| 10,000 entries | ~5 seconds | Severe |

**Why it doesn't break immediately:** Cache entries have 1-hour TTL, so the active cache size is naturally bounded by recent request volume.

### 3. Duplicate Embedding Model (~200MB Wasted RAM)

Both `cache.py` and `memory.py` instantiate separate `SentenceTransformer("all-MiniLM-L6-v2")` instances, each consuming ~100 MB of RAM.

### 4. No Connection Pooling for TensorZero

`_tz_call_once` creates a new `httpx.AsyncClient` for each LLM call. While localhost connections are fast, connection pooling would reduce overhead for high-throughput scenarios.

### 5. Unbounded Worker Concurrency

The worker loop calls `asyncio.create_task` for every job without any concurrency limit. A burst of jobs causes:
- All jobs processed simultaneously
- Hundreds of concurrent LLM API calls
- Provider rate limits → mass retries → quota waste

---

## What Happens at 10x Traffic

**Scenario:** 100 research requests/hour instead of ~10

| Component | Behavior | Breaks? |
|-----------|----------|---------|
| ALB | Handles easily | ✅ No |
| API (job submission) | Redis XADD is fast | ✅ No |
| Rate limiting | 10 req/min per IP may be hit by power users | ⚠️ Configurable |
| Redis (queue/cache) | cache.t3.micro handles ~25K ops/sec | ✅ No |
| Semantic cache scan | 100× more cache entries → O(n) becomes painful | ⚠️ Slow |
| Worker concurrency | 10+ concurrent jobs → 80+ concurrent LLM calls | ⚠️ Risky |
| OpenAI API | ~800 calls/hour, within most tier limits | ✅ Probably fine |
| PostgreSQL | ~200 queries/hour, db.t3.micro handles this | ✅ No |
| ECS auto-scaling | Scales 1→5 at CPU 70% | ✅ Designed for this |
| Embedding model | 2 instances per task × 5 tasks = 1 GB wasted | ⚠️ Wasteful |
| Evaluation | 400 extra LLM calls/hour (eval on every job) | ⚠️ Expensive |

**Primary concern at 10x:** LLM API costs and evaluation overhead. The infrastructure handles it, but API costs increase linearly.

**Recommended actions for 10x:**
1. Add `asyncio.Semaphore` to limit concurrent job processing
2. Skip evaluation on cache/LTM hits
3. Route evaluation to a cheaper model
4. Fix O(n) cache scan

---

## What Happens at 100x Traffic

**Scenario:** 1,000 research requests/hour

| Component | Behavior | Breaks? |
|-----------|----------|---------|
| ALB | Fine | ✅ |
| Redis cache.t3.micro | Memory pressure (~500 MB limit) | ⚠️ Upgrade needed |
| Semantic cache O(n) | 1000+ entries × linear scan = seconds per lookup | ❌ **Breaks** |
| Worker concurrency | 100+ concurrent jobs = 800+ concurrent LLM calls | ❌ **Breaks** |
| OpenAI API | 8,000+ calls/hour, may hit RPM limits | ❌ **Likely breaks** |
| PostgreSQL db.t3.micro | 2,000+ writes/hour, IOPS limit | ⚠️ May struggle |
| ECS (max 5 tasks) | CPU may not scale enough | ⚠️ Increase max |
| LLM costs | ~$40–120/hour at 100x | ❌ **Cost prohibitive** |
| Groq fallback | 8K/month quota = irrelevant at this scale | ❌ Useless |

**What breaks first:**
1. Semantic cache O(n) scan
2. Unbounded worker concurrency causing LLM quota storms
3. OpenAI RPM limits
4. LLM costs

**Recommended redesign for 100x:**
1. Replace O(n) cache scan with proper vector index (Redis Vector, pgvector for cache)
2. Implement concurrency limits + job prioritization queue
3. Route non-critical tasks to cheaper models (GPT-4o-mini for search, summarize, critic, eval)
4. Add Redis sentinel/cluster for HA and capacity
5. Upgrade RDS to db.t3.medium or larger
6. Increase ECS max instances beyond 5
7. Implement request batching
8. Add LLM response caching at TensorZero level
9. Consider async/streaming responses

---

## Caching Strategy Analysis

### Current Caching

| Cache | Hit Rate (estimated) | Savings |
|-------|---------------------|---------|
| Semantic cache (Redis, 1hr TTL) | Low–Medium (same topic within 1 hour) | Saves full pipeline (4–12 LLM calls) |
| LTM search (pgvector, 7-day window) | Medium (similar topic within 7 days) | Saves full pipeline |
| LTM related search (pgvector) | Medium | Provides context, doesn't skip pipeline |

### Missing Caching Opportunities

| What Could Be Cached | Expected Savings |
|----------------------|-----------------|
| Skip eval on cache/LTM hits | 4 LLM calls per cached job |
| TensorZero-level response cache | Duplicate prompts across jobs |
| Embedding computation | Repeated embeddings for same text |
| Bedrock guardrail results | Same topic checked multiple times |

---

## Cost Analysis

### Per-Job Cost Breakdown

| Component | Calls | Est. Token Usage | Est. Cost (GPT-4o) |
|-----------|-------|-----------------|-------------------|
| SearchAgent | 1 | ~500 in / 500 out | ~$0.006 |
| SummarizeAgent | 1 | ~600 in / 400 out | ~$0.005 |
| WriterAgent | 1 | ~1000 in / 2000 out | ~$0.023 |
| CriticAgent | 1 | ~1500 in / 100 out | ~$0.005 |
| 4× Eval judges | 4 | ~1000 in / 200 out each | ~$0.018 |
| **Total per job** | **8** | **~6,500 in / 3,900 out** | **~$0.055** |

### Monthly Cost Projections

| Traffic | Jobs/Month | LLM Cost | Infra Cost | **Total** |
|---------|-----------|----------|------------|-----------|
| Low (10/day) | 300 | ~$17 | ~$140 | **~$157** |
| Medium (50/day) | 1,500 | ~$83 | ~$140 | **~$223** |
| High (200/day) | 6,000 | ~$330 | ~$160 | **~$490** |
| Very High (1000/day) | 30,000 | ~$1,650 | ~$250 | **~$1,900** |

### Cost Reduction Strategies

| Strategy | Estimated Savings | Complexity |
|----------|------------------|------------|
| Skip eval on cache/LTM hits | ~30-50% of eval costs | Low |
| Use GPT-4o-mini for eval judges | ~90% of eval costs | Low (TZ config change) |
| Use GPT-4o-mini for Search + Summarize | ~60% of pipeline costs | Low (TZ config change) |
| Extend cache TTL (1hr → 24hr) | Variable (depends on repeat traffic) | Trivial |
| Rule-based completeness check | 25% of eval costs | Low |
| Concurrency limiter to prevent quota storms | Prevents cost spikes | Low |
