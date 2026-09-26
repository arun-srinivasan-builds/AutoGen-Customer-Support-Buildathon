# 🤖 AutoGen Customer Support Buildathon


## 🎯 Buildathon Objective

The goal of this implementation is not only to execute three agents sequentially, but to make the **AutoGen orchestration model observable, secure, testable, and deployable**.

> **Build → Observe → Guard → Evaluate → Deploy**
> 
A production-minded **three-agent customer support workflow** built using **Microsoft AutoGen AgentChat** and **Streamlit**.

The application demonstrates sequential multi-agent orchestration, AutoGen Group Chat Manager coordination, isolated agent tools, live web grounding, grounded response consolidation, explicit termination, input/output guardrails, deterministic evaluations, persistence, and live workflow observability.

---

## 🏗️ Architecture


![Alt text](https://github.com/arun-srinivasan-builds/AutoGen-Customer-Support-Buildathon/blob/a56be6edd06e29f8f6e37e47384d461c279e2b40/docs/images/architecture.png)


---

## ✅ Buildathon Implementation

| Requirement | Implementation |
|---|---|
| Framework | Microsoft AutoGen AgentChat |
| Agents | Exactly 3 `AssistantAgent`s |
| Team | `RoundRobinGroupChat` |
| Assistant Agent | Direct model-knowledge response |
| Web Search Assistant | Current web-grounded response |
| Entry Agent | Grounded consolidation + persistence |
| Web Search Tool | Available only to Web Search Assistant |
| File Tool | Available only to Entry Agent |
| Termination | Explicit `TextMentionTermination` + safety fallback |
| UI | Streamlit |
| Persistence | `.txt` support record |
| Secrets | Environment variables |
| Application Code | Single `app.py` |
| Deployment | Docker + VPS |

---

## 🧠 Three-Agent Workflow

### 1. Assistant Agent
Answers the customer query directly using model knowledge.

**Tools:** None

### 2. Web Search Assistant
Uses live web search to produce a current, evidence-grounded response.

**Tool:** `web_search()`

### 3. Entry Agent
Reads both previous responses, produces a **grounded consolidated response**, and persists the complete support record.

**Tool:** `save_to_file()`

The Entry Agent completion marker triggers the explicit AutoGen termination condition.

---

## 🛡️ Security & Validation

The application includes layered protection around the agent workflow:

- Input guardrails
- Prompt-injection detection
- Secret/API-key extraction protection
- Agent/manager orchestration manipulation protection
- Unsafe-content screening
- Output guardrails
- Secret-leakage validation
- Pre-persistence validation
- Temporal-currentness checks
- Deterministic workflow evaluations
- Tool-isolation validation
- Explicit termination validation

Unsafe requests are blocked **before the AutoGen team, web-search tool, or persistence tool executes**.

---

## 👀 Live AutoGen Observability

The Streamlit UI visualizes the actual workflow as it executes:

![Alt text](https://github.com/arun-srinivasan-builds/AutoGen-Customer-Support-Buildathon/blob/fd4417b940e6d27bfb6156abf059176a6b66909d/docs/images/live-autogen-observability.png)

Only safe execution metadata is displayed. Private model reasoning is not exposed.

---

## 📚 Explore the Project

Detailed implementation notes are separated from the main README to keep the repository easy to review.

- [Architecture & AutoGen Concepts](docs/ARCHITECTURE.md)
- [Security Guardrails & Evaluations](docs/GUARDRAILS-EVALS.md)
- [Testing & Validation](docs/TESTING.md)
- [Docker & VPS Deployment](docs/DOCKER-VPS.md)
- [Key Learnings & Engineering Findings](docs/LEARNINGS.md)

---

## 🚀 Run Locally

### 1. Create the environment

```bash
python -m venv venv
```

Activate on Windows:

```bash
venv\Scripts\activate
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure environment variables

Create a `.env` file from `.env.example`.

```env
OPENAI_API_KEY=your_openai_api_key
SERPER_API_KEY=your_serper_api_key
```

Never commit `.env` or real API keys.

### 4. Start the application

```bash
streamlit run app.py
```

---

## 🐳 Docker

Build the image:

```bash
docker build -t autogen-customer-support:1.0 .
```

Run the container:

```bash
docker run -d \
  --name autogen-customer-support \
  --env-file .env \
  -p 8501:8501 \
  autogen-customer-support:1.0
```

Verify container health:

```bash
docker inspect autogen-customer-support --format="{{.State.Health.Status}}"
```

Local Docker validation completed successfully with:

```text
healthy
```

The Entry Agent's `.txt` persistence was also validated inside the running container.

---

## 🔐 Secrets & Repository Hygiene

The repository intentionally excludes:

```text
.env
venv/
__pycache__/
outputs/*.txt
backup files
temporary files
```

Only `outputs/.gitkeep` is committed so generated customer-support records remain runtime artifacts.

---

## 💡 Key Engineering Learnings

A few important issues were discovered and corrected during implementation:

- Generic content moderation alone does not prevent **agent-orchestration attacks**; application-specific manager/tool guardrails were required.
- Relative queries such as **"this year"** require explicit temporal grounding to prevent stale answers.
- Agent tool execution must be controlled to prevent duplicate persistence calls.
- Unsafe generated content should be validated **before persistence**, not only before display.
- Explicit termination is essential in a round-robin multi-agent workflow.
- Tool isolation provides a clear least-privilege boundary between agents.

See [Key Learnings & Engineering Findings](docs/LEARNINGS.md) for details.

---

## 📦 Technology Stack

- Python 3.11
- Microsoft AutoGen AgentChat
- OpenAI
- Serper Web Search
- Streamlit
- Docker
- Input & Output Guardrails
- Deterministic Evaluations

---

## 📌 Project Status

| Stage | Status |
|---|---|
| Three-Agent AutoGen Workflow | ✅ Complete |
| Group Chat Manager Visualization | ✅ Complete |
| Web Search Tool | ✅ Complete |
| Entry Agent Persistence | ✅ Complete |
| Grounded Consolidated Response | ✅ Complete |
| Explicit Termination | ✅ Complete |
| Input & Output Guardrails | ✅ Complete |
| Evaluations | ✅ Complete |
| Live Workflow Visualization | ✅ Complete |
| Local Docker Validation | ✅ Complete |
| VPS Deployment | ⏳ In Progress |

---

