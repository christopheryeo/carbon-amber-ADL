# Cross-Reference Analysis Report

**Sentient Agentic AI Platform — DSTA Video Analysis Application**

**Date:** 21 February 2026  
**Prepared by:** Claude (AI Assistant)

---

## 1. Executive Summary

This report presents the results of a comprehensive cross-reference analysis across all files in the Sentient Agentic AI Platform project. The analysis examined **20+ files** spanning governance, operational, executional agent prompts, context documents, schema definitions, and templates.

The review identified **27 findings** across five categories: terminology/naming inconsistencies, version and staleness issues, structural and reference gaps, content contradictions, and schema-to-spec misalignment. Of these:
- **8 are rated HIGH severity** (requiring immediate attention)
- **12 MEDIUM** (should be addressed in the next revision cycle)
- **7 LOW** (informational or minor)

---

## 2. Scope and Methodology

### Scope
All main project files (excluding worktree copies). Coverage includes agent prompt files, context/governance documents, the JSON schema, the application template, and the README.

### Analysis dimensions
- (A) Terminology consistency
- (B) Version/staleness tracking
- (C) Structural alignment and reference integrity
- (D) Contradiction detection
- (E) Schema-to-spec validation

---

## 3. File Inventory

| File Path | Version | Date | Status |
|-----------|---------|------|--------|
| agent/governance/objective.md | v1.4.0 | Feb 21 | Active — Governance agent #1 |
| agent/governance/goal.md | v1.7.0 | Feb 21 | Active — Governance agent #2 |
| agent/operational/planning.md | v1.2.0 | Feb 21 | Active — Operational agent #3 |
| agent/operational/dispatch.md | v1.0.0 | Feb 21 | Active — Operational agent #4 |
| agent/operational/reasoning.md | v1.0.0 | Feb 21 | Active — CAP-SYN handler |
| agent/operational/memory.md | v1.0.0 | Feb 13 | Active — Post-chain memory |
| agent/operational/learning.md | v0.1.0 | Feb 9 | PLACEHOLDER — Not implemented |
| agent/executional/execution.md | v1.0.0 | Feb 21 | Active — Unified tool agent |
| agent/executional/action.md | v0.1.0 | Feb 9 | STALE — Superseded by execution.md |
| agent/executional/interpretation.md | v0.1.0 | Feb 9 | STALE — Superseded by execution.md |
| agent/executional/perception.md | v0.1.0 | Feb 9 | STALE — Superseded by execution.md |
| context/application.md | v1.3 | Feb 21 | Active — DSTA application context |
| context/instructions.md | v2.0.0 | Feb 10 | OUTDATED — Predates newer agents |
| context/governance/message_format.md | v1.6.0 | Feb 20 | Active — Message spec |
| context/governance/audit.md | v1.4.0 | Feb 20 | Active — Audit requirements |
| context/governance/fileformat.md | v1.0.0 | Feb 9 | Active — Format governance |
| schema/message_schema.json | v1.2.0 | Feb 19 | OUTDATED — Behind message_format.md |
| template/application_template.md | v1.0.0 | Feb 9 | OUTDATED — Missing newer agents |

---

## 4. Findings

### 4.1 Category A: Terminology and Naming Inconsistencies

These findings identify cases where the same concept is referred to using different names, IDs, or conventions across files, creating ambiguity for both LLMs and human maintainers.

| # | Severity | File A | File B | Description |
|---|----------|--------|--------|-------------|
| A1 | **HIGH** | memory.md | dispatch.md | Memory agent ID mismatch: memory.md defines agent.name as "agent_memory" but dispatch.md routes post-chain work to "memory_agent". One must be corrected for routing to work. |
| A2 | **MEDIUM** | application.md | goal.md | CAP-AUD-R identifier format inconsistency: application.md uses "CAP-AUD-R001/R002/R003" (no dash before number), goal.md uses "CAP-AUD-Rxxx", dispatch.md and planning.md use "CAP-AUD-R*" wildcard. Standardise on one convention. |
| A3 | **MEDIUM** | memory.md | instructions.md | Orchestration layer naming: memory.md references "ARL" as the orchestration platform, while instructions.md and application.md use "n8n". If ARL was renamed to n8n, memory.md needs updating. |
| A4 | **HIGH** | memory.md | (architecture) | memory.md references a "Reviewer Agent" that does not exist anywhere in the current agent architecture. This is either a stale reference or an undocumented agent. |
| A5 | **LOW** | goal.md | goal.md | Common Mistakes numbering in goal.md is non-sequential: runs 1–6, then jumps to 10, 11, back to 7, 8, 9, then 12, 13. While functional, it hinders readability and maintenance. |

### 4.2 Category B: Version and Staleness Issues

These findings track files that are outdated, superseded, or significantly behind the versions of related documents.

| # | Severity | File A | File B | Description |
|---|----------|--------|--------|-------------|
| B1 | **HIGH** | schema v1.2.0 | msg_format v1.6.0 | message_schema.json (v1.2.0, Feb 19) is significantly behind message_format.md (v1.6.0, Feb 20). Schema may not validate newer message fields or constraints correctly. |
| B2 | **MEDIUM** | instructions.md | dispatch/reasoning/execution | instructions.md (v2.0.0, Feb 10) predates Dispatch Agent, Reasoning Agent, and Action Agent (all Feb 21). The prompt assembly contract does not mention these newer agents. |
| B3 | **MEDIUM** | action.md / interpretation.md / perception.md | execution.md | Three placeholder files (v0.1.0, Feb 9) in agent/executional/ are superseded by the unified execution.md (v1.0.0, Feb 21). Stale files should be archived or deleted to avoid confusion. |
| B4 | **LOW** | learning.md | (architecture) | learning.md remains a placeholder (v0.1.0, Feb 9) with no substantive content. If the Learning Agent is not yet implemented, consider adding a note to that effect in the README or architecture docs. |
| B5 | **MEDIUM** | application_template.md | application.md | Template (v1.0.0, Feb 9) is outdated: Section 3 lists only Objective and Goal agents (missing Planning Agent). Section 9 Operational Core omits Dispatch Agent. Executional Core uses placeholder names instead of Action Agent. |

### 4.3 Category C: Structural and Reference Issues

These findings identify missing references, broken cross-links, and structural gaps in how files reference one another.

| # | Severity | File A | File B | Description |
|---|----------|--------|--------|-------------|
| C1 | **HIGH** | message_format.md | application.md | Agent Chain Flow diagram (steps [4]–[5]) lists Action Agent capabilities as "CAP-ACQ, CAP-PRE, CAP-AUD, CAP-SPK, CAP-VIS, CAP-DAT" but omits CAP-AUD-R entirely. application.md defines CAP-AUD-R001/R002/R003 as valid capabilities. |
| C2 | **MEDIUM** | instructions.md | schema/message_schema.json | instructions.md defines the 6-step prompt assembly order but does not mention loading schema/message_schema.json. If the schema is used for validation, its loading should be documented. |
| C3 | **LOW** | memory.md | (architecture) | memory.md references concepts not defined in any other file: "circuit breaker", "10-hop limit", and "SLM to LLM escalation". These should either be documented in a shared glossary or removed if obsolete. |
| C4 | **LOW** | memory.md | audit.md | memory.md Knowledge Filing section says files are loaded by "ARL during Context Assembly stage (Station 3)" — this references the old orchestration name and an undocumented "Station 3" concept. |
| C5 | **MEDIUM** | application.md Sec 3 | template Sec 3 | application.md Section 3 correctly lists Objective, Goal, and Planning agents. Template Section 3 only lists Objective and Goal, missing the Planning Agent entry. |

### 4.4 Category D: Content Contradictions

These findings identify cases where two files make conflicting claims about the same behaviour, format, or requirement. These are the most critical to resolve as they can cause runtime failures or incorrect LLM outputs.

| # | Severity | File A | File B | Description |
|---|----------|--------|--------|-------------|
| D1 | **HIGH** | message_format.md Ex.1 | objective.md | Example 1 Objective Agent output shows objectives WITHOUT URL annotations on src_N (e.g., "Obtain video content from src_1 and store as store_1"). objective.md v1.4.0 REQUIRES parenthetical URL annotations (e.g., "from src_1 (https://...)"). The example must be updated. |
| D2 | **HIGH** | message_format.md Ex.2 | goal.md | Example 2 Goal Agent output does not include a "Detect language(s) spoken in the audio" goal before transcription. goal.md v1.7.0 Rule #11 makes language detection MANDATORY before any transcription goal. The example violates this rule. |
| D3 | **HIGH** | memory.md | message_format.md | Memory agent output example uses flat field format ("agent": "agent_memory", "status": "success") instead of the nested object structure required by message_format.md (agent: {name, version, type}, status: {code, message}). |
| D4 | **MEDIUM** | memory.md | audit.md | Log path conflict: memory.md references "system/logs/txn_20250601_001.jsonl" (transaction-based JSONL). audit.md specifies "system/logs/YYYYMMDD.log" (date-based log files). These are incompatible log storage conventions. |
| D5 | **MEDIUM** | memory.md | (architecture) | memory.md output example uses a 2025 date ("2025-06-01") while all other files use 2026 dates. This suggests the memory.md example was written for an earlier version and not updated. |
| D6 | **LOW** | planning.md | dispatch.md | planning.md specifies parent_message_id construction as "msg-goal-{YYYYMMDD}-{HHMMSS}" (deterministic from Goal Agent timestamp). Dispatch.md does not reference this convention, though it consumes the Planning Agent output. |

### 4.5 Category E: Schema vs Specification Alignment

These findings compare message_schema.json against message_format.md and agent prompt files to identify validation gaps.

| # | Severity | File A | File B | Description |
|---|----------|--------|--------|-------------|
| E1 | **HIGH** | schema v1.2.0 | msg_format v1.6.0 | Schema version (v1.2.0) is 4 minor versions behind spec (v1.6.0). Any fields, constraints, or validation rules added in v1.3.0–v1.6.0 are not enforced by the schema. Requires a schema update pass. |
| E2 | **LOW** | schema | msg_format | Positive alignment: agent.type enum ["governance", "operational", "executional"], status.code enum ["success", "partial", "failed", "pending"], and all 6 error codes match between schema and spec. |
| E3 | **LOW** | schema | agent files | Positive alignment: ref_id patterns (src_N, store_N, derived_N) in the schema match conventions used in objective.md, goal.md, and planning.md. |
| E4 | **MEDIUM** | schema | msg_format | Schema requires 11 top-level fields matching message_format.md. However, sub-object constraints (e.g., mandatory audit.compliance_notes, audit.governance_files_consulted per audit.md) may not be fully validated at schema level. |
| E5 | **MEDIUM** | schema | application.md | Schema does not include capability ID validation (e.g., pattern matching for CAP-XXX-NNN format). Capability IDs in DAG tasks are currently free-text strings with no schema enforcement. |
| E6 | **MEDIUM** | schema | planning.md | Planning Agent DAG task schema (8 fields including execution_group, estimated_weight, dependencies) is defined in planning.md but not formally captured in message_schema.json for validation. |

---

## 5. Summary by Severity

| Severity | Count | Categories | Key Action |
|----------|-------|-----------|-----------|
| **HIGH** | 8 | A(2), B(1), C(1), D(3), E(1) | Fix immediately — runtime/validation failures |
| **MEDIUM** | 12 | A(2), B(3), C(2), D(2), E(3) | Address in next revision cycle |
| **LOW** | 7 | A(1), B(1), C(2), D(1), E(2) | Informational / minor cleanup |

---

## 6. Priority Recommendations

### Immediate Actions (HIGH findings)

- **Resolve memory agent ID mismatch (A1):** Align memory.md agent.name with dispatch.md routing target. Recommended: standardise on "memory_agent".

- **Update message_format.md examples (D1, D2):** Add URL annotations to Example 1 objectives. Add language detection goal to Example 2 Goal Agent output. These examples are the primary reference for LLM behaviour.

- **Fix memory.md message format (D3):** Rewrite the output example to use the nested object structure (agent: {name, version, type}, status: {code, message}) per message_format.md.

- **Add CAP-AUD-R to Agent Chain Flow (C1):** Include CAP-AUD-R in the capability list at steps [4]–[5] of message_format.md.

- **Update message_schema.json to v1.6.0 (E1/B1):** Synchronise the JSON schema with the current message_format.md specification. Add any new field constraints introduced in v1.3.0 through v1.6.0.

- **Remove or document Reviewer Agent (A4):** If the Reviewer Agent was deprecated, remove all references from memory.md. If planned, add it to the architecture documentation.

### Next Revision Cycle (MEDIUM findings)

- Standardise CAP-AUD-R identifier format across all files (A2).

- Update instructions.md to reflect the full agent chain including Dispatch, Reasoning, and Execution agents (B2).

- Archive or delete superseded placeholder files: action.md, interpretation.md, perception.md (B3).

- Update application_template.md to match the current architecture (B5).

- Align memory.md orchestration references from "ARL" to "n8n" (A3).

- Resolve log path convention conflict between memory.md and audit.md (D4).

- Add schema validation for capability IDs, DAG task structure, and audit sub-fields (E4–E6).

---

*End of Report*
