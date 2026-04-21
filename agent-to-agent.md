# Agent-to-Agent (A2A) Protocol Implementation Guide

A2A (Agent-to-Agent) is an open, transport-agnostic protocol that defines how autonomous agents discover, negotiate with, and delegate tasks to other agents. Unlike REST APIs — which are synchronous request/response between a human client and a server — A2A enables long-running, stateful, multi-turn interactions between agents that may each be operated by different teams, written in different languages, and deployed on different infrastructure.

## Why not just use REST or gRPC?

> Standard RPC frameworks assume you know the full API surface upfront and that calls complete quickly. Agents are different: they run for seconds to hours, may need to pause and resume, produce partial results via streaming, and must describe their own capabilities dynamically so other agents can decide whether to delegate to them.

## A2A vs Traditional API Calls

| Dimension              | REST / gRPC                               | A2A Protocol                                     |
| ---------------------- | ----------------------------------------- | ------------------------------------------------ |
| Interaction model      | Synchronous request/response              | Asynchronous, stateful tasks                     |
| Discovery              | OpenAPI / proto files checked in manually | Agent Card served at `/.well-known/x-agent.json` |
| Capability negotiation | None — caller hardcodes contract          | Skills + modalities declared; caller selects     |
| Long-running work      | Timeout, polling hacks, webhooks          | Native task states + SSE streaming               |
| Multi-turn interaction | Application-level session management      | Built into task model (input-required)           |
| Auth                   | API key or OAuth per endpoint             | Declared in Agent Card; negotiated per task      |

## Core Concepts & Terminology

| Term                      | Definition                                                                                                                                 |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| **Client agent**          | The agent that originates a task and delegates it to a remote agent. Also called the orchestrator when it coordinates multiple sub-agents. |
| **Remote agent (server)** | The agent that receives and executes a task. Exposes an A2A endpoint. May itself act as a client to further sub-agents.                    |
| **Agent Card**            | A machine-readable JSON document that describes the agent's identity, capabilities, skills, and authentication requirements.               |
| **Task**                  | The fundamental unit of work. Has a unique ID, a lifecycle (submitted → working → done), and carries messages in both directions.          |
| **Message**               | A single turn in a task conversation. Has a role (user or agent) and one or more typed parts (text, file, data).                           |
| **Part**                  | The atomic content unit inside a message. Types: TextPart, FilePart, DataPart                                                              |

## Mental Model

> Think of A2A like email between agents — each task is a thread with a unique ID. Messages go back and forth. Either side can attach files or structured data. The thread lives until the task reaches a terminal state.

## Agent Card — The Identity Contract

The Agent Card is the single source of truth about what your agent does and how to talk to it. It **MUST** be served at `https://<your-agent-host>/.well-known/x-agent.json` with `Content-Type: application/json` and **no authentication required**. Client agents fetch this before initiating any task.

### Mandatory Endpoint

> If your agent does not serve a valid Agent Card at `/.well-known/x-agent.json`, it **cannot participate in A2A**. This is non-negotiable for all agents.

### Required vs Optional Fields

| Field                | Type            | Required | Description                                                                                             |
| -------------------- | --------------- | -------- | ------------------------------------------------------------------------------------------------------- |
| `name`               | string          | Required | Human-readable name. Use team/agent-name format. e.g. `data-eng/sql-agent`                              |
| `description`        | string          | Required | One-sentence plain English capability summary. Used by LLM orchestrators to decide whether to delegate. |
| `url`                | string (URL)    | Required | The HTTPS base URL of the A2A endpoint. Must be same origin as Agent Card host.                         |
| `version`            | string (semver) | Required | Agent implementation version. Format: MAJOR.MINOR.PATCH                                                 |
| `capabilities`       | object          | Required | Declares streaming and pushNotifications support booleans.                                              |
| `skills`             | array<Skill>    | Required | List of specific abilities. Each skill has id, name, description, optional inputModes and outputModes   |
| `authentication`     | object          | Optional | Declares schemes array (e.g. bearer, oauth2) and scopes required per skill.                             |
| `defaultInputModes`  | array<string>   | Optional | MIME types the agent accepts by default. e.g. `["text/plain", "application/json"]`                      |
| `defaultOutputModes` | array<string>   | Optional | MIME types the agent produces by default. Clients use this to check compatibility before sending.       |
| `provider`           | object          | Optional | Contains creator, team. Mandatory if multiple teams are building agents.                                |

### Canonical Agent Card Example

```json
{
  "name": "data-eng/sql-agent",
  "description": "Translates natural language to SQL, executes read-only queries, and returns structured results.",
  "url": "https://sql-agent.internal.corp.com",
  "version": "2.1.0",
  "capabilities": {
    "streaming": true,
    "pushNotifications": false
  },
  "defaultInputModes": ["text/plain"],
  "defaultOutputModes": ["application/json", "text/plain"],
  "authentication": {
    "schemes": ["bearer"],
    "credentials": null
  },
  "skills": [
    {
      "id": "nl-to-sql",
      "name": "Natural language to SQL",
      "description": "Converts a plain English question into a SQL query against a specified schema.",
      "inputModes": ["text/plain"],
      "outputModes": ["application/json"]
    }
  ],
  "provider": {
    "creator": "Kiara",
    "team": "AI Assistant"
  }
}
```

## Task Lifecycle — States & Transitions

Every unit of work in A2A is modelled as a Task. A task has a unique ID, a history of messages, and a status that transitions through a defined state machine. Understanding this lifecycle is the most critical part of A2A compliance.

### State flow:

`submitted → working → input-required → working → completed`
(or terminal states: `failed`, `canceled`)

### All Task States

| State | Terminal? | Meaning | Agent must… |
|-------|-----------|---------|-------------|
| submitted | No | Task received; agent has not yet started processing. | Persist the task ID immediately. Respond within 500ms. |
| working | No | Agent is actively processing. May stream partial artifacts. | Emit SSE events for streaming-capable tasks. Must not stay in this state >10 min without a heartbeat event. |
| input-required | No | Agent needs more information before proceeding. Blocks until client sends more. | Include a clear message describing exactly what is needed. Set inputSchema if the expected input is structured. |
| completed | Yes | Task finished successfully. Final artifacts available. | Include all output in the final task response. No further mutations allowed. |
| failed | Yes | Task failed due to agent error or unrecoverable upstream failure. | Include an error object with code and message. Log the cause internally. |
| canceled | Yes | Client requested cancellation via tasks/cancel. | Stop processing immediately. Release all held resources. Respond within 2s. |

## Illegal Transitions — Enforce These

> Once a task reaches a terminal state (completed, failed, canceled) it MUST NOT transition to any other state. Any attempt to mutate a terminal task must return HTTP 409 Conflict.

### The Heartbeat Rule

Any task in `working` state for longer than 60 seconds MUST emit a heartbeat event via the SSE stream (or push notification if streaming is not in use). A task silent for more than 10 minutes is considered hung and the platform's watchdog will mark it failed.
JSON

```json
{
  "id": "task_01HXYZ",
  "status": {
    "state": "working",
    "message": {
      "role": "agent",
      "parts": [
        {
          "type": "text",
          "text": "Still processing... fetched 4/10 pages."
        }
      ]
    },
    "final": false
  }
}
```

## Message Envelope & Part Types

All agent communication travels inside the Message envelope. Messages carry content as Parts — typed payloads that represent text, files, or structured data. A message may contain multiple parts of mixed types.

### Sending a Task — tasks/send

```json
{
  "jsonrpc": "2.0",
  "id": "rpc-001",
  "method": "tasks/send",
  "params": {
    "id": "task_01HXYZ", // client-generated UUID
    "sessionId": "ses_abc", // groups related tasks; optional
    "message": {
      "role": "user",
      "parts": [
        {
          "type": "text",
          "text": "How many orders shipped last week?"
        }
      ]
    },
    "metadata": {
      // always include these
      "callerAgent": "orchestrator/planner-agent",
      "traceId": "abc-123-xyz",
      "spanId": "span-456"
    }
  }
}
```

### Part Types Reference

| Part type | Required fields                                                      | When to use                                                             | Max size                       |
| --------- | -------------------------------------------------------------------- | ----------------------------------------------------------------------- | ------------------------------ |
| TextPart  | type: "text", text                                                   | Natural language instructions, status messages, markdown output         | 1 MB                           |
| FilePart  | type: "file", file.mimeType + either file.bytes (base64) or file.uri | Binary files, PDFs, images. Prefer uri for files > 1 MB.                | 5 MB inline; unlimited via URI |
| DataPart  | type: "data", data (any JSON value)                                  | Structured inputs/outputs, schema-validated payloads, tool call results | 2 MB                           |

### Don't Embed Large Files Inline

> FileParts with `bytes` >1 MB will be rejected by the platform gateway. Upload to S3/GCS first and pass a pre-signed uri instead. The remote agent must be able to fetch that `URI` without credentials unless you negotiate auth separately.

## Transport Layer — HTTP + JSON-RPC

A2A is transport-agnostic in theory. In practice, this mandates HTTPS + JSON-RPC 2.0 for all agent communication.

### Standard Endpoint Methods

| Method                     | HTTP       | Purpose                                                                     | Idempotent?         |
| -------------------------- | ---------- | --------------------------------------------------------------------------- | ------------------- |
| tasks/send                 | POST       | Create a new task or add a message to an existing task (by reusing task ID) | No                  |
| tasks/stream               | POST → SSE | Create task AND open an SSE stream for updates in the same request          | No                  |
| tasks/get                  | POST       | Retrieve current task state + full message history                          | Yes                 |
| tasks/cancel               | POST       | Request cancellation of a running task                                      | Yes — safe to retry |
| tasks/pushNotification/set | POST       | Register a webhook URL to receive async task state updates                  | Yes                 |
| tasks/pushNotification/get | POST       | Retrieve the registered webhook config for a task                           | Yes                 |

```json
{
  "jsonrpc": "2.0",
  "id": "rpc-001",
  "error": {
    "code": -32001, // A2A error codes: -32001 to -32099
    "message": "Task not found",
    "data": {
      "taskId": "task_01HXYZ",
      "traceId": "abc-123-xyz"
    }
  }
}
```

## Streaming & Push Notifications

Long-running tasks must not keep the client blocked. A2A provides two mechanisms: SSE streaming (server-sent events on a persistent HTTP connection) and push notifications (webhooks to a client-registered URL). Declare which you support in your Agent Card.

### SSE Streaming via tasks/stream

Use this when the client can hold an open connection. The server MUST set `Content-Type: text/event-stream` and stream `TaskStatusUpdateEvent` or `TaskArtifactUpdateEvent` objects as SSE data frames.

```json
data: {"id":"task_01HXYZ","status":{"state":"working","final":false}}
data: {"id":"task_01HXYZ","artifact":{"index":0,"append":true,"parts":[{"type":"text","text":"SELECT COUNT(*)"}]}}
data: {"id":"task_01HXYZ","artifact":{"index":0,"append":true,"lastChunk":true,"parts":[{"type":"text","text":" FROM orders WHERE..."}]}}
data: {"id":"task_01HXYZ","status":{"state":"completed","final":true}}
```

### Push Notifications for Async Clients

Use when the client cannot hold an open connection (e.g. serverless functions, mobile). The client registers a webhook URL via `tasks/pushNotification/set`. The agent POSTs `TaskStatusUpdateEvent` JSON to that URL on every state change.

### Webhook Validation is Mandatory

> Before the agent starts POSTing to a webhook, it MUST send a validation challenge (a short-lived HMAC-signed token) and verify the endpoint echoes it back. This prevents SSRF attacks via malicious webhook registration. Use the platform's agent-sdk webhook validator — do not implement this yourself.

### Capability Declaration in Agent Card

```json
{
  "capabilities": {
    "streaming": true, // supports tasks/sendSubscribe
    "pushNotifications": true // supports webhook registration
  }
}
```

## Error Handling & Retry Standards

Errors in A2A have two layers: transport errors (HTTP status codes) and protocol errors (JSON-RPC error objects). Both must be handled. Tasks that fail must propagate a structured error into the task's terminal state.

### A2A Error Code Registry

| Code   | Name                          | Retryable? | Action                                                                    |
| ------ | ----------------------------- | ---------- | ------------------------------------------------------------------------- |
| -32700 | Parse error                   | No         | Fix the request body. Log with full request for debugging.                |
| -32600 | Invalid request               | No         | Schema validation failed. Check required fields.                          |
| -32601 | Method not found              | No         | Check Agent Card capabilities — you may be calling an unsupported method. |
| -32001 | Task not found                | No         | Task ID does not exist or has been garbage collected.                     |
| -32002 | Task not cancelable           | No         | Task is already in terminal state.                                        |
| -32003 | Push notification unsupported | No         | Agent Card declares pushNotifications: false                              |
| -32004 | Unsupported operation         | No         | Agent does not support this skill/modality combination.                   |
| -32005 | Incompatible content type     | No         | Part type not in agent's defaultInputModes                                |
| -32008 | Task timeout                  | Yes        | Retry with exponential backoff. Alert if persists >3 retries.             |
| -32009 | Push notification failed      | Yes        | Retry webhook delivery. After 5 failures, mark notification as dead.      |

### Idempotency Keys Prevent Duplicate Task Creation on Retry

> Always include `Idempotency-Key` header on `tasks/send` calls. If your retry arrives at the server after the original succeeded (network timeout), the server returns the original task instead of creating a duplicate.

## Agent Discovery Protocol Standard

- Direct Access Prohibited — Agents must not GET peer /.well-known/\* resources directly; this avoids latency and security exposure.
- Registry-First Lookup — Agents must query A2A-Registry-Service via /find-capability.
- Manifest Refresh Policy — Updates to /.well-known/x-agent.json must trigger POST /registry/refresh so the global index updates immediately.
