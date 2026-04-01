# Langfuse Local Setup Guide

This guide provides step-by-step instructions for setting up and configuring Langfuse for local deployment with LiteLLM in the Heretek OpenClaw system.

## Overview

Langfuse is an open-source LLM observability platform that provides tracing, monitoring, and analytics for your LLM applications. This guide covers the local deployment configuration.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Heretek OpenClaw Stack                        │
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │   LiteLLM    │────▶│   Langfuse   │────▶│  ClickHouse  │    │
│  │   Gateway    │     │   :3000      │     │  Analytics   │    │
│  │     :4000    │     │              │     │              │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│         │                    │                                  │
│         │                    ▼                                  │
│         │            ┌──────────────┐                          │
│         └───────────▶│  PostgreSQL  │                          │
│                      │   (Langfuse) │                          │
│                      └──────────────┘                          │
└─────────────────────────────────────────────────────────────────┘
```

## Configuration Files

### 1. `.env` File Settings

The following environment variables are configured in [`heretek-openclaw-core/.env`](../../../heretek-openclaw-core/.env):

```bash
# Langfuse Observability (Local Deployment)
LANGFUSE_ENABLED=true
LANGFUSE_PUBLIC_KEY=pk-lf-local-deployment-key
LANGFUSE_SECRET_KEY=sk-lf-local-deployment-key
LANGFUSE_HOST=http://langfuse:3000
LANGFUSE_POSTGRES_PASSWORD=<your-password>
LANGFUSE_SALT=<your-salt>
LANGFUSE_NEXTAUTH_SECRET=<your-secret>
```

### 2. `docker-compose.yml` Settings

The Langfuse service is configured in [`heretek-openclaw-core/docker-compose.yml`](../../../heretek-openclaw-core/docker-compose.yml:86):

```yaml
langfuse:
  image: langfuse/langfuse:latest
  container_name: heretek-langfuse
  ports:
    - "${LANGFUSE_PORT:-3000}:3000"
  environment:
    - DATABASE_URL=postgresql://langfuse:${LANGFUSE_POSTGRES_PASSWORD}@langfuse-postgres:5432/langfuse
    - CLICKHOUSE_URL=${CLICKHOUSE_URL:-clickhouse://default:clickhouse_password@clickhouse:9000}
    # ... additional settings
```

### 3. `litellm_config.yaml` Settings

LiteLLM is configured to send observability data to Langfuse in [`heretek-openclaw-core/litellm_config.yaml`](../../../heretek-openclaw-core/litellm_config.yaml:828):

```yaml
observability:
  langfuse:
    enabled: true
    public_key: os.environ/LANGFUSE_PUBLIC_KEY
    secret_key: os.environ/LANGFUSE_SECRET_KEY
    host: os.environ/LANGFUSE_HOST
```

---

## Step-by-Step Setup Instructions

### Step 1: Start the Docker Services

Start all services including Langfuse, ClickHouse, and PostgreSQL:

```bash
cd heretek-openclaw-core
docker compose up -d
```

Verify all services are running:

```bash
docker compose ps
```

You should see:
- `heretek-langfuse` - Langfuse application
- `heretek-langfuse-db` - Langfuse PostgreSQL database
- `heretek-clickhouse` - ClickHouse analytics database
- `heretek-litellm` - LiteLLM gateway

### Step 2: Access Langfuse UI

Open your browser and navigate to:

**http://localhost:3000**

> **Note:** The first time you access Langfuse, you will need to create an organization.

### Step 3: Create Your Organization (First-Time Setup)

1. On the Langfuse welcome page, click **"Sign Up"** or **"Create Account"**
2. Fill in the registration form:
   - **Email**: Your email address
   - **Password**: Create a secure password
   - **Organization Name**: e.g., "Heretek OpenClaw" or your preferred name
3. Click **"Create Organization"**
4. Verify your email if required (depending on configuration)

### Step 4: Create a Project

1. After logging in, you will be prompted to create your first project
2. Click **"Create Project"**
3. Enter a project name: e.g., **"OpenClaw Agents"**
4. Optionally add a description
5. Click **"Create"**

### Step 5: Get API Keys

1. Navigate to your project by clicking on it in the dashboard
2. Click on **"Settings"** (gear icon) in the left sidebar
3. Scroll down to **"API Keys"** section
4. You will see:
   - **Public Key** (starts with `pk-`)
   - **Secret Key** (starts with `sk-`)
5. Copy both keys

### Step 6: Update `.env` File

Update the [`heretek-openclaw-core/.env`](../../../heretek-openclaw-core/.env) file with your actual API keys:

```bash
# Open the .env file
nano heretek-openclaw-core/.env

# Update these lines with your actual keys from Langfuse:
LANGFUSE_PUBLIC_KEY=pk-your-actual-public-key-here
LANGFUSE_SECRET_KEY=sk-your-actual-secret-key-here
```

### Step 7: Restart LiteLLM Service

Apply the changes by restarting the LiteLLM service:

```bash
docker compose restart litellm
```

Verify LiteLLM has picked up the new configuration:

```bash
docker compose logs litellm | grep -i langfuse
```

You should see logs indicating Langfuse integration is active.

---

## Verification Steps

### Verify Langfuse is Running

1. **Check Langfuse health endpoint:**
   ```bash
   curl http://localhost:3000/api/health
   ```
   Expected response: `{"status":"ok"}`

2. **Check Langfuse logs:**
   ```bash
   docker compose logs langfuse
   ```

### Verify LiteLLM → Langfuse Integration

1. **Check LiteLLM logs for Langfuse connection:**
   ```bash
   docker compose logs litellm | grep -i "langfuse"
   ```
   Look for messages like:
   - "Langfuse client initialized"
   - "Successfully connected to Langfuse"

2. **Make a test LLM request:**
   ```bash
   curl -X POST http://localhost:4000/chat/completions \
     -H "Authorization: Bearer d03bdce07c029021bcb9a2d6834c0c33009ff8d68c5d1b7124d93b32f081c338" \
     -H "Content-Type: application/json" \
     -d '{
       "model": "minimax/MiniMax-M2.7",
       "messages": [{"role": "user", "content": "Hello"}]
     }'
   ```

3. **Check Langfuse Dashboard:**
   - Navigate to http://localhost:3000
   - Go to your project
   - Click on **"Traces"** in the left sidebar
   - You should see the trace from your test request appear within a few seconds

4. **Verify metrics are being collected:**
   - In Langfuse, navigate to **"Dashboard"** or **"Analytics"**
   - You should see:
     - Request counts
     - Token usage
     - Latency metrics
     - Cost tracking (if configured)

### Verify ClickHouse Integration

Langfuse V3 requires ClickHouse for analytics. Verify it's working:

```bash
# Check ClickHouse health
curl http://localhost:8123/ping
# Expected: "Ok."

# Check ClickHouse logs
docker compose logs clickhouse
```

---

## Troubleshooting

### Langfuse UI Not Accessible

**Problem:** Cannot access http://localhost:3000

**Solutions:**
1. Check if Langfuse container is running:
   ```bash
   docker compose ps langfuse
   ```
2. Check Langfuse logs for errors:
   ```bash
   docker compose logs langfuse
   ```
3. Verify port is not in use by another service:
   ```bash
   lsof -i :3000
   ```

### API Keys Not Working

**Problem:** LiteLLM logs show authentication errors with Langfuse

**Solutions:**
1. Verify keys are correctly copied (no extra spaces)
2. Ensure keys are from the correct project
3. Regenerate keys in Langfuse if needed:
   - Go to Project Settings → API Keys
   - Click "Regenerate Secret Key"
4. Restart LiteLLM after updating keys

### No Traces Appearing in Langfuse

**Problem:** LLM requests are not showing up in Langfuse dashboard

**Solutions:**
1. Verify `LANGFUSE_ENABLED=true` in `.env`
2. Check LiteLLM logs for Langfuse errors:
   ```bash
   docker compose logs litellm | grep -i langfuse
   ```
3. Verify network connectivity between containers:
   ```bash
   docker compose exec litellm curl http://langfuse:3000/api/health
   ```
4. Check that callbacks are enabled in `litellm_config.yaml`:
   ```yaml
   success_callback: ["prometheus", "langfuse", "log_cost"]
   ```

### ClickHouse Connection Errors

**Problem:** Langfuse logs show ClickHouse connection failures

**Solutions:**
1. Verify ClickHouse is healthy:
   ```bash
   docker compose ps clickhouse
   ```
2. Check ClickHouse logs:
   ```bash
   docker compose logs clickhouse
   ```
3. Verify ClickHouse URL in `.env` matches docker-compose configuration
4. Wait for ClickHouse to fully initialize (can take 30-60 seconds)

---

## Access URLs

| Service | URL | Description |
|---------|-----|-------------|
| Langfuse UI | http://localhost:3000 | Main Langfuse dashboard |
| LiteLLM UI | http://localhost:4000/ui | LiteLLM admin dashboard |
| LiteLLM API | http://localhost:4000 | LiteLLM API endpoint |
| ClickHouse HTTP | http://localhost:8123 | ClickHouse HTTP interface |

---

## Security Notes

1. **Default Credentials:** Change default passwords in `.env` for production deployments
2. **API Keys:** Keep your Langfuse secret key secure and never commit it to version control
3. **Network Binding:** Services are bound to `127.0.0.1` by default for local security
4. **Authentication:** Langfuse requires authentication to access the dashboard

---

## Additional Resources

- [Langfuse Documentation](https://langfuse.com/docs)
- [Langfuse Self-Hosting Guide](https://langfuse.com/self-hosting)
- [LiteLLM Observability Docs](https://docs.litellm.ai/docs/observability/langfuse_integration)
- [Heretek OpenClaw Documentation](../README.md)

---

## Quick Reference

```bash
# Start all services
docker compose up -d

# View Langfuse logs
docker compose logs -f langfuse

# View LiteLLM logs
docker compose logs -f litellm

# Restart services
docker compose restart litellm langfuse

# Stop all services
docker compose down

# Check service health
docker compose ps
```
