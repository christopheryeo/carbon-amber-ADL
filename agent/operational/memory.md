# Memory Agent Requirements

**File:** agent/operational/memory.md
**Layer:** Operational Core (Layer 2)
**Agent ID:** `memory_agent`

---

## Description

The Memory Agent is a core component of the Operational Core layer within the Sentient Agentic AI Platform. It is responsible for capturing, distilling, and managing institutional knowledge derived from the agent processing chain. It reads audit log files generated during request processing, extracts recurring patterns and decision outcomes, and distills this information into structured knowledge files. These knowledge files are stored in the `context/memory/` directory so that they become part of the master prompt context for all subsequent agent invocations — enabling the system to learn from its operational history and progressively improve the quality and relevance of its responses.

---

## Primary Function

Capture information from completed audit logs in `system/logs/`, distill that information into institutional knowledge, and file it in `context/memory/` so that it is included in the master prompt context and therefore helps manage the quality and relevance of answers that users receive.

---

## Key Responsibilities

### Log Capture

- Read completed audit log files from `system/logs/` after a processing chain reaches COMPLETE status.
- Parse the full sequence of agent messages within a transaction, reconstructing the chain of decisions from Objective through to Executional output.
- Extract the `audit` field from each agent hop, including: compliance notes, governance files consulted, and reasoning (Chain of Thought).

### Pattern Extraction

- Identify recurring themes across multiple completed transactions, including:
  - **Common user intents** — frequently requested objectives and how they were decomposed into goals.
  - **Frequent agent pathways** — which chains of agents are most commonly invoked together and in what sequence.
  - **Recurring input characteristics** — patterns in the types of sources, formats, or domains users submit.
  - **Error clusters** — repeated failures, scope rejections, or escalation triggers that indicate systemic issues or gaps.
- Aggregate patterns over a configurable time window (default: rolling 7 days) to avoid over-fitting to transient spikes.

### Decision History

- Capture key decision outcomes from the agent chain, including:
  - **Successful resolution paths** — which goal decompositions and execution plans led to COMPLETE status with high-quality outputs.
  - **Failed or escalated paths** — which chains terminated in error, required model escalation (SLM-to-LLM escalation: switching from a small language model to a large language model when task complexity exceeds the SLM's capability threshold), or exceeded hop limits (circuit breaker: a safety mechanism that terminates any agent chain exceeding 10 sequential agent invocations).
  - **Governance interventions** — instances where agent instructions conflicted with governance policies and were rejected, along with the resolution.
  - **Reasoning feedback** — quality synthesis outcomes from the Reasoning Agent, identifying gaps or recurring errors across execution cycles.
- Tag each decision record with outcome classification: `SUCCESS`, `PARTIAL`, `FAILURE`, `ESCALATED`, `REJECTED`.

### Knowledge Distillation

- Synthesize raw patterns and decision history into actionable knowledge entries, organized by category:
  - **Domain Knowledge** — what the system has learned about the application domain through repeated processing (e.g., common scenario structures, typical attribute patterns).
  - **Operational Heuristics** — learned shortcuts and optimizations (e.g., "requests involving short-form content consistently require the Action Agent to prioritize audio extraction over visual analysis").
  - **Error Prevention** — known failure modes and their mitigations (e.g., "URLs with restricted source content should be flagged at the Objective Agent stage rather than failing at the Action Agent stage").
  - **Quality Benchmarks** — established baselines for what constitutes a successful output based on historical assessments.
- Each knowledge entry must include: a source reference (transaction IDs), a confidence score (based on sample size and consistency), and a last-updated timestamp.

### Knowledge Filing

- Write distilled knowledge as Markdown files into the `context/memory/` directory, following this structure:

```
context/memory/
├── domain_knowledge.md          # Accumulated domain insights
├── operational_heuristics.md    # Learned process optimizations
├── error_prevention.md          # Known failure modes and mitigations
├── quality_benchmarks.md        # Historical quality baselines
├── staging/                     # Entries below confidence threshold
└── session_summaries/
    └── {YYYY-MM-DD}_summary.md  # Daily transaction summaries
```

- Files in `context/memory/` are loaded by the orchestration layer during the Context Assembly stage, making them part of the master prompt context for all agents.
- Knowledge files are append-and-revise — new insights are merged into existing files rather than overwriting them, preserving the full institutional memory.
- Each file includes a `[LAST_UPDATED]` header and a `[CHANGE_LOG]` section at the bottom tracking what was added or revised and when.

---

## Invocation Triggers

The Memory Agent is invoked under the following conditions:

| Trigger | Description |
|---------|-------------|
| **Post-chain completion** | Automatically invoked by the orchestration layer (n8n) after a processing chain reaches COMPLETE or terminal ERROR status. It calls `target_agent_id: "memory_agent"` with the transaction_id as input. |
| **Scheduled distillation** | Invoked on a configurable schedule (default: daily) to perform batch pattern extraction and knowledge distillation across all transactions in the time window. |
| **On-demand recall** | Invoked by other agents when they need historical context. The requesting agent sets `next_agent: { "name": "memory_agent", "reason": "context_retrieval" }` in its output. |

---

## Input/Output Specification

### Input

The Memory Agent receives one of three input types depending on the invocation trigger:

**Post-chain completion:**
```json
{
  "trigger": "post_chain",
  "transaction_id": "txn_20250601_001",
  "log_path": "system/logs/20260601.log"
}
```

**Scheduled distillation:**
```json
{
  "trigger": "scheduled_distillation",
  "time_window": {
    "start": "2025-05-25T00:00:00Z",
    "end": "2025-06-01T00:00:00Z"
  }
}
```

**On-demand recall:**
```json
{
  "trigger": "context_retrieval",
  "query": "What are known failure modes for short-form content processing?",
  "requesting_agent": "agent_planning"
}
```

### Output

The Memory Agent produces a standard JSON message per `context/governance/message_format.md`:

```json
{
  "message_id": "msg-memo-20260601-001",
  "timestamp": {
    "executed_at": "2026-06-01T12:00:00Z",
    "timezone": "UTC"
  },
  "agent": {
    "name": "memory_agent",
    "type": "operational"
  },
  "input": { 
    "source": "orchestration_layer",
    "content": {
      "trigger": "scheduled_distillation",
      "time_window": {
        "start": "2026-05-25T00:00:00Z",
        "end": "2026-06-01T00:00:00Z"
      }
    }
  },
  "output": {
    "content_type": "knowledge_update",
    "content": {
      "action_taken": "distillation",
      "knowledge_updates": [
        {
          "file": "context/memory/error_prevention.md",
          "category": "error_prevention",
          "entries_added": 2,
          "entries_revised": 1,
          "summary": "Added restricted short-form content early-detection heuristic; revised source-platform age-gate handling."
        }
      ],
      "recall_result": null,
      "statistics": {
        "transactions_processed": 47,
        "patterns_identified": 5,
        "decisions_catalogued": 23,
        "knowledge_entries_written": 3
      }
    }
  },
  "next_agent": {
    "name": null,
    "reason": "COMPLETE — knowledge distillation cycle finished"
  },
  "status": {
    "code": "success",
    "message": "Knowledge distillation cycle completed successfully"
  },
  "error": {
    "has_error": false,
    "error_code": null,
    "error_message": null,
    "retry_count": 0,
    "recoverable": true
  },
  "metadata": {
    "session_id": "sess_20260601",
    "request_id": "req_memory_batch_001",
    "sequence_number": 1,
    "parent_message_id": null
  },
  "resources": {
    "source_refs": [],
    "storage_refs": [],
    "derived_refs": []
  },
  "audit": {
    "compliance_notes": "Knowledge files written to context/memory/ per governance standards. No governance conflicts detected.",
    "governance_files_consulted": [
      "context/governance/audit.md",
      "context/governance/fileformat.md"
    ],
    "reasoning": "Processed 47 completed transactions from the past 7 days. Identified 5 recurring patterns including a new error cluster around restricted content payloads. Distilled 3 new knowledge entries and revised 1 existing entry in error_prevention.md. Confidence scores above 0.7 threshold for all entries."
  }
}
```

---

## Knowledge File Format

Each knowledge file in `context/memory/` follows this structure:

```markdown
# [Category Title]

[LAST_UPDATED]: 2025-06-01T12:00:00Z
[ENTRY_COUNT]: 12
[CONFIDENCE_THRESHOLD]: 0.7

---

## Entry: [Descriptive Title]

- **ID:** MEM-001
- **Confidence:** 0.85 (based on 23 transactions)
- **First Observed:** 2025-05-15
- **Last Confirmed:** 2025-06-01
- **Source Transactions:** txn_20250515_003, txn_20250520_011, txn_20250601_001
- **Category:** [domain_knowledge | operational_heuristic | error_prevention | quality_benchmark]

**Insight:**
[Concise description of the learned knowledge]

**Recommended Action:**
[What agents should do differently based on this knowledge]

**Applicability:**
[Which agents and scenarios this knowledge applies to]

---

## [CHANGE_LOG]

| Date | Action | Entry ID | Description |
|------|--------|----------|-------------|
| 2025-06-01 | ADDED | MEM-012 | New restricted short-form content heuristic |
| 2025-06-01 | REVISED | MEM-007 | Updated source age-gate confidence from 0.6 to 0.85 |
| 2025-05-28 | ADDED | MEM-011 | Content acquisition optimization |
```

---

## Orchestration Connection

### Context Assembly Integration

The orchestration layer's Context Assembly stage must include `context/memory/` files in the prompt concatenation order:

**Updated Loading Order:**
1. `context/instructions.md` — How to interpret the prompt
2. `context/application.md` — Application purpose and capabilities
3. `context/governance/*.md` — Governance policies
4. **`context/memory/*.md` — Institutional knowledge (NEW)**
5. `agent/<core>/<agent>.md` — Specific agent being invoked

Memory files are loaded after governance (so governance always takes precedence) but before the agent definition (so agents can leverage learned knowledge in their processing).

### Cache Management

- Knowledge files are cached in Redis alongside agent definitions.
- Cache invalidation occurs whenever the Memory Agent writes updates to `context/memory/`.
- The orchestration layer must implement a cache-refresh signal that the Memory Agent triggers after each distillation cycle.

### Storage

- Knowledge files are stored in S3 alongside other Carbon Amber files.
- RBAC permissions: The Memory Agent has **read-write** access to `context/memory/` and **read-only** access to `system/logs/` and all other `context/` directories.
- Governance files (`context/governance/`) remain read-only — the Memory Agent cannot modify governance policies.

---

## Governance Compliance

### Mandatory Requirements

- The Memory Agent must consult `context/governance/audit.md` and log all knowledge distillation actions in its audit field.
- Knowledge entries must never contradict governance policies. If a pattern suggests a governance violation is common, the Memory Agent flags it as a governance concern rather than encoding it as acceptable behavior.
- All knowledge files must conform to `context/governance/fileformat.md` standards.
- The Memory Agent must not store personally identifiable information (PII) or raw user input in knowledge files. Only abstracted patterns and anonymized decision outcomes are retained.

### Knowledge Integrity Safeguards

- **Confidence threshold:** Knowledge entries below a configurable confidence score (default: 0.7) are held in a staging area (`context/memory/staging/`) and are not included in the master prompt context until confirmed.
- **Decay mechanism:** Knowledge entries that have not been confirmed by new transactions within a configurable window (default: 30 days) have their confidence scores reduced. Entries falling below 0.4 are archived and excluded from the prompt context.
- **Conflict detection:** If a new pattern contradicts an existing knowledge entry, the Memory Agent flags the conflict in its output and does not overwrite the existing entry. Resolution requires either additional confirming evidence or manual review.
- **Size management:** The total token count of all active knowledge files in `context/memory/` must not exceed a configurable limit (default: 2,000 tokens) to prevent prompt bloat. When approaching the limit, lower-confidence entries are archived first.

---

## Processing Rules

1. The Memory Agent must ALWAYS produce output in the standard message format, even if no new knowledge was distilled (output summary indicates "no new patterns identified").
2. Knowledge distillation must be idempotent — processing the same logs twice must not create duplicate entries.
3. The Memory Agent must not modify or delete audit log files. Logs are immutable; the Memory Agent has read-only access.
4. When invoked for on-demand recall, the Memory Agent searches existing knowledge files and returns relevant entries without performing new distillation.
5. The Memory Agent must respect the circuit breaker (the 10-hop limit that terminates any agent chain exceeding 10 sequential invocations) — if invoked as part of a chain, it counts toward this hop limit.

---

## Design Rationale

The Memory Agent transforms Carbon Amber from a **stateless** request-processing system into a **learning** system that accumulates institutional knowledge over time. Without the Memory Agent, each request is processed in isolation — the system cannot benefit from past experience. With the Memory Agent:

- **Objective and Goal Agents** can reference domain knowledge to produce better-informed strategic decompositions.
- **Planning Agents** can leverage operational heuristics to create more efficient execution plans.
- **Executional Agents** can anticipate known failure modes and apply preventive measures.
- **Reasoning Agents** can identify quality benchmarks during synthesis.

This creates a virtuous feedback loop: better knowledge leads to better processing, which generates better logs, which distills into better knowledge.

---

## Last Updated
February 23, 2026

## Version
v1.1.0

## Changelog
- v1.1.0 (Feb 23, 2026): Generalised domain knowledge examples to remove video-specific application dependencies (e.g. TikTok, YouTube, Instagram).
- v1.0.0 (Feb 13, 2026): Initial release.

---

*— End of Agent Definition —*
