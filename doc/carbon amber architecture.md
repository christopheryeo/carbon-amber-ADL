# Carbon Amber

*Architecture and Design Documentation*

## Overview

Carbon Amber is a prompt engineering architecture that defines how multiple AI agents collaborate to process user requests. It functions as an intelligent, modular system where agents work together in a coordinated chain to transform user inputs into actionable outputs.

Carbon Amber serves as the **"agent brain"** that the Agent Runtime Layer (ARL) orchestration engine dynamically loads and executes at runtime. Together, they form a complete AI agent system:

- **ADL provides the intelligence** — agent definitions, governance rules, and application context
- **ARL provides the execution framework** — n8n orchestration, caching, routing, and audit logging

---

## Architecture and Runtime Relationship

### ADL: The Agent Definitions

ADL provides the **"what to execute"** — the actual agent logic that defines intelligent behavior:

- Strategic agent definitions (Objective Agent, Goal Agent)
- Operational agent definitions (Planning, Review, Memory)
- Executional agent definitions (application-specific tasks)
- Governance policies (audit logging, messaging standards, file formats)
- Application context (capabilities, scope, constraints)

### ARL: The Orchestration Engine

The ARL workflow provides the **"how to execute"** — the n8n infrastructure that:

- Dynamically loads agent definitions from storage (via S3/Redis)
- Concatenates files into prompts for LLM execution
- Routes messages between agents based on their outputs
- Manages sessions, caching, and comprehensive audit logging
- Implements polymorphic execution (one workflow, many agent types)

### Integration Point

When ARL receives a target_agent_id parameter (e.g., "agent_objective"), it:

1. Retrieves the corresponding agent file from Carbon Amber
2. Concatenates it with context files
3. Sends the combined prompt to a Small Language Model (SLM)
4. Routes the output to the next agent based on the response

---

## Architectural Philosophy

### Application-Agnostic Design

Carbon Amber separates universal agent logic from application-specific configuration, enabling one framework to support multiple use cases:

**Universal Components** (same across all applications):

- LLM execution instructions
- Governance policies (audit, messaging, file formats)
- Governance Core agents (Objective, Goal)
- Operational Core agents (Planning, Review, Memory)

**Application-Specific Components** (customized per use case):

- application.md file defining purpose, capabilities, and constraints
- Executional Core agents for specialized tasks

**Current Configuration:** DSTA Video Analysis Platform — analyzing video content from social media sources (YouTube, Instagram, TikTok).

### Prompt Concatenation Strategy

ARL implements a specific file loading pattern that creates a unified prompt for LLM execution:

**Loading Order:**

1. `01_llm_instructions.md` — How to interpret the prompt
2. `02_application.md` — Application purpose and capabilities
3. `03_governance/*.md` — Governance policies
4. `agents/<core>/<agent>.md` — Specific agent being invoked

**File Markers:** Each file is wrapped with markers (e.g., `[FILE: context/03_governance/audit.md]`) so the LLM can locate referenced content within the concatenated prompt.

---

## Directory Structure

| Directory/File | Description |
|---|---|
| **context/** | Always loaded for ALL agents |
| `01_llm_instructions.md` | LLM execution instructions |
| `02_application.md` | Application context (TEMPLATE) |
| **03_governance/** | Governance policies |
| `audit.md` | Audit logging requirements |
| `message_format.md` | JSON message specification |
| `fileformat.md` | Markdown file standards |
| **agents/** | Only SPECIFIC agent loaded per invocation |
| **01_governance_core/** | Strategic agents (Objective, Goal) |
| **02_operational_core/** | Mid-level agents (Planning, Review, Memory) |
| **03_executional_core/** | Application-specific agents |
| **system/** | Runtime artifacts (NOT concatenated) |
| `logs/` | Daily audit logs |

---

## Agent Processing Chain

Carbon Amber defines a hierarchical agent chain that processes user requests through three distinct layers:

### Layer 1: Governance Core (Strategic Layer)

High-level agents that determine strategic direction:

- **Objective Agent** — Receives user requests and outputs strategic objectives (*what* to achieve)
- **Goal Agent** — Decomposes objectives into actionable goals (*how* to achieve them)

**Example:**

- **User Input:** "Download this YouTube video and analyze the speaker's sentiment"
  - **Objective Agent Output:** "Obtain video content from YouTube", "Analyze speaker sentiment"
  - **Goal Agent Output:** Goals for each objective with specific steps

### Layer 2: Operational Core (Orchestration Layer)

Mid-level agents that manage workflow coordination:

- **Planning Agent** — Creates detailed execution plans and workflows
- **Reviewer Agent** — Validates outputs and ensures quality standards
- **Memory Agent** — Manages context and historical information

### Layer 3: Executional Core (Application-Specific)

Specialized agents that produce deliverables. These agents are customized for each application.

**Current DSTA Video Analysis Configuration:**

- **Perception Agent** — Identifies speakers, analyzes expressions/gestures, detects objects, analyzes audio
- **Interpretation Agent** — Processes analysis requests and contextualizes results
- **Action Agent** — Executes analysis models and generates structured outputs

### Processing Flow

Requests flow through the layers in sequence:

1. User submits a request
2. **Governance Core** processes the request into objectives and goals
3. **Operational Core** creates execution plans and manages context
4. **Executional Core** performs the actual work and generates outputs
5. Final output is returned to the user

### ARL Implementation

The ARL orchestration engine implements this chain through its *"Round-Robin Routing"* mechanism:

- Each agent outputs a JSON message with a next_agent field
- ARL reads this field and recursively invokes itself with the new target_agent_id
- This creates the multi-hop processing chain automatically
- When next_agent is null or COMPLETE, the final output returns to the user

---

## Key Components

### 1. LLM Execution Instructions

**File:** `context/01_llm_instructions.md`

Teaches the LLM how to interpret the concatenated prompt. Key content includes:

- How to locate referenced files using file markers
- What the LLM can and cannot do (cannot read files, execute code, or route messages)
- Instructions to consult governance files before processing requests
- Emphasis that the LLM's role is to produce JSON output; orchestration handles execution

**ARL Connection:** This file is always loaded first, ensuring the LLM understands the prompt structure before processing any agent logic.

### 2. Application Context

**File:** `context/02_application.md`

Defines the application's identity, capabilities, and constraints using a standardized template format:

- **[STANDARD]** sections — Same for all applications (execution context, governance precedence, agent roles)
- **[APPLICATION-SPECIFIC]** sections — Customized per application

**Current DSTA Video Analysis Configuration:**

- **Primary Objective:** Video content analysis and insight generation
- **Capabilities:** Audio transcription, speaker sentiment/stance analysis, audience sentiment, object detection, OCR, action recognition, scene understanding
- **Supported Sources:** YouTube, Instagram, TikTok, direct uploads
- **Technology Stack:** n8n, CrewAI, Gemini/Llama 4, Whisper-X, Elasticsearch, Wasabi

**ARL Connection:** ARL's Context Assembly station merges this file with agent definitions to create the complete prompt.

### 3. Governance Policies

**Directory:** `context/03_governance/`

#### A. Audit Logging (audit.md)

Defines how agent actions must be logged for explainability and compliance:

- Every agent output must include an audit field in its JSON response
- Audit field contains: compliance notes, governance files consulted, reasoning
- Logs must capture the "Chain of Thought" — why decisions were made

**ARL Connection:** ARL implements "Glass Box Audit Logging" by extracting the audit field from each agent's JSON output and appending it to an immutable log file, creating a complete audit trail.

#### B. Message Format (message_format.md)

Defines the standardized JSON structure for inter-agent communication. Required fields include:

- `message_id`, `timestamp`, `agent` — Identity information
- `input`, `output` — Content
- `next_agent` — Routing instruction
- `status`, `error` — Execution state
- `metadata` — Session tracking (session_id, request_id, sequence_number)
- `audit` — Governance compliance

**ARL Connection:** ARL's Router parses the next_agent.name field to determine routing, implementing the recursive agent chain.

#### C. File Format Standards (fileformat.md)

Defines documentation standards for agent definition files.

### 4. Agent Definitions

**Directory:** `agents/`

#### A. Objective Agent

**File:** `agents/01_governance_core/objective_agent.md`

The first agent in the chain; translates user requests into strategic objectives.

**Key Responsibilities:**

- Receives raw user input
- Outputs strategic objectives (the *what*)
- Initializes the standard message format
- Validates requests against application scope
- Must ALWAYS produce output (or error message if out of scope)

**ARL Connection:** ARL invokes this agent first by calling itself with target_agent_id: "agent_objective". ARL retrieves the agent definition from S3/Redis cache.

#### B. Goal Agent

Decomposes strategic objectives into actionable goals (the *how*).

#### C. Planning, Review, Memory Agents

Mid-level coordination agents for workflow creation, quality validation, and context management.

#### D. Executional Agents

Application-specific agents customized for each deployment. The current DSTA configuration includes Perception, Interpretation, and Action agents for video analysis tasks.

---

## Critical Design Features

### 1. Mandatory Governance Compliance

**Principle:** Governance files are authoritative and cannot be overridden.

**Enforcement:**

- Every agent must evaluate instructions against governance policies
- Conflicting instructions must be rejected and logged
- The audit field must document compliance checks

**ARL Connection:** ARL implements RBAC (Role-Based Access Control) on the S3 bucket containing governance files to prevent prompt injection attacks from modifying agent behavior. The governance files are read-only at runtime.

### 2. Session Tracking and Traceability

**Principle:** Every action must be traceable across the agent chain.

**Implementation:**

- `transaction_id` (ARL term) = `session_id` + `request_id` (platform term) created at chain start
- `sequence_number` increments with each agent hop
- `parent_message_id` links messages in the chain
- Audit logs capture full reasoning trail

**ARL Connection:** ARL propagates the transaction ID through all recursive calls, enabling complete reconstruction of the decision chain. This supports the "Insight AI" mentioned in ARL requirements, which can analyze audit logs for performance reporting and anomaly detection.

### 3. Cost Optimization via Small Language Models

**Principle:** Use Small Language Models (SLMs) by default to minimize computational cost and energy consumption ("digital ATP").

**Configuration:**

- **Default Models:** Llama-3-8B, Qwen-2.5-7B
- **Escalation Only:** GPT-4, Claude 3.5 (used only when necessary)

**ARL Connection:** ARL's Reasoning Core station executes SLMs via Ollama or Groq. Carbon Amber's agent definitions are designed to work effectively with smaller models through clear, structured prompts.

### 4. Circuit Breaker Protection

**Principle:** Prevent infinite loops and resource exhaustion.

**ARL Connection:** ARL implements a hop_count limiter. If the agent chain exceeds 10 hops, ARL force-terminates the chain to prevent "digital ischemia" (infinite resource consumption).

---

## How ARL Executes This Architecture

### ARL's 6-Station Workflow

The ARL orchestration engine implements Carbon Amber through a sequential 6-station workflow:

#### Station 1: Webhook Receiver

- **Receives:** { transaction_id, target_agent_id, input_context }
- **Maps to platform:** target_agent_id points to a file in the agents/ directory

#### Station 2: Definition Retrieval (Cache-First)

- Checks Redis for cached agent definition
- If cache miss: Fetches from S3 storage
- **Maps to platform:** Retrieves files from context/ and agents/

#### Station 3: Context Assembly

- Merges: System Prompt (context files) + Agent Definition + User Input
- **Maps to platform:** Concatenates files in the specified order with [FILE: ...] markers

#### Station 4: Reasoning Core

- Executes: SLM (Ollama/Groq) with concatenated prompt
- Enforces: JSON output schema (from message_format.md)

#### Station 5: Immutable Logging

- Extracts: audit field from JSON output
- Appends: Record to system/logs/{transaction_id}.jsonl
- **Maps to platform:** Implements requirements from context/03_governance/audit.md

#### Station 6: The Router

- Reads: next_agent.name from JSON output
- Routes:
  - If next_agent exists → Recursive call to Station 1 with new target_agent_id
  - If null/COMPLETE → Return final output to user
- **Maps to platform:** Implements the agent chain defined in this architecture

---

## Deployment Model

### Creating a New Application

To deploy a new application using this architecture:

1. **Copy application.md:** Create a new application configuration file from `context/02_application.md`
2. **Keep [STANDARD] sections unchanged:** Sections 1-3, parts of Section 9, and Section 10
3. **Customize [APPLICATION-SPECIFIC] sections:**
   - Section 4: Application identity (name, customer, version)
   - Section 5: Primary objective and scope
   - Section 6: Capabilities matrix
   - Section 7: Supported input sources
   - Section 8: Technology stack
   - Section 9: Executional Core agents (keep Governance/Operational standard)
   - Section 11: Constraints and boundaries
4. **Define Executional agents:** Create agent definitions in `agents/03_executional_core/` for application-specific tasks
5. **Deploy:** Upload to S3 bucket; ARL automatically discovers and loads the new configuration

**No changes to ARL required** — the orchestration engine is truly polymorphic and adapts to any application configuration.

### Example Applications

| Application | Primary Objective | Key Executional Agents |
|---|---|---|
| **DSTA Video Analysis** (current) | Video content analysis and insight generation | Perception, Interpretation, Action |
| **Document Processing** | Document classification and extraction | OCR, Entity Extraction, Classification |
| **Social Media Monitoring** | Social media trend analysis | Sentiment Tracking, Topic Extraction, Influencer ID |

---

## Success Metrics

Carbon Amber and the ARL orchestration engine are evaluated against the following success criteria:

1. **Reusability:** 100% of standard agent types (Objective, Goal, Planning) run on the single ARL template without code changes
2. **Performance:** Redis cache hits reduce agent instantiation time by >90% compared to S3 fetches
3. **Auditability:** 100% of agent routing decisions are traceable via generated audit log files, enabling the "Insight AI" to analyze performance

---

## Summary

Carbon Amber is a comprehensive prompt engineering architecture that defines intelligent agent behavior, governance policies, and application context. It functions as the "constitution" for an AI agent system — establishing how agents should behave, communicate, and maintain accountability.

When combined with the ARL orchestration engine described in the referenced PDF, Carbon Amber forms a complete AI agent system:

- **ADL provides the intelligence** — agent definitions, governance rules, application configuration
- **ARL provides the execution framework** — n8n orchestration, caching, routing, audit logging

Together, they enable the deployment of multiple AI applications using a single, polymorphic architecture that separates universal agent logic from application-specific configuration. This design ensures scalability, maintainability, and comprehensive governance compliance across all deployments.

---

*— End of Document —*
