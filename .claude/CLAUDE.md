# CLAUDE.md — Carbon Amber ADL

## Session Startup

At the beginning of every morning working session, read and follow `.claude/startup.md` before doing anything else.

## Project Overview

Carbon Amber is a **prompt engineering architecture** (Agent Definition Layer / ADL) that defines how multiple AI agents collaborate to process user requests. It is the "agent brain" that the Agent Runtime Layer (ARL) orchestration engine (n8n) dynamically loads and executes at runtime.

- **ADL provides the intelligence** — agent definitions, governance rules, application context
- **ARL provides the execution framework** — n8n orchestration, caching, routing, audit logging

The architecture is **application-agnostic**: the same agent framework supports different applications by swapping `context/application.md`. The current deployment is configured for the **DSTA Video Analysis Platform**.

## ⚠️ Core Principle: Agent Genericity

**ALL agent definitions in `agent/` MUST be fully generic and application-agnostic.** Every piece of application-specific content — capability IDs, tool names, platform names, storage URIs, domain terminology, dependency chains, I/O asset types — belongs exclusively in `context/application.md`. Agent files must never contain hardcoded references to any specific application, tool, model, platform, or domain concept. Swapping `context/application.md` must be sufficient to repurpose the entire agent framework for a different application with zero changes to agent files. See the detailed separation table under **Editing Guidelines → Architectural Principle: Agent-Application Separation** for the full boundary specification.

## Directory Structure

```
carbon-amber-ADL/
├── context/                         # Always loaded for ALL agents
│   ├── instructions.md              # Prompt assembly contract (NOT concatenated)
│   ├── application.md               # Active application context
│   ├── governance/                  # Governance policies (authoritative, cannot be overridden)
│   │   ├── audit.md                 # Audit logging requirements
│   │   ├── message_format.md        # JSON message format spec + Agent Chain Flow
│   │   └── fileformat.md            # Markdown/file format standards
│   └── memory/                      # Institutional knowledge (populated by Memory Agent)
│       ├── staging/                 # Low-confidence entries (NOT concatenated)
│       └── session_summaries/       # Daily transaction summaries
├── agent/                           # Only SPECIFIC agent loaded per invocation
│   ├── governance/                  # Layer 1: Strategic (Objective, Goal)
│   ├── operational/                 # Layer 2: Orchestration (Planning, Dispatch, Reasoning, Memory, Learning)
│   └── executional/                 # Layer 3: Application-specific (Action)
├── schema/                          # Machine-readable validation schemas
│   └── message_schema.json          # JSON Schema for agent message format
├── template/                        # Blank templates
│   └── application_template.md      # Template for new application contexts
├── application/                     # Archived application contexts
│   └── archive/
├── system/                          # Runtime artifacts (NOT concatenated)
│   └── logs/                        # Daily audit logs (YYYYMMDD.md)
├── plans/                           # Planned changes and improvements to the project
├── doc/                             # Updated project documentation on the Carbon Amber Agentic AI project (NOT concatenated)
└── temp/                            # Temporary working files
```

## Agent Chain Flow

The canonical processing sequence (defined in `context/governance/message_format.md`):

```
User Request
    |
[1] Objective Agent (governance)        → strategic objectives (the "what")
    |
[2] Goal Agent (governance)             → actionable goals (the "how")
    |
[3] Planning Agent (operational)        → execution plan (static DAG of tasks)
    |
[4] Dispatch Agent (operational)        → manages runtime DAG execution
    |
[5] Action Agent / Reasoning Agent   → task results / synthesis results
    ^___| (loop: completed tasks return to Dispatch for next dispatch)
    |
[6] Final Output → response to user
    |
[7] Memory Agent (operational, post-chain) → distills institutional knowledge
```

**Routing rules at Dispatch:**
- **Action Agent** (`action_agent`): tool-calling tasks via MCP (CAP-ACQ, CAP-PRE, CAP-AUD, CAP-SPK, CAP-VIS, CAP-DAT)
- **Reasoning Agent** (`reasoning_agent`): cross-task synthesis requiring LLM inference (CAP-SYN)
- **Memory Agent** (`memory_agent`): on workflow completion

## Key Conventions

### Prompt Concatenation Order

The orchestration layer assembles a master prompt per invocation:

1. `context/application.md`
2. `context/governance/audit.md`
3. `context/governance/message_format.md`
4. `context/governance/fileformat.md`
5. All `.md` files in `context/memory/` (excluding `staging/`)
6. `agent/<core>/<agent>.md` (only the invoked agent)

Each file is wrapped with `[FILE: <relative path>]` markers. `context/instructions.md` is read by the orchestration layer but is NOT concatenated.

### Governance Compliance

- Governance files are **authoritative** and cannot be overridden
- Every agent output must include an `audit` field with compliance notes, governance files consulted, and reasoning
- All governance file references must use **full repository-root-relative paths** (e.g., `context/governance/audit.md`, not `audit.md`)

### Agent Message Format

All agents communicate via a standardized JSON message defined in `context/governance/message_format.md`. Required top-level fields: `message_id`, `timestamp`, `agent`, `input`, `output`, `next_agent`, `status`, `error`, `metadata`, `resources`, `audit`.

### Resource Reference System

URLs and assets use a structured ref system (not raw URLs):
- `src_N` — source references (original URLs)
- `store_N` — storage references (downloaded/stored assets)
- `derived_N` — derived references (extracted audio, frames, transcripts, etc.)

The `resources` field in messages tracks all refs with their resolution status.

### Schema Validation

`schema/message_schema.json` provides JSON Schema for validating agent message output programmatically.

## Editing Guidelines

### Architectural Principle: Agent-Application Separation

**CRITICAL**: Agent definitions (`agent/`) must remain **application-agnostic**. All application-specific content belongs in `context/application.md`.

| Belongs in `application.md` | Does NOT belong in agent files |
|------------------------------|-------------------------------|
| Capability IDs (CAP-XXX-NNN) and their descriptions | Hardcoded CAP-ID lists or category-specific names |
| Tool names (ffmpeg, Whisper-X, yt-dlp, YOLO) | References to specific tools or models |
| Platform names (YouTube, Instagram, TikTok) | Platform-specific URLs or examples |
| Storage backends (Wasabi, Redis, Elasticsearch) | Hardcoded storage URIs (e.g., `wasabi://dsta-bucket/...`) |
| Dependency chains between capabilities | Application-specific task type names (e.g., "audio extraction", "transcription") |
| I/O asset types (audio_track, frame_set, transcript) | Application-domain terminology baked into rules |

**What agent files SHOULD contain:**
- Generic process descriptions (e.g., "deduplicate goals", "resolve dependencies", "dispatch tasks")
- References to `application.md` sections (e.g., "consult Section 6.9 for capability dependencies")
- Format specifications and schema definitions
- Generic examples using placeholder capability IDs and descriptions
- Error handling, validation, and governance compliance rules

**Why this matters:** The architecture is designed so that swapping `context/application.md` enables a completely different application without modifying any agent files. Application-specific content in agents breaks this portability.

### When editing agent definitions (`agent/`)

- Each agent file is self-contained with: Primary Function, Required Output, schema, process steps, interaction metadata, scope validation, examples, common mistakes, and version/changelog
- Agent files define `next_agent.name` routing — check `message_format.md` Agent Chain Flow for the canonical chain
- Increment the version number and add a changelog entry when modifying agent files
- The `sequence_number` in the Interaction section must match the agent's position in the chain

### When editing governance files (`context/governance/`)

- These are authoritative — changes affect ALL agents
- `message_format.md` is the single source of truth for message structure and agent chain flow
- Update `schema/message_schema.json` when changing the message format

### When editing application context (`context/application.md`)

- Sections marked `[STANDARD]` must not be changed (they are shared across all applications)
- Sections marked `[APPLICATION-SPECIFIC]` are customized per deployment
- The Capabilities Matrix (Section 6) is referenced by Planning Agent for task-to-capability mapping

### When creating a new application

1. Archive current `context/application.md` to `application/archive/`
2. Copy `template/application_template.md` to `context/application.md`
3. Fill in `[APPLICATION-SPECIFIC]` sections
4. Define new executional agents in `agent/executional/`
5. No changes to ARL or governance/operational agents needed

## Technology Context

- **Orchestration**: n8n (ARL)
- **Default LLMs**: Llama-3-8B, Qwen-2.5-7B (SLMs for cost optimization)
- **Escalation LLMs**: GPT-4, Claude 3.5 (only when necessary)
- **Execution**: Ollama, Groq
- **Storage**: Wasabi (S3-compatible), Redis (caching)
- **Circuit Breaker**: 10-hop limit to prevent infinite loops
- **Current Application**: DSTA Video Analysis — audio transcription, speaker sentiment/stance analysis, audience sentiment, object detection, OCR, action recognition, scene understanding
- **Supported Sources**: YouTube, Instagram, TikTok, direct uploads

## Daily Plans

When the user asks "what's the plan for today?" or similar, read the latest file in `plans/` (by date in filename, e.g., `2026-02-23-plans.md`) and present a **summary table grouped by priority**. Each priority group should be a table with columns: `#`, `Task`, `Status`, `Effort`. Use status icons: 🆕 New, 🔄 In Progress, ✅ Done. Include totals at the bottom.

### Development Auditing

When an item in the plans file is executed during a development session, its results **must be audited** at the end of execution. The findings and verification checks must be written to the appropriate accompanying audit file (e.g., `plans/2026-02-23 Audit.md`) to capture a permanent trial of development progress. This ensures transparency and prevents plan status icons from diverging from the actual codebase state. Always update the audit file with specific pass/fail outcomes against the task's objectives.

### Audit File Formatting (`plans/YYYY-MM-DD Audit.md`)

Each completed task must be recorded as an individual block in the following exact format — **do not use summary bullet lists**:

```markdown
## [Task Number] [Task name]
**Status:** ✅ Done
**Files:** `path/to/file1.md`, `path/to/file2.md`
**Objective:** One sentence describing the goal of the task.
**Verification Checklist:**
- [x] Specific thing that was verified or done
- [x] Another specific verification item
```

Rules:
- Every task gets its **own `## N.N` heading** — never group multiple tasks under one heading
- `Status` is always `✅ Done` once executed
- `Files` lists the exact files modified (use backtick-quoted relative paths)
- `Objective` is a single clear sentence of what the task accomplished
- `Verification Checklist` items start with `- [x]` (checked) and describe specific, concrete outcomes — not vague summaries
- Tasks in the same priority group share a single `## Priority N: <Name>` parent header with no sub-introduction or summary section; individual `## N.N` blocks follow directly

### Plans File Formatting (`plans/YYYY-MM-DD-plans.md`)

When a task is completed, both the **task-level icon** and the **priority section heading** must be updated:

1. **Task icon**: Change `🆕` → `✅` for each completed item (e.g., `#### 2.1 ✅ Fix memory agent ID mismatch`)
2. **Priority heading**: Append `✅ Done` to the heading once all items in a priority group are complete (e.g., `### 🔴 Priority 2: HIGH Severity Fixes ✅ Done`)
3. **Status tracking table** (where present): Update `❌` rows to `✅ Done` for items completed during the session

Both the audit file and the plans file must be updated **together** at the end of each task — never update one without the other.

### Check Repo Sync Status

When the user says "Check Repo", perform the following procedure to ensure the Git repository is synchronized:

1. **Verify No Uncommitted Changes** — Check `git status` to ensure the working directory is clean.
2. **Fetch Remote Changes** — Run `git fetch` to retrieve the latest remote state without merging.
3. **Compare Local and Remote** — Check if the local branch is ahead, behind, or has diverged from the remote tracking branch (e.g., using `git status` after fetching).
4. **Report Status** — Provide the user with a concise summary. If synced, confirm it. If not, state exactly what needs to be committed, pushed, or pulled.

## Log Files

When the user refers to "the latest log file" or "the log file", always look in `system/logs/`. Log files are named `YYYYMMDD-N.md` (e.g., `20260213-1.md`). Select the file with the most recent date; if multiple logs exist for the same day, use the highest sequence number `N`.

### Committing Log Files
Whenever you are asked to commit and push changes to the repository, you **must always** include any new or modified log files in `system/logs/` in your commit. Log files are considered an integral part of the project's history.

### Validate the Latest Log File

When the user says "Please validate the log" or asks to validate the latest log file, perform the following procedure:

1. **Locate the latest log file** — List all files in `system/logs/` and select the one with the most recent date (filename format `YYYYMMDD-N.md`). If multiple logs exist for the same day, use the highest sequence number.

2. **Execute Validation Instructions** — Read and strictly follow the analysis and formatting instructions defined in `.claude/log_validation.md`. Generate your report using the exact structure specified in that template.

## Version

Platform version: v2.3.0 (February 20, 2026)
See README.md Version History for full changelog.
