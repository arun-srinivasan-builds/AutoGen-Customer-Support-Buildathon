# Testing & Validation

## Functional Workflow

A successful normal request validates the complete path:

```text
Input Guardrail
→ Manager
→ Assistant
→ Web Search
→ Entry
→ Persistence
→ Termination
→ Output Guardrail
→ Evaluations
```

Expected evidence:

- Both source responses generated
- Grounded Entry Agent response generated
- Web-search tool executed
- File persistence completed
- Explicit termination detected
- Evaluation checks passed

## Security Tests

Representative tests included:

| Scenario | Expected Result |
|---|---|
| Normal support query | Full workflow executes |
| Prompt injection | Blocked before AutoGen |
| System prompt extraction | Blocked |
| `.env` / API-key extraction | Blocked |
| Manager manipulation | Blocked before dispatch |
| Direct Entry Agent invocation | Blocked |
| Tool manipulation | Blocked |
| Obfuscated unsafe input | Blocked |
| Unsafe content | Blocked |

A blocked request must not trigger:

```text
AutoGen Agent execution
Web Search
File Persistence
```

## Temporal Grounding Test

Relative requests such as:

```text
When is Mahalaya Amavasya this year?
```

exposed the risk of stale model/search information.

The application now supplies current-date context and validates temporal currentness before accepting the grounded result.

## Docker Validation

The application was successfully built and executed using Docker.

Validated:

```text
Image:     autogen-customer-support:1.0
Runtime:   Python 3.11
App Port:  8501
Health:    healthy
User:      non-root appuser
```

Persistence was verified inside the running container:

```text
/app/outputs/support_run_<timestamp>.txt
```

## Repository Validation

Before committing, repository exclusions were explicitly verified:

```text
.env              → ignored
venv/             → ignored
outputs/*.txt     → ignored
outputs/.gitkeep  → retained
```

This prevents secrets, runtime records and local development artifacts from entering the repository.