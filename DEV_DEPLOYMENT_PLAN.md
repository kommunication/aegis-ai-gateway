# AEGIS AI Gateway - Dev Deployment Plan

**Status**: Ready for first deployment  
**Target Environment**: Development/Staging  
**Date**: 2025-01-22

---

## 🎯 Deployment Overview

We'll deploy aegis-ai-gateway to a dev environment with:
- PostgreSQL + Redis (Docker)
- AEGIS Gateway binary
- Test scenarios to validate all major features
- Monitoring setup (Prometheus metrics)

**Time Estimate**: 2-3 hours

---

## 📋 Prerequisites Checklist

### Infrastructure
- [ ] Server/VM with Docker installed
- [ ] Docker Compose v2+ available
- [ ] Go 1.25+ installed (or mise)
- [ ] Ports available: 8080 (gateway), 9090 (metrics), 5432 (postgres), 6379 (redis)
- [ ] SSL certificate (optional for dev, required for production)

### Credentials
- [ ] OpenAI API key (for gpt-4o, gpt-4o-mini testing)
- [ ] Anthropic API key (for claude-3.5-sonnet testing)
- [ ] Azure OpenAI credentials (optional)
- [ ] Database credentials decided

### Access
- [ ] SSH access to deployment server
- [ ] GitHub access to clone repo
- [ ] Ability to set environment variables

---

## 🚀 Deployment Steps

### Step 1: Prepare the Environment (15 min)

**On the deployment server:**

```bash
# Clone the repository
cd /opt  # or your preferred location
git clone https://github.com/kommunication/aegis-ai-gateway.git
cd aegis-ai-gateway

# Checkout main branch (Sprint 7-10 already merged)
git checkout main
git pull origin main

# Verify we're on the latest
git log --oneline -5
```

**Install mise (if not already present):**

```bash
# Install mise
curl https://mise.run | sh
echo 'eval "$(~/.local/bin/mise activate bash)"' >> ~/.bashrc
source ~/.bashrc

# Verify
mise --version
```

**Setup project dependencies:**

```bash
# mise will auto-install Go 1.25 + golangci-lint
mise install

# Install Go dependencies
mise run setup
```

---

### Step 2: Configure Environment (15 min)

**Create `.env` file:**

```bash
cp .env.example .env
nano .env  # or vim, emacs, etc.
```

**Fill in your API keys:**

```bash
# Required
OPENAI_API_KEY=sk-proj-...
ANTHROPIC_API_KEY=sk-ant-...

# Optional (for Azure testing)
AZURE_OPENAI_KEY=
AZURE_OPENAI_ENDPOINT=

# Optional (for OpenAI org scoping)
OPENAI_ORG_ID=
```

**Review database credentials:**

The defaults in `.mise.toml` are:
- DB_USER=aegis
- DB_PASSWORD=aegis-dev
- DB_NAME=aegis

For dev, these are fine. For production, use strong credentials.

---

### Step 3: Start Infrastructure (10 min)

**Start PostgreSQL, Redis, and NLP filter service:**

```bash
# Start all services in Docker
mise run services:up

# Verify services are healthy
docker ps
docker compose -f deploy/docker-compose.yaml ps

# Check logs
mise run services:logs
# Press Ctrl+C when satisfied
```

**Expected output:**
```
✔ Container aegis-postgres     Healthy
✔ Container aegis-redis         Healthy
✔ Container aegis-filter-nlp    Healthy
```

**Test database connection:**

```bash
docker exec -it aegis-postgres psql -U aegis -d aegis -c "SELECT version();"
```

**Test Redis connection:**

```bash
docker exec -it aegis-redis redis-cli ping
# Expected: PONG
```

---

### Step 4: Run Database Migrations (5 min)

**Apply all migrations:**

```bash
mise run db:migrate
```

**Expected output:**
```
Running migrations up...
Applied migration: 001_initial_schema.up.sql
Applied migration: 002_add_api_keys.up.sql
Applied migration: 003_add_audit_logs.up.sql
Applied migration: 004_create_usage_records_detailed.up.sql
Applied migration: 005_create_audit_events.up.sql
✓ All migrations applied
```

**Verify tables created:**

```bash
docker exec -it aegis-postgres psql -U aegis -d aegis -c "\dt"
```

**Expected tables:**
- api_keys
- audit_logs
- audit_events
- usage_records
- schema_migrations

---

### Step 5: Build the Gateway (5 min)

**Compile binaries:**

```bash
mise run build
```

**Verify binaries:**

```bash
ls -lh bin/
# Should see: gateway, keygen, migrate
```

**Check version:**

```bash
./bin/gateway -version
# Or if not implemented:
./bin/gateway -h
```

---

### Step 6: Generate API Keys (5 min)

**Generate a test API key:**

```bash
mise run keygen
```

**Expected output:**
```
Generated API key: ak_abcd1234efgh5678ijkl9012mnop3456qrst7890
Organization: dev-org
Team: dev-team
Name: dev-key
Classification: INTERNAL
Expires: 2026-01-22

⚠️  Save this key — it's shown only once!
```

**Save this key** — you'll need it for testing.

**Generate keys for different scenarios:**

```bash
# High classification key (for testing gating)
go run ./cmd/keygen \
  -org security-team \
  -team red-team \
  -name high-sec-key \
  -classification CONFIDENTIAL \
  -expires 90d

# Budget-limited key (for testing rate limits)
go run ./cmd/keygen \
  -org finance-team \
  -team analytics \
  -name budget-test-key \
  -classification INTERNAL \
  -expires 30d
```

---

### Step 7: Start the Gateway (5 min)

**Run the gateway:**

```bash
# Option 1: Using mise (recommended for dev)
mise run run

# Option 2: Direct binary
./bin/gateway -config configs

# Option 3: With verbose logging
LOG_LEVEL=debug mise run run
```

**Expected startup logs:**

```
2025-01-22T10:00:00Z INFO  Starting AEGIS AI Gateway
2025-01-22T10:00:00Z INFO  Database pool initialized (max_conns=25)
2025-01-22T10:00:00Z INFO  Redis cache connected
2025-01-22T10:00:00Z INFO  Loaded 8 model configurations
2025-01-22T10:00:00Z INFO  Loaded 4 provider configurations
2025-01-22T10:00:00Z INFO  HTTP server listening on :8080
2025-01-22T10:00:00Z INFO  Metrics server listening on :9090
```

**In another terminal, verify it's running:**

```bash
# Health check
curl http://localhost:8080/aegis/v1/health

# Expected response:
{
  "status": "healthy",
  "database": {
    "connected": true,
    "pool_stats": { ... }
  }
}
```

**Check metrics endpoint:**

```bash
curl http://localhost:9090/metrics | grep aegis
```

---

## ✅ Test Scenarios

Run these to validate the deployment. Each scenario tests a critical feature.

### Scenario 1: Basic Chat Completion (OpenAI)

**Test**: Simple request to OpenAI model through the gateway

```bash
API_KEY="ak_your_key_here"  # Replace with your generated key

curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-fast",
    "messages": [
      {"role": "user", "content": "Say hello in exactly 5 words"}
    ]
  }' | jq
```

**Expected response:**
```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1705924800,
  "model": "gpt-4o-mini",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Hello there, how are you?"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 12,
    "completion_tokens": 7,
    "total_tokens": 19
  },
  "estimated_cost_usd": 0.0000285
}
```

**Verify**:
- [ ] Response includes `estimated_cost_usd`
- [ ] `usage` object has token counts
- [ ] Response is valid JSON
- [ ] Content makes sense

**Check database:**

```bash
docker exec -it aegis-postgres psql -U aegis -d aegis -c \
  "SELECT request_id, model_used, prompt_tokens, completion_tokens, cost_usd 
   FROM usage_records 
   ORDER BY created_at DESC 
   LIMIT 1;"
```

**Verify**:
- [ ] Record exists with matching token counts
- [ ] Cost is calculated correctly
- [ ] Timestamp is recent

---

### Scenario 2: Streaming Response (Anthropic)

**Test**: Streaming chat completion with Claude

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-smart",
    "messages": [
      {"role": "user", "content": "Count from 1 to 5, one number per line"}
    ],
    "stream": true
  }'
```

**Expected output** (SSE chunks):
```
data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1705924800,"model":"claude-3.5-sonnet","choices":[{"index":0,"delta":{"role":"assistant","content":"1"},"finish_reason":null}]}

data: {"id":"chatcmpl-...","object":"chat.completion.chunk","created":1705924800,"model":"claude-3.5-sonnet","choices":[{"index":0,"delta":{"content":"\n2"},"finish_reason":null}]}

...

data: [DONE]
```

**Verify**:
- [ ] Chunks arrive in real-time (not all at once)
- [ ] Content is streamed progressively
- [ ] Final chunk has `[DONE]`
- [ ] No errors in gateway logs

**Check metrics:**

```bash
curl -s http://localhost:9090/metrics | grep -E 'aegis_streaming_(chunk|time_to_first|tokens_per_second|duration)'
```

**Verify**:
- [ ] `aegis_streaming_chunk_total` incremented
- [ ] `aegis_streaming_time_to_first_token_ms` recorded
- [ ] `aegis_streaming_tokens_per_second` calculated
- [ ] `aegis_streaming_duration_ms` tracked

---

### Scenario 3: Content Filtering (Secrets Scanner)

**Test**: Request with AWS credentials should be blocked

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-fast",
    "messages": [
      {"role": "user", "content": "My AWS key is AKIAIOSFODNN7EXAMPLE"}
    ]
  }' | jq
```

**Expected response** (HTTP 400):
```json
{
  "error": {
    "message": "Content filter violation: secrets detected",
    "type": "content_filter_error",
    "code": "secrets_detected"
  }
}
```

**Verify**:
- [ ] Request is blocked (400 status)
- [ ] Error message mentions secrets
- [ ] No request sent to upstream provider (check provider logs)

**Check audit log:**

```bash
docker exec -it aegis-postgres psql -U aegis -d aegis -c \
  "SELECT event_type, action, result, details 
   FROM audit_events 
   WHERE event_type = 'filter_block' 
   ORDER BY timestamp DESC 
   LIMIT 1;"
```

**Verify**:
- [ ] Audit event logged
- [ ] Event type is `filter_block`
- [ ] Details include reason (secrets detected)

---

### Scenario 4: Rate Limiting

**Test**: Exceed rate limit for a key

First, check the rate limit config:

```bash
cat configs/gateway.yaml | grep -A 5 rate_limit
```

**Flood the endpoint:**

```bash
# Send 20 requests rapidly
for i in {1..20}; do
  curl -X POST http://localhost:8080/v1/chat/completions \
    -H "Authorization: Bearer $API_KEY" \
    -H "Content-Type: application/json" \
    -d '{"model":"aegis-fast","messages":[{"role":"user","content":"Test"}]}' \
    -w "\nStatus: %{http_code}\n" \
    -o /dev/null -s &
done
wait
```

**Expected behavior**:
- First ~10 requests succeed (200 OK)
- Subsequent requests return 429 (Too Many Requests)

**Check one 429 response:**

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"aegis-fast","messages":[{"role":"user","content":"Test"}]}' \
  -i
```

**Expected headers:**
```
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 10
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705924860
```

**Verify**:
- [ ] 429 status returned when limit exceeded
- [ ] `Retry-After` header present
- [ ] Rate limit headers show remaining = 0

**Check metrics:**

```bash
curl -s http://localhost:9090/metrics | grep aegis_ratelimit
```

**Verify**:
- [ ] `aegis_ratelimit_exceeded_total` incremented

---

### Scenario 5: Classification Gating

**Test**: Request requiring high classification with low-class key

Generate a low-classification key if you haven't:

```bash
go run ./cmd/keygen \
  -org public-team \
  -team external \
  -name public-key \
  -classification PUBLIC \
  -expires 7d
```

**Attempt to use a CONFIDENTIAL model:**

```bash
LOW_CLASS_KEY="ak_..."  # Your PUBLIC classification key

curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $LOW_CLASS_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-secure",
    "messages": [
      {"role": "user", "content": "Test"}
    ]
  }' | jq
```

**Expected response** (HTTP 403):
```json
{
  "error": {
    "message": "Insufficient classification level",
    "type": "classification_error",
    "code": "classification_too_low"
  }
}
```

**Verify**:
- [ ] Request is blocked (403 status)
- [ ] Error message clear about classification
- [ ] Audit event logged

---

### Scenario 6: Provider Fallback

**Test**: Primary provider fails, fallback succeeds

**Simulate OpenAI failure** by using an invalid API key temporarily:

```bash
# Stop the gateway
# Edit .env: set OPENAI_API_KEY=sk-invalid
# Restart gateway
mise run run
```

**Send request to aegis-fast (primary: OpenAI, fallback: Anthropic):**

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-fast",
    "messages": [
      {"role": "user", "content": "Hello"}
    ]
  }' | jq
```

**Expected response:**
- Response succeeds (200 OK)
- `model` field shows fallback provider was used (e.g., "claude-3.5-sonnet")
- Gateway logs show fallback triggered

**Verify gateway logs:**
```
WARN  Primary provider failed, attempting fallback
INFO  Fallback to anthropic successful
```

**Check metrics:**

```bash
curl -s http://localhost:9090/metrics | grep aegis_provider_fallback
```

**Verify**:
- [ ] `aegis_provider_fallback_total` incremented
- [ ] Response successful despite primary failure

**Restore OpenAI key** and restart gateway.

---

### Scenario 7: Cost Tracking

**Test**: Verify cost calculation across providers

**Send 3 requests** to different models:

```bash
# Request 1: gpt-4o-mini (cheap)
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-fast",
    "messages": [{"role": "user", "content": "Hi"}]
  }' | jq '.estimated_cost_usd'

# Request 2: gpt-4o (expensive)
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hi"}]
  }' | jq '.estimated_cost_usd'

# Request 3: claude-3.5-sonnet (mid-range)
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-smart",
    "messages": [{"role": "user", "content": "Hi"}]
  }' | jq '.estimated_cost_usd'
```

**Verify**:
- [ ] gpt-4o cost > claude cost > gpt-4o-mini cost
- [ ] Costs are reasonable (check against provider pricing)

**Check cumulative cost:**

```bash
docker exec -it aegis-postgres psql -U aegis -d aegis -c \
  "SELECT 
     COUNT(*) as total_requests,
     SUM(cost_usd) as total_cost,
     AVG(cost_usd) as avg_cost,
     model_used
   FROM usage_records
   GROUP BY model_used
   ORDER BY total_cost DESC;"
```

**Check Prometheus metrics:**

```bash
curl -s http://localhost:9090/metrics | grep aegis_cost_usd_total
```

**Verify**:
- [ ] `aegis_cost_usd_total` matches database SUM
- [ ] Per-model costs broken down correctly

---

### Scenario 8: Model Listing

**Test**: List available models

```bash
curl -X GET http://localhost:8080/v1/models \
  -H "Authorization: Bearer $API_KEY" | jq
```

**Expected response:**
```json
{
  "object": "list",
  "data": [
    {
      "id": "aegis-fast",
      "object": "model",
      "created": 1705924800,
      "owned_by": "aegis"
    },
    {
      "id": "aegis-smart",
      "object": "model",
      "created": 1705924800,
      "owned_by": "aegis"
    },
    ...
  ]
}
```

**Verify**:
- [ ] All configured models listed
- [ ] Response is OpenAI-compatible format
- [ ] No errors

---

### Scenario 9: Invalid Requests

**Test**: Malformed input validation

**Missing required field (messages):**

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "aegis-fast"
  }' | jq
```

**Expected response** (HTTP 400):
```json
{
  "error": {
    "message": "Invalid request: messages field is required",
    "type": "invalid_request_error",
    "code": "missing_required_field"
  }
}
```

**Invalid model:**

```bash
curl -X POST http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "nonexistent-model",
    "messages": [{"role": "user", "content": "Test"}]
  }' | jq
```

**Expected response** (HTTP 404):
```json
{
  "error": {
    "message": "Model not found: nonexistent-model",
    "type": "invalid_request_error",
    "code": "model_not_found"
  }
}
```

**Verify**:
- [ ] Validation errors clear and actionable
- [ ] No request sent to upstream providers
- [ ] Appropriate HTTP status codes

---

### Scenario 10: Health & Monitoring

**Test**: All monitoring endpoints functional

**Health check:**

```bash
curl http://localhost:8080/aegis/v1/health | jq
```

**Expected response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-01-22T10:00:00Z",
  "database": {
    "connected": true,
    "pool_stats": {
      "max_conns": 25,
      "open_conns": 3,
      "in_use": 1,
      "idle": 2
    }
  },
  "redis": {
    "connected": true,
    "latency_ms": 1.2
  }
}
```

**Prometheus metrics:**

```bash
# Raw metrics
curl http://localhost:9090/metrics

# Specific metric families
curl -s http://localhost:9090/metrics | grep -E '^aegis_' | head -20
```

**Expected metrics:**
- `aegis_requests_total` - Request counter
- `aegis_request_duration_ms` - Latency histogram
- `aegis_tokens_total` - Token usage
- `aegis_cost_usd_total` - Cost tracking
- `aegis_ratelimit_exceeded_total` - Rate limit hits
- `aegis_streaming_chunk_total` - Streaming metrics
- `aegis_provider_fallback_total` - Fallback events

**Verify**:
- [ ] Health endpoint returns 200
- [ ] Database connection healthy
- [ ] Redis connection healthy
- [ ] Metrics endpoint accessible
- [ ] Key metrics present and incrementing

---

## 🐛 Troubleshooting

### Gateway Won't Start

**Symptom**: `mise run run` fails immediately

**Check**:
```bash
# Database accessible?
docker exec -it aegis-postgres psql -U aegis -d aegis -c "SELECT 1;"

# Redis accessible?
docker exec -it aegis-redis redis-cli ping

# Migrations applied?
docker exec -it aegis-postgres psql -U aegis -d aegis -c "\dt"

# Config files present?
ls -la configs/
```

**Common fixes**:
- Ensure Docker services running: `mise run services:up`
- Apply migrations: `mise run db:migrate`
- Check `.env` file exists and has valid API keys
- Check port conflicts: `lsof -i :8080`

---

### Authentication Failures

**Symptom**: `curl` returns 401 Unauthorized

**Check**:
```bash
# Key exists in database?
docker exec -it aegis-postgres psql -U aegis -d aegis -c \
  "SELECT api_key_hash, organization, expires_at FROM api_keys;"

# Redis cache accessible?
docker exec -it aegis-redis redis-cli KEYS "auth:*"

# Using correct header format?
# Correct:   Authorization: Bearer ak_...
# Incorrect: Authorization: ak_...
# Incorrect: X-API-Key: ak_...
```

**Common fixes**:
- Regenerate key: `mise run keygen`
- Use full `Bearer ak_...` format in header
- Check key hasn't expired

---

### Provider Errors

**Symptom**: Gateway returns 502 Bad Gateway

**Check**:
```bash
# API keys valid?
echo $OPENAI_API_KEY
echo $ANTHROPIC_API_KEY

# Provider reachable?
curl -I https://api.openai.com/v1/models
curl -I https://api.anthropic.com/v1/messages

# Check gateway logs
docker logs aegis-gateway
```

**Common fixes**:
- Verify API keys in `.env` are current
- Check network connectivity
- Test provider APIs directly with `curl`
- Check provider status pages

---

### High Latency

**Symptom**: Requests take >5 seconds

**Check**:
```bash
# Database pool exhausted?
curl http://localhost:8080/aegis/v1/health | jq '.database.pool_stats'

# Redis slow?
docker exec -it aegis-redis redis-cli --latency

# Provider slow?
curl -w "\nTime: %{time_total}s\n" \
  -X POST https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"Hi"}]}'
```

**Common fixes**:
- Increase DB pool size in config
- Check Redis memory usage: `docker exec aegis-redis redis-cli INFO memory`
- Use faster models (gpt-4o-mini vs gpt-4o)
- Check network latency to providers

---

## 📊 Success Criteria

Deployment is successful when:

### Core Functionality
- [x] Gateway starts without errors
- [x] Health endpoint returns 200
- [x] Database and Redis connections healthy
- [x] API keys can be generated
- [x] Chat completions work (non-streaming)
- [x] Streaming completions work
- [x] Model listing works

### Security & Governance
- [x] Authentication enforced (401 without key)
- [x] Rate limiting active (429 when exceeded)
- [x] Content filtering blocks secrets
- [x] Classification gating enforced
- [x] Audit logs written to database

### Observability
- [x] Usage records written to database
- [x] Cost calculated and tracked
- [x] Prometheus metrics exposed
- [x] All key metrics incrementing correctly

### Reliability
- [x] Provider fallback works
- [x] Invalid requests rejected gracefully
- [x] Timeout handling works
- [x] No memory leaks (monitor over 1 hour)

---

## 🎉 Next Steps After Successful Deployment

1. **Set up monitoring**:
   - Configure Prometheus scraping
   - Set up Grafana dashboards
   - Create alerts (rate limit exceeded, high error rate, etc.)

2. **Load testing**:
   - Use `hey`, `wrk`, or `k6` to simulate production load
   - Measure throughput (requests/sec)
   - Identify bottlenecks

3. **Documentation**:
   - Update deployment docs with any learnings
   - Document any config tweaks made
   - Create runbook for common issues

4. **Production planning**:
   - SSL/TLS setup
   - Strong database credentials
   - Firewall rules
   - Backup strategy
   - High availability (multiple instances, load balancer)

5. **Feature work**:
   - Cost analytics dashboard
   - Multi-tenancy & RBAC
   - Advanced routing
   - Caching layer

---

**Good luck with the deployment!** 🚀

If you encounter issues not covered here, check:
- Gateway logs: `docker logs aegis-gateway`
- Service logs: `mise run services:logs`
- GitHub issues: https://github.com/kommunication/aegis-ai-gateway/issues

**Questions?** Open an issue or contact the team.
