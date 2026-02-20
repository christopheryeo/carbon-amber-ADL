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
  "resources": {
    "source_refs": [
      {
        "ref_id": "<src_N identifier>",
        "url": "<original URL or file path>",
        "platform": "<source platform>",
        "media_type": "<media type>",
        "provided_by": "<user or agent name>",
        "extracted_at_sequence": <sequence number of extracting agent>
      }
    ],
    "storage_refs": [
      {
        "ref_id": "<store_N identifier>",
        "source_ref_id": "<src_N this was acquired from>",
        "storage_uri": "<storage URI or null if pending>",
        "storage_backend": "<storage system e.g. wasabi>",
        "media_type": "<media type>",
        "created_at_sequence": <sequence number or null>,
        "status": "<pending|stored|failed>"
      }
    ],
    "derived_refs": [
      {
        "ref_id": "<derived_N identifier>",
        "parent_ref_id": "<ref_id this was derived from>",
        "storage_uri": "<storage URI or null if pending>",
        "asset_type": "<asset type e.g. audio_track, frame_set>",
        "capability_id": "<CAP-ID that produces this asset>",
        "created_at_sequence": <sequence number or null>,
        "status": "<pending|created|failed>"
      }
    ]
  },
  "audit": {
    "compliance_notes": "<governance compliance observations and validations>",
    "governance_files_consulted": ["<list of governance files referenced>"],
    "reasoning": "<chain of thought explaining why decisions were made>"
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

### Resources Fields (Required)

| Field | Type | Description |
|-------|------|-------------|
| `resources.source_refs` | array | URLs and file paths from user input. Each entry: `ref_id` (src_N), `url`, `platform`, `media_type`, `provided_by`, `extracted_at_sequence` |
| `resources.storage_refs` | array | Platform storage locations for acquired content. Each entry: `ref_id` (store_N), `source_ref_id`, `storage_uri` (null if pending), `storage_backend`, `media_type`, `created_at_sequence`, `status` |
| `resources.derived_refs` | array | Intermediate assets derived from stored content. Each entry: `ref_id` (derived_N), `parent_ref_id`, `storage_uri` (null if pending), `asset_type`, `capability_id`, `created_at_sequence`, `status` |

**Ref ID Conventions:**
- Source refs: `src_1`, `src_2`, etc. — assigned by the Objective Agent (sequence 1)
- Storage refs: `store_1`, `store_2`, etc. — assigned by the Objective Agent (sequence 1), URI populated by executional agents
- Derived refs: `derived_1`, `derived_2`, etc. — registered by the Planning Agent (sequence 3), created by executional agents

**First-Mention Provenance Rule:** When objective or goal text references a storage ref (e.g., `store_1`), the FIRST mention within each scope must include parenthetical provenance: `store_1 (acquired from src_1)`. Scope varies by agent: for the Objective Agent, provenance appears on the first analysis objective referencing a store_N; for the Goal Agent, provenance appears on the first goal within each objective's goal list. Subsequent mentions within the same scope use the bare ref ID `store_1`.

### Audit Fields (Required)

| Field | Type | Description |
|-------|------|-------------|
| `audit.compliance_notes` | string | Governance compliance observations, validations performed, and any flags |
| `audit.governance_files_consulted` | array | List of governance files referenced during processing, using repository-root-relative paths (e.g., `["context/governance/audit.md", "context/governance/message_format.md", "agent/governance/objective.md"]`) |
| `audit.reasoning` | string | Chain of Thought explaining why decisions were made — the rationale behind routing, scope validation, output choices, and any trade-offs considered |

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
    "content": "Download this YouTube video and analyze the speaker's sentiment in https://www.youtube.com/watch?v=abc123"
  },
  "output": {
    "content": [
      "Obtain video content from src_1 and store as store_1",
      "From store_1 (acquired from src_1), analyze speaker sentiment through multi-modal analysis"
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
  "resources": {
    "source_refs": [
      {
        "ref_id": "src_1",
        "url": "https://www.youtube.com/watch?v=abc123",
        "platform": "youtube",
        "media_type": "video",
        "provided_by": "user",
        "extracted_at_sequence": 1
      }
    ],
    "storage_refs": [
      {
        "ref_id": "store_1",
        "source_ref_id": "src_1",
        "storage_uri": null,
        "storage_backend": "wasabi",
        "media_type": "video",
        "created_at_sequence": null,
        "status": "pending"
      }
    ],
    "derived_refs": []
  },
  "audit": {
    "compliance_notes": "Request within video analysis scope per context/application.md; objectives align with Audio Analysis and Speaker Analysis capabilities; validated against supported video sources (YouTube). Resource extraction: identified 1 URL in user input, assigned src_1 → store_1. Acquisition-First Pattern applied with ref IDs.",
    "governance_files_consulted": ["context/application.md", "context/governance/message_format.md", "context/governance/audit.md", "agent/governance/objective.md"],
    "reasoning": "Scanned user input for source URLs. Found 1 URL: https://www.youtube.com/watch?v=abc123. Assigned ref IDs: src_1 (source) → store_1 (storage target). Platform detection: youtube.com domain maps to YouTube (supported per application.md Section 6.1). User request contains two distinct actions (download and analyze) requiring separate objectives. Applied Acquisition-First Pattern: Objective 1 acquires from src_1 to store_1; Objective 2 references store_1 (acquired from src_1) for analysis."
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
      "Obtain video content from src_1 and store as store_1",
      "From store_1 (acquired from src_1), analyze speaker sentiment through multi-modal analysis"
    ]
  },
  "output": {
    "content": {
      "objective_1": {
        "objective": "Obtain video content from src_1 and store as store_1",
        "goals": [
          "Validate that src_1 is a reachable YouTube URL",
          "Download video content from src_1 to platform file storage (Wasabi) as store_1",
          "Verify downloaded file integrity and format compatibility for store_1",
          "Extract video metadata including duration, resolution, and frame rate from store_1"
        ]
      },
      "objective_2": {
        "objective": "From store_1 (acquired from src_1), analyze speaker sentiment through multi-modal analysis",
        "goals": [
          "Extract audio track from store_1 (acquired from src_1)",
          "Transcribe audio content to text with timestamps",
          "Identify and label distinct speakers through diarization",
          "Analyse vocal characteristics for speech emotion recognition per speaker",
          "Run sentiment analysis on transcript segments per speaker",
          "Correlate text sentiment, vocal emotion, and facial expression results per speaker"
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
    "message": "Successfully decomposed 2 objectives into 10 actionable goals"
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
  "resources": {
    "source_refs": [
      {
        "ref_id": "src_1",
        "url": "https://www.youtube.com/watch?v=abc123",
        "platform": "youtube",
        "media_type": "video",
        "provided_by": "user",
        "extracted_at_sequence": 1
      }
    ],
    "storage_refs": [
      {
        "ref_id": "store_1",
        "source_ref_id": "src_1",
        "storage_uri": null,
        "storage_backend": "wasabi",
        "media_type": "video",
        "created_at_sequence": null,
        "status": "pending"
      }
    ],
    "derived_refs": []
  },
  "audit": {
    "compliance_notes": "Objectives successfully decomposed into actionable goals. Storage Resolution Rule applied using ref IDs: acquisition goals reference src_1 with explicit store_1 target; analysis goals reference store_1 (acquired from src_1) on first mention per First-Mention Provenance Rule. All goals map to platform capabilities.",
    "governance_files_consulted": ["context/application.md", "context/governance/message_format.md", "context/governance/audit.md", "agent/governance/goal.md"],
    "reasoning": "Objective 1 (video acquisition) decomposed into 4 goals: validate src_1, download src_1 to Wasabi as store_1, verify store_1 integrity, extract store_1 metadata. Objective 2 (sentiment analysis) decomposed into 6 goals following audio-first pipeline. First goal in objective 2 includes provenance: 'store_1 (acquired from src_1)'; subsequent goals use bare ref IDs per First-Mention Provenance Rule."
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
  "resources": {
    "source_refs": [],
    "storage_refs": [],
    "derived_refs": []
  },
  "audit": {
    "compliance_notes": "Request rejected - video editing and audio addition fall outside platform scope per context/application.md Constraints and Boundaries; platform supports analysis only, not content modification. No resource refs assigned (out-of-scope request).",
    "governance_files_consulted": ["context/application.md", "context/governance/message_format.md", "context/governance/audit.md", "agent/governance/objective.md"],
    "reasoning": "User request asks for video editing (content modification) and audio addition (content creation). Both actions fall outside the platform scope which is limited to video analysis and insight generation. No partial match to any supported capability. No source URLs to extract. Returning OUT_OF_SCOPE with no next_agent to terminate the chain."
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
    ↓
[6] Memory Agent (operational, post-chain) → captures audit logs, distills institutional knowledge into context/memory/
```

Steps [1]–[5] form the main request chain. Each agent receives the message from the previous agent, processes it, updates the relevant fields, and passes it to the next agent.

Step [6] is a post-chain process: after the chain reaches COMPLETE or terminal ERROR status, the orchestration layer invokes the Memory Agent to process the transaction's audit logs and distill institutional knowledge. The Memory Agent may also be invoked on a schedule for batch distillation, or on-demand by other agents for context retrieval. See `agent/operational/memory.md` for full specification.

---

## Compliance Requirements

1. **All agents MUST use this message format** for inter-agent communication
2. **All fields are required** — use `null` for fields that don't apply; use empty arrays `[]` for resource ref arrays with no entries
3. **Messages MUST include audit data** — the `audit` field is mandatory for every message
4. **Messages MUST include resources** — the `resources` field is mandatory for every message, even if all arrays are empty
5. **Resource refs are immutable** — once a ref ID is assigned, it must not be reassigned or reused within the same session
6. **First-Mention Provenance Rule** — the first reference to a storage_ref within each agent's message scope must include parenthetical provenance (e.g., `store_1 (acquired from src_1)`)
7. **Message integrity must be maintained** — do not alter previous agent's data
8. **Sequence numbers must be consecutive** — no gaps in the chain

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
│     - resources.source_refs (ref_id and url pairs)              │
│     - resources.storage_refs (ref_id and status)                │
│     - resources.derived_refs (ref_id and asset_type)            │
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
[RESOURCES]: source_refs=<ref_id:url pairs>, storage_refs=<ref_id:status pairs>, derived_refs=<ref_id:asset_type pairs>
[COMPLIANCE NOTES]: <audit.compliance_notes>
[GOVERNANCE FILES]: <audit.governance_files_consulted>
[MESSAGE_ID: <message_id>]
[SESSION_ID: <metadata.session_id>]
---
```

See `context/governance/audit.md` for complete audit logging governance requirements.

---

## Orchestration Layer: Message Validation

Before passing a message to the next agent, the orchestration layer SHOULD perform the following validations and flag non-compliant messages:

### Metadata Continuity Checks

| Check | Rule | Severity |
|-------|------|----------|
| `request_id` inheritance | For sequence_number > 1: the orchestration layer MUST overwrite `metadata.request_id` with the value from the parent message (identified by `parent_message_id`) before forwarding. Do not rely on the LLM to copy this field correctly. | ERROR |
| `session_id` inheritance | For sequence_number > 1: the orchestration layer MUST overwrite `metadata.session_id` with the value from the parent message before forwarding. | ERROR |
| `sequence_number` continuity | Must equal parent's `sequence_number + 1`. | ERROR |
| `parent_message_id` validity | Must reference an existing `message_id` from the current session. | ERROR |

### Output Integrity Checks

| Check | Rule | Severity |
|-------|------|----------|
| Goal count accuracy | For goal_agent messages: count all goals across all objectives in `output.content` and verify the count in `status.message` matches. | WARNING |
| Resource ref consistency | All `ref_id` values in `resources` must be consistent with the parent message (no dropped or renamed refs). | ERROR |

### Audit Quality Checks

| Check | Rule | Severity |
|-------|------|----------|
| `governance_files_consulted` paths | All entries must be repository-root-relative paths (e.g., `"context/governance/audit.md"`, not `"audit.md"`). | WARNING |
| `audit.reasoning` completeness | Must be non-empty and reference specific governance rules applied. | WARNING |

---

## Version
v1.6.0

## Last Updated
February 20, 2026

## Changelog
- v1.6.0 (Feb 20, 2026): Removed LLM-targeted `request_id`/`session_id` inheritance instructions from Metadata Fields table and metadata inheritance rule paragraph. Removed Example 2 annotation about request_id inheritance. Reframed Metadata Continuity Checks to specify orchestration-layer overwrite (not LLM compliance) for `request_id` and `session_id`. Spec-level instructions proved insufficient across four log runs (20260220-1.md through 20260220-4.md); enforcement is now delegated to the orchestration layer. Retained all Issue 1 (goal count) and Issue 3 (file path) fixes from v1.4.0.
- v1.5.0 (Feb 20, 2026): Updated Metadata Fields table to explicitly mark `session_id` and `request_id` as inherited (generated by first agent only, copied by all subsequent agents). Added metadata inheritance rule summary paragraph below the table. Added annotation after Example 2 highlighting that `request_id` is inherited (not regenerated) even though `message_id` is generated per agent. These changes address the persistent `request_id` regeneration observed in 20260220-2.md logs despite v1.4.0 fixes.
- v1.4.0 (Feb 20, 2026): Added "Orchestration Layer: Message Validation" section defining metadata continuity checks (request_id/session_id inheritance, sequence_number continuity), output integrity checks (goal count accuracy), and audit quality checks (governance file path format). Fixed Example 3 governance_files_consulted to use full paths. These address systematic violations observed in 20260220.md logs.
- v1.3.0 (Feb 19, 2026): Previous release.
