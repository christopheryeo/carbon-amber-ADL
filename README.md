# Sentient Agentic AI Platform

## What This Folder Does

This folder contains a **prompt engineering architecture** that defines how multiple AI agents collaborate to process requests. It is essentially a "constitution" for an AI agent system — defining how agents should behave, communicate, and maintain accountability.

The architecture is **application-agnostic**: the same agent framework, governance policies, and LLM instructions can be used for different applications by simply changing the `application.md` file.

### How It Works

1. **Prompt Concatenation**: Files are concatenated with section markers and loaded into an LLM's context window
2. **Agent Chain**: User requests flow through agents in sequence:
   - **Objective Agent** → receives user requests, outputs strategic objectives (*what* to achieve)
   - **Goal Agent** → decomposes objectives into actionable goals (*how* to achieve them)
   - **Planning Agent** → creates execution workflows
   - **Executional Agents** → perform the actual tasks
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

```
Sentient Agentic AI/
├── context/                         ← Always loaded for ALL agents
│   ├── 01_llm_instructions.md       ← LLM execution instructions (first file loaded)
│   ├── 02_application.md            ← Application context (TEMPLATE — customize per application)
│   └── 03_governance/               ← Governance policies
│       ├── audit.md
│       ├── message_format.md
│       └── fileformat.md
├── agents/                          ← Only SPECIFIC agent loaded per invocation
│   ├── 01_governance_core/
│   │   └── objective_agent.md
│   ├── 02_operational_core/
│   └── 03_executional_core/
├── system/                          ← Runtime artifacts (NOT concatenated)
│   └── logs/
└── README.md                        ← Human documentation (NOT concatenated)
```

---

## Application Template Format

The `application.md` file follows a **standardized template format** that enables different applications to be deployed using the same agent framework. When creating a new application, replace the content in sections marked `[APPLICATION-SPECIFIC]` while keeping sections marked `[STANDARD]` unchanged.

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

1. Copy `context/02_application.md` to create your new application file
2. Keep all `[STANDARD]` sections unchanged
3. Replace all `[APPLICATION-SPECIFIC]` sections with your application's content:
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
2. **Agent files** (`agents/`) — Only the SPECIFIC agent definition is loaded per invocation

### Loading Order

The orchestration layer concatenates files into a master prompt in this order:

| Order | Path | Purpose |
|-------|------|---------|
| 1 | `context/01_llm_instructions.md` | LLM execution instructions — how to interpret the prompt |
| 2 | `context/02_application.md` | Application context — purpose, capabilities, constraints |
| 3 | `context/03_governance/*.md` | Governance policies (audit, message format, file format) |
| 4 | `agents/<core>/<agent>.md` | Specific agent definition (only the agent being invoked) |

**Note**: This README.md is **NOT concatenated** — it is for human reference only.

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
[FILE: context/01_llm_instructions.md]
================================================================================

# LLM Execution Instructions

## Description
...

================================================================================
[FILE: context/02_application.md]
================================================================================

# Application Context

## Section 1: LLM Execution Context [STANDARD]
...

================================================================================
[FILE: context/03_governance/audit.md]
================================================================================

# AI Audit and Logging Governance
...

================================================================================
[FILE: agents/01_governance_core/objective_agent.md]
================================================================================

# Objective Agent Requirements
...
```

These markers allow the LLM to locate specific files when documents reference them (e.g., "See `audit.md`").

---

## context/

Contains files that are **always loaded for ALL agents**. This provides the common context that every agent needs.

### Files:
- **01_llm_instructions.md** — Explains how the LLM should interpret the prompt, locate referenced files using markers, and format its JSON output
- **02_application.md** — Application context template defining purpose, capabilities, and constraints (customize per application)
- **03_governance/** — Governance policies folder:
  - **audit.md** — Audit logging requirements
  - **message_format.md** — JSON message format specification
  - **fileformat.md** — Markdown file format standards

---

## agents/

Contains agent definitions organized into three core layers. **Only the specific agent being invoked is loaded** (not all agents).

### Governance Core (01_governance_core/)
High-level strategic agents responsible for objective and goal setting:
- **objective_agent.md** — Receives user requests and outputs strategic objectives (*what* to achieve)
- **goal_agent/** — Decomposes objectives into actionable goals (*how* to achieve them)

### Operational Core (02_operational_core/)
Mid-level agents that manage planning, review, and memory:
- **planning_agent/** — Creates execution plans and workflows
- **reviewer_agent/** — Validates outputs and ensures quality standards
- **memory_agent/** — Manages context and historical information

### Executional Core (03_executional_core/)
Specialized agents that produce deliverables. **These agents are application-specific** — different applications define different executional agents based on their capabilities.

For the current DSTA Video Analysis application (see `application.md`):
- **perception_agent/** — Identifies speakers, analyzes expressions/gestures, detects objects, analyzes audio
- **interpretation_agent/** — Processes analysis requests and contextualizes results
- **action_agent/** — Executes analysis models and generates structured outputs

---

## system/

System-level infrastructure supporting the Agentic AI framework. **NOT concatenated into prompts.**

### Subdirectories:
- **logs/** — Daily audit logs written by the orchestration layer

---

## Governance Compliance

**CRITICAL REQUIREMENT**: All AI agents must comply with governance policies in `context/03_governance/`.

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

```
User Request
     ↓
┌─────────────────────────────────────────────────────────────────┐
│  GOVERNANCE CORE                                                │
│  ┌─────────────────┐      ┌─────────────────┐                  │
│  │ Objective Agent │ ───→ │   Goal Agent    │                  │
│  │ (what to do)    │      │ (how to do it)  │                  │
│  └─────────────────┘      └─────────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  OPERATIONAL CORE                                               │
│  ┌─────────────────┐  ┌─────────────────┐                      │
│  │ Planning Agent  │  │ Reviewer Agent  │                      │
│  │ (workflows)     │  │ (quality)       │                      │
│  └─────────────────┘  └─────────────────┘                      │
│  ┌─────────────────┐                                           │
│  │  Memory Agent   │                                           │
│  │ (context)       │                                           │
│  └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  EXECUTIONAL CORE (Application-Specific)                        │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐                  │
│  │ Perception │ │Interprettn │ │   Action   │                  │
│  │   Agent    │ │   Agent    │ │   Agent    │                  │
│  └────────────┘ └────────────┘ └────────────┘                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                      Final Output
```

---

## Orchestration Layer Responsibilities

The orchestration layer (n8n, CrewAI, or similar) handles:

| Task | Description |
|------|-------------|
| **File concatenation** | Concatenate context files + specific agent file with `[FILE: <path>]` markers |
| **Prompt delivery** | Send concatenated prompt to LLM |
| **Audit log extraction** | Extract `audit` field from JSON response and write to `system/logs/YYYYMMDD.log` |
| **Message routing** | Read `next_agent.name` from JSON response and route to next agent |
| **Session management** | Track session state across agent chain |
| **Model execution** | Execute application-specific models as directed by agents |

---

## Last Updated
January 28, 2026

---

## Change Log

| Date | Change |
|------|--------|
| 2026-01-28 | Updated Executional Core agent list to match DSTA Video Analysis application (perception, interpretation, action agents) |
