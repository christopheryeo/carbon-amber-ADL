# Agent Message Format Specification

## Description

This document defines the mandatory JSON message format used for communication between all agents within the Sentient Agentic AI Platform. The Objective Agent establishes this format when processing user requests, and all subsequent agents must adhere to this structure when passing information downstream.

---

## Message Format Specification

All agent communications must use the following JSON structure:

```json
{
  "message_id": "<unique identifier>",
  "timestamp": {
    "executed_at": "<ISO 8601 datetime>",
    "timezone": "<timezone identifier>"
  },
  "agent": {
    "name": "<name of the executing agent>",
    "type": "<agent type/core layer>"
  },
  "input": {
    "source": "<source of input - 'user' or agent name>",
    "content": "<the input received by this agent>"
  },
  "output": {
    "content": "<the output/recommendation produced by this agent>",
    "content_type": "<type of output - 'objectives', 'goals', 'plan', 'analysis', etc.>"
  },
  "next_agent": {
    "name": "<name of the next agent to execute>",
    "reason": "<brief explanation why this agent is next>"
  },
  "status": {
    "code": "<status code - 'success', 'partial', 'failed', 'pending'>",
    "message": "<human-readable status description>"
  },
  "error": {
    "has_error": <boolean>,
    "error_code": "<error code if applicable>",
    "error_message": "<error description if applicable>",
    "retry_count": <number of retry attempts>,
    "recoverable": <boolean - whether error is recoverable>
  },
  "metadata": {
    "session_id": "<session identifier>",
    "request_id": "<original user request identifier>",
    "sequence_number": <position in agent chain>,
    "parent_message_id": "<message_id of previous agent's message>"
  },
  "audit": {
    "compliance_notes": "<governance compliance observations and validations>",
    "governance_files_consulted": ["<list of governance files referenced>"]
  }
}
```

---

## Field Definitions

### Core Fields (Required)

| Field | Type | Description |
|-------|------|-------------|
| `message_id` | string | Unique identifier for this message (UUID format recommended) |
| `timestamp.executed_at` | string | ISO 8601 formatted datetime when agent executed |
| `timestamp.timezone` | string | Timezone identifier (e.g., "Asia/Singapore", "UTC") |
| `agent.name` | string | Name of the agent producing this message |
| `agent.type` | string | Agent's core layer (governance, operational, executional) |
| `input.source` | string | Where the input came from ("user" or previous agent name) |
| `input.content` | string/object | The actual input received |
| `output.content` | string/array/object | The output produced by the agent |
| `output.content_type` | string | Classification of output type |
| `next_agent.name` | string | Name of the next agent in the chain |
| `next_agent.reason` | string | Brief explanation for routing decision |

### Status Fields (Required)

| Field | Type | Description |
|-------|------|-------------|
| `status.code` | string | One of: "success", "partial", "failed", "pending" |
| `status.message` | string | Human-readable status description |

### Error Fields (Required)

| Field | Type | Description |
|-------|------|-------------|
| `error.has_error` | boolean | True if an error occurred |
| `error.error_code` | string | Error code (null if no error) |
| `error.error_message` | string | Error description (null if no error) |
| `error.retry_count` | integer | Number of retry attempts made |
| `error.recoverable` | boolean | Whether the error can be recovered from |

### Metadata Fields (Required)

| Field | Type | Description |
|-------|------|-------------|
| `metadata.session_id` | string | Identifier for the current session |
| `metadata.request_id` | string | Identifier linking back to original user request |
| `metadata.sequence_number` | integer | Position in the agent execution chain (starts at 1) |
| `metadata.parent_message_id` | string | message_id of the previous agent's message (null for first agent) |

### Audit Fields (Required)

| Field | Type | Description |
|-------|------|-------------|
| `audit.compliance_notes` | string | Governance compliance observations, validations performed, and any flags |
| `audit.governance_files_consulted` | array | List of governance files referenced during processing (e.g., `["audit.md", "message_format.md"]`) |

---

## Status Codes

| Code | Description |
|------|-------------|
| `success` | Agent completed processing successfully |
| `partial` | Agent completed with partial results (some objectives/goals processed) |
| `failed` | Agent failed to process the request |
| `pending` | Agent processing is in progress or waiting for external input |

---

## Error Codes

| Code | Description |
|------|-------------|
| `OUT_OF_SCOPE` | Request falls outside platform capabilities |
| `INVALID_INPUT` | Input format or content is invalid |
| `PROCESSING_ERROR` | Error occurred during agent processing |
| `TIMEOUT` | Agent processing exceeded time limit |
| `DEPENDENCY_ERROR` | Required upstream data is missing or invalid |
| `VALIDATION_ERROR` | Output failed validation checks |

---

## Example Messages

### Example 1: Objective Agent Output (Success)

```json
{
  "message_id": "msg-obj-20260127-143052-001",
  "timestamp": {
    "executed_at": "2026-01-27T14:30:52+08:00",
    "timezone": "Asia/Singapore"
  },
  "agent": {
    "name": "objective_agent",
    "type": "governance"
  },
  "input": {
    "source": "user",
    "content": "Download this YouTube video and analyze the speaker's sentiment"
  },
  "output": {
    "content": [
      "Obtain video content from YouTube",
      "Analyze speaker sentiment"
    ],
    "content_type": "objectives"
  },
  "next_agent": {
    "name": "goal_agent",
    "reason": "Objectives require decomposition into actionable goals"
  },
  "status": {
    "code": "success",
    "message": "Successfully generated 2 strategic objectives from user request"
  },
  "error": {
    "has_error": false,
    "error_code": null,
    "error_message": null,
    "retry_count": 0,
    "recoverable": false
  },
  "metadata": {
    "session_id": "session-20260127-1430",
    "request_id": "req-20260127-143050",
    "sequence_number": 1,
    "parent_message_id": null
  },
  "audit": {
    "compliance_notes": "Request within video analysis scope per context/application.md; objectives align with Audio Analysis and Speaker Analysis capabilities; validated against supported video sources (YouTube)",
    "governance_files_consulted": ["context/application.md", "message_format.md", "audit.md"]
  }
}
```

### Example 2: Goal Agent Output (Success)

```json
{
  "message_id": "msg-goal-20260127-143055-001",
  "timestamp": {
    "executed_at": "2026-01-27T14:30:55+08:00",
    "timezone": "Asia/Singapore"
  },
  "agent": {
    "name": "goal_agent",
    "type": "governance"
  },
  "input": {
    "source": "objective_agent",
    "content": [
      "Obtain video content from YouTube",
      "Analyze speaker sentiment"
    ]
  },
  "output": {
    "content": {
      "objective_1": {
        "objective": "Obtain video content from YouTube",
        "goals": [
          "Extract video URL from user input",
          "Validate URL is from supported source (YouTube)",
          "Download video file to storage",
          "Verify video file integrity"
        ]
      },
      "objective_2": {
        "objective": "Analyze speaker sentiment",
        "goals": [
          "Extract audio track from video",
          "Transcribe audio using Whisper-X",
          "Identify speaker segments",
          "Run sentiment analysis model on transcript",
          "Correlate sentiment with speaker timestamps"
        ]
      }
    },
    "content_type": "goals"
  },
  "next_agent": {
    "name": "planning_agent",
    "reason": "Goals require execution planning and workflow orchestration"
  },
  "status": {
    "code": "success",
    "message": "Successfully decomposed 2 objectives into 9 actionable goals"
  },
  "error": {
    "has_error": false,
    "error_code": null,
    "error_message": null,
    "retry_count": 0,
    "recoverable": false
  },
  "metadata": {
    "session_id": "session-20260127-1430",
    "request_id": "req-20260127-143050",
    "sequence_number": 2,
    "parent_message_id": "msg-obj-20260127-143052-001"
  },
  "audit": {
    "compliance_notes": "Objectives successfully decomposed into actionable goals; all goals map to platform capabilities defined in context/application.md; goal sequence supports efficient execution",
    "governance_files_consulted": ["context/application.md", "message_format.md", "audit.md"]
  }
}
```

### Example 3: Objective Agent Output (Error - Out of Scope)

```json
{
  "message_id": "msg-obj-20260127-150012-001",
  "timestamp": {
    "executed_at": "2026-01-27T15:00:12+08:00",
    "timezone": "Asia/Singapore"
  },
  "agent": {
    "name": "objective_agent",
    "type": "governance"
  },
  "input": {
    "source": "user",
    "content": "Edit this video and add background music"
  },
  "output": {
    "content": null,
    "content_type": "error"
  },
  "next_agent": {
    "name": null,
    "reason": "Request is out of scope - no further processing"
  },
  "status": {
    "code": "failed",
    "message": "Request falls outside platform capabilities"
  },
  "error": {
    "has_error": true,
    "error_code": "OUT_OF_SCOPE",
    "error_message": "Video editing and audio addition are not supported. This platform only supports video analysis capabilities including transcription, sentiment analysis, object detection, and content understanding.",
    "retry_count": 0,
    "recoverable": false
  },
  "metadata": {
    "session_id": "session-20260127-1500",
    "request_id": "req-20260127-150010",
    "sequence_number": 1,
    "parent_message_id": null
  },
  "audit": {
    "compliance_notes": "Request rejected - video editing and audio addition fall outside platform scope per context/application.md Constraints and Boundaries; platform supports analysis only, not content modification",
    "governance_files_consulted": ["context/application.md", "message_format.md", "audit.md"]
  }
}
```

---

## Agent Chain Flow

The typical message flow through the agent chain:

```
User Request
    ↓
[1] Objective Agent (governance) → outputs: objectives
    ↓
[2] Goal Agent (governance) → outputs: goals and sub-goals
    ↓
[3] Planning Agent (operational) → outputs: execution plan
    ↓
[4] Executional Agents → outputs: analysis results
    ↓
[5] Final Output → formatted response to user
```

Each agent receives the message from the previous agent, processes it, updates the relevant fields, and passes it to the next agent.

---

## Compliance Requirements

1. **All agents MUST use this message format** for inter-agent communication
2. **All fields are required** — use `null` for fields that don't apply
3. **Messages MUST include audit data** — the `audit` field is mandatory for every message
4. **Message integrity must be maintained** — do not alter previous agent's data
5. **Sequence numbers must be consecutive** — no gaps in the chain

---

## Orchestration Layer: Audit Log Extraction

The orchestration layer is responsible for extracting audit data from the JSON message and writing it to the daily log file. This ensures every agent response is logged without requiring the LLM to perform file operations.

### Extraction and Logging Process

```
┌─────────────────────────────────────────────────────────────────┐
│  LLM outputs JSON message (includes audit field)                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  ORCHESTRATION LAYER                                            │
│                                                                 │
│  1. Receive JSON message from LLM                               │
│  2. Extract audit-relevant fields:                              │
│     - timestamp.executed_at                                     │
│     - agent.name                                                │
│     - input.source, input.content                               │
│     - output.content_type                                       │
│     - status.code, status.message                               │
│     - error (if has_error is true)                              │
│     - audit.compliance_notes                                    │
│     - audit.governance_files_consulted                          │
│  3. Format and append to system/logs/YYYYMMDD.log               │
│  4. Pass the SAME JSON message to next agent (next_agent.name)  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Next Agent receives the JSON message as input                  │
└─────────────────────────────────────────────────────────────────┘
```

### Log Entry Format (Written by Orchestration Layer)

The orchestration layer writes entries to `system/logs/YYYYMMDD.log` in this format:

```
[TIMESTAMP: HH:MM:SS]
[AGENT: <agent.name>]
[INPUT SOURCE: <input.source>]
[INPUT]: <input.content>
[OUTPUT TYPE: <output.content_type>]
[STATUS: <status.code>] <status.message>
[COMPLIANCE NOTES]: <audit.compliance_notes>
[GOVERNANCE FILES]: <audit.governance_files_consulted>
[MESSAGE_ID: <message_id>]
[SESSION_ID: <metadata.session_id>]
---
```

See `audit.md` for complete audit logging governance requirements.

---

## Version
v1.0.0

## Last Updated
February 9, 2026
