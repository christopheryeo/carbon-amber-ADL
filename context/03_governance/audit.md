# AI Audit and Logging Governance

## Description

This document establishes mandatory audit and logging requirements for all AI agent instructions and responses within the Sentient Agentic AI Platform to ensure accountability, traceability, and compliance monitoring.

---

## How Audit Logging Works

Audit logging is a **two-step process** that leverages the JSON message format:

1. **LLM Responsibility**: Include the `audit` field in every JSON message output (per `message_format.md`)
2. **Orchestration Layer Responsibility**: Extract audit-relevant fields from the JSON message and write to the daily log file

This approach ensures every LLM response is logged while maintaining a single, consistent output format.

---

## LLM Audit Requirements

**Every JSON message MUST include the `audit` field.**

The `audit` field is part of the standard message format defined in `message_format.md`:

```json
{
  "audit": {
    "compliance_notes": "<governance compliance observations and validations>",
    "governance_files_consulted": ["<list of governance files referenced>"]
  }
}
```

### Audit Field Definitions

| Field | Required | Description |
|-------|----------|-------------|
| `audit.compliance_notes` | Yes | Governance compliance observations, validations performed, scope confirmations, and any flags or concerns |
| `audit.governance_files_consulted` | Yes | Array of governance file names referenced during processing (e.g., `["application.md", "message_format.md", "audit.md"]`) |

### What to Include in compliance_notes

- Confirmation that the request is within platform scope (per `application.md`)
- Validation that output aligns with platform capabilities
- Any governance concerns or flags identified
- Reason for rejection (if status is `failed` or `rejected`)
- References to specific constraints or boundaries applied

### Example

```json
"audit": {
  "compliance_notes": "Request within video analysis scope per application.md; objectives align with Audio Analysis and Speaker Analysis capabilities; validated against supported video sources (YouTube)",
  "governance_files_consulted": ["application.md", "message_format.md", "audit.md"]
}
```

---

## Orchestration Layer: Log Extraction and Writing

The orchestration layer extracts audit data from each JSON message and writes it to the daily log file.

### Fields Extracted for Logging

From each JSON message, the orchestration layer extracts:

| JSON Field | Audit Purpose |
|------------|---------------|
| `timestamp.executed_at` | When the agent processed the request |
| `agent.name` | Which agent produced the response |
| `input.source` | Origin of the input (user or upstream agent) |
| `input.content` | The instruction/request received |
| `output.content_type` | Type of output produced |
| `status.code` | Execution status (success, failed, etc.) |
| `status.message` | Human-readable status description |
| `error.*` | Error details (if `has_error` is true) |
| `audit.compliance_notes` | Governance compliance observations |
| `audit.governance_files_consulted` | Files referenced |
| `message_id` | Unique message identifier |
| `metadata.session_id` | Session identifier for traceability |

### Daily Log Files

- **Location**: `system/logs/`
- **File Naming Convention**: YYYYMMDD.log (reverse date format)
- **Examples**:
  - January 27, 2026 → `20260127.log`
  - February 15, 2026 → `20260215.log`
- **New Log File Creation**: A new log file is created daily at the start of each calendar day
- **Log Rotation**: Each day uses its own dedicated log file

### Log Entry Format

The orchestration layer writes entries in this format:

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

---

## Mandatory Logging Requirements

- **Every JSON message** must include the `audit` field
- **Every message** is logged by the orchestration layer — no exceptions
- **Failed/rejected requests** must include rejection rationale in `compliance_notes`
- **Error conditions** must be logged with full error details from the `error` field

### Compliance Verification

- The orchestration layer should flag any JSON message missing the `audit` field as non-compliant
- Messages without audit data should be logged with a compliance violation note
- System administrators should monitor for audit field omissions

---

## Audit Trail Integrity

- Log files are immutable records and must not be altered retroactively
- Log entries must be appended sequentially in chronological order
- Any corruption or loss of log data must be reported immediately
- Regular audit reviews should be conducted to ensure logging completeness

---

## Enforcement

| Responsibility | Owner |
|----------------|-------|
| Include `audit` field in every JSON message | LLM / AI Agent |
| Extract audit data and write to log files | Orchestration Layer |
| Monitor log completeness and compliance | System Administrators |
| Flag missing audit fields | Orchestration Layer |

Non-compliant operations (messages without audit data) are prohibited and must be flagged for review.

---

## Last Updated
January 27, 2026
