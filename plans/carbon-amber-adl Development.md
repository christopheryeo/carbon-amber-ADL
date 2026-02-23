# Carbon Amber ADL Analysis Report
**Date:** 2026-02-21  
**Subject:** Log File Validation & Agent Architecture Review

---

## Part 1: Log File Validation Analysis

### Objective
Analyse the first log file for today (20260221-1.md) and determine whether it validates the behaviour of the agents in the carbon-amber-ADL and whether agents are performing as expected.

---

### Log File Summary

The log contains **12 agent executions** across **4 distinct user requests**, all processing the same YouTube video (`pcaYkGY996o`). Each request flows through the three-agent governance/operational chain:

```
objective_agent → goal_agent → planning_agent
```

No executional core agents (perception, interpretation, action) appear in this log.

#### The 4 User Requests

| # | Request Type | Time Range | Description |
|---|---|---|---|
| 1 | Comprehensive analysis | 1:04:04–1:04:23 | Transcribe, identify speakers, analyze sentiment |
| 2 | Re-run analysis | 1:04:52 | Same request, second planning run with different decomposition |
| 3 | Conflict analysis | 1:05:56–1:06:33 | Visual/narrative turning point identification |
| 4 | Speaker analysis | 1:07:06–1:08:08 | Audio-focused sentiment and tone analysis |

---

### Compliance Assessment

#### ✅ What's Working Well

**Message Format Compliance (Strong)**
- Every single message contains all required fields from `message_format.md`
- Required fields: `message_id`, `timestamp`, `agent`, `input`, `output`, `next_agent`, `status`, `error`, `metadata`, `resources`, `audit`
- No fields are missing across the entire log

**Ref ID Handling (Consistently Correct)**
- Objective Agent properly extracts URLs and assigns `src_1` / `store_1` refs
- YouTube platform detection is accurate
- First-Mention Provenance Rule is applied correctly:
  - First analysis objective references `store_1 (acquired from src_1)`
  - Subsequent objectives use bare ref
- Goal Agent correctly propagates this within each objective's goal list

**Audit Fields (Thorough)**
- All `governance_files_consulted` arrays use repository-root-relative paths
  - Correct: `context/governance/audit.md`
  - Incorrect: `audit.md` (not used)
- Pattern fix from v1.4.0 (Feb 20) properly implemented
- `reasoning` field provides genuine chain-of-thought with pattern references (Pattern A, B, C) and CAP-ID mappings
- `compliance_notes` confirm scope validation against `application.md`

**Acquisition-First Pattern (Consistently Applied)**
- Every request generates Objective 1 as video acquisition before any analysis objectives
- Goal Agent decomposes acquisition using standard pattern:
  - URL validation → download → verify → metadata

**Planning Agent Deduplication (Works Well)**
- Request 1 correctly identified 6 duplicate goals across 4 objectives
- Merged: audio extraction, language detection, transcription, diarization
- Deduplication logs are detailed with rationale

**Agent Chain Routing (Correct)**
- Routing sequence: `objective_agent → goal_agent → planning_agent → perception_agent`
- `next_agent` field and reasoning are consistent throughout

---

#### ⚠️ Issues Identified

##### Issue 1: Duplicate Planning Execution for Request 1
**Severity: Medium**

- Two separate `planning_agent` runs for the same user request
- First run (1:04:23): processes 4 objectives/18 goals → produces 11 tasks
- Second run (1:04:52): processes different inputs:
  - 3 objectives/13 goals → produces 9 tasks
  - Goal Agent input materially different between runs
  - First run: Objectives 2/3/4
  - Second run: Consolidated Objective 2/3
- Neither planning run references the other

**Investigation needed:** Was orchestration layer invoked twice with different intermediate results, or was there a retry with modified Goal Agent output?

**Expected behavior:** Single user request should produce one deterministic chain execution

---

##### Issue 2: Inconsistent `parent_message_id` Values
**Severity: Medium**

- Planning Agent (1:04:23, sequence 3) references `parent_message_id: "msg-goal-20260221-123453"`
  - Timestamp doesn't correspond to any goal_agent execution in this log
  - Goal Agent actually ran at 13:04:19, not 12:34:53
- Second planning run (1:04:52) references `"msg-goal-20260221-100000"`
- Planning run (1:06:33) references same `"msg-goal-20260221-100000"`
  - This timestamp doesn't appear in the log

**Root cause:** Audit reasoning mentions "parent_message_id constructed deterministically from Goal Agent's timestamp" but constructed values don't match actual executions

**Impact:** Breaks audit trail traceability; `metadata.parent_message_id` integrity compromised

---

##### Issue 3: `message_id` Prefix Mismatch for Planning Agent
**Severity: Low**

- Planning Agent uses prefix `msg-goal-` instead of `msg-plan-` in `message_id`

**Examples:**
- `msg-goal-20260221-130423`
- `msg-goal-20260221-130452`
- `msg-goal-20260221-130633`
- `msg-goal-20260221-130808`

**Impact:** Creates confusion; hinders debugging since message_id should uniquely identify producing agent

---

##### Issue 4: Stale Input Warning Inconsistency
**Severity: Low**

- Final planning_agent run (1:08:08) flags `STALE_INPUT_WARNING`
- Claims Goal Agent timestamp was `2026-02-21T10:00:00+08:00` ("5 seconds before Planning Agent execution")
- **Actual timing:** Planning Agent ran at 13:08:08, which is 3+ hours after 10:00:00, not 5 seconds

**Root cause:** Goal Agent timestamp passed to planner appears to be a synthetic value rather than actual execution timestamp

**Other instances:** Similar discrepancies in other planning runs, all referencing same `10:00:00` placeholder timestamp

---

##### Issue 5: Objective Over-Splitting in Request 4
**Severity: Low (Debatable)**

- User query: speaker sentiment and tone analysis
- Objective Agent generated 3 objectives:
  1. Acquire video
  2. Determine speaker sentiment
  3. Analyze speaker emotional tone
- Audit reasoning: separated because they are "distinct analysis types per Pre-Output Self-Check 2"

**Problem:** Sentiment and emotional tone are closely related; user made a single analytical ask

**Result:** Goal Agent generated near-duplicate goal lists for Objectives 2 and 3, creating unnecessary complexity

**Later correction:** Third planning run (1:08:33) sensibly consolidated these into single objective with 7 goals, suggesting a later retry corrected this

**Recommendation:** Monitor objective over-splitting pattern; consider consolidation rules

---

### Verdict

**Overall Assessment: Agents are largely performing as expected**

**Strengths:**
- Core governance chain follows defined patterns faithfully (objective decomposition, goal generation, execution planning)
- Message format compliance is strong
- Ref ID handling is accurate
- Audit documentation is thorough
- Capability mapping is solid

**Critical Concerns:**
- **Parent message ID integrity** (Issue 2) undermines the audit trail
- **Duplicate planning execution** (Issue 1) warrants orchestration layer investigation

**Root Cause Hypothesis:** Both critical issues are likely orchestration-layer or timestamp-construction bugs rather than LLM reasoning failures

---

## Part 2: The Missing Agent Between Planning and Execution

### The Gap Analysis

**Current Chain:**
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

## Part 3: Action Agents After Dispatch Implementation

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

## Part 4: Revised Executional Architecture

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

## Part 5: Agent Responsibilities Summary

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

### Responsibility Mapping by Task Capability

| Capability ID Range | Task Type | Responsible Agent | Example |
|---|---|---|---|
| CAP-ACQ-\* | Video acquisition | action_agent | Download video from YouTube |
| CAP-PRE-\* | Preprocessing | action_agent | Extract audio, convert format |
| CAP-AUD-\* | Audio analysis | action_agent | Transcribe, diarize, detect language |
| CAP-VIS-\* | Visual analysis | action_agent | Detect scenes, identify objects, extract frames |
| CAP-SPK-\* | Speaker analysis | action_agent | Identify speakers, extract embeddings |
| CAP-DAT-\* | Data structuring | action_agent | Format outputs, validate structures |
| CAP-SYN-\* | Synthesis/reasoning | reasoning_agent | Correlate modalities, build timelines, answer questions |

---

## Appendix: Implementation Priorities

### Phase 1: Foundation (Critical)
- [ ] Implement `dispatch_agent` with task selection and ref resolution
- [ ] Create `action_agent` by consolidating perception + action responsibilities
- [ ] Test single-task execution pipeline with dispatch → execution → dispatch flow

### Phase 2: Operational (High)
- [ ] Extend dispatch_agent with failure handling and retry logic
- [ ] Implement parallel execution group coordination
- [ ] Add comprehensive workflow state tracking

### Phase 3: Refinement (Medium)
- [ ] Reposition reasoning_agent for synthesis task invocation
- [ ] Deprecate perception_agent and action_agent
- [ ] Retire interpretation_agent; migrate responsibilities

### Phase 4: Validation (Ongoing)
- [ ] Test with multi-task workflows (11+ tasks)
- [ ] Validate ref resolution across deep dependency chains
- [ ] Monitor dispatch_agent decision logs for audit compliance

---

## Document Metadata

- **Report Date:** 2026-02-21
- **Log Analyzed:** 20260221-1.md (89KB)
- **Agents Evaluated:** objective_agent, goal_agent, planning_agent
- **Recommended Changes:** 3 new agents (dispatch), 2 consolidated agents (execution), 1 repositioned agent (reasoning)
- **Critical Issues:** 2 (parent_message_id integrity, duplicate planning execution)
- **Medium Issues:** 2
- **Low Issues:** 2

---

*End of Report*
