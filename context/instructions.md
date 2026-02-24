# LLM Execution Instructions

## Description

This is the **first file** the orchestration workflow must read when building a master prompt for any agent.

It defines:
1. The required file load order for prompt assembly
2. What the executing LLM must do after prompt assembly
3. What responsibilities stay with orchestration vs. the LLM

---

## Prompt Assembly Contract (Orchestration Layer)

**Purpose**: n8n (or equivalent orchestration) must follow this contract before calling the LLM.

### Step 1 — Read this file first
Always read `context/instructions.md` before all other files.

### Step 2 — Read and concatenate core files in this exact order

1. `context/application.md`
2. `context/governance/audit.md`
3. `context/governance/message_format.md`
4. `context/governance/fileformat.md`
5. All files found under `context/memory/` and its subdirectories (excluding `context/memory/staging/`). Load every `.md` file present at the time of prompt assembly. This directory is populated by the Memory Agent and may be empty initially.
6. One agent file selected for the current invocation:
   - `agent/governance/objective.md`, or
   - `agent/governance/goal.md`, or
   - `agent/operational/planning.md`, or
   - `agent/operational/dispatch.md`, or
   - `agent/operational/memory.md`, or
   - `agent/operational/reasoning.md`, or
   - `agent/executional/action.md`

> **Note**: This list defines valid agent files for prompt assembly. Agent execution order is not determined by this list — see `context/governance/message_format.md` § Agent Chain Flow for the canonical flow definition.

### Step 3 — Wrap each file with file markers

Each concatenated file must be wrapped in the marker format below:

```
================================================================================
[FILE: <relative path to file>]
================================================================================

<file content here>
```

### Step 4 — Send one merged prompt to the target LLM
Do not ask the LLM to fetch files dynamically. The orchestration layer must provide all required context in the concatenated prompt.

### Step 5 — Validate the LLM Output
After receiving the JSON response from the LLM, the orchestration layer must validate it against `schema/message_schema.json` before routing it to the next agent. This ensures structural compliance with the `message_format.md` specification. If validation fails, the orchestration layer handles the error (e.g., triggering a retry or aborting the transaction).

---

## Executing LLM Responsibilities

After receiving the concatenated prompt, the LLM must:

1. Use `context/application.md` as application scope and capability source
2. Enforce governance from:
   - `context/governance/audit.md`
   - `context/governance/message_format.md`
   - `context/governance/fileformat.md`
3. Apply institutional knowledge from any `context/memory/` files included in the prompt. These contain learned patterns, heuristics, and error prevention insights. Governance always takes precedence over memory.
4. Follow the loaded agent file as role-specific behavior
5. Output exactly one JSON message that complies with `context/governance/message_format.md`

---

## Responsibility Boundary

| Task | LLM | Orchestration Layer |
|------|-----|---------------------|
| Interpret user/agent input | ✅ | |
| Apply governance rules | ✅ | |
| Produce JSON output per `message_format.md` | ✅ | |
| Select which agent file to load for this run | | ✅ |
| Read files and concatenate master prompt | | ✅ |
| Route output to next agent | | ✅ |
| Persist logs/session state | | ✅ |
| Execute external tools, APIs, models, file IO | | ✅ |

---

## Processing Sequence (LLM Side)

1. Read `context/application.md`
2. Read all governance files
3. Read any `context/memory/` files for institutional knowledge
4. Read your agent file
5. Validate scope and constraints
6. Generate role-appropriate output
7. Format as one JSON message

---

## Version
v2.1.0

## Last Updated
February 23, 2026

## Changelog
- v2.1.0 (Feb 23, 2026): Updated the valid agent file list in Step 2 to explicitly name all 7 active agents (`dispatch.md`, `reasoning.md`, `action.md`, `planning.md`, `memory.md`, `objective.md`, `goal.md`) following the removal of deprecated placeholder files (`perception.md`, `interpretation.md`).
- v2.0.0 (Feb 10, 2026): Added memory integration instructions in prompt assembly and execution responsibilities.
