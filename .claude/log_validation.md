# Log Validation Template

This template defines the standard format for analyzing new n8n runtime log files. When prompted by `startup.md` to run a log validation analysis, review the log file and generate a report using the exact structure below.

---

## Part 1: Log File Validation Analysis

### Objective
Analyse the log file (`<filename>`) and determine whether it validates the behaviour of the agents in the carbon-amber-ADL and whether agents are performing as expected.

---

### Log File Summary

The log contains **[X] agent executions** across **[Y] distinct user requests**.

#### The [Y] User Requests

| # | Request Type | Time Range | Description |
|---|---|---|---|
| 1 | <Short type> | <HH:MM:SS-HH:MM:SS> | <Description> |
| 2 | ... | ... | ... |

---

### Compliance Assessment

#### ✅ What's Working Well

<List 4-6 areas of strong compliance. Examples:>
**Message Format Compliance (Strong)**
- Every single message contains all required fields from `message_format.md`

**Ref ID Handling (Consistently Correct)**
- Objective Agent properly extracts URLs and assigns `src_1` / `store_1` refs

**Planning Agent Deduplication (Works Well)**
- Correctly identified X duplicate goals across Y objectives

---

#### ⚠️ Issues Identified

<Identify anomalies, bugs, or non-compliant behaviors. For each, use the format:>

##### Issue 1: <Short Title>
**Severity: <High/Medium/Low>**

- <Description of what happened>
- <Evidence from log>

**Root cause:** <Hypothesis on why it happened (e.g. LLM reasoning failure vs. orchestration bug)>
**Impact:** <Consequence of the failure>

##### Issue 2: <Short Title>
<...>

---

### Verdict

**Overall Assessment:** <e.g., Agents are largely performing as expected>

**Strengths:**
- <List top 3 strengths>

**Critical Concerns:**
- <List top concerns required fixing>

**Root Cause Hypothesis:** <Summary of where the issues are stemming from>
