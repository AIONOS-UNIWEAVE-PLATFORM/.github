# IntelliRAG Integration

**Priority**: #3
**Deadline**: ASAP — partial start now; full spec after wireframes
**Owner**: Himanshu Arora
**Status**: Devs have swagger + platform access. UI spec blocked on Agam's wireframes. Backend exploration can start now.
**Trello**: #98

---

## What to build

UniWeave voice/chat agents query IntelliRAG's knowledge base during conversations to retrieve relevant information — policies, product details, FAQs, procedures — and use it in their responses. Instead of embedding all knowledge in the system prompt, the agent pulls it dynamically from IntelliRAG at query time.

This makes agents smarter (access to a real knowledge base) and easier to maintain (knowledge updates in IntelliRAG automatically flow to the agent — no prompt editing needed).

**Architecture**: IntelliRAG will expose a native **MCP server**. The `livekit-agent` consumes it via `mcp_client.py`. See [livekit-agent-mcp-integration.md](livekit-agent-mcp-integration.md) for the full integration spec.

---

## What this is NOT

- UniWeave hosting or managing the knowledge base — IntelliRAG owns that
- Custom RAG pipeline built by UniWeave — we use IntelliRAG's API, not build our own
- UI for uploading documents to IntelliRAG — not in scope for v1

---

## P0 Requirements

### Query integration (backend — can start now)

- [ ] IntelliRAG exposes MCP server with `query_knowledge_base` tool (see MCP spec)
- [ ] UniWeave agent can call IntelliRAG query API mid-conversation
  - Input: user's message (the question being asked)
  - Output: relevant document chunks / answer from IntelliRAG
  - This result is injected into the agent's context before it generates a response

- [ ] Auth: set up API key / token from IntelliRAG (check swagger for auth method)

- [ ] Latency budget: IntelliRAG query must complete in <1 second to avoid noticeable voice delay
  - If >1s: run query in parallel with agent thinking (not sequentially)
  - Test latency with IntelliRAG team before full integration

- [ ] Fallback: if IntelliRAG is unavailable, agent falls back to system prompt knowledge gracefully (no error shown to user)

### Configuration

- [ ] IntelliRAG API key in ENV
- [ ] Knowledge base ID / namespace configurable per agent deployment
- [ ] Integration can be enabled/disabled per agent

### UI (blocked on wireframes — do not build yet)

- Agam to create wireframes showing how IntelliRAG is configured per agent
- UI spec will be added here once wireframes are done

---

## Acceptance criteria

**Happy path:**
- Given an active voice/chat conversation
- When the user asks a question that exists in the IntelliRAG knowledge base
- Then the agent's response includes accurate information from the knowledge base within normal response latency

**Fallback:**
- Given IntelliRAG API is down
- When the user asks a question
- Then the agent responds using system prompt knowledge only — no error, no degraded UX visible to user

**Latency:**
- Given a typical query
- When IntelliRAG is called
- Then the round-trip adds no more than 500ms to the agent's response time

---

## Day-by-day tasks (backend — start now)

**Day 1 (today)** — MCP server setup
- Read [livekit-agent-mcp-integration.md](livekit-agent-mcp-integration.md) — this is your integration contract
- Stand up IntelliRAG MCP server with `query_knowledge_base` tool
- Test: `curl http://intellirag-service:PORT/mcp` → tool list returns
- Share URL with Harsh for env var config

**Day 2** — Query function
- Build `queryIntelliRAG(userMessage)` function wired to IntelliRAG backend
  - Calls IntelliRAG with user's message
  - Returns top 1–3 relevant chunks
- Test with 10 sample questions, review quality of results

**Day 3** — Integration test
- Connect livekit-agent MCP client to IntelliRAG MCP server
- Test end-to-end: user asks question → IntelliRAG queried via MCP → agent answers with retrieved info
- Measure latency impact

**Day 4 onwards** — UI (waiting on Agam's wireframes)
- Agam to deliver wireframes → then build configuration UI

---

## Open questions

| Question | Answer needed from |
|---|---|
| IntelliRAG query endpoint and payload format | Swagger (already shared) |
| Auth method: API key? OAuth? | Swagger + IntelliRAG team |
| Latency: what is typical p95 response time for a query? | IntelliRAG team |
| Do they have a sandbox environment for testing? | IntelliRAG team |
| What knowledge bases are pre-loaded for our first use case? | IntelliRAG team / Agam |
| UI wireframes for per-agent IntelliRAG configuration | **Agam — blocking item** |

---

## Related

- [livekit-agent MCP Integration](livekit-agent-mcp-integration.md) — **read this first** — MCP contract between IntelliRAG and livekit-agent
- [Analytics & User Mgmt Explainer](../analytics-user-mgmt-explainer.md) — current ragQuery tool path (legacy)
- [System Architecture](../system-architecture.md)
- Trello #98
