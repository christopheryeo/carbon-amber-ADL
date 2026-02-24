# Agent Task Formulation and Decomposition

This document explains how the generic framework agents (Objective, Goal, Planning) rely on the rules defined in `context/application.md` to formulate, decompose, and schedule tasks specific to the DSTA Video Analysis platform.

---

## 1. Objective Decomposition
**Where it's guided:** `application.md` (Section 4: Application Identity & Section 5: Primary Objective)

**How it works:** 
These sections define the strict boundaries of the application. The **Objective Agent** references Section 5 to determine whether a user Request is in-scope. If a request does not align with the stated objectives (e.g., "The platform transforms raw video inputs into actionable intelligence..."), it gets rejected.

---

## 2. Goal Decomposition
**Where it's guided:** `application.md` (Section 6.11: Capability-to-Goal Decomposition Patterns)

**How it works:** 
This section provides explicit templates (e.g., *Pattern A: Video Acquisition Objectives*) that the **Goal Agent** must use. It provides exact phrasing instructions, such as:
*"If you see an objective about downloading a video, decompose it into exactly these 4 goals using this exact wording."*
This ensures complete consistency in how complex user requests are broken down into actionable steps.

---

## 3. Task Formulation
**Where it's guided:** `application.md` (Sections 6.1 - 6.8: Capabilities Matrix & Section 6.10: Capability-to-Tool Mapping)

**How it works:** 
The actual *wording* of tasks is inherited blindly from the Goal Agent. However, the **Planning Agent** uses the Capabilities Matrix to identify what the task technically *is*, mapping the plain text to a specific `CAP-ID` (like `CAP-ACQ-001`). Later in the pipeline, the **Action Agent** uses Section 6.10 to translate that `CAP-ID` into a specific tool or script (like `url_validator`) for physical execution.

---

## 4. Scheduling & Dependencies
**Where it's guided:** `application.md` (Section 6.9: Capability Dependencies and I/O Reference)

**How it works:** 
This section enforces the rules of physics for the pipeline via a strict prerequisite table. For example, it explicitly states that `CAP-AUD-001` (Transcription) **must** be preceded by `CAP-AUD-004` (Language Detection). The **Planning Agent** reads this table to build its Directed Acyclic Graph (DAG), order its execution groups, and determine which tasks can run in parallel and which must wait for prerequisites.

---

*Summary: `application.md` serves as the configuration rulebook, while the generic agent definitions (`objective.md`, `goal.md`, `planning.md`) serve as the execution engine that parses and enforces that configuration.*
