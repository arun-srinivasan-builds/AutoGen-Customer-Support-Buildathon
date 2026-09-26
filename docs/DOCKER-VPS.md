# Docker & VPS Deployment

## Docker Architecture

```text
Browser
   ↓
Streamlit :8501
   ↓
AutoGen Application
   ↓
External APIs
   ├── OpenAI
   └── Web Search
```

The application runs inside a Python 3.11 container.

## Security Decisions

The Docker image:

- Uses `python:3.11-slim`
- Runs the application as a non-root user
- Does not copy `.env` into the image
- Receives secrets at runtime
- Excludes local virtual environments
- Excludes generated support records
- Includes an application health check

## Build

```bash
docker build -t autogen-customer-support:1.0 .
```

## Local Run

```bash
docker run -d \
  --name autogen-customer-support \
  --env-file .env \
  -p 8502:8501 \
  autogen-customer-support:1.0
```

## Health Validation

```bash
docker inspect autogen-customer-support \
  --format="{{.State.Health.Status}}"
```

Validated result:

```text
healthy
```

## Persistence Validation

```bash
docker exec autogen-customer-support \
  ls -la /app/outputs
```

A generated support record was successfully verified inside the running container.

## Docker Issue Resolved

An initial Docker build failed because the JSON-form `CMD` was split incorrectly across Dockerfile lines.

The runtime command was corrected to a valid exec-form instruction:

```dockerfile
CMD ["streamlit", "run", "app.py", "--server.address=0.0.0.0", "--server.port=8501", "--server.headless=true", "--browser.gatherUsageStats=false"]
```

The subsequent image build and health validation succeeded.

---

## VPS Deployment

**Status:** Pending

This section will be updated after deployment with:

- VPS directory structure
- Docker deployment commands
- Runtime environment configuration
- Reverse-proxy configuration
- HTTPS endpoint
- Container health validation
- Live application evidence