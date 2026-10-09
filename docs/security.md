# Security

## Security Overview

The Research Agent implements production security at multiple layers: API authentication, content safety guardrails, secrets management, network isolation, and AI safety testing. This document separates what is actually implemented from what is recommended.

---

## Implemented Security

### API Authentication

| Mechanism | Implementation | Location |
|-----------|---------------|----------|
| API Key | Single shared `X-API-Key` header | [auth.py](file:///d:/ai-resarch-agent/app/auth.py) |
| Key storage | AWS Secrets Manager | `research-agent/config` |
| Key validation | String comparison in FastAPI dependency | All protected routes |

**How it works:**
```python
async def require_api_key(request):
    config = request.app.state.config
    if not config.api_key:
        return  # auth disabled when no key configured
    key = request.headers.get("X-API-Key", "")
    if key != config.api_key:
        raise HTTPException(status_code=401)
```

> [!WARNING]
> If `API_KEY` is not set in Secrets Manager, authentication is completely disabled. This is by design for development but should always be set in production.

### Rate Limiting

| Setting | Value |
|---------|-------|
| Requests per window | 10 (configurable) |
| Window duration | 60 seconds (configurable) |
| Scope | Per client IP |
| Mechanism | Redis INCR + EXPIRE |

Rate limiting protects against abuse and prevents a single client from overwhelming the LLM quota.

### Content Safety (Bedrock Guardrails)

**Implemented filters:**

| Filter Type | Threshold | Blocked Content |
|-------------|-----------|-----------------|
| Content filters | HIGH | HATE, VIOLENCE, SEXUAL, INSULTS, MISCONDUCT, PROMPT_ATTACK |
| Topic denials | Blocked | Weapons, illegal activities, self-harm |
| PII blocking | Blocked | SSN, credit cards, AWS keys |
| PII anonymization | Anonymized | Email, phone numbers |
| Word filtering | Blocked | Profanity |

Both input (user topics) and output (generated reports) pass through guardrails:
- **Input**: blocked topics return HTTP 400 immediately
- **Output**: blocked reports set job status to "blocked", never returned to user
- **Fail-closed**: when in doubt, the guardrail blocks content

### Secrets Management

| Aspect | Implementation |
|--------|---------------|
| Storage | AWS Secrets Manager (`research-agent/config`) |
| Access | IAM role-based, scoped to ECS task execution role |
| Injection | ECS task definition injects secrets at container runtime |
| Source code | No hardcoded API keys found in repository |
| `.env` files | Listed in `.gitignore` |
| Logging | No credential logging detected in application code |

Terraform creates the secret with `REPLACE_ME` placeholder values. Users manually update the 3 API keys (OpenAI, Groq, LangSmith) after initial deployment.

### Network Security

| Layer | Implementation |
|-------|---------------|
| VPC | Isolated 10.0.0.0/16 network |
| Database subnet | Private subnets (no internet access) |
| VPC Endpoints | AWS services accessed via private endpoints (no NAT) |
| ALB | Internet-facing, health check on `/health` |
| Security groups | Scoped ingress/egress per service |

### IAM

| Role | Purpose | Scoping |
|------|---------|---------|
| ECS Execution Role | Pull images from ECR, read secrets | ECR + Secrets Manager |
| ECS Task Role | Bedrock, CloudWatch Logs | Bedrock guardrail + CW Logs |
| EventBridge Role | Trigger ECS tasks | ECS RunTask |

### CORS

```python
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_methods=["GET", "POST"],
    allow_headers=["*"],
)
```

> [!NOTE]
> CORS is set to `allow_origins=["*"]` which allows requests from any origin. This is acceptable for a public-facing API but should be tightened if the frontend is served from a known domain.

### AI Safety — Red Team Testing

The PyRIT dashboard provides automated adversarial testing:
- **Jailbreak**: Direct bypass attempts
- **XPIA**: Cross-prompt injection attacks
- **Crescendo**: Gradual escalation attacks
- **Skeleton Key**: Authority-claiming bypass attempts

Automated red team runs every Monday at 2 AM UTC via EventBridge.

---

## Security Concerns

### Critical

| Finding | Risk | Location |
|---------|------|----------|
| `REPLACE_ME` placeholder API keys in Terraform | If Secrets Manager is not manually updated, the app starts with invalid keys | [main.tf:649–650](file:///d:/ai-resarch-agent/terraform/main.tf#L649-L650) |
| RDS `backup_retention_period = 0` | No automated database backups; data loss on failure | [main.tf](file:///d:/ai-resarch-agent/terraform/main.tf) |

### High

| Finding | Risk | Location |
|---------|------|----------|
| PyRIT dashboard publicly accessible (port 8001, 0.0.0.0/0) | Anyone who discovers the URL can trigger attacks that consume API quota | ALB listener rule |
| No authentication on PyRIT dashboard | No access control on red team tool | [pyrit_dashboard/main.py](file:///d:/ai-resarch-agent/pyrit_dashboard/main.py) |
| AWS credentials as GitHub Secrets (not OIDC) | Static credentials are less secure than federated identity | [deploy.yml:24–25](file:///d:/ai-resarch-agent/.github/workflows/deploy.yml#L24-L25) |

### Medium

| Finding | Risk |
|---------|------|
| Auth disabled when API_KEY is empty | Accidental unauthenticated deployment |
| Docker containers run as root | Container escape risk; no `USER` directive in Dockerfiles |
| Single shared API key for all clients | No per-client tracking or revocation |
| No WAF on ALB | No protection against common web attacks (SQL injection, XSS in headers) |

---

## Recommended Improvements

### Short-term
1. **Restrict PyRIT dashboard** — add IP allowlist or authentication, or remove the ALB listener for port 8001
2. **Add non-root Docker user** — add `USER appuser` to Dockerfiles
3. **Enable RDS backups** — set `backup_retention_period = 7`
4. **Switch to OIDC for GitHub Actions** — use `aws-actions/configure-aws-credentials` with OIDC role instead of static keys

### Medium-term
5. **Per-client API keys** — support multiple API keys for different clients
6. **Add WAF** — AWS WAF on ALB for rate limiting and common attack protection
7. **Move ECS tasks to private subnets** — use NAT gateway or VPC endpoints for outbound traffic
8. **HTTPS enforcement** — configure ALB with ACM certificate, redirect HTTP to HTTPS

### Long-term
9. **JWT-based authentication** — replace API key with JWT tokens for proper session management
10. **RBAC** — role-based access control for admin vs user operations
11. **Audit logging** — log all API access with client identity and actions
