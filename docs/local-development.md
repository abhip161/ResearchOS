# Local Development Guide

This guide walks you through setting up and running the **Autonomous Research Agent** on your local workstation for development, testing, and debugging.

---

## 1. Prerequisites

Ensure you have the following installed on your local machine:

| Component | Minimum Version | Purpose | Verification |
|-----------|-----------------|---------|--------------|
| **Python** | 3.12+ (app), 3.11+ (PyRIT) | Application runtime | `python --version` |
| **Docker & Docker Compose** | 24.0+ | Running local Redis, PostgreSQL (pgvector), and TensorZero | `docker compose version` |
| **Git** | 2.40+ | Version control | `git --version` |
| **curl / HTTP client** | Any | Testing REST API endpoints | `curl --version` |
| **AWS CLI** *(Optional)* | 2.x | Only if interacting with AWS Bedrock or Secrets Manager | `aws --version` |

### Required API Keys

You will need the following API keys for external services:

1. **OpenAI API Key**: Primary LLM for agents and evaluation ([platform.openai.com](https://platform.openai.com/api-keys)).
2. **Groq API Key**: Fallback LLM provider routed via TensorZero ([console.groq.com](https://console.groq.com/keys)).
3. **LangSmith API Key**: Tracing and evaluation logging ([smith.langchain.com](https://smith.langchain.com)).

---

## 2. Repository Setup

Clone the repository and set up a Python virtual environment:

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/research-agent.git
cd research-agent

# Create Python 3.12 virtual environment
python -m venv venv

# Activate virtual environment
# Windows (PowerShell):
.\venv\Scripts\Activate.ps1
# Windows (CMD):
.\venv\Scripts\activate.bat
# Linux / macOS:
source venv/bin/activate

# Upgrade pip and install core dependencies
pip install --upgrade pip
pip install -r requirements.txt
```

> [!NOTE]
> During installation, `sentence-transformers` and PyTorch will be installed. On first launch, the `all-MiniLM-L6-v2` embedding model (~90 MB) will be downloaded to your local cache (`~/.cache/huggingface/hub`). In Docker builds, this model is pre-baked into the image.

---

## 3. Local Infrastructure with Docker Compose

To run the full stack locally without provisioning AWS cloud resources, start local instances of **Redis 7.1**, **PostgreSQL 15 with pgvector**, and the **TensorZero Gateway**.

Create a `docker-compose.local.yml` file in the project root:

```yaml
version: '3.8'

services:
  redis:
    image: redis:7.1-alpine
    container_name: local-redis
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    command: ["redis-server", "--appendonly", "yes"]

  postgres:
    image: pgvector/pgvector:pg15
    container_name: local-postgres
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgrespassword
      POSTGRES_DB: research_agent
    volumes:
      - pg-data:/var/lib/postgresql/data

  tensorzero:
    image: tensorzero/gateway:latest
    container_name: local-tensorzero
    ports:
      - "3000:3000"
    volumes:
      - ./tensorzero/tensorzero.toml:/config/tensorzero.toml:ro
    environment:
      TENSORZERO_CONFIG_PATH: /config/tensorzero.toml
      OPENAI_API_KEY: ${OPENAI_API_KEY}
      GROQ_API_KEY: ${GROQ_API_KEY}

volumes:
  redis-data:
  pg-data:
```

Start the containers:

```bash
docker compose -f docker-compose.local.yml up -d
```

Verify that all three services are healthy:
```bash
docker compose -f docker-compose.local.yml ps
```

---

## 4. Local Configuration & Environment Variables

In production, [app/config.py](file:///d:/ai-resarch-agent/app/config.py) pulls settings from AWS Secrets Manager (`research-agent/config`). For local execution, configure environment variables directly in your terminal or a `.env` file:

```bash
# Core API & Service URLs
export REDIS_URL="redis://localhost:6379"
export DATABASE_URL="postgresql://postgres:postgrespassword@localhost:5432/research_agent"
export TENSORZERO_GATEWAY_URL="http://localhost:3000"

# LLM & Observability Keys
export OPENAI_API_KEY="sk-..."
export GROQ_API_KEY="gsk_..."
export LANGSMITH_API_KEY="ls__..."
export LANGSMITH_PROJECT="research-agent"
export LANGSMITH_DATASET_NAME="research-agent-reports"

# Optional App Authentication & Tuning
export API_KEY="dev-test-key"
export RATE_LIMIT_PER_MINUTE="120"
export AGENT_MAX_ITERATIONS="2"
export SEMANTIC_CACHE_TTL="3600"
export SEMANTIC_CACHE_THRESHOLD="0.92"
export LTM_SIMILARITY_THRESHOLD="0.88"
export LTM_RELATED_THRESHOLD="0.50"
export EMBEDDING_MODEL="all-MiniLM-L6-v2"

# Bedrock Guardrail (Set to dummy string if running offline)
export BEDROCK_GUARDRAIL_ID=""
export BEDROCK_GUARDRAIL_VERSION=""
export AWS_DEFAULT_REGION="us-east-1"
```

> [!TIP]
> If you do not have AWS Bedrock credentials configured locally, the guardrail calls can be bypassed or mocked in development by leaving `BEDROCK_GUARDRAIL_ID` empty or modifying [app/guardrails.py](file:///d:/ai-resarch-agent/app/guardrails.py) to return `("ALLOWED", None)`.

---

## 5. Database Schema Initialization

Connect to your local PostgreSQL instance and initialize the `pgvector` extension and the `reports` table:

```bash
docker exec -i local-postgres psql -U postgres -d research_agent << 'EOF'
-- Enable the vector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create the reports table
CREATE TABLE IF NOT EXISTS reports (
    id SERIAL PRIMARY KEY,
    topic TEXT NOT NULL,
    report TEXT NOT NULL,
    embedding vector(384),
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Create IVFFlat index for fast cosine similarity search
CREATE INDEX IF NOT EXISTS reports_embedding_idx 
ON reports 
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
EOF
```

---

## 6. Running the Applications Locally

### 1. Run the Main FastAPI Application & Background Worker

The FastAPI application manages API endpoints and spins up the background Redis Streams worker on startup:

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

You should see startup logs indicating:
- PostgreSQL pool initialized (`app.pool`)
- Redis client connected (`app.queue`, `app.cache`)
- Redis Stream consumer group initialized (`research-agent-group`)
- Sentence-transformers model loaded (`all-MiniLM-L6-v2`)
- Background worker task started (`asyncio.create_task(worker_loop())`)

### 2. Run the PyRIT Red Team Dashboard (Optional)

In a separate terminal window, activate the virtual environment and install PyRIT dependencies:

```bash
cd pyrit_dashboard
pip install -r requirements.txt

# Run the PyRIT dashboard
uvicorn main:app --host 0.0.0.0 --port 8001 --reload
```

Access the dashboard in your browser at `http://localhost:8001`.

### 3. Serving the Frontend UI

Open [index.html](file:///d:/ai-resarch-agent/index.html) in your browser:
- You can serve it directly with Python's HTTP server:
  ```bash
  python -m http.server 8080
  ```
- Navigate to `http://localhost:8080`.
- In the frontend UI, configure the API Base URL to `http://localhost:8000` and enter your development API key (`dev-test-key`).

---

## 7. Testing Your Local Setup

### Step 1: Health Check

```bash
curl http://localhost:8000/health
```
**Expected Response:**
```json
{"status": "healthy"}
```

### Step 2: System Stats

```bash
curl http://localhost:8000/stats -H "X-API-Key: dev-test-key"
```
**Expected Response:**
```json
{
  "redis_connected": true,
  "queue_depth": 0,
  "cache_keys_count": 0,
  "embedding_model": "all-MiniLM-L6-v2"
}
```

### Step 3: Submit a Research Job

```bash
curl -X POST http://localhost:8000/research \
  -H "Content-Type: application/json" \
  -H "X-API-Key: dev-test-key" \
  -d '{
    "topic": "Neuromorphic computing hardware architectures 2025",
    "session_id": "local_dev_session_1",
    "output_format": "text"
  }'
```
**Expected Response:**
```json
{
  "job_id": "job-a1b2c3d4e5",
  "session_id": "local_dev_session_1"
}
```

### Step 4: Poll for Results

```bash
curl http://localhost:8000/result/job-a1b2c3d4e5 -H "X-API-Key: dev-test-key"
```

While running, the endpoint returns `{"status": "pending"}`. Once complete (usually 25–45 seconds), it returns the structured report object with all sections:
- `executive_summary`
- `key_findings`
- `detailed_analysis`
- `conclusion`
- `critic_score`
- `eval_scores`

### Step 5: Test Output Formats

**Download PDF:**
```bash
curl http://localhost:8000/result/job-a1b2c3d4e5/pdf \
  -H "X-API-Key: dev-test-key" \
  -o local_report.pdf
```

**View Report Diff:**
```bash
curl "http://localhost:8000/diff/Neuromorphic%20computing%20hardware%20architectures%202025" \
  -H "X-API-Key: dev-test-key"
```

---

## 8. Troubleshooting Common Issues

### Issue 1: `pgvector` extension not recognized
- **Symptom:** `type "vector" does not exist` or `syntax error at or near "vector"`
- **Fix:** Ensure you are using the `pgvector/pgvector:pg15` Docker image, not the vanilla `postgres:15` image. Run `CREATE EXTENSION IF NOT EXISTS vector;` in the target database.

### Issue 2: TensorZero Gateway `Connection Refused` on port 3000
- **Symptom:** `httpx.ConnectError: [Errno 111] Connection refused` in `app.agents`
- **Fix:** Check that the TensorZero container is running (`docker compose ps`). Verify that `tensorzero/tensorzero.toml` contains valid OpenAI and Groq API keys and that the ports are properly mapped.

### Issue 3: Redis `NOGROUP` or Consumer Group Error
- **Symptom:** `NOGROUP No such key 'research:jobs' or consumer group 'research-agent-group'`
- **Fix:** The queue initialization creates the stream and group with `MKSTREAM`. If Redis was restarted without persistence, restart the FastAPI app so the lifespan hook re-creates the consumer group.

### Issue 4: Sentence-Transformers Hub Timeout
- **Symptom:** `ReadTimeout` or network error while loading `all-MiniLM-L6-v2`
- **Fix:** Ensure internet connectivity on first run. If offline, copy an existing model directory to `~/.cache/huggingface/hub/` and set `export HF_HUB_OFFLINE=1`.

### Issue 5: Bedrock Guardrails `NoCredentialsError`
- **Symptom:** `botocore.exceptions.NoCredentialsError: Unable to locate credentials`
- **Fix:** If not actively testing AWS Bedrock locally, unset `BEDROCK_GUARDRAIL_ID` in your environment or set AWS development credentials via `aws configure`.
