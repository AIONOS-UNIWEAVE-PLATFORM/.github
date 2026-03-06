# Chat Feature

**Priority**: #1 — customer commit
**Deadline**: ~March 19, 2026 (2 weeks)
**Owner**: Nomaan Khan
**Stack**: Azure OpenAI (NOT Ultravox) — text only, channel-agnostic Chat API + WhatsApp connector

---

## What to build

A chat widget that can be embedded on any website using a JS snippet. Users type messages; the agent responds in text using the same system prompt architecture already used for voice agents. The chat must be multi-turn (it remembers the conversation). The goal is to let enterprise clients deploy a UniWeave text agent on their website the same way they deploy a voice agent — just a different interface.

Same backend system prompt format. Different front-end. Chat is a **platform capability** — ships for March launch regardless of customer timelines.

---

## Phased channel rollout (confirmed)

| Phase | Channel | Status |
|---|---|---|
| v1 (now) | Web widget | Build first |
| v2 | Mobile app | After web widget ships |
| v3 | SDK / API kit | So customers can build their own UI |
| v4 | Contained UI (white-label) | Final phase |

**Customer needs**: RAG integration + tool calling. Design backend to support both from day one even if v1 doesn't expose them fully.

## What this is NOT in v1

- Mobile app — v2
- SDK — v3
- Custom branding/theming — v4
- Analytics dashboard — separate initiative

---

## P0 Requirements (must-have for v1)

### Backend

- [ ] New `/chat` API endpoint on UniWeave backend
  - Accepts: `session_id`, `message`, `system_prompt` (or `agent_id` pointing to a stored prompt)
  - Returns: agent text response (streaming preferred; sync acceptable for v1)
  - Maintains conversation history per `session_id`

- [ ] Session management
  - Each chat session has a unique ID
  - Session stores message history (minimum: last 20 turns)
  - Session expires after 30 minutes of inactivity

- [ ] Same system prompt format as voice
  - Engineers use the same 9-section prompt structure already in use
  - No new prompt format to invent — reuse what exists

### Frontend widget

- [ ] React chat widget component
  - Message input box + send button
  - Scrollable message history (user messages right, agent messages left)
  - Typing indicator while agent is responding
  - Auto-scroll to latest message

- [ ] Embed script
  - Single `<script>` tag that can be dropped onto any webpage
  - Configurable: `agent_id`, `primary_color`, `widget_position` (bottom-right default)
  - Widget opens as floating bubble; expands on click

- [ ] Basic error handling
  - If API call fails: show "Something went wrong, please try again"
  - If session expires: show "This conversation has ended. Start a new chat."

---

## Acceptance criteria

**Happy path:**
- Given a webpage with the embed script installed
- When a user clicks the chat bubble and types a message
- Then the agent responds within 3 seconds and the conversation continues correctly across multiple turns

**Session persistence:**
- Given an ongoing conversation
- When the user sends a second message without refreshing the page
- Then the agent's response reflects context from earlier in the conversation

**Error state:**
- Given an API timeout
- When the user sends a message
- Then the widget shows an error message — it does not hang indefinitely

**Embed:**
- Given the embed script with a valid `agent_id`
- When dropped on any HTML page
- Then the chat widget appears in the bottom-right corner with no additional configuration

---

## Day-by-day tasks

**Day 1** — Backend foundation
- Set up `/chat` endpoint skeleton (accepts message, returns hardcoded response)
- Define session schema + store (in-memory or Redis)
- Test with curl: send a message, get a response, send a second message with same session ID

**Day 2** — LLM integration
- Wire `/chat` endpoint to Azure OpenAI
- Pass session history as context on each call
- Test: 5-turn conversation where the agent remembers what was said in turn 1

**Day 3** — Widget UI (bare bones)
- Build React chat component: input, send, message list
- Connect to `/chat` endpoint
- Test locally: type a message, see the response

**Day 4** — Widget polish + embed script
- Typing indicator, auto-scroll, basic styling
- Build embed script (`<script>` tag → loads widget)
- Test: embed on a blank HTML page, full conversation works

**Day 5** — Error handling + QA
- Add error states (timeout, session expired)
- End-to-end test with real system prompt (use an existing voice agent prompt)
- Send demo link to Agam for review

---

## Open questions / assumptions

| Assumption | Status |
|---|---|
| Web widget is the right first channel | CONFIRMED |
| Customer needs RAG + tool calling | CONFIRMED — design backend to support both |
| Phased rollout: widget → app → SDK → contained UI | CONFIRMED |
| English only for v1 | Assumed — confirm |
| Same system prompt format as voice (9-section) | Assumed — confirm with engineering |
| Session stored server-side | Assumed — confirm with engineering |

---

## Related

- [System Architecture](../system-architecture.md)
- Trello #94 (Chat Engine), #95 (WhatsApp connector), #108 (WABA provisioning)
