# Architecture & AutoGen Concepts

## System Flow

```text
Customer Query
      ↓
Input Guardrail
      ↓
AutoGen RoundRobinGroupChat
      ↓
Group Chat Manager
      │
      ├──► Assistant Agent
      │        └── Direct response
      │
      ├──► Web Search Assistant
      │        └── web_search()
      │
      ├──► Entry Agent
      │        ├── Grounded consolidation
      │        └── save_to_file()
      │
      ▼
TextMentionTermination
      ↓
Output Guardrail
      ↓
Evaluations
      ↓
Validated UI Response
```

## AutoGen Team

The implementation uses exactly three AutoGen `AssistantAgent` instances inside a `RoundRobinGroupChat`.

| Agent | Responsibility | Tool Access |
|---|---|---|
| Assistant Agent | Direct response using model knowledge | None |
| Web Search Assistant | Current web-grounded response | `web_search()` |
| Entry Agent | Consolidates both responses and persists the record | `save_to_file()` |

Tool access follows least privilege: each agent receives only the capability required for its responsibility.

## Group Chat Manager

`RoundRobinGroupChat` maintains a shared conversation and coordinates speaker turns through AutoGen's group-chat management mechanism.

The required execution order is deterministic:

```text
Assistant Agent
      ↓
Web Search Assistant
      ↓
Entry Agent
```

The Streamlit interface makes this orchestration observable by showing Manager → Agent dispatches, Agent → Manager returns, tool activity and termination status.

## Shared Conversation

Each agent operates using the conversation accumulated before its turn.

This enables:

- Agent 2 to see the original task and Agent 1 response.
- Agent 3 to see both earlier responses.
- Agent 3 to create the final grounded consolidated response.
- The termination condition to observe the completion marker.

## Explicit Termination

The Entry Agent completes persistence and produces the workflow completion marker.

```text
Entry Agent
      ↓
save_to_file()
      ↓
AUTOGEN_WORKFLOW_COMPLETE
      ↓
TextMentionTermination
      ↓
STOP
```

A safety message-count limit is also retained as a fallback.

Without explicit termination, a round-robin conversation could continue into another agent turn.

## Live Observability

The UI exposes safe execution events such as:

```text
Manager → Assistant Agent
Assistant Agent → Manager
Manager → Web Search Assistant
Web Search Assistant → web_search()
Web Search Assistant → Manager
Manager → Entry Agent
Entry Agent → save_to_file()
Entry Agent → Manager
Manager → Termination Gate
```

The application does not expose private chain-of-thought or model reasoning.