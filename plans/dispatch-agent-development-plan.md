# Dispatch Agent Development Plan

**Date:** February 23, 2026
**Status:** Draft — Pending Review
**Author:** AI Assistant + Chris Yeo

---

## 1. Overview

The Dispatch Agent is defined in `agent/operational/dispatch.md` (v1.0.0) as the runtime DAG executor — it receives the static execution plan from the Planning Agent and manages live task execution. This plan describes how to implement the Dispatch Agent as a working n8n workflow in the Agent Runtime Layer (ARL).

### What Already Exists (ADL — Agent Definition Layer)

The ADL specification in `dispatch.md` is **comprehensive** and defines:

| Area | Status | Description |
|------|--------|-------------|
| Phase A output schema | ✅ Defined | `task_dispatch` message format with resolved refs |
| Phase B output schema | ✅ Defined | `workflow_complete` message format with results summary |
| Workflow state schema | ✅ Defined | Task states, ref registry, failure log |
| 7-step dispatch process | ✅ Defined | Ingest → Update refs → Handle failures → Ready check → Select → Construct → Complete |
| Agent routing rules | ✅ Defined | CAP-ACQ/PRE/AUD/SPK/VIS/DAT → action_agent; CAP-SYN → reasoning_agent |
| Retry policy | ✅ Defined | 3 retries for recoverable errors, immediate fail for non-recoverable |
| Downstream impact assessment | ✅ Defined | Walk dependency graph forward, skip transitively blocked tasks |
| Parallel dispatch rules | ✅ Defined | Group-level parallelism, cross-group blocking |
| 4 examples | ✅ Defined | Initial dispatch, callback with ref resolution, CAP-SYN routing, failure cascade |
| 10 common mistakes | ✅ Defined | Dispatch discipline, routing errors, ref resolution gaps |

### What Needs to Be Built (ARL — Agent Runtime Layer)

| Component | Description |
|-----------|-------------|
| **n8n Dispatch Workflow** | The n8n workflow that implements the dispatch loop |
| **Workflow State Persistence** | Redis-backed state management for tracking task statuses and ref registry between invocations |
| **Callback Routing** | n8n webhook/trigger mechanism for receiving Action Agent and Reasoning Agent results |
| **LLM Prompt Assembly** | How the dispatch.md prompt is concatenated and invoked for each dispatch decision |
| **Parallel Dispatch Mechanism** | How n8n handles dispatching multiple tasks simultaneously |
| **Failure Detection & Retry** | n8n error handling nodes for retry logic and failure escalation |
| **Completion Detection** | Logic to recognise terminal workflow states and emit Phase B output |

---

## 2. Architecture Decision: LLM-Driven vs. Deterministic Dispatch

> **Key Question:** Should the Dispatch Agent use LLM inference for every dispatch decision, or should the dispatching logic be implemented as deterministic code in n8n?

### Option A: Full LLM-Driven Dispatch

Every invocation (initial + every callback) sends the full dispatch.md prompt + workflow state to an LLM. The LLM decides which tasks to dispatch, how to update state, and what to do on failures.

| Pros | Cons |
|------|------|
| True to the ADL design — the agent "thinks" at every step | Expensive: 10-15+ LLM calls per workflow |
| Can handle edge cases the code might miss | Slower: LLM latency on every dispatch decision |
| Adaptive: LLM can reason about novel failure scenarios | Risk: LLM may hallucinate dispatch decisions |
| Simple n8n workflow (just calls the LLM) | Unpredictable: same input may produce different decisions |

### Option B: Deterministic Code + LLM for Edge Cases

Implement the 7-step dispatch process as deterministic n8n/JavaScript logic. Only invoke the LLM for complex decisions (failure impact assessment, partial-input synthesis decisions).

| Pros | Cons |
|------|------|
| Fast: no LLM latency for routine dispatch | More complex n8n workflow to build |
| Predictable: same input always produces same dispatch | May not handle novel edge cases |
| Cheap: only LLM calls for genuinely complex decisions | Dual logic: code + prompt must stay in sync |
| Reliable: no hallucination risk for routine operations | |

### Option C: Hybrid — LLM Validates Code Decisions

Deterministic code makes the dispatch decision. LLM validates the decision before execution (optional validation gate).

| Pros | Cons |
|------|------|
| Best of both worlds: fast + validated | Slightly slower than pure deterministic |
| Audit trail: LLM can explain why a decision was made | Extra complexity |

> **Recommendation:** Start with **Option B** (deterministic). The dispatch logic is well-defined and algorithmic (graph traversal, state updates, retry counters). LLM inference adds latency and cost with no benefit for routine DAG execution. Reserve LLM calls for the `partial-input synthesis` decision only (Example 4 in dispatch.md — deciding whether a fusion task can proceed with reduced input).

---

## 3. Development Phases

### Phase 1: Core Dispatch Loop (MVP)

**Goal:** A working n8n workflow that can dispatch a simple linear DAG (no parallelism, no failures).

**Components:**

1. **Webhook Trigger** — Receives the Planning Agent's workflow output
2. **State Initializer** — Parses the DAG, initializes all tasks as `pending`, populates ref_registry
3. **Ready Task Scanner** — Identifies tasks whose dependencies are all `complete` and input_refs resolved
4. **Agent Router** — Routes tasks to `action_agent` or `reasoning_agent` based on capability_ids
5. **Dispatch Message Builder** — Constructs the Phase A output message per dispatch.md schema
6. **Callback Webhook** — Receives results from Action/Reasoning agents
7. **State Updater** — Updates task status, ref_registry, and failure_log on callback
8. **Completion Checker** — Detects terminal state and emits Phase B output

**Tech stack:**
- n8n workflow with JavaScript Code nodes for state logic
- Redis for workflow state persistence (keyed by `session_id`)
- n8n Webhook nodes for receiving callbacks

**Deliverables:**
- n8n workflow JSON file
- Redis state schema documentation
- End-to-end test with a 3-task linear DAG

**Estimated effort:** 2–3 days

---

### Phase 2: Parallel Dispatch & Group Management

**Goal:** Support dispatching multiple tasks within the same execution group simultaneously.

**Components:**

1. **Group-Level Dispatch** — Dispatch all `ready` tasks in the current execution group at once
2. **Group Completion Tracking** — Track when all tasks in a group reach terminal state before advancing
3. **Partial Group Failure** — Handle the case where some tasks in a group fail while others succeed

**Considerations:**
- n8n's `SplitInBatches` node or `Execute Workflow` (sub-workflow) for parallel dispatch
- Each dispatched task runs in its own callback chain
- Need to track pending callbacks per group

**Deliverables:**
- Updated n8n workflow with parallel dispatch
- Test with a 5-task DAG containing 2 parallel groups

**Estimated effort:** 1–2 days

---

### Phase 3: Failure Handling & Retry Logic

**Goal:** Implement the full failure handling policy from dispatch.md.

**Components:**

1. **Retry Mechanism** — Re-dispatch failed tasks up to `max_retries` (3) with incremented attempt counter
2. **Failure Classification** — Check `error.recoverable` from the returning agent
3. **Downstream Impact Assessment** — Walk the dependency graph forward and mark transitively blocked tasks as `skipped`
4. **Partial Execution for Synthesis** — Allow CAP-SYN tasks to proceed with reduced input when some branches fail

**Considerations:**
- Retry backoff timing is managed by n8n's built-in retry logic (or custom delay nodes)
- The `failure_log` in workflow_state must track all failure details for the Phase B output
- Need graph traversal logic (BFS/DFS from failed task node)

**Deliverables:**
- Updated n8n workflow with retry and failure handling
- Test scenarios: single task failure, cascading failure, partial synthesis

**Estimated effort:** 2–3 days

---

### Phase 4: Ref Resolution & Asset Tracking

**Goal:** Implement the ref resolution lifecycle described in dispatch.md.

**Components:**

1. **Ref Registry Management** — Maintain the ref_registry in Redis with live status updates
2. **Dispatch-Time Resolution** — Populate `input_refs_resolved` with actual storage URIs from the registry
3. **Callback-Time Updates** — Extract `output_refs_produced` from callback results and update registry
4. **Dispatch Blocking** — Prevent dispatch when any input_ref has `status: "pending"`

**Considerations:**
- The ref_registry is the critical data structure — it bridges the Planning Agent's abstract refs to actual storage URIs
- Race condition: two callbacks could arrive simultaneously for tasks in the same group, both trying to update the registry
- Use Redis transactions or Lua scripts for atomic updates

**Deliverables:**
- Updated n8n workflow with ref resolution
- Redis-backed ref_registry with atomic operations
- Test with a DAG where `derived_1` flows from task_5 to task_7

**Estimated effort:** 1–2 days

---

### Phase 5: Agent-Application Separation (dispatch.md)

**Goal:** Apply the same genericisation to `dispatch.md` as done for `planning.md`. This is item 1.3 on today's plans.

**Components:**
- Replace Wasabi URIs with `storage://...` placeholders
- Replace YouTube URLs with generic source URLs
- Replace tool names (ffmpeg, Whisper-X, yt-dlp) with generic descriptions
- Genericise the CAP-ID routing table descriptions
- Genericise failure examples

**Note:** This phase is ADL work, not ARL. It should be done alongside or before the ARL implementation to ensure the prompt the LLM sees (for edge case decisions) is clean.

**Estimated effort:** ~30 minutes (this is item 1.3 on the current plans)

---

### Phase 6: Integration & End-to-End Testing

**Goal:** Full end-to-end test with a real workflow flowing through the complete agent chain.

**Test scenarios:**

1. **Happy Path** — User request → Objective → Goal → Planning → Dispatch → Action (all tasks succeed) → Memory
2. **Single Failure with Recovery** — One task fails once, retry succeeds
3. **Cascading Failure** — Audio extraction fails, all downstream audio/text tasks skipped, visual branch continues
4. **Partial Synthesis** — Multi-modal fusion proceeds with reduced input
5. **Parallel Execution** — Multiple tasks in the same group dispatched and completed in parallel
6. **Large DAG** — 13+ task workflow with 8 execution groups (the Example 1 from planning.md)

**Estimated effort:** 2–3 days

---

## 4. Verification & Checks

### 4.1 ADL Compliance Verification

Before any ARL implementation, verify that `dispatch.md` is internally consistent and aligns with the rest of the ADL:

| Check | Method | Status |
|-------|--------|--------|
| Output schema matches `message_format.md` | Cross-reference Phase A and Phase B schemas against `message_format.md` | 🆕 |
| Agent routing covers all capabilities | Verify all CAP-IDs in `application.md` have routing rules in `dispatch.md` | 🆕 |
| Workflow state schema covers all task status transitions | Verify `pending → ready → in_progress → complete/failed/skipped` are exhaustive | 🆕 |
| Examples follow the dispatch process steps | Walk through each example step-by-step against the 7-step process | 🆕 |
| Message schema compatibility | Validate Phase A and Phase B outputs against `message_schema.json` | 🆕 |
| Interaction metadata consistency | Verify sequence_number, parent_message_id, next_agent logic against `message_format.md` | 🆕 |

### 4.2 ARL Logic Verification

For each development phase, verify the n8n workflow logic:

| Phase | Verification Method |
|-------|-------------------|
| Phase 1 | Run a 3-task linear DAG through the workflow; verify task_1 dispatches before task_2, etc. |
| Phase 2 | Run a 5-task DAG with parallel groups; verify both tasks in a group dispatch simultaneously |
| Phase 3 | Inject a failure response; verify retry, downstream skipping, and failure_log population |
| Phase 4 | Run a DAG with derived_refs; verify `input_refs_resolved` contains actual URIs |
| Phase 6 | Full chain from user request to Memory Agent; verify all intermediate states |

### 4.3 State Consistency Checks

After each callback processing:

1. **No orphan tasks**: Every task in the DAG is accounted for in workflow_state
2. **No invalid transitions**: Tasks can only move forward in the state machine (`pending → ready → in_progress → complete/failed/skipped`)
3. **Ref consistency**: Every `input_refs` reference in a dispatched task has a corresponding registry entry with a resolved URI
4. **Group integrity**: No Group N+1 task dispatched while Group N has non-terminal tasks
5. **Completion integrity**: Phase B output only emitted when all tasks are in terminal state

---

## 5. Testing Strategy

### 5.1 Unit Tests (JavaScript Logic)

Test the core dispatch logic functions in isolation:

| Test | Input | Expected Output |
|------|-------|-----------------|
| `initializeState(dag)` | A Planning Agent DAG | Workflow state with all tasks `pending` |
| `findReadyTasks(state)` | State where task_1 is `complete` | Tasks in group 2 whose deps are all `complete` |
| `routeTask(task)` | Task with `capability_ids: ["CAP-AUD-001"]` | `"action_agent"` |
| `routeTask(task)` | Task with `capability_ids: ["CAP-SYN-001"]` | `"reasoning_agent"` |
| `handleFailure(state, taskId)` | State where task_5 failed | task_7, task_8 marked `skipped` |
| `updateRefRegistry(state, callback)` | Callback with `output_refs_produced` | Registry updated with new URI |
| `checkCompletion(state)` | State where all tasks are terminal | `{ isComplete: true, status: "complete" }` |
| `checkCompletion(state)` | State with failed + skipped tasks | `{ isComplete: true, status: "partial" }` |

**Framework:** Jest or Vitest (standalone, outside n8n)
**Location:** `tests/dispatch/` directory

### 5.2 Integration Tests (n8n Workflow)

Test the n8n workflow with mock agent responses:

| Test Scenario | DAG | Mock Responses | Verified Outcome |
|--------------|-----|----------------|-----------------|
| Linear happy path | 3 tasks, sequential | All succeed | Phase B: status "complete", 3/3 completed |
| Parallel happy path | 5 tasks, 2 parallel groups | All succeed | Both parallel tasks dispatch simultaneously |
| Single retry | 3 tasks | task_2 fails once, succeeds on retry | task_2 has 2 attempts, workflow completes |
| Non-recoverable failure | 3 tasks | task_2 fails non-recoverably | task_3 skipped, Phase B: status "partial" |
| Cascading failure | 8 tasks, branching DAG | task_5 fails | Audio branch skipped, visual branch continues |
| Ref resolution | 5 tasks with derived_refs | task_3 produces derived_1 | task_4 dispatched with resolved URI |

**Framework:** n8n's built-in test runner + custom test harness
**Location:** `tests/dispatch/integration/` directory

### 5.3 Schema Validation Tests

Validate that every output message conforms to `message_schema.json`:

| Test | Action |
|------|--------|
| Phase A output | Validate dispatched task message against schema |
| Phase B output | Validate workflow_complete message against schema |
| Callback parsing | Verify the workflow correctly parses Action Agent response format |

**Framework:** AJV (JSON Schema validator) in JavaScript
**Location:** `tests/schema/` directory

### 5.4 Manual Testing Checklist

For the initial deployment:

- [ ] Deploy the n8n dispatch workflow to the staging environment
- [ ] Trigger with a sample Planning Agent output (copy from dispatch.md Example 1)
- [ ] Verify the dispatch message arrives at the Action Agent webhook
- [ ] Mock an Action Agent success response and verify callback processing
- [ ] Mock a failure response and verify retry behaviour
- [ ] Run a complete 12-task workflow from planning.md Example 1
- [ ] Verify the Phase B completion message is correctly formatted
- [ ] Verify the Memory Agent receives the completion message
- [ ] Check Redis state is cleaned up after workflow completion

---

## 6. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| LLM hallucination in edge case decisions | Medium | High | Use deterministic logic for routine dispatch; only LLM for partial-synthesis decisions |
| Redis state corruption | Low | Critical | Atomic operations via Lua scripts; state snapshots for recovery |
| Callback ordering issues | Medium | Medium | Sequence numbers on callbacks; idempotent state transitions |
| n8n workflow complexity | High | Medium | Start with Phase 1 MVP; iterate. Keep n8n workflow modular with sub-workflows. |
| Timeout on long-running tasks | Medium | Medium | Configurable `task_timeout` in dispatch config; escalation to failure handling |
| Race conditions in parallel dispatch | Medium | High | Redis-based locking for concurrent callback processing |

---

## 7. Open Questions

1. **n8n version**: Which version of n8n is currently deployed? Does it support sub-workflows and webhook-based callbacks?
2. **Redis deployment**: Is Redis already available in the infrastructure, or does it need to be provisioned?
3. **Action/Reasoning Agent readiness**: Are the Action Agent and Reasoning Agent n8n workflows already built, or do they also need development?
4. **Monitoring**: What monitoring/alerting infrastructure is available for tracking workflow progress and failures?
5. **Timeout policy**: What should the timeout be for individual task execution? (Not defined in dispatch.md)
6. **Persistence**: Should workflow state survive n8n restarts? (Redis handles this, but needs to be configured correctly)

---

## 8. Dependencies

| Dependency | Required Before |
|-----------|----------------|
| `dispatch.md` agent-app separation (item 1.3) | Phase 1 — so LLM prompts are generic |
| `action.md` ARL implementation | Phase 6 — needed as the task executor |
| `reasoning.md` ARL implementation | Phase 6 — needed for CAP-SYN tasks |
| Redis provisioning | Phase 1 — state storage |
| n8n deployment | Phase 1 — workflow engine |
| `message_schema.json` updates (item 2.6) | Phase 6 — for schema validation tests |

---

*End of Plan*
