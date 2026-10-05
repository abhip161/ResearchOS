# API Documentation

## Base URL

```
http://<alb_dns>/
```

All endpoints except `/health` and `/` require the `X-API-Key` header when an API key is configured in Secrets Manager. If no API key is configured, authentication is disabled.

## Authentication

| Header | Value | Required |
|--------|-------|----------|
| `X-API-Key` | The API key set in Secrets Manager | Yes (when configured) |
| `Content-Type` | `application/json` | For POST requests |

---

## Endpoints

### `POST /research` — Submit Research Job

Submits a new research topic for processing. Returns immediately with a `job_id` for polling.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |
| **Rate Limited** | Yes (10 req/min per IP, configurable) |
| **Input Guardrail** | Yes — Bedrock validates topic before accepting |

**Request Body:**
```json
{
  "topic": "AI chip market 2025",
  "session_id": "abc123",
  "output_format": "text"
}
```

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `topic` | string | Yes | — | Research topic to investigate |
| `session_id` | string | No | auto-generated UUID | Groups multiple requests into a conversation |
| `output_format` | string | No | `"text"` | Output format: `text`, `pdf`, or `json` |

**Response (202-style, async):**
```json
{
  "job_id": "a1b2c3d4-...",
  "session_id": "abc123"
}
```

**Error Responses:**

| Status | Condition |
|--------|-----------|
| 400 | Topic blocked by Bedrock Guardrails (harmful content) |
| 401 | Invalid or missing API key |
| 429 | Rate limit exceeded |

---

### `GET /result/{job_id}` — Get Job Result

Polls for the result of a submitted research job.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |

**Response (pending):**
```json
{
  "status": "pending"
}
```

**Response (completed):**
```json
{
  "status": "done",
  "topic": "AI chip market 2025",
  "report": "# Executive Summary\n...",
  "diff": "--- previous (2025-10-03)\n+++ latest (2025-10-04)\n..."
}
```

If `output_format` was `pdf`, includes `pdf_base64` field. If `json`, includes `structured` field with metadata.

**Response (blocked by output guardrail):**
```json
{
  "status": "blocked",
  "error": "Output blocked by safety guardrail."
}
```

**Response (error):**
```json
{
  "status": "error",
  "error": "Error message"
}
```

---

### `GET /result/{job_id}/pdf` — Download PDF

Downloads the completed report as a PDF file.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |

**Response:** PDF binary file with `Content-Disposition: attachment; filename={job_id}.pdf`

| Status | Condition |
|--------|-----------|
| 404 | Job not found or not completed yet |

---

### `GET /session/{session_id}` — Get Session History

Retrieves the conversation history for a session.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |

**Response:**
```json
{
  "session_id": "abc123",
  "messages": [
    {"role": "user", "content": "AI chip market 2025"},
    {"role": "assistant", "content": "# Executive Summary..."}
  ]
}
```

---

### `GET /diff/{topic}` — Get Report Diff

Shows what changed between the latest and previous report for a given topic.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |

**Response:**
```json
{
  "topic": "AI chip market 2025",
  "diff": "--- previous (2025-10-03)\n+++ latest (2025-10-04)\n@@ -1,5 +1,5 @@\n-Old finding\n+Updated finding"
}
```

---

### `GET /stats` — System Statistics

Returns Redis statistics and system configuration.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |

**Response:**
```json
{
  "redis": {
    "total_keys": 42,
    "cache_entries": 5,
    "active_sessions": 3,
    "memory_used_mb": 12.5,
    "connected_clients": 2,
    "uptime_hours": 168.3
  },
  "tensorzero_url": "http://localhost:3000",
  "guardrail_id": "abc123def"
}
```

---

### `GET /evaluate/{job_id}` — Evaluate Specific Job

Manually triggers LLM-as-judge evaluation on a completed job.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |

**Response:**
```json
{
  "job_id": "a1b2c3d4-...",
  "topic": "AI chip market 2025",
  "scores": {
    "relevance": 0.9,
    "completeness": 0.8,
    "hallucination_risk": 0.2,
    "overall_quality": 0.85
  }
}
```

| Status | Condition |
|--------|-----------|
| 404 | Job not found or not completed |

> [!IMPORTANT]
> This triggers 4 LLM calls. Use sparingly.

---

### `POST /run-evaluation` — Batch Evaluation

Triggers evaluation across multiple topics. Re-runs the entire research pipeline for each topic.

| Attribute | Detail |
|-----------|--------|
| **Auth** | Required |

**Request Body:**
```json
{
  "topics": ["quantum computing", "AI regulations"]
}
```

If `topics` is empty, fetches the 10 most recent topics from the database.

**Response:**
```json
{
  "message": "Batch evaluation started in background",
  "topics": 2
}
```

> [!WARNING]
> Batch evaluation re-runs the full agent pipeline + 4 eval judges per topic. A batch of 10 topics generates 80–160 LLM calls.

---

### `GET /health` — Health Check

Health check endpoint used by ALB. Does **not** require authentication.

**Response:**
```json
{
  "status": "ok",
  "redis": "ok"
}
```

Or if Redis is down:
```json
{
  "status": "degraded",
  "redis": "error"
}
```

> [!NOTE]
> The health check only verifies Redis connectivity. It does not check TensorZero, PostgreSQL, or Bedrock availability.

---

### `GET /` — Frontend

Serves the static HTML frontend (`index.html`).

| Attribute | Detail |
|-----------|--------|
| **Auth** | Not required |

---

## API Summary Table

| Method | Endpoint | Purpose | Auth | Rate Limited | LLM Calls |
|--------|----------|---------|------|-------------|-----------|
| GET | `/` | Serve frontend | No | No | 0 |
| GET | `/health` | Health check | No | No | 0 |
| POST | `/research` | Submit research job | Yes | Yes | 0 (async) |
| GET | `/result/{job_id}` | Get job result | Yes | No | 0 |
| GET | `/result/{job_id}/pdf` | Download PDF | Yes | No | 0 |
| GET | `/session/{session_id}` | Get session history | Yes | No | 0 |
| GET | `/diff/{topic}` | Report diff | Yes | No | 0 |
| GET | `/stats` | System stats | Yes | No | 0 |
| GET | `/evaluate/{job_id}` | Evaluate one job | Yes | No | 4 |
| POST | `/run-evaluation` | Batch evaluate | Yes | No | 8–16 per topic |

## PyRIT Dashboard API (Port 8001)

| Method | Endpoint | Purpose |
|--------|----------|---------|
| GET | `/` | Red team dashboard UI |
| GET | `/run-attacks` | Run all attack types |
| GET | `/run-attacks?types=jailbreak,xpia` | Run specific attacks |
| GET | `/results` | Get attack results |
