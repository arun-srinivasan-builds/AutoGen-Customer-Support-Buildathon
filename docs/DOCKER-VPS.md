# Docker & VPS Deployment

## Deployment Status

**Status: ✅ Live and validated**

**Public endpoint:** https://autogen-support.srv1965124.hstgr.cloud

The application was validated end to end on the VPS with a healthy Docker container, Traefik reverse proxy routing, HTTPS, runtime `.txt` persistence, explicit AutoGen termination, guardrails, and deterministic workflow evaluations.

---

## Deployment Architecture

```text
Browser
   ↓ HTTPS :443
Traefik Reverse Proxy
   ↓ n8n_default network
autogen-customer-support :8501
   ↓
Streamlit + AutoGen AgentChat
   ├── OpenAI
   ├── Serper Web Search
   └── /app/outputs/*.txt
```

The Streamlit container is not exposed through a public host port on the VPS. Traefik reaches it through the shared Docker network.

---

## Docker Security Decisions

The image:

- uses `python:3.11-slim`
- runs as a non-root `appuser`
- does not copy `.env` into the image
- receives secrets only at runtime
- excludes the local virtual environment
- excludes generated `.txt` support records from Git
- includes an HTTP health check
- exposes only container port `8501`
- keeps the application code in the required single `app.py`

The runtime user validated on the VPS was:

```text
uid=999(appuser) gid=999(appgroup)
```

---

## Local Docker Validation

Build:

```bash
docker build -t autogen-customer-support:1.0 .
```

Run:

```bash
docker run -d \
  --name autogen-customer-support \
  --env-file .env \
  -p 8502:8501 \
  autogen-customer-support:1.0
```

Health check:

```bash
docker inspect autogen-customer-support \
  --format='{{.State.Health.Status}}'
```

Validated result:

```text
healthy
```

Persistence was also validated inside the local running container.

---

## Dockerfile Issue Resolved

An early Docker build failed because the JSON-form `CMD` was split incorrectly across Dockerfile lines.

The runtime command was corrected to a valid exec-form instruction:

```dockerfile
CMD ["streamlit", "run", "app.py", "--server.address=0.0.0.0", "--server.port=8501", "--server.headless=true", "--browser.gatherUsageStats=false"]
```

The subsequent build and health validation succeeded.

---

## VPS Location

The repository was deployed under:

```text
/docker/apps/AutoGen-Customer-Support-Buildathon
```

Repository:

```text
https://github.com/arun-srinivasan-builds/AutoGen-Customer-Support-Buildathon.git
```

Clone:

```bash
cd /docker/apps
git clone https://github.com/arun-srinivasan-builds/AutoGen-Customer-Support-Buildathon.git
cd AutoGen-Customer-Support-Buildathon
```

---

## Runtime Environment

Create the VPS `.env` locally on the server. Do not commit it.

```env
OPENAI_API_KEY=your_runtime_openai_key
SERPER_API_KEY=your_runtime_serper_key
```

Restrict access:

```bash
chmod 600 .env
```

Verify Git ignores it:

```bash
git check-ignore .env
```

Expected:

```text
.env
```

### Security lesson from deployment

`docker compose config` can render values sourced from `env_file` in terminal output.

Do not share or capture that output when it contains runtime secrets. If a credential is exposed through logs, screenshots, copied terminal output, or another channel, rotate it immediately.

---

## Production Compose Pattern

The VPS deployment uses a private application network plus the existing Traefik network:

```yaml
services:
  autogen-customer-support:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: autogen-customer-support
    restart: unless-stopped
    env_file:
      - .env
    volumes:
      - ./outputs:/app/outputs
    networks:
      - autogen-network
      - traefik-network
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=n8n_default"
      - "traefik.http.routers.autogen-support.rule=Host(`autogen-support.srv1965124.hstgr.cloud`)"
      - "traefik.http.routers.autogen-support.entrypoints=web,websecure"
      - "traefik.http.routers.autogen-support.tls=true"
      - "traefik.http.routers.autogen-support.tls.certresolver=mytlschallenge"
      - "traefik.http.services.autogen-support.loadbalancer.server.port=8501"

networks:
  autogen-network:
    name: autogen-customer-support-network
  traefik-network:
    external: true
    name: n8n_default
```

The `.env` file remains VPS-local and is not included in the image.

---

## Build and Start on VPS

```bash
docker compose build
docker compose up -d
```

The VPS build completed successfully and created:

```text
autogen-customer-support-network
autogen-customer-support
```

Container health was confirmed:

```bash
docker ps --filter name=autogen-customer-support
```

Validated state:

```text
Up ... (healthy)
```

---

## Docker Network Validation

The application container is connected to both the application network and Traefik's network:

```bash
docker inspect autogen-customer-support \
  --format '{{range $k,$v := .NetworkSettings.Networks}}{{$k}}{{"\n"}}{{end}}'
```

Validated networks:

```text
autogen-customer-support-network
n8n_default
```

Traefik-to-Streamlit connectivity was validated from the proxy container:

```bash
docker exec n8n-traefik-1 \
  wget -qO- http://autogen-customer-support:8501/_stcore/health
```

Validated response:

```text
ok
```

---

## HTTPS Validation

Public endpoint:

```text
https://autogen-support.srv1965124.hstgr.cloud
```

Validation:

```bash
curl -I https://autogen-support.srv1965124.hstgr.cloud
```

Validated response included:

```text
HTTP/2 200
content-type: text/html; charset=utf-8
server: uvicorn
```

---

## VPS Persistence Permission Issue

### Symptom

The application container was healthy and the three AutoGen agents executed, but the Entry Agent failed during `save_to_file()` with a permission error for `/app/outputs`.

This caused three downstream validation failures:

```text
Entry save tool called once   → FAIL
Text file persisted           → FAIL
Explicit termination reached  → FAIL
```

Because the save tool could not finish, it did not return `AUTOGEN_WORKFLOW_COMPLETE`; therefore `TextMentionTermination` did not fire and the max-message safety fallback stopped the team.

### Root Cause

The Docker image correctly changed `/app/outputs` ownership to the non-root application user.

However, the production compose configuration bind-mounted the host folder:

```text
./outputs:/app/outputs
```

A bind mount replaces the image directory metadata with the host directory metadata.

The host folder was:

```text
root:root
drwxr-xr-x
```

while the container ran as:

```text
uid=999(appuser) gid=999(appgroup)
```

The non-root container therefore had no write permission.

### Fix

Confirm the runtime identity:

```bash
docker exec autogen-customer-support id
```

Align host directory ownership and permissions:

```bash
APP_UID=$(docker exec autogen-customer-support id -u)
APP_GID=$(docker exec autogen-customer-support id -g)

chown -R "$APP_UID:$APP_GID" outputs
chmod 775 outputs
```

Validate write access from the running container:

```bash
docker exec autogen-customer-support \
  sh -c 'touch /app/outputs/.write_test && rm /app/outputs/.write_test && echo "outputs directory is writable"'
```

Expected:

```text
outputs directory is writable
```

No image rebuild was required because the fix changed permissions on the bind-mounted host directory.

---

## Final Live Workflow Validation

After correcting persistence permissions, the public VPS application passed the complete workflow:

```text
Input Guardrail
      ↓
Group Chat Manager
      ↓
Assistant Agent
      ↓
Web Search Assistant
      └── web_search() once
      ↓
Entry Agent
      └── save_to_file() once
      ↓
.txt persisted
      ↓
AUTOGEN_WORKFLOW_COMPLETE
      ↓
TextMentionTermination
      ↓
Output Guardrail
      ↓
11/11 deterministic evaluations
```

Final live validation:

| Check | Result |
|---|---|
| Container health | ✅ Healthy |
| Traefik internal route | ✅ PASS |
| Public HTTPS | ✅ HTTP/2 200 |
| Required agent sequence | ✅ PASS |
| Web search exact use | ✅ PASS |
| Entry save exact use | ✅ PASS |
| Runtime persistence | ✅ PASS |
| Temporal currentness | ✅ PASS |
| Explicit `TextMentionTermination` | ✅ PASS |
| Output Guardrail | ✅ PASS |
| Evaluations | ✅ 11/11 |
| Compliance | ✅ 100% |
| Malicious orchestration tests | ✅ Blocked |

---

## Updating the Deployment

After pushing an application update to `main`:

```bash
cd /docker/apps/AutoGen-Customer-Support-Buildathon

git pull origin main
docker compose build
docker compose up -d
```

Validate:

```bash
docker ps --filter name=autogen-customer-support

docker inspect autogen-customer-support \
  --format '{{.State.Health.Status}}'

curl -I https://autogen-support.srv1965124.hstgr.cloud
```

If the `outputs` host folder is recreated, re-check its ownership before running persistence tests.

---

## Production Takeaways

- a healthy container does not prove writable bind-mounted persistence
- image-level `chown` does not override host bind-mount ownership
- application health, network connectivity, persistence, termination, and guardrails should be validated independently
- maximum-message termination should remain a fallback rather than being treated as successful explicit completion
- runtime secrets should never be committed or copied into images
- reverse-proxy routing allows the Streamlit port to remain private on the VPS
