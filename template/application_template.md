# Application Context

<!--
================================================================================
APPLICATION.MD TEMPLATE STRUCTURE
================================================================================
This file follows a standardized template format. When creating a new application,
replace the content in sections marked [APPLICATION-SPECIFIC] while keeping
sections marked [STANDARD] unchanged.

Section Reference:
- [STANDARD]              = Do not modify — same for all applications
- [APPLICATION-SPECIFIC]  = Customize for each application
================================================================================
-->

---

## Section 1: LLM Execution Context [STANDARD]

**IMPORTANT**: This file (context/application.md) is concatenated into the master prompt before being sent to the executing LLM. It defines the overall purpose of the application being built.

**For the Executing LLM**: You are operating within the Sentient Agentic AI Platform. This file provides your primary orientation and application context. You must:
1. Parse and internalize the application purpose and capabilities defined below
2. Use this context when decomposing user requests into objectives (Objective Agent) and goals (Goal Agent)
3. Ensure all decomposed objectives and goals align with the Primary Objective and Capabilities stated in this document
4. Comply with all governance policies in `context/governance/`
5. Consult the appropriate agent definitions in `agent/` before executing tasks

---

## Section 2: Governance Files [STANDARD]

**CRITICAL**: All governance files in `context/governance/` are **always included** in the master prompt alongside this application context. These files govern the behaviour of ALL agents and define mandatory standards that must be followed.

### Governance Files Included in Every Prompt

| File | Purpose |
|------|---------|
| `context/governance/audit.md` | Audit logging requirements — defines how all agent actions must be logged |
| `context/governance/message_format.md` | JSON message format specification — defines the mandatory structure for all agent outputs |
| `context/governance/fileformat.md` | Markdown file format standards — defines documentation requirements |

### Governance Precedence

Governance files are **authoritative** and take precedence over any conflicting instructions:

1. **Mandatory Compliance**: Every agent must evaluate its actions against governance policies before execution
2. **No Exceptions**: Governance requirements cannot be overridden by user requests or other instructions
3. **Conflict Resolution**: If any instruction conflicts with governance policies, the governance policy prevails and the conflict must be logged
4. **Audit Trail**: All governance compliance (or non-compliance) must be recorded in the `audit` field of the JSON output

---

## Section 3: Agent Roles [STANDARD]

**For Objective Agent and Goal Agent**: This document serves as your foundational reference:
- **Objective Agent**: Receives user requests and outputs one or more strategic objectives required to fulfill the request. Objectives define *what* needs to be achieved.
- **Goal Agent**: Takes each objective from the Objective Agent and decomposes it into a series of goals and sub-goals. Goals define *how* to achieve each objective.
- All objectives and goals must align with the Capabilities defined in Section 6 and stay within the application scope.

---

## Section 4: Application Identity [APPLICATION-SPECIFIC]

<!--
GUIDANCE: Fill in the following fields with your application's details.
Replace the bracketed placeholders with actual values.
-->

| Field | Value |
|-------|-------|
| **Application Name** | [YOUR APPLICATION NAME] |
| **Customer** | [YOUR CUSTOMER NAME] |
| **Deployment** | Sentient Agentic AI Platform |
| **Version** | [VERSION NUMBER, e.g., 1.0] |
| **Last Updated** | [LAST UPDATE DATE, e.g., February 9, 2026] |

### Description

<!--
GUIDANCE: Provide a 2-3 sentence description of your application.
- What problem does it solve?
- What are the main capabilities?
- What is the scope of work?

Example structure:
"This application enables [WHAT IT DOES] through [HOW IT WORKS].
It integrates [KEY TECHNOLOGIES] and leverages [KEY FEATURES] to perform
[PRIMARY FUNCTION]. The platform is designed for [TARGET USE CASE]."
-->

[PROVIDE 2-3 SENTENCE DESCRIPTION OF YOUR APPLICATION HERE]

---

## Section 5: Primary Objective [APPLICATION-SPECIFIC]

<!--
GUIDANCE: Define the core mission of your application in 1-2 sentences.
- What transformative capability does it enable?
- What categories of outcomes does it produce?
- What is the business or operational value?

Then provide a Scope Statement that defines what IS and ISN'T included in this application.
-->

[DESCRIBE THE PRIMARY OBJECTIVE AND TRANSFORMATIVE CAPABILITY YOUR APPLICATION PROVIDES]

**Scope Statement**: [DEFINE WHAT TASKS ARE IN-SCOPE FOR THIS APPLICATION AND WHAT THE BOUNDARY CONDITIONS ARE]

---

## Section 6: Capabilities Matrix [APPLICATION-SPECIFIC]

<!--
GUIDANCE: List all the specific capabilities your application can perform.
Organize them by category (e.g., Analysis Type, Function Category, Domain).

For each capability, provide:
1. The capability name
2. A clear description of what it does
3. How it contributes to the Primary Objective

You can have as many capability categories as needed. Use a table format
similar to the example below.

Example categories:
- Audio Analysis
- Text Analysis
- Data Processing
- Reporting
- Integration Capabilities
-->

### [CAPABILITY CATEGORY 1]
| Capability | Description |
|------------|-------------|
| [CAPABILITY NAME] | [What this capability does and how it contributes to objectives] |
| [CAPABILITY NAME] | [What this capability does and how it contributes to objectives] |

### [CAPABILITY CATEGORY 2]
| Capability | Description |
|------------|-------------|
| [CAPABILITY NAME] | [What this capability does and how it contributes to objectives] |
| [CAPABILITY NAME] | [What this capability does and how it contributes to objectives] |

<!--
GUIDANCE NOTES:
- Include at least 2-3 capability categories
- Each category should contain 2-5 specific capabilities
- Capabilities should align with the Primary Objective
- Consider what outputs or insights each capability produces
-->

---

## Section 7: Supported Sources [APPLICATION-SPECIFIC]

<!--
GUIDANCE: List all the data sources or input types your application can accept.
Include:
1. Source Type (e.g., API, File Upload, Database, Real-time Stream)
2. Specific examples or formats

If your application accepts multiple input types, organize by category.
-->

| Source Type | Examples or Details |
|-------------|-------------|
| [SOURCE CATEGORY] | [Specific sources or formats] |
| [SOURCE CATEGORY] | [Specific sources or formats] |

---

## Section 8: Technology Stack [APPLICATION-SPECIFIC]

<!--
GUIDANCE: List the key technologies, tools, and platforms used by your application.
Organize by category (Orchestration, LLMs, Databases, APIs, etc.).

For agents: This helps them understand what tools and integrations are available.
This information is important when planning how to execute requests.
-->

| Category | Technologies |
|----------|--------------|
| [CATEGORY, e.g., Orchestration] | [Technology 1, Technology 2, ...] |
| [CATEGORY, e.g., Databases] | [Technology 1, Technology 2, ...] |
| [CATEGORY, e.g., APIs/Integrations] | [Technology 1, Technology 2, ...] |
| [CATEGORY, e.g., Specialized Services] | [Technology 1, Technology 2, ...] |

<!--
GUIDANCE NOTES:
- Include technologies relevant to agents (orchestration, LLMs, databases)
- Include external APIs or services your application integrates with
- Include any specialized or custom-built models/services
- Typical categories: Orchestration, LLMs, Transcription/NLP, Databases,
  Data Collection, File Storage, Specialized Models, Monitoring
-->

---

## Section 9: Agent Architecture [STANDARD WITH APPLICATION-SPECIFIC EXECUTIONAL AGENTS]

The platform utilizes a three-core agent architecture. User requests should be decomposed with awareness of which agents will execute the work:

### Governance Core [STANDARD]
| Agent | Role |
|-------|------|
| Objective Agent | Receives user requests and outputs strategic objectives (*what* needs to be achieved) |
| Goal Agent | Decomposes objectives into goals and sub-goals (*how* to achieve them) |

### Operational Core [STANDARD]
| Agent | Role |
|-------|------|
| Planning Agent | Generates deterministic execution DAGs from decomposed goals |
| Dispatch Agent | Manages runtime DAG execution and task dispatching |
| Reasoning Agent | Synthesizes insights from completed executional tasks |
| Memory Agent | Captures audit log data, distills institutional knowledge, and files it in `context/memory/` for inclusion in the master prompt |

### Executional Core [APPLICATION-SPECIFIC]

<!--
GUIDANCE: Define the application-specific agents that will execute tasks.
These are agents unique to your application domain.

For each agent, provide:
1. Agent name
2. Role (what does it do?)
3. Key responsibilities or specialized functions

Typical agents in an analysis application:
- Action Agent: Unified tool-invocation agent via MCP
- Reporting Agent: Formatting and presenting outputs

Customize these based on your application's needs.
-->

| Agent | Role |
|-------|------|
| [AGENT NAME] | [Description of what this agent does and its key responsibilities] |
| [AGENT NAME] | [Description of what this agent does and its key responsibilities] |
| [AGENT NAME] | [Description of what this agent does and its key responsibilities] |

---

## Section 10: Execution Process [STANDARD]

When processing requests, the system follows this execution flow:

1. **Receive Request**: User submits a request
2. **Define Objectives (Objective Agent)**: Translate the user request into strategic objectives (*what* needs to be achieved)
3. **Decompose into Goals (Goal Agent)**: For each objective, create goals and sub-goals (*how* to achieve it)
4. **Plan Execution (Planning Agent)**: Generate a deterministic execution DAG from the goals
5. **Dispatch Tasks (Dispatch Agent)**: Manage runtime DAG execution and route tasks
6. **Execute Tasks (Action Agent)**: Run capability-specific tool-calling tasks via MCP
7. **Synthesize Results (Reasoning Agent)**: Synthesize insights from multiple task outputs
8. **Memory Capture (Memory Agent)**: Distill operational history into context files

---

## Section 11: Constraints and Boundaries [APPLICATION-SPECIFIC]

<!--
GUIDANCE: Define operational constraints and boundaries for your application.

Create two sections:
1. A "Constraint" table listing all limits, rules, and requirements
2. Two subsections listing "In-Scope Requests" and "Out-of-Scope Requests"

For the constraint table, consider:
- Scope boundaries (what domain of work?)
- Compliance and governance requirements
- Data handling and privacy rules
- Technical limitations
- Performance or quality requirements
- Security constraints
-->

When decomposing requests, agents must observe these boundaries:

| Constraint | Description |
|------------|-------------|
| **Scope** | [Define the primary domain/scope of your application] |
| **Compliance** | [Any regulatory, privacy, or governance requirements] |
| **Data Handling** | [How should data be processed, stored, or secured?] |
| **[CONSTRAINT NAME]** | [Add additional constraints as needed] |

### In-Scope Requests

<!--
GUIDANCE: List 4-6 examples of requests that ARE appropriate for this application.
Focus on requests that clearly map to your Capabilities and Primary Objective.
-->

- [Example of an in-scope request that aligns with Primary Objective]
- [Example of an in-scope request that leverages multiple capabilities]
- [Example of an in-scope request]
- [Example of an in-scope request]

### Out-of-Scope Requests (Reject or Escalate)

<!--
GUIDANCE: List 4-6 examples of requests that are NOT appropriate for this application.
These should represent common requests that fall outside the scope or boundaries.
-->

- [Example of an out-of-scope request outside the primary domain]
- [Example of a request that violates constraints or compliance requirements]
- [Example of an out-of-scope request]
- [Example of an out-of-scope request]

---

<!--
TEMPLATE METADATA
Last Updated: February 23, 2026
Version: v2.0.0
Template Purpose: Blank template for creating new application context files in the Sentient Agentic AI Platform
-->
