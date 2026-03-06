# IntelliPulse Integration

**Priority**: #2
**Deadline**: ASAP
**Owner**: Upender
**Complexity**: Low. Estimated: 2–3 days.
**Trello**: #96 (transport layer), #97 (Voice AI enrichment)

---

## What to build

At the end of every voice call, UniWeave sends the call transcript and audio recording to IntelliPulse via a POST request to their API. IntelliPulse processes these for quality analysis, coaching, and insights. UniWeave's job is purely to deliver the data — what IntelliPulse does with it is their concern.

This is a post-call webhook. No real-time streaming. No changes to the call flow itself.

**IntelliPulse API endpoint**: `POST /api/audit-publisher/publish` (port 7000 in `analytics-user-mgmt`)

---

## What this is NOT

- Real-time audio streaming to IntelliPulse during a call — not in scope
- IntelliPulse UI inside UniWeave — not in scope
- Receiving data back from IntelliPulse into UniWeave — not in scope for v1

---

## P0 Requirements

### Trigger

- [ ] UniWeave fires a `call.ended` event when a call completes
- [ ] Event includes: `call_id`, `transcript` (full text), `audio_recording_url` (or audio file), `call_duration`, `timestamp`

### Integration handler

- [ ] On `call.ended` event, call IntelliPulse API with:
  - Transcript (full conversation, formatted as array of turns: role + content)
  - Audio recording URL (or raw audio if IntelliPulse requires file upload)
  - Call metadata: call ID, timestamp, duration
- [ ] Auth: API key in request header (confirm key format with IntelliPulse)
- [ ] Async: integration runs after call ends — does NOT block or delay the call

### Error handling

- [ ] If IntelliPulse POST fails: log the error + retry once after 60 seconds
- [ ] If retry fails: log to error queue for manual review. Do not silently drop.
- [ ] IntelliPulse downtime must not affect call quality or stability

### Configuration

- [ ] IntelliPulse API key stored in ENV (not hardcoded)
- [ ] Integration can be enabled/disabled per deployment via config flag

---

## Acceptance criteria

**Happy path:**
- Given a completed voice call with a transcript and recording
- When the call ends
- Then within 5 seconds, IntelliPulse receives a POST with transcript + audio, and returns a 2xx response

**Failure handling:**
- Given IntelliPulse returns a 500
- When the first POST fails
- Then UniWeave retries once after 60 seconds and logs the outcome

**No call impact:**
- Given the IntelliPulse API is down
- When a call ends
- Then the call ends cleanly with no error shown to the end user; only internal logging is affected

---

## Day-by-day tasks

**Day 1** — Align with IntelliPulse + scaffold
- Meet/message IntelliPulse team: get API endpoint, auth format, required payload structure
- Confirm: do they want audio as URL or file upload?
- Scaffold the integration handler (receives `call.ended`, formats payload)
- Test POST manually with Postman/curl using a sample transcript

**Day 2** — Wire into UniWeave call lifecycle
- Hook handler into UniWeave's `call.ended` event
- Test end-to-end with a real call: confirm IntelliPulse receives data
- Add ENV config for API key + on/off flag

**Day 3** — Error handling + QA
- Add retry logic (1 retry after 60s)
- Add error logging
- Run 3 end-to-end test calls; confirm IntelliPulse dashboard shows data

---

## Open questions

| Question | Answer needed from |
|---|---|
| IntelliPulse API endpoint URL | IntelliPulse team |
| Auth format: API key in header? Bearer token? | IntelliPulse team |
| Audio: URL or file upload? | IntelliPulse team |
| Does IntelliPulse have a test/sandbox environment? | IntelliPulse team |
| Where is the `call.ended` event in the current UniWeave codebase? | Harsh / engineering |

---

## Related

- [Analytics & User Mgmt Explainer](../analytics-user-mgmt-explainer.md) — full audit pipeline details
- [System Architecture](../system-architecture.md) — post-call data flow section
- Trello #96 (transport), #97 (enrichment)
