# Agent Intelligence Dashboard — Outcome-Centric Platform Analytics

**Card**: Trello #97 (repurposed from "Pulse enrichment")
**Status**: Ready to Build
**Module**: UniWeave Platform Analytics

---

## Context

The Saudia/Whilter fork built a Voice AI Operations Portal — a solid operational baseline but
entirely descriptive and passive. It tells you what happened, not what to do about it. More
critically, it measures the wrong primary metric: automation rate (did a human answer?) rather
than outcome completion rate (did the agent finish the task?).

UniWeave sells agents. Agents are evaluated on outcomes. The analytics layer must reflect this.

Card #97 ("[Pulse] Voice AI enrichment") was confirmed out-of-scope per 2026-03-07 code review —
Pulse has no schema fields for operational metrics. This card repurposes #97 to build what was
always needed: an outcome-centric Agent Intelligence Dashboard, separate from IntelliPulse.

**Separation of concerns:**
- **IntelliPulse (Pulse)**: rubric-based QA scoring, sentiment, NPS, agent coaching from QA angle.
- **Agent Intelligence Dashboard (this spec)**: agent execution performance — did the agent complete the task? Which tools failed? Which prompts work better?

---

## Part 1 — Saudia Baseline KPIs (adopt, re-implement cleanly)

Source: existing audit pipeline (`audit-publisher → Kafka → MongoDB audit_store`).
These mirror the Saudia/Whilter portal but are re-implemented in the UniWeave platform UI.

| KPI | Definition | Source field |
|---|---|---|
| **Automation Rate** | % of calls handled without human transfer | `escalated == false` |
| **Average Handling Time (AHT)** | Avg call duration across selected period | `duration_ms` |
| **Escalation Rate** | % of calls transferred to human agent | `escalated == true` |
| **CSAT Score** | Avg sentiment score (0–5) from interactions | `sentiment_score` |
| **AI Containment Rate** | % conversations fully resolved by AI (no transfer, no escalation) | `escalated == false AND completed == true` |
| **Call Intent Frequency** | Count of each detected intent over time period | `intent` |
| **Calls by Duration** | Distribution: Short (<60s), Medium (60–300s), Long (>300s) | `duration_ms` bucket |
| **Daily Calls Trend** | AI-handled vs human-handled calls by day | `escalated`, `created_at` |
| **Peak Hours Volume** | Hourly call traffic distribution | `created_at` hour |
| **Call Logs** | Per-call record: intent, sentiment, status, duration, transcript, audio | full `audit_store` record |
| **Intent Detail Panel** | Per-call: intent, sentiment, expected vs actual result, entities, narrative summary | `intent`, `outcome_completed`, `entities`, `summary` |

---

## Part 2 — New Outcome-Centric KPIs

These are net-new metrics not present in the Saudia portal. They answer: *did the agent actually complete the task?*

| KPI | Definition | Why it matters |
|---|---|---|
| **Outcome Completion Rate (OCR)** | % of calls where agent fully completed the stated intent | **Primary metric.** Containment ≠ completion. AI can handle a call and still fail the task. |
| **Tool Success Rate (TSR)** | % of tool calls returning valid response (2xx + non-error payload) | Tool failures = silent task failures. Shown per tool + aggregate. |
| **Intent Resolution Rate (IRR)** | OCR broken down by intent type | Which intents the agent handles well vs poorly. Directly feeds prompt improvement work. |
| **Agent Effectiveness Score (AES)** | Composite = OCR × Containment Rate × TSR | Single number per agent version. Comparable across agents and over time. |
| **Turns to Completion** | Avg conversation turns to complete a task | Lower = more efficient. High turn count = prompt clarity issue. |
| **Clarification Loop Rate** | % of calls where agent asks for the same entity twice | Indicates prompt gap — agent doesn't collect information correctly the first time. |
| **Drop-off Stage** | For incomplete calls: at what stage did completion fail? | Root cause analysis. Tells you exactly where to fix the prompt or tool. |
| **Prompt Version Delta** | Any metric compared across agent versions (v1.0 vs v1.1) | Did the prompt change make things better or worse? Quantified. |

### Drop-off Stage enum values

```
entity_collection   — agent failed to collect required entities (passenger name, flight number, etc.)
tool_call           — tool call failed or returned error
confirmation        — agent completed tool call but failed at final confirmation step
unknown             — could not determine failure stage
```

### Agent Effectiveness Score calculation

```
AES = OCR × Containment Rate × TSR

Where:
  OCR = outcome_completed / total_calls
  Containment Rate = (total_calls - escalated_calls) / total_calls
  TSR = successful_tool_calls / total_tool_calls

Range: 0.0–1.0. Higher = more effective.
```

---

## Part 3 — Agentic Intelligence Layer

The Saudia portal produces 1 automated insight ("most frequent intent is one_way_booking"). This
is trivial. UniWeave runs LLM analysis across interactions, clusters failures, and surfaces
actionable recommendations for prompt improvement.

### 3.1 Automated Insight Engine

**How it runs**: Cron job (daily at 02:00 UTC) or triggered on-demand from dashboard.

**What it does**:
1. Reads `audit_store` for the last 24 hours (or configurable window)
2. Filters for incomplete calls (`outcome_completed == false`)
3. Groups by intent + drop_off_stage + error patterns
4. Calls ai-gateway (gpt-4o-mini) with failure cluster data
5. LLM produces: finding, affected intent, root cause hypothesis, suggested action
6. Writes results to `insights_store` (MongoDB, new collection)
7. Dashboard reads `insights_store` for insight cards

**Output format per insight card**:
```json
{
  "insight_id": "...",
  "severity": "high | medium | low",
  "finding": "73% of One Way Booking intents fail at entity_collection",
  "detail": "Passenger last name provided as first name only in 62% of calls",
  "affected_intent": "one_way_booking",
  "affected_calls": 47,
  "sample_call_ids": ["...", "..."],
  "suggested_action": "Add explicit instruction: require full name (first + last) before proceeding to flight selection",
  "created_at": "2026-03-08T02:00:00Z"
}
```

### 3.2 Agent Coaching Recommendations

- Failure clusters surface as specific prompt improvement suggestions in the dashboard
- Each recommendation is tied to the agent version running during the failure period
- Suggestions are **not auto-applied** — surfaced for operator review and approval
- Operator can: Dismiss / Mark as applied / Copy to clipboard (for pasting into agent prompt editor)

### 3.3 Prompt Performance Benchmarking

- Track OCR, AES, and TSR per agent version
- Show delta when a new version is deployed
- Timeline view: "v1.1 deployed Mar 10. OCR: 58% → 74% (+16pp)"
- Exposed in the Agent Config screen (version history table gains analytics columns: OCR, AES, TSR)
- Comparison mode: select any two versions and see metric diff

### 3.4 Real-time Failure Signal (v2 scope — do not build now)

During a live call: if pattern matches known failure signatures (clarification loop, entity not
collected after 3 attempts), surface a risk indicator in the Live Calls / Live Rooms view.
Ambient signal only — no call interruption. Flag for v2 planning.

---

## Part 4 — Data Model Additions

New fields required in the call-end audit event, emitted by the LiveKit agent service (Python).
These flow into the existing `audit-publisher → Kafka → audit-listener → audit_store` pipeline.

### New fields on call.ended event

```json
{
  "outcome_completed": true,
  "drop_off_stage": "entity_collection | tool_call | confirmation | unknown | null",
  "intent": "one_way_booking",
  "turns_count": 12,
  "clarification_loops": 2,
  "agent_version": "v1.1",
  "tool_calls": [
    {
      "tool_name": "search_flights",
      "success": true,
      "latency_ms": 340,
      "error_code": null,
      "called_at": "2026-03-08T14:23:11Z"
    },
    {
      "tool_name": "book_flight",
      "success": false,
      "latency_ms": 1200,
      "error_code": "INVALID_PASSENGER_NAME",
      "called_at": "2026-03-08T14:24:05Z"
    }
  ]
}
```

### Field definitions

| Field | Type | Description | Who sets it |
|---|---|---|---|
| `outcome_completed` | boolean | Did the agent fully complete the stated intent? | LiveKit agent, tracked state |
| `drop_off_stage` | enum | Where did an incomplete call fail? Null if completed. | LiveKit agent, inferred from call state |
| `intent` | string | Primary intent detected at call start | LiveKit agent, first classification |
| `turns_count` | integer | Total conversation turns (agent + user) | LiveKit agent, counter |
| `clarification_loops` | integer | Times agent asked for same entity twice | LiveKit agent, entity tracker |
| `agent_version` | string | Active agent version (e.g. "v1.1") | LiveKit agent, from agent config |
| `tool_calls[]` | array | All tool executions during the call | LiveKit agent, intercepted at tool dispatch |

### Implementation note for LiveKit engineer

These are **lightweight tracked state** — no additional LLM calls needed to collect them.
`outcome_completed` is set by the agent's completion state machine (already knows if it finished).
`drop_off_stage` is inferred from which state the call was in when it ended.
`tool_calls[]` is populated by intercepting tool dispatch/response at the framework level.

### MongoDB schema addition (analytics-user-mgmt)

Add to existing `AuditStoreRequest` in `analytics-user-mgmt`:

```java
// In AuditStoreRequest.java
private Boolean outcomeCompleted;
private String dropOffStage;  // enum: entity_collection, tool_call, confirmation, unknown
private String intent;
private Integer turnsCount;
private Integer clarificationLoops;
private String agentVersion;
private List<ToolCallRecord> toolCalls;

// New inner class
public static class ToolCallRecord {
    private String toolName;
    private Boolean success;
    private Integer latencyMs;
    private String errorCode;
    private Instant calledAt;
}
```

---

## Part 5 — Architecture

```
LiveKit Agent Service (Python)
  ↓ call.ended + enriched audit payload
  ↓ (outcomeCompleted, dropOffStage, toolCalls[], intent, turnsCount,
  ↓  clarificationLoops, agentVersion)

audit-publisher (Java, port 7000)
  POST /api/audit-publisher/publish

Kafka (audit-queue-voicebot-dev)

audit-listener (Java, port 7001)
  → audit_store (MongoDB, existing collection — schema extended)

audit-analytics (Java, port 7002)
  → Dashboard API queries (new endpoints for outcome KPIs)

UniWeave Platform UI
  → Agent Intelligence Dashboard
  → Baseline KPIs (Part 1) + Outcome KPIs (Part 2)
  → Agent Config screen (Prompt Version Benchmarking)

Automated Insight Engine (separate cron service)
  → reads audit_store (MongoDB)
  → calls ai-gateway (gpt-4o-mini)
  → writes to insights_store (MongoDB, new collection)
  → Dashboard reads insights_store → Insight cards panel
```

### New API endpoints required (audit-analytics, port 7002)

| Endpoint | Returns |
|---|---|
| `GET /analytics/outcome?agentId=&from=&to=` | OCR, AES, TSR, drop-off breakdown |
| `GET /analytics/tools?agentId=&from=&to=` | TSR per tool, latency, error codes |
| `GET /analytics/intents?agentId=&from=&to=` | IRR by intent, turns avg by intent |
| `GET /analytics/versions?agentId=` | Per-version metric comparison |
| `GET /insights?agentId=&status=` | Insight cards (pending / dismissed / applied) |
| `PATCH /insights/:id` | Update insight status (dismiss, mark applied) |

---

## Part 6 — Acceptance Criteria

### Done when

- [ ] Engineering reviews and approves data model additions (call.ended event + AuditStoreRequest)
- [ ] LiveKit agent service emits enriched fields on call.ended
- [ ] Baseline KPIs (Part 1 — Saudia parity) visible in Agent Intelligence Dashboard
- [ ] OCR visible per call (intent detail panel) and in aggregate (dashboard summary)
- [ ] Drop-off stage visible per call and in aggregate (breakdown chart by stage)
- [ ] Tool Success Rate visible per tool and in aggregate
- [ ] Turns to Completion and Clarification Loop Rate visible in dashboard
- [ ] AES calculated and shown per agent + per agent version
- [ ] Automated Insight Engine v1 live: daily cron, insight cards appear on dashboard
- [ ] Coaching recommendations surfaced with dismiss / mark applied actions
- [ ] Prompt version comparison functional: select v1.0 vs v1.1, see OCR/AES/TSR delta
- [ ] Agent Config screen shows analytics columns in version history table

### Not included

- IntelliPulse QA scoring (separate module)
- Real-time call intervention (v2 scope)
- Telephony-level metrics
- Outbound call analytics (future)

---

## Part 7 — Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | Who owns the LiveKit agent enrichment work? Harsh to assign. | Harsh | Open |
| 2 | Is `outcome_completed` reliably determinable from the agent's existing state machine? | LiveKit engineer | Open |
| 3 | Insights Engine: separate microservice or extend audit-analytics? | Harsh | Open |
| 4 | UI: Agent Intelligence Dashboard — new top-level nav item or tab inside Agent screen? | Agam + design | Open |
| 5 | `insights_store` collection: same MongoDB instance as `audit_store`? Assume yes. | Engineering | Open |
| 6 | Agent version: set in platform UI — does LiveKit pick it up dynamically or injected at deploy? | Harsh + Platform | Open |
