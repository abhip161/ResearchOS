# Testing

## Current Testing Status

| Test Type | Status | Details |
|-----------|--------|---------|
| Unit tests | **Not implemented** | No test files found |
| Integration tests | **Not implemented** | — |
| End-to-end tests | **Not implemented** | — |
| Load tests | **Not implemented** | — |
| CI test step | **Not implemented** | deploy.yml has no test stage |
| Type checking | **Not configured** | No mypy/pyright config |
| Linting | **Not configured** | No ruff/flake8/pylint config |
| Test framework | **Not installed** | No pytest in requirements.txt |

> [!WARNING]
> The project has **zero automated tests**. Any refactoring could silently break behavior. This is the single largest quality gap.

---

## AI/LLM Evaluation (Implemented)

While traditional tests are absent, the project has a built-in **LLM-as-judge evaluation system** that assesses every generated report:

### Implemented Evaluation Metrics

| Metric | What It Measures | Type |
|--------|-----------------|------|
| Relevance | Is the report relevant to the topic? | LLM judge (0.0–1.0) |
| Completeness | Does it have all 4 required sections? | LLM judge (0.0–1.0) |
| Hallucination Risk | Are there fabricated facts? | LLM judge (0.0–1.0, inverted) |
| Overall Quality | Depth, accuracy, clarity, usefulness | LLM judge (0.0–1.0) |

### How Evaluation Results Are Stored

Results are logged to the LangSmith dataset `research-agent-reports`. Each entry contains:
- Topic
- Report preview (first 400 chars)
- All 4 evaluation scores
- Job ID

### Red Team Testing (Implemented)

The PyRIT dashboard provides adversarial testing:
- 4 attack types (jailbreak, XPIA, crescendo, skeleton key)
- Automated weekly runs via EventBridge
- Results stored in Redis

---

## Recommended Testing Strategy

### Unit Tests (Priority 1)

| Component | What to Test | Complexity |
|-----------|-------------|------------|
| `_parse_score()` | Score parsing from LLM output | Low |
| `cache_set/cache_get` | Cache storage and retrieval | Low |
| `session_add/session_get` | Session memory CRUD | Low |
| `generate_pdf()` | PDF generation from text | Low |
| `generate_json_report()` | JSON structure correctness | Low |
| `with_retry()` | Retry behavior, backoff timing | Low |
| `require_api_key()` | Auth validation | Low |
| `_rate_limit()` | Rate limit enforcement | Medium |
| `OrchestratorAgent.route()` | Routing logic (retry vs. end) | Low |

### Integration Tests (Priority 2)

| Test | What It Validates |
|------|-------------------|
| API → Redis → Worker → Result | Full job lifecycle |
| Cache hit path | Cached results returned correctly |
| LTM hit path | LTM results returned correctly |
| Guardrail blocking | Harmful input rejected |
| Rate limiting | 429 after threshold |

### AI Evaluation Tests (Priority 3)

| Metric | Implementation | Purpose |
|--------|---------------|---------|
| Retrieval accuracy | Compare search results against known-good answers | Measure SearchAgent quality |
| Report structure validation | Rule-based check for required sections | Replace `eval_completeness` LLM call |
| Regression testing | Compare scores across model/prompt versions | Catch quality degradation |
| Latency tracking | Measure per-stage timing | Performance regression detection |

### Recommended Evaluation Metrics (Not Yet Implemented)

| Metric | What It Would Measure | Framework |
|--------|----------------------|-----------|
| Context relevance | Are search results relevant to the topic? | RAGAS |
| Faithfulness | Is the report grounded in search results? | RAGAS |
| Answer correctness | Does the report accurately answer the query? | DeepEval |
| Token efficiency | Tokens used vs. quality achieved | Custom |
| Latency percentiles | P50, P95, P99 response times | Custom |
