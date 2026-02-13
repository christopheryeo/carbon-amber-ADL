# Sentient Agentic AI Platform

## What This Folder Does

This folder contains a **prompt engineering architecture** that defines how multiple AI agents collaborate to process requests. It is essentially a "constitution" for an AI agent system — defining how agents should behave, communicate, and maintain accountability.

The architecture is **application-agnostic**: the same agent framework, governance policies, and LLM instructions can be used for different applications by changing `context/application.md` and archiving prior versions under `application/archive/`.

### How It Works

1. **Prompt Concatenation**: Files are concatenated with section markers and loaded into an LLM's context window
2. **Agent Chain**: User requests flow through agents in sequence, with each agent routing to the next via the `next_agent` field. The canonical agent chain flow is defined in `context/governance/message_format.md` § Agent Chain Flow.
3. **Standardized Communication**: Agents communicate using a defined JSON message format with session tracking, error handling, and audit metadata
4. **Governance Enforcement**: All operations must comply with policies in the governance folder

### Key Design Features

- **Application-agnostic architecture** — same framework supports different applications
- **Hierarchical agent architecture** separating strategy from execution
- **Mandatory governance compliance** — all AI agents must evaluate instructions against governance policies before execution
- **Comprehensive audit logging** — every action is logged with timestamps, compliance notes, and references
- **Structured message passing** between agents with metadata for traceability

---

## Directory Structure

### Canonical core context filenames

The primary context and template documents use the following canonical filenames:

- `instructions.md` (path: `context/instructions.md`)
- `application.md` (path: `context/application.md`)
- `application_template.md` (path: `template/application_template.md`)

All cross-references should point to these canonical filenames and paths.

```
Sentient Agentic AI/
├── context/                         ← Always loaded for ALL agents
│   ├── instructions.md              ← Prompt assembly contract (read by orchestration only, NOT concatenated)
│   ├── application.md               ← Application context (active application)
│   ├── governance/                  ← Governance policies
│   │   ├── audit.md
│   │   ├── message_format.md
│   │   └── fileformat.md
│   └── memory/                      ← Institutional knowledge (populated by Memory Agent)
│       ├── domain_knowledge.md      ← Accumulated domain insights
│       ├── operational_heuristics.md ← Learned process optimizations
│       ├── error_prevention.md      ← Known failure modes and mitigations
│       ├── quality_benchmarks.md    ← Historical quality baselines
│       ├── staging/                 ← Low-confidence entries (NOT concatenated)
│       └── session_summaries/       ← Daily transaction summaries
│           └── {YYYY-MM-DD}_summary.md
├── agent/                           ← Only SPECIFIC agent loaded per invocation
│   ├── governance/
│   │   ├── objective.md
│   │   └── goal.md
│   ├── operational/
│   │   ├── planning.md
│   │   ├── reasoning.md
│   │   ├── learning.md
│   │   └── memory.md
│   └── executional/
│       ├── perception.md
│       ├── interpretation.md
│       └── action.md
├── schema/                          ← Machine-readable validation schemas
│   └── message_schema.json
├── template/                        ← Blank templates for creating new content
│   └── application_template.md
├── application/                     ← Archived application contexts
│   └── archive/
├── system/                          ← Runtime directories (NOT concatenated)
│   └── logs/                        ← Daily audit log files (YYYYMMDD.log)
├── doc/                             ← Human documentation (NOT concatenated)
│   └── Carbon Amber Agentic AI Platform.docx
└── README.md                        ← Human documentation (NOT concatenated)
```

---

## Application Template Format

The `context/application.md` file follows a **standardized template format** that enables different applications to be deployed using the same agent framework. When creating a new application, replace the content in sections marked `[APPLICATION-SPECIFIC]` while keeping sections marked `[STANDARD]` unchanged.

### Template Sections

| Section | Tag | Description |
|---------|-----|-------------|
| **Section 1: LLM Execution Context** | `[STANDARD]` | Instructions for the LLM on how to use the application context |
| **Section 2: Governance Files** | `[STANDARD]` | Declares that governance files are always included and authoritative |
| **Section 3: Agent Roles** | `[STANDARD]` | Defines how Objective and Goal Agents use this document |
| **Section 4: Application Identity** | `[APPLICATION-SPECIFIC]` | Application name, customer, version, description |
| **Section 5: Primary Objective** | `[APPLICATION-SPECIFIC]` | What the application does and its scope statement |
| **Section 6: Capabilities Matrix** | `[APPLICATION-SPECIFIC]` | What the system can do — agents map requests to these capabilities |
| **Section 7: Supported Sources** | `[APPLICATION-SPECIFIC]` | What inputs the system accepts |
| **Section 8: Technology Stack** | `[APPLICATION-SPECIFIC]` | What tools/models are available for agents to use |
| **Section 9: Agent Architecture** | `[MIXED]` | Governance and Operational cores are standard; Executional core is application-specific |
| **Section 10: Execution Process** | `[STANDARD]` | The 9-step workflow for processing requests |
| **Section 11: Constraints and Boundaries** | `[APPLICATION-SPECIFIC]` | Scope limits, privacy rules, in-scope/out-of-scope definitions |

### Creating a New Application

To deploy a new application:

1. Copy `template/application_template.md` to `context/application.md` (move the previous one into `application/archive/`)
2. Keep all `[STANDARD]` sections unchanged
3. Replace all `[APPLICATION-SPECIFIC]` placeholder sections with your application's content:
   - **Section 4**: Set your application name, customer, and description
   - **Section 5**: Define the primary objective and scope statement
   - **Section 6**: List all capabilities the application supports
   - **Section 7**: Specify supported input sources
   - **Section 8**: List the technology stack
   - **Section 9**: Define Executional Core agents (keep Governance and Operational cores standard)
   - **Section 11**: Define constraints, in-scope requests, and out-of-scope requests
4. Deploy using the same n8n workflow — no other changes required

### Example Applications

| Application | Primary Objective | Key Capabilities |
|-------------|-------------------|------------------|
| **DSTA Video Analysis** | Video content analysis and insight generation | Transcription, sentiment analysis, object detection, scene understanding |
| **Document Processing** | Document classification and extraction | OCR, entity extraction, summarization, classification |
| **Social Media Monitoring** | Social media trend analysis | Sentiment tracking, topic extraction, influencer identification |

---

## Prompt Concatenation

### Concatenation Strategy

The folder is organized to support two loading patterns:

1. **Context files** (`context/`) — Always loaded for ALL agent invocations
2. **Agent files** (`agent/`) — Only the SPECIFIC agent definition is loaded per invocation

### Loading Order

The orchestration layer reads `context/instructions.md` first to determine the assembly contract, then concatenates the following files into a master prompt in this order:

| Order | Path | Purpose |
|-------|------|---------|
| 1 | `context/application.md` | Application context — purpose, capabilities, constraints |
| 2 | `context/governance/audit.md` | Audit governance requirements |
| 3 | `context/governance/message_format.md` | JSON output format requirements |
| 4 | `context/governance/fileformat.md` | Markdown and file formatting standards |
| 5 | All `.md` files in `context/memory/` and subdirectories (excluding `context/memory/staging/`) | Institutional knowledge — learned patterns, heuristics, error prevention. May be empty initially. |
| 6 | `agent/<core>/<agent>.md` | Specific agent definition (only the agent being invoked) |

**Note**: `context/instructions.md` is read by the orchestration layer to determine which files to load but is **NOT** itself concatenated into the master prompt. This README.md is also **NOT concatenated** — it is for human reference only.

### File Markers (Orchestration Layer Requirement)

When concatenating files, the orchestration layer **MUST** wrap each file with markers:

```
================================================================================
[FILE: <relative path to file>]
================================================================================

<file content>
```

**Example concatenated output:**

```
================================================================================
[FILE: context/application.md]
================================================================================

# Application Context

## Section 1: LLM Execution Context [STANDARD]
...

================================================================================
[FILE: context/governance/audit.md]
================================================================================

# AI Audit and Logging Governance
...

================================================================================
[FILE: context/memory/error_prevention.md]
================================================================================

# Error Prevention

[LAST_UPDATED]: 2025-06-01T12:00:00Z
...

================================================================================
[FILE: agent/governance/objective.md]
================================================================================

# Objective Agent Requirements
...
```

These markers ensure each source is explicitly identifiable inside the merged master prompt.

### Executing LLM Responsibilities

After receiving the concatenated prompt, the LLM should:

1. Use `context/application.md` as the authoritative application scope and capability source
2. Enforce governance from:
   - `context/governance/audit.md`
   - `context/governance/message_format.md`
   - `context/governance/fileformat.md`
3. Apply institutional knowledge from any `context/memory/` files included in the prompt. These contain learned patterns, heuristics, and error prevention insights. Governance always takes precedence over memory.
4. Follow the selected agent file as role-specific behavior
5. Return exactly one JSON message compliant with `context/governance/message_format.md`

### Responsibility Boundary

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

### Processing Sequence (LLM Side)

1. Read `context/application.md`
2. Read all governance files
3. Read any `context/memory/` files for institutional knowledge
4. Read the loaded agent file
5. Validate scope and constraints
6. Generate role-appropriate output
7. Format as one JSON message

---

## context/

Contains files that are **always loaded for ALL agents**. This provides the common context that every agent needs.

### Files:
- **instructions.md** — Prompt assembly contract read by the orchestration layer to determine which files to load. Not concatenated into the master prompt itself.
- **application.md** — Active application context defining purpose, capabilities, and constraints (archive prior versions under `application/archive/`)
- **governance/** — Governance policies folder:
  - **audit.md** — Audit logging requirements
  - **message_format.md** — JSON message format specification
  - **fileformat.md** — Markdown file format standards
- **memory/** — Institutional knowledge folder populated by the Memory Agent. All `.md` files here (excluding `staging/`) are concatenated into the master prompt. Contains:
  - **domain_knowledge.md** — Accumulated domain insights from repeated processing
  - **operational_heuristics.md** — Learned shortcuts and process optimizations
  - **error_prevention.md** — Known failure modes and their mitigations
  - **quality_benchmarks.md** — Historical quality baselines from Reviewer Agent assessments
  - **staging/** — Low-confidence knowledge entries awaiting confirmation (NOT concatenated)
  - **session_summaries/** — Daily transaction summaries

---

## agent/

Contains agent definitions organized into three core layers. **Only the specific agent being invoked is loaded** (not all agents).

### Governance Core (governance/)
High-level strategic agents responsible for objective and goal setting:
- **objective.md** — Receives user requests and outputs strategic objectives (*what* to achieve)
- **goal.md** — Decomposes objectives into actionable goals (*how* to achieve them)

### Operational Core (operational/)
Mid-level agents that manage planning, reasoning, learning, and memory:
- **planning.md** — Devises strategies for complex tasks involving multiple data points or agents
- **reasoning.md** — Makes logical inferences from analyzed elements
- **learning.md** — Adapts analysis models based on feedback and new data patterns
- **memory.md** — Captures audit log data, distills it into institutional knowledge (patterns, decision history, error prevention, quality benchmarks), and files it in `context/memory/` for inclusion in the master prompt

### Executional Core (executional/)
Specialized agents that produce deliverables. **These agents are application-specific** — different applications define different executional agents based on their capabilities.

For the current DSTA Video Analysis application (see `context/application.md`):
- **perception.md** — Identifies speakers, analyzes expressions/gestures, detects objects, analyzes audio
- **interpretation.md** — Processes analysis requests and contextualizes results
- **action.md** — Executes analysis models and generates structured outputs

---


## Governance Compliance

**CRITICAL REQUIREMENT**: All AI agents must comply with governance policies in `context/governance/`.

### Compliance Mandate
- Every instruction must be evaluated against governance policies
- Conflicting instructions must be rejected and escalated
- Governance files are authoritative — they override conflicting directives
- Non-compliance is prohibited

### Current Governance Standards

| File | Purpose |
|------|---------|
| `audit.md` | Audit logging requirements — LLM includes `audit` field in JSON output; orchestration layer writes to log files |
| `message_format.md` | JSON message format specification with all required fields |
| `fileformat.md` | Markdown file format and structure standards |

---

## Agent Communication Flow

The agent chain flow — including the full execution sequence and routing — is defined in `context/governance/message_format.md` § Agent Chain Flow. That file is the single source of truth for agent ordering. Each agent determines its successor at runtime via the `next_agent` field in its JSON output.

---

## Orchestration Layer Responsibilities

The orchestration layer (n8n) handles:

| Task | Description |
|------|-------------|
| **File concatenation** | Concatenate context files + specific agent file with `[FILE: <path>]` markers |
| **Prompt delivery** | Send concatenated prompt to LLM |
| **Audit log extraction** | Extract `audit` field from JSON response and write to the orchestration layer's configured audit log location |
| **Message routing** | Read `next_agent.name` from JSON response and route to next agent |
| **Session management** | Track session state across agent chain |
| **Model execution** | Execute application-specific models as directed by agents |

---

## Last Updated
February 10, 2026

---

## Version History

| Version | Date       | Description                                                                              |
|:--------|:-----------|:-----------------------------------------------------------------------------------------|
| v2.1.0  | 2026-02-13 | Added Memory Agent definition (`agent/operational/memory.md`) and `context/memory/`      |
|         |            | directory for institutional knowledge. Updated loading order and LLM responsibilities    |
|         |            | to include memory files. Clarified that `instructions.md` is not concatenated into       |
|         |            | master prompt. Removed CrewAI references — n8n is the sole orchestration engine.         |
| v2.0.0  | 2026-02-10 | Updated `context/instructions.md` documentation with explicit file loading contract,     |
|         |            | executing LLM responsibilities, responsibility boundary, and processing sequence         |
| v1.1.0  | 2026-02-09 | Added placeholder agent files, template/, schema/, system/logs/; refactored audit.md;    |
|         |            | added version numbers to governance files; standardized agent file naming                |
| v1.0.0  | 2026-01-29 | Initial Release: Sentient Agentic AI Platform                                            |
| v0.9.0  | 2026-01-28 | Updated Executional Core agent list to match DSTA Video Analysis application             |
|         |            | (perception, interpretation, action agents)                                              |

---

## schema/

Machine-readable validation schemas for programmatic enforcement of platform standards.

- **message_schema.json** — JSON Schema for validating the agent message format defined in `context/governance/message_format.md`. Can be used by the orchestration layer to validate every agent output automatically.

---

## template/

Blank templates for creating new content. These are starting points — copy and customize.

- **template/application_template.md** — Blank application context template with all `[STANDARD]` sections pre-filled and `[APPLICATION-SPECIFIC]` sections containing placeholder guidance. Copy to `context/application.md` when deploying a new application.

---

## system/

Runtime directories used by the orchestration layer. Not concatenated into prompts.

- **logs/** — Daily audit log files in `YYYYMMDD.log` format, written by the orchestration layer per `context/governance/audit.md`

---

## application/

Archived application context files. Move older versions of `context/application.md` here as new applications are deployed.

---

## doc/

Human documentation and reference material. Not concatenated into prompts.

- **Carbon Amber Agentic AI Platform.docx** — Platform documentation

---
