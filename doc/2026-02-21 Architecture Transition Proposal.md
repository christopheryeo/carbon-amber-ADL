# Carbon Amber ADL Analysis Report
**Date:** 2026-02-21 (Updated 2026-02-23)
**Subject:** Agent Architecture Review & Transition Proposal (V1 → V2)

> **Historical Context:** This document, originally named `carbon-amber-adl Development.md`, was produced on Feb 21, 2026 after analyzing the first live n8n orchestration logs. It identified a critical gap in execution capability and proposed the transition to the V2 architecture, including the creation of the Dispatch Agent and the consolidation of the executional layer into the Action Agent. It is preserved here as a historical record of architectural intent.

---

## Part 1: The Missing Agent Between Planning and Execution

### The Gap Analysis

**Current Chain (V1):**
```
Objective Agent → Goal Agent → Planning Agent → [?] → Perception Agent → Interpretation Agent → Action Agent
```

**The Problem:**
- Planning Agent produces rich output: full DAG with 8–11 tasks, execution groups, parallelism flags, dependency chains, derived_ref registrations, and deduplication logs
- Action Agent (and other executional agents) execute **one specific analysis task at a time** via MCP server calls
- **No agent bridges this gap**

#### Critical Runtime Questions Left Unanswered

1. **Which task runs next?**
   - Plan defines execution groups and topological order
   - No agent selects next task at runtime or checks dependency completion
   - No dispatch mechanism to executional agents

2. **What happens when a task fails?**
   - If `task_5` (audio extraction) fails, tasks 6–11 all depend on it
   - No failure handling strategy defined
   - Planning Agent produced static plan — not invoked at runtime

3. **Where do derived_refs get resolved?**
   - Plan registers `derived_1`, `derived_2` with `status: "pending"` and `storage_uri: null`
   - After `task_5` extracts audio track to Wasabi, something must update refs
   - Need: actual URI and `status: "created"` for downstream tasks

4. **Who manages parallel execution?**
   - Plan marks execution groups 3, 5, 7 as `parallel: true`
   - Nothing dispatches tasks concurrently or waits for group completion

5. **Who tracks overall workflow state?**
   - 11 task executions produce 11 JSON messages
   - No mechanism for cumulative state tracking
   - What's done? What's pending? What refs are resolved?

---

### The Solution: Dispatch Agent

#### Recommendation: Introduce `dispatch_agent`

**Position:** Operational Core layer (alongside planning_agent)

**Name Rationale:**
- ❌ Not "orchestration" — n8n layer already handles infrastructure
- ❌ Not "coordinator" — too generic, doesn't convey pattern
- ❌ Not "scheduler" — implies time-based triggering; this is dependency-based
- ✅ **"Dispatch"** — take static plan, maintain runtime state, dispatch tasks one-at-a-time (or in parallel groups)

---

#### Architecture: Where Dispatch Agent Sits

```
Planning Agent
    ↓ (produces static workflow DAG)
    
Dispatch Agent
    ↓ (manages runtime execution of the DAG)
    ├─ dispatches individual tasks to:
    │
    ├─→ Perception Agent / Action Agent
    │       ↓ (execute one task each via MCP)
    │       ↓ returns result to:
    │
    └─→ Dispatch Agent
            ↓ (updates state, resolves refs, picks next task)
            ↓ when all tasks complete:
            
Interpretation Agent
    ↓ (synthesises results)
```

---

#### Core Responsibilities

**1. Task Selection and Dispatch**
- Walk `execution_groups` in order
- For each group: check all `depends_on` tasks have `status: "success"`
- Dispatch ready tasks to appropriate executional agent
- For `parallel: true` groups: dispatch all tasks concurrently

**2. Ref Resolution**
- When task completes with output (e.g., audio at `wasabi://bucket/audio_123.wav`):
  - Update corresponding `derived_ref` from `{status: "pending", storage_uri: null}`
  - Update to `{status: "created", storage_uri: "wasabi://..."}`
- Inject resolved URI into `input_refs` of downstream tasks before dispatch

**3. Failure Handling**
- If task fails with `recoverable: true`: retry up to N times
- If permanent failure:
  - Mark all downstream dependents as `"skipped"`
  - Determine if workflow can produce partial result or must abort
  - Log failure chain in audit

**4. Workflow State Management**
- Maintain runtime state object tracking each task:
  - Current status: `pending` → `dispatched` → `success` / `failed` / `skipped`
  - Attempt counts
  - Timing data
- This state is the **single source of truth** for workflow progress

**5. Completion Signalling**
- When final execution group completes (or workflow aborts):
  - Compile cumulative results
  - Pass to Interpretation Agent with summary:
    - What succeeded
    - What failed
    - What refs were resolved

---

#### Message Format

**Outbound task assignments:** `content_type: "dispatch"`

**State updates:** `content_type: "dispatch_status"`

**Resources field:** Authoritative, up-to-date version
- Only place where `storage_uri` and `status` transition from `pending` to runtime values

---

#### Why This Must Be an Agent (Not Just n8n Logic)

The dispatch agent needs **LLM reasoning** for:

- **Failure handling decisions:** "This audio extraction failed because the video has no audio track — should I skip all audio-dependent tasks and still produce the visual analysis?"

- **Dynamic re-planning:** "Task_7 timed out; can I substitute a lighter-weight transcription model?"

- **Audit compliance:** Every dispatch decision needs governance-compliant `audit.reasoning`

These are judgment calls, not mechanical loops.

---

#### Summary Table

| Aspect | Detail |
|--------|--------|
| **Name** | `dispatch_agent` |
| **Core Layer** | Operational (alongside planning_agent) |
| **Position** | Between planning_agent and executional agents |
| **Agent Type** | `operational` |
| **Input** | Static workflow DAG from planning_agent |
| **Output** | Individual task dispatches + runtime state updates |
| **Key Capabilities** | Task selection, ref resolution, failure handling, parallel coordination, state management |

---

## Part 2: Action Agents After Dispatch Implementation

### The Foundational Question

Once dispatch_agent is implemented, **which executional agent handles which task?**

Current architecture assumes: Perception → Interpretation → Action, but the task-level DAG doesn't map cleanly to this decomposition.

**Two possible models exist:**

---

### Model A: Three Agents in Sequence per Task (Micro-Pipeline)

```
Dispatch Agent sends task
    ↓
Perception Agent
    → prepares inputs, validates readiness
    ↓
Action Agent
    → calls the MCP tool / model
    ↓
Interpretation Agent
    → parses raw output into structured result
    ↓
Result returns to Dispatch Agent
```

**Characteristics:**
- Every single task goes through three-step sub-pipeline
- Perception: "eyes and ears" — reads input refs, verifies formats, configures parameters
- Action: makes actual tool call
- Interpretation: structures raw model output for downstream consumption

**Cost Analysis:**
- 3 LLM invocations per task
- For 11-task workflow: 33 LLM calls just for execution
- Plus 3 governance/planning calls + dispatch calls
- For simple tasks (e.g., "Validate YouTube URL" = single HTTP HEAD): massive overhead

**Verdict:** Wildly excessive for most tasks

---

### Model B: One Agent per Task, Role by Task Type

```
Dispatch Agent examines task capability_id
    ↓
Routes to ONE of:
  - Perception Agent      (for CAP-PRE-*, CAP-ACQ-* tasks)
  - Action Agent          (for CAP-AUD-*, CAP-VIS-*, CAP-SPK-* tasks)
  - Interpretation Agent  (for CAP-SYN-* tasks)
```

**Characteristics:**
- Each executional agent owns a capability category
- Dispatch routes task to exactly one agent
- Agent handles full lifecycle: input preparation, tool invocation, output structuring

**Problem:** Boundaries are artificial
- What makes "extract audio track" (CAP-PRE-002, ffmpeg) a Perception task but "transcribe audio" (CAP-AUD-001, Whisper-X) an Action task?
- Both call an MCP tool on input ref and produce derived_ref
- "Perceiving" vs "acting" doesn't map cleanly

**Verdict:** Category assignment is arbitrary

---

### Recommended Solution: Rethink the Executional Agents

The original Perception → Interpretation → Action decomposition was designed **before task-level DAGs existed**. With individual tasks now well-defined (explicit `capability_ids`, `input_refs`, `output_refs`), the execution model should reflect this clarity.

**Proposal: Replace three agents with two plus one repositioned**

---

## Part 3: Revised Executional Architecture

### New Architecture Overview

```
Planning Agent
    ↓ (static DAG)
Dispatch Agent
    ↓ dispatches task-by-task
    ├── action_agent  (for tool-calling tasks)
    │       ↓ returns result
    │   Dispatch Agent (updates refs, picks next task)
    │
    └── reasoning_agent  (for synthesis tasks)
            ↓ returns synthesised insights
        Dispatch Agent (records final output)
    ↓ (all tasks complete)
Final Output to user
```

---

### Agent 1: `action_agent` (New)
**Replaces: Perception Agent + Action Agent**

#### Role
Execute a single task by calling the appropriate MCP tool and producing structured output.

#### Responsibilities

**1. Input Resolution**
- Receive task from dispatch_agent with resolved `input_refs` (URIs populated by dispatch)
- Validate all inputs exist and are in expected format
- *Absorbs the useful part of old Perception Agent*

**2. Tool Selection and Invocation**
- Map task's `capability_ids` to correct MCP server tool
- Examples:
  - `CAP-AUD-001` → Whisper-X MCP tool
  - `CAP-PRE-002` → ffmpeg audio extraction tool
  - `CAP-VIS-001` → YOLO object detection tool
- Call tool with parameters derived from task definition and input refs

**3. Output Structuring**
- Take raw tool response (model output, file path, JSON payload)
- Structure into standard message format with populated `output_refs`
- If task produces `derived_ref`:
  - Include actual `storage_uri`
  - Set `status: "created"`

**4. Error Handling**
- Capture error details on tool failure:
  - Timeout
  - Invalid input
  - Model error
- Return structured failure message for dispatch_agent to make retry/skip decisions

#### Why Merge Perception and Action?
At single-task level, "perceiving" input and "acting" on it are two halves of same operation. Separating into two LLM calls doubles cost without meaningful reasoning gain.

The action_agent is fundamentally a **tool-calling agent**:
- Understands what tool to call
- Knows how to call it correctly
- Packages result properly

---

### Agent 2: `reasoning_agent` (Kept & Repositioned)
**Current Status: Exists; Repositioned for clarity**

#### Role
Synthesise results across multiple completed tasks to produce higher-level conclusions and insights.

#### Key Difference from action_agent
- **Does NOT call MCP tools**
- **Reasons across already-produced outputs**
- Sits in Operational Core (as currently defined)
- Invoked at specific DAG points (synthesis/fusion tasks)

#### Invocation Trigger
Dispatch_agent invokes reasoning_agent when synthesis tasks appear:
- `CAP-SYN-001`: Cross-modal synthesis
- `CAP-SYN-002`: Temporal synthesis
- `CAP-SYN-003`: User-question synthesis

#### Responsibilities

**1. Cross-Modal Correlation**
- Example: "Correlate text sentiment and vocal emotion results per speaker"
- Requires LLM reasoning across `derived_4` (emotion_scores) and `derived_5` (sentiment_scores)
- Produces fused insight — not MCP tool call

**2. Timeline Reconstruction**
- Building narrative timeline from:
  - Scene understanding outputs
  - Transcription timestamps
  - Diarization maps
- Requires synthesis, not tool invocation

**3. Answering User's Actual Question**
- Example query: "At what timestamp does the main conflict begin?"
- Takes structured analysis outputs and formulates actual answer
- Execution_agent couldn't do this (only handles single-tool operations)

**4. Conflict Resolution**
- When modalities disagree (text sentiment: positive, vocal emotion: negative)
- Reasoning_agent applies judgment to:
  - Resolve contradiction
  - Explain discrepancy
  - Determine ground truth

---

### Agent 3: `interpretation_agent` (Removed)
**Disposition: Deprecated; functionality split**

#### Why Removed?

Original function: "Contextualizes results based on user queries, transforming raw perceptual data into meaningful, query-relevant interpretations"

This splits between two new agents:
- **action_agent:** Result structuring (raw model output → clean JSON with populated refs)
- **reasoning_agent:** Contextualisation (results → meaningful in context of user question)

Standalone Interpretation Agent becomes **redundant**.

---

## Part 4: Agent Responsibilities Summary

### Quick Reference Table

| Agent | Layer | Responsibility | Tool-Calling | Input | Output |
|-------|-------|-----------------|--------------|-------|--------|
| **action_agent** | Executional | Call MCP tools, validate inputs, structure results | Yes | Single task + resolved input_refs | Structured result with output_refs |
| **reasoning_agent** | Operational | Synthesise across tasks, answer user questions, resolve conflicts | No | Multiple completed task outputs | High-level insights, user-facing answers |
| **dispatch_agent** | Operational | Manage DAG execution, route tasks, resolve refs, track state | No | Static DAG from planning_agent | Task dispatches + runtime state updates |
| ~~perception_agent~~ | ~~Executional~~ | ~~Merged into action_agent~~ | — | — | — |
| ~~action_agent~~ | ~~Executional~~ | ~~Merged into action_agent~~ | — | — | — |
| ~~interpretation_agent~~ | ~~Operational~~ | ~~Split into action_agent + reasoning_agent~~ | — | — | — |

---

## Document Metadata

- **Report Date:** 2026-02-21 (Architecture Overhaul)
- **Log Analyzed:** 20260221-1.md (89KB)
- **Implemented Changes:** 3 new agents (dispatch), 2 consolidated agents (action), 1 repositioned agent (reasoning)
