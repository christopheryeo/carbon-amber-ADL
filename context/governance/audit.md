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
    "governance_files_consulted": ["<list of governance files referenced>"],
    "reasoning": "<chain of thought explaining why decisions were made>"
  }
}
```

### Audit Field Definitions

| Field | Required | Description |
|-------|----------|-------------|
| `audit.compliance_notes` | Yes | Governance compliance observations, validations performed, scope confirmations, and any flags or concerns |
| `audit.governance_files_consulted` | Yes | Array of governance file names referenced during processing (e.g., `["context/application.md", "message_format.md", "audit.md"]`) |
| `audit.reasoning` | Yes | Chain of Thought explaining why decisions were made — the rationale behind routing, scope validation, output choices, and any trade-offs considered |

### What to Include in compliance_notes

- Confirmation that the request is within platform scope (per `context/application.md`)
- Validation that output aligns with platform capabilities
- Any governance concerns or flags identified
- Reason for rejection (if status is `failed` or `rejected`)
- References to specific constraints or boundaries applied

### What to Include in reasoning

- Why the agent chose to route to a specific next agent
- How scope validation was determined (what was checked, what matched)
- Rationale behind output choices (why these objectives, goals, or plans)
- Trade-offs considered and why one approach was selected over alternatives
- How institutional knowledge from `context/memory/` influenced decisions (if applicable)
- **Resource extraction audit** (Objective Agent only): How many URLs/file paths were found in user input, which URL was assigned to which ref ID (e.g., `src_1 → https://youtu.be/xyz`), platform detection results for each URL, and the rationale for ref ID assignment order
- **Storage ref mapping** (Objective Agent only): Which `store_N` was created for each `src_N` and why
- **Derived ref registration** (Planning Agent only): Which `derived_N` refs were created, what `parent_ref_id` each derives from, and which capability produces the asset

### Example (Objective Agent)

```json
"audit": {
  "compliance_notes": "Request within video analysis scope per context/application.md; objectives align with Audio Analysis and Speaker Analysis capabilities; validated against supported video sources (YouTube). Resource extraction: 1 URL found, assigned src_1 → store_1.",
  "governance_files_consulted": ["context/application.md", "message_format.md", "audit.md", "objective.md"],
  "reasoning": "User request contains one YouTube URL: https://youtu.be/pcaYkGY996o. Assigned src_1 (platform: youtube, media_type: video). Created corresponding store_1 for Wasabi storage. User request contains two distinct actions (download and analyze) requiring separate objectives. Applied Acquisition-First Pattern: Objective 1 uses ref IDs (src_1, store_1) for video acquisition. Objective 2 references store_1 with first-mention provenance (acquired from src_1) for sentiment analysis. Sentiment analysis maps to Speaker Analysis capability."
}
```

### Example (Planning Agent)

```json
"audit": {
  "compliance_notes": "All goals mapped to valid capabilities in Capabilities Matrix. 0 duplicate goals detected. Registered 4 derived_refs (derived_1 through derived_4) for intermediate assets.",
  "governance_files_consulted": ["context/application.md", "message_format.md", "audit.md", "planning.md"],
  "reasoning": "Received 2 objectives with 11 total goals from goal_agent. All goals map to CAP-IDs in the Capabilities Matrix. Derived ref registration: derived_1 (audio_track from store_1, CAP-PRE-002), derived_2 (frame_set from store_1, CAP-PRE-003), derived_3 (transcript from derived_1, CAP-AUD-001), derived_4 (diarization_map from derived_1, CAP-AUD-002). DAG validation passed: no circular dependencies. Execution groups: 7 tiers with parallelism in groups 3, 4, 5, 6."
}
```

---

## Log Storage and Retention

Logs are stored in daily files at `system/logs/YYYYMMDD.log` (reverse date format). For example, January 27, 2026 → `20260127.log`, February 15, 2026 → `20260215.log`. A new log file is created daily at the start of each calendar day.

For complete details on how the orchestration layer extracts audit data from JSON messages, the fields that are logged, and the log entry format, refer to the "Orchestration Layer: Audit Log Extraction" section in `message_format.md`.

---

## Mandatory Logging Requirements

- **Every JSON message** must include the `audit` field
- **Every message** is logged by the orchestration layer — no exceptions
- **Failed/rejected requests** must include rejection rationale in `compliance_notes`
- **Error conditions** must be logged with full error details from the `error` field
- **Resource extraction** (Objective Agent): `audit.reasoning` must document all ref ID assignments (src_N, store_N) and platform detection results
- **Derived ref registration** (Planning Agent): `audit.reasoning` must document all derived_N refs created, their parent refs, and producing capabilities

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

## Version
v1.3.0

## Last Updated
February 19, 2026

## Changelog
- v1.3.0 (Feb 19, 2026): Added resource extraction audit requirements for Objective Agent (ref ID assignment documentation) and Planning Agent (derived_refs registration documentation). Updated examples to show resource tracking in audit reasoning. Added mandatory logging requirements for resources.
- v1.2.0 (Feb 13, 2026): Previous release.
