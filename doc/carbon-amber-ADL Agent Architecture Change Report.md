# Agent Architecture Change Report

**Dispatch, Execution, and Reasoning Agent Restructure**

Version 1.0.0 | 21 February 2026

**Classification:** Internal

---

## 1. Executive Summary

This report documents a significant restructuring of the carbon-amber-ADL agent architecture. The changes address an architectural gap identified between the Planning Agent (which produces a static DAG-based execution plan) and the former Executional Core agents (which could only process one task at a time). The restructure introduces a new Dispatch Agent, consolidates three deprecated agents into a unified Action Agent, and repositions the Reasoning Agent for synthesis tasks.

### Key Changes at a Glance

| Change Type | Agent | Description |
|---|---|---|
| **NEW** | **Dispatch Agent** | Runtime DAG executor bridging Planning and Execution |
| **NEW** | **Action Agent** | Unified tool-calling agent replacing three deprecated agents |
| **REWRITTEN** | **Reasoning Agent** | Repositioned from placeholder to dedicated CAP-SYN synthesis agent |
| **DEPRECATED** | **Perception Agent** | Replaced by Action Agent |
| **DEPRECATED** | **Interpretation Agent** | Eliminated; responsibilities absorbed by Execution and Reasoning agents |
| **DEPRECATED** | **Action Agent** | Replaced by Action Agent |

---

## 2. Problem Statement

Analysis of production log files revealed a structural gap in the agent chain. The Planning Agent produces a multi-task DAG with execution groups and dependency chains, but the former Executional Core agents (Perception, Interpretation, Action) were designed to handle individual tasks without any runtime orchestration logic. This created two critical issues:

- **No runtime DAG executor:** The static execution plan had no agent responsible for managing task sequencing, parallel dispatch, failure handling, or ref resolution at runtime. The n8n orchestration layer handled message routing but lacked DAG-awareness.

- **Unclear agent boundaries:** The Perception, Interpretation, and Action agents had overlapping responsibilities. Perception and Action both invoked MCP tools, while Interpretation served as a pass-through contextualiser with no clear value-add in an automated pipeline.

These issues resulted in fragile workflow execution where task failures could silently cascade, parallel execution groups were not properly coordinated, and the agent chain had unnecessary complexity.

---

## 3. Architecture: Before and After

### 3.1 Previous Agent Chain

| Seq | Agent | Core | Role |
|---|---|---|---|
| [1] | Objective Agent | Governance | Strategic objectives |
| [2] | Goal Agent | Governance | Goals and sub-goals |
| [3] | Planning Agent | Operational | Static DAG execution plan |
| [4] | Perception Agent | Executional | Speaker ID, expressions, object detection |
| [5] | Interpretation Agent | Executional | Contextualise results |
| [6] | Action Agent | Executional | Execute analysis models, structured outputs |
| [7] | Memory Agent | Operational | Post-chain audit and knowledge capture |

*Red rows indicate agents that have been deprecated.*

### 3.2 New Agent Chain

| Seq | Agent | Core | Role |
|---|---|---|---|
| [1] | Objective Agent | Governance | Strategic objectives |
| [2] | Goal Agent | Governance | Goals and sub-goals |
| [3] | Planning Agent | Operational | Static DAG execution plan |
| [4] | **Dispatch Agent** | Operational | Runtime DAG execution, task dispatch, failure handling |
| [5] | **Action Agent** | Executional | MCP tool invocation for all tool-calling capabilities |
| [5] | **Reasoning Agent** | Operational | CAP-SYN multi-modal synthesis, timelines, reports |
| [6] | Memory Agent | Operational | Post-chain audit and knowledge capture |

*Green rows are new agents. Blue row is the rewritten Reasoning Agent. Steps [4]-[5] form an iterative loop.*

---

## 4. New Agent: Dispatch Agent

| Property | Value |
|---|---|
| **File** | `agent/operational/dispatch.md` |
| **Version** | 1.0.0 |
| **Core** | Operational |
| **Sequence** | [4] — immediately after Planning Agent |

### 4.1 Primary Function

The Dispatch Agent receives the execution workflow (DAG) from the Planning Agent and manages its runtime execution. It is the bridge between the static execution plan and live task execution. The Planning Agent defines what to run and in what order; the Dispatch Agent decides when to run each task, which agent runs it, and what to do when things go wrong.

### 4.2 Key Responsibilities

- **Workflow State Management:** Maintains a running record of task states (pending, ready, in_progress, complete, failed, skipped) and a ref_registry mapping ref IDs to resolved storage URIs.

- **Task Dispatch:** Selects the next ready task(s) from the current execution group and routes them to the correct downstream agent based on capability type.

- **Agent Routing:** Routes CAP-ACQ, CAP-PRE, CAP-AUD, CAP-SPK, CAP-VIS, CAP-DAT tasks to the Action Agent; routes CAP-SYN tasks to the Reasoning Agent.

- **Ref Resolution:** Resolves input_refs for each task by looking up the ref_registry to provide concrete storage URIs. Refs are immutable within a session.

- **Failure Handling:** Classifies failures (transient vs. permanent), applies retry logic with configurable limits, walks transitive dependencies to skip blocked downstream tasks, and escalates when retry limits are exhausted.

- **Parallel Coordination:** All tasks within an execution group may run in parallel. Group N+1 begins only after all tasks in Group N reach a terminal state (complete, failed, or skipped).

### 4.3 Output Phases

The Dispatch Agent produces two categories of output: Phase A (task_dispatch) is emitted once per task or parallel group during execution. Phase B (workflow_complete) is emitted once at the end when all tasks have reached terminal state, containing the full workflow result with final ref_registry and execution summary.

---

## 5. New Agent: Action Agent

| Property | Value |
|---|---|
| **File** | `agent/executional/execution.md` |
| **Version** | 1.0.0 |
| **Core** | Executional (application-specific) |
| **Replaces** | Perception Agent, Interpretation Agent, Action Agent |

### 5.1 Primary Function

The Action Agent receives a single task from the Dispatch Agent and executes it by invoking the appropriate tools via MCP (Model Context Protocol). It resolves input references, selects and configures the correct tool for the task's capability, executes the tool, structures the output, and returns results to the Dispatch Agent. Each invocation handles exactly one task.

### 5.2 Capability Coverage

The Action Agent covers all tool-calling capabilities in the platform's Capability Matrix:

| Capability Group | Example Capabilities | MCP Tools |
|---|---|---|
| **CAP-ACQ** | URL validation, media download | url_validator, yt-dlp, wget |
| **CAP-PRE** | Audio extraction, video segmentation | ffmpeg, scenedetect |
| **CAP-AUD** | Transcription, diarisation | Whisper-X, pyannote |
| **CAP-SPK** | Speaker embedding, voice matching | Resemblyzer, speaker-db |
| **CAP-VIS** | Object detection, facial analysis | YOLO, DeepFace, InsightFace |
| **CAP-DAT** | Storage upload, metadata extraction | Wasabi S3, mediainfo |

### 5.3 Quality Checks and Error Handling

Every task result includes a quality_checks object verifying output_exists, output_non_empty, and format_valid. Errors are classified by recoverability (transient: retry at Dispatch level; permanent: mark failed immediately) with structured error codes and messages returned to the Dispatch Agent.

---

## 6. Rewritten Agent: Reasoning Agent

| Property | Value |
|---|---|
| **File** | `agent/operational/reasoning.md` |
| **Previous** | v0.1.0 placeholder — generic "logical inferences" stub |
| **New Version** | v1.0.0 — dedicated CAP-SYN synthesis agent |
| **Core** | Operational (repositioned from Executional consideration) |

### 6.1 Supported Capabilities

| Capability | Name | Description |
|---|---|---|
| **CAP-SYN-001** | Multi-Modal Fusion | Fuses cross-modal analysis results (text, audio, visual) into unified assessment findings with sentiment, emotion, stance, and credibility dimensions |
| **CAP-SYN-002** | Timeline Reconstruction | Reconstructs event timelines from timestamped evidence across multiple modalities, annotating each event with supporting sources |
| **CAP-SYN-003** | Structured Report Generation | Generates structured reports with sourced claims, evidence cross-references, and confidence assessments suitable for downstream presentation |

### 6.2 Key Design Decisions

- **LLM-based reasoning, not tool invocation:** Unlike the Action Agent which calls MCP tools, the Reasoning Agent performs synthesis through LLM reasoning across multiple completed task outputs.

- **Cross-modal consistency checks:** Every synthesis output includes validation that findings are consistent across modalities, with flagging when contradictions are detected.

- **Confidence assessment:** Output includes overall confidence (high/medium/low) based on modality coverage, with explicit noting of missing modalities and their impact.

- **Partial-input handling:** The agent can produce results even when some input modalities are missing (e.g., no visual data), marking affected findings with reduced confidence.

---

## 7. Cross-Reference Updates

The following existing files were updated to reflect the new architecture:

### 7.1 planning.md

- **Change:** Line 711 — next_agent.name changed from "perception_agent" to "dispatch_agent"

- **Rationale:** The Planning Agent now hands off to the Dispatch Agent, not directly to an executional agent.

### 7.2 message_format.md

- **Change:** Agent Chain Flow diagram updated from 6 steps to 7 steps. Dispatch Agent inserted at position [4]. Execution/Reasoning Agent at [5] with iterative loop notation. Memory Agent moved to [7].

- **Change:** Explanatory text updated to describe the [4]-[5] dispatch loop and the CAP-based routing split between Execution and Reasoning agents.

### 7.3 application.md

- **Section 9:** Operational Core table updated to include Dispatch Agent and rewritten Reasoning Agent description. Executional Core table replaced three deprecated agents with unified Action Agent.

- **Section 10:** Phase 3 execution steps rewritten to describe the Dispatch Agent's role in DAG execution, task routing to Action Agent and Reasoning Agent, and failure handling with downstream impact analysis.

---

## 8. Remaining Work

The following items contain stale references to the deprecated agents and should be addressed in a follow-up effort:

| File / Location | Issue | Priority |
|---|---|---|
| `agent/executional/perception.md` | Full deprecated agent file still exists | Medium — add deprecation header or archive |
| `agent/executional/interpretation.md` | Full deprecated agent file still exists | Medium — add deprecation header or archive |
| `agent/executional/action.md` | Full deprecated agent file still exists | Medium — add deprecation header or archive |
| `agent/operational/memory.md` | References perception_agent and action_agent | High — update references |
| `temp/agentPrompt.md` | References deprecated agent names | Medium — regenerate from updated source |
| `template/application_template.md` | Template still lists old agent names | Medium — update template |
| `.claude/worktrees/*` | Worktree copies contain old references | Low — will sync on next branch update |
| `system/logs/*` | Historical logs reference old agent names | None — historical records, do not modify |

---

## 9. File Inventory

Complete list of files created or modified as part of this architecture change:

| File Path | Action | Version | Lines (approx.) |
|---|---|---|---|
| `agent/operational/dispatch.md` | **Created** | 1.0.0 | ~350 |
| `agent/executional/execution.md` | **Created** | 1.0.0 | ~400 |
| `agent/operational/reasoning.md` | **Rewritten** | 0.1.0 → 1.0.0 | ~400 |
| `agent/operational/planning.md` | **Edited** | 1.2.0 | 1 line changed |
| `context/governance/message_format.md` | **Edited** | 1.6.0 | ~20 lines changed |
| `context/application.md` | **Edited** | 1.3 | ~25 lines changed |

---

*End of Report*
