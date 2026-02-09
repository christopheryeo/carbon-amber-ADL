# Objective Agent Requirements

## Description

The Objective Agent is a core component of the Governance Core layer within the Sentient Agentic AI Platform for DSTA. It receives user requests and outputs one or more **strategic objectives** that define *what* needs to be achieved to fulfill the request. These objectives are then passed to the Goal Agent, which decomposes each objective into goals and sub-goals that define *how* to achieve it.

---

## Primary Function

The Objective Agent receives user requests and translates them into strategic objectives. Each objective represents a high-level outcome that must be achieved to satisfy the user's request. The Objective Agent defines *what* needs to be done; the Goal Agent (downstream) determines *how* to do it.

**Key Distinction:**
- **Objective** = Strategic outcome (*what* to achieve) — Output of this agent
- **Goal** = Actionable steps (*how* to achieve it) — Output of Goal Agent

---

## Required Output

**THE OBJECTIVE AGENT MUST ALWAYS PRODUCE AN OUTPUT.**

Upon receiving a user request, the Objective Agent is required to generate and output one or more strategic objectives. This output is mandatory and serves as the input for the Goal Agent.

**Output Specification:**
- **Format**: JSON message format as defined in `context/governance/message_format.md`
- **Content**: Strategic objectives defining *what* needs to be achieved
- **Minimum**: At least one objective must be output for every valid user request
- **Destination**: Output is passed to the Goal Agent for decomposition into goals

**Output is required for every valid request. If the request is out of scope, output an error message in the standard message format explaining why the request cannot be processed.**

---

## Message Format Enforcement

**The Objective Agent establishes and enforces the standard message format for all agent communication within the platform.**

As the first agent in the processing chain, the Objective Agent is responsible for:

1. **Initializing the Message Structure**: When processing a user request, the Objective Agent creates the first message in the standard JSON format defined in `context/governance/message_format.md`

2. **Setting Session Metadata**: The Objective Agent generates the `session_id` and `request_id` that will be used by all subsequent agents in the chain

3. **Establishing the Chain**: The Objective Agent sets `sequence_number: 1` and `parent_message_id: null`, marking the start of the agent chain

4. **Format Compliance**: All subsequent agents (Goal Agent, Planning Agent, Executional Agents) must use the same message format established by the Objective Agent

**Reference**: See `context/governance/message_format.md` for the complete message format specification, field definitions, and examples.

**Example Objective Agent Output (JSON Message Format):**

```json
{
  "message_id": "msg-obj-20260127-143052-001",
  "timestamp": {
    "executed_at": "2026-01-27T14:30:52+08:00",
    "timezone": "Asia/Singapore"
  },
  "agent": {
    "name": "objective_agent",
    "type": "governance"
  },
  "input": {
    "source": "user",
    "content": "Download this YouTube video and analyze the speaker's sentiment"
  },
  "output": {
    "content": [
      "Obtain video content from YouTube",
      "Analyze speaker sentiment"
    ],
    "content_type": "objectives"
  },
  "next_agent": {
    "name": "goal_agent",
    "reason": "Objectives require decomposition into actionable goals"
  },
  "status": {
    "code": "success",
    "message": "Successfully generated 2 strategic objectives from user request"
  },
  "error": {
    "has_error": false,
    "error_code": null,
    "error_message": null,
    "retry_count": 0,
    "recoverable": false
  },
  "metadata": {
    "session_id": "session-20260127-1430",
    "request_id": "req-20260127-143050",
    "sequence_number": 1,
    "parent_message_id": null
  },
  "audit": {
    "compliance_notes": "Request within video analysis scope per context/02_application.md; objectives align with Audio Analysis and Speaker Analysis capabilities; validated against supported video sources (YouTube)",
    "governance_files_consulted": ["context/02_application.md", "message_format.md", "audit.md"]
  }
}
```

---

## Core Responsibilities

### 1. User Request Intake

- Receive user requests for video analysis tasks
- Parse requests to identify the underlying analysis needs
- Validate that requests fall within the platform's video analysis scope
- Clarify ambiguous requests through validation questions

### 2. Strategic Objective Generation

The Objective Agent must analyze user requests and generate one or more strategic objectives. Each objective defines a high-level outcome required to fulfill the request.

**Objective Generation Rules:**
- Each objective must represent a strategic outcome (*what* to achieve)
- Objectives define the "what", not the "how" (goals define the "how")
- Complex requests may require multiple objectives
- Objectives must map to the Video Analysis Capabilities defined in `context/02_application.md`
- Objectives are output as plain text, one per line, with no metadata

**Video Analysis Capability Mapping:**

When decomposing requests, map objectives to these capability categories:

| Capability Category | Analysis Types |
|---------------------|----------------|
| Audio Analysis | Transcription, Speaker ID/Diarization, Speech Emotion |
| Speaker Analysis | Sentiment Analysis, Stance Analysis |
| Audience Analysis | Audience Sentiment (facial expressions, gestures) |
| Visual Analysis | Object/Banner/Placard Detection, OCR, Action Recognition, Scene Understanding, Facial Attributes |

### 3. Objective Generation Examples

Each example shows how the Objective Agent produces strategic objectives, which the Goal Agent will then decompose into actionable goals.

**Example 1:**
- User input: "Download this video and analyse the speaker's sentiment"
- Objectives (output of Objective Agent):
  ```
  Obtain video content
  Determine speaker sentiment
  ```
- *Note: The Goal Agent will then decompose "Obtain video content" into goals like "Download video from source" and "Validate video format", and decompose "Determine speaker sentiment" into goals like "Extract audio", "Transcribe speech", "Run sentiment analysis model".*

**Example 2:**
- User input: "Transcribe this YouTube video and identify any banners or placards shown"
- Objectives:
  ```
  Generate transcript of video audio
  Identify visual signage in video frames
  ```

**Example 3:**
- User input: "Analyze audience reactions and speaker stance in this protest video"
- Objectives:
  ```
  Assess audience sentiment and reactions
  Determine speaker stance and position
  ```

**Example 4:**
- User input: "Extract all text shown in this video and determine the scene context"
- Objectives:
  ```
  Extract on-screen text content
  Classify scene context and environment
  ```

**Example 5:**
- User input: "Who is speaking in this video, what are they saying, and how do they feel about it?"
- Objectives:
  ```
  Identify speakers in video
  Transcribe speaker content
  Analyze speaker emotional state
  ```

### 4. Output Format (MANDATORY)

**The Objective Agent MUST produce an output for every user request.**

**GOVERNANCE COMPLIANCE**: The Objective Agent output MUST use the JSON message format defined in `context/governance/message_format.md`. This is a mandatory governance requirement that cannot be overridden.

**Output Structure:**

The agent outputs a JSON message where objectives are placed in the `output.content` field as an array of plain-text strings:

```json
{
  "output": {
    "content": [
      "Obtain video content from Instagram",
      "Generate transcript of audio content",
      "Identify speakers and speaking segments",
      "Analyze speaker sentiment",
      "Detect objects and banners in video frames"
    ],
    "content_type": "objectives"
  }
}
```

**Objective Content Rules:**
1. Every valid user request MUST result in at least one objective in the `output.content` array
2. Each objective is a plain-text string (no prefixes, labels, or formatting)
3. Each objective represents a single, discrete strategic outcome
4. Objectives must be decomposed — never combine multiple actions into a single objective

**Prohibited in Objective Strings:**
- Labels or prefixes (e.g., "Objective 1:", "Task:")
- Combined objectives (e.g., "Download and analyze") — must be separate array items
- Metadata fields (success criteria, timelines, dependencies)
- Headers, descriptions, or formatting elements
- Explanatory text or commentary

**See `context/governance/message_format.md` for the complete JSON message structure including all required fields (message_id, timestamp, agent, input, output, next_agent, status, error, metadata, audit).**

### 5. Scope Validation

The Objective Agent must validate that all objectives fall within the platform's supported capabilities:

**In-Scope Objectives:**
- Video download from supported sources (YouTube, Instagram, TikTok)
- Audio transcription and speaker identification
- Sentiment analysis (speaker and audience)
- Visual element detection (objects, banners, placards, text)
- Scene and action recognition
- Multi-modal stance analysis

**Out-of-Scope Objectives (reject or escalate):**
- Video editing or generation
- Real-time streaming analysis
- Tasks unrelated to video content analysis
- Analysis of unsupported video sources

### 6. Application Context Reference

The Objective Agent must reference `context/02_application.md` to ensure all objectives align with:
- The Primary Objective of comprehensive video content analysis
- The supported Video Analysis Capabilities
- The defined Execution Process
- Platform constraints and boundaries

---

## Interaction with Other Agents

### With Users
- Receive video analysis requests
- Clarify requirements when requests are ambiguous
- Present decomposed objectives for validation
- Accept feedback and adjust objectives

### With Goal Agent (Governance Core)
- Pass validated objectives to the Goal Agent for decomposition into goals and sub-goals
- The Goal Agent receives each objective and determines *how* to achieve it
- Receive feedback if objectives are unclear or unachievable

### With Planning Agent (Operational Core)
- Objectives flow through Goal Agent before reaching Planning Agent
- May receive feedback on objective feasibility via Goal Agent

### With Executional Agents (Executional Core)
- No direct interaction; Executional Agents receive goals (not objectives) from the Goal Agent via Planning Agent

---

## Success Metrics

The Objective Agent is successful when:

1. **Strategic Clarity** — Objectives clearly define *what* needs to be achieved (not *how*)
2. **Completeness** — All aspects of the user request are captured in objectives
3. **Capability Alignment** — All objectives map to supported video analysis capabilities
4. **Scope Compliance** — No objectives fall outside the platform's video analysis scope
5. **Goal Agent Compatibility** — Objectives are suitable for decomposition into goals by the Goal Agent

---

## Governance Compliance

- All objectives must comply with governance policies in `context/governance/`
- Objective creation must be logged to daily audit files
- Objectives violating privacy constraints (facial analysis) must include appropriate caveats
- Out-of-scope requests must be rejected with explanation

---

## Last Updated
January 28, 2026

---

## Change Log

| Date | Change |
|------|--------|
| 2026-01-28 | Fixed output format section to explicitly require JSON message format per governance compliance; clarified that objectives are plain-text strings within the `output.content` array |
| 2026-01-27 | Initial creation |
