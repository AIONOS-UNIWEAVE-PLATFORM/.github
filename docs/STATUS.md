# UniWeave Engineering Status

Last updated: 2026-03-07

---

## In Progress

| Card | What | Owner | Due | Checklist | Notes |
|---|---|---|---|---|---|
| #94 | Chat Engine — Azure OpenAI LLM + session management | Nomaan | Mar 14 | **8/12** | Major progress. 4 items remaining — confirm what's left Monday. |
| #96 | IntelliPulse — Post-call webhook | Upender | Mar 10 | 0/4 (misleading) | **Functionally complete** per Upender. Cron-based approach (not event-driven as specced). Blocked on Java service routing (#122). Edge cases pending. **Monday: live demo + review. Discuss: add force-refresh trigger alongside cron for immediate post-call flush.** |
| #98 | IntelliRAG — Query integration (MCP server) | Himanshu | Mar 10 | **11/11** | Checklist complete. v0.1 built. Review + feedback cycle next. May be ready to move to Done after review. |
| #118 | UniScript — voice AI prompt engineering service | Agam | — | — | Planning phase. |
| #120 | Unified auth — SSO for all services | Himanshu + Upender | Mar 12 | 0/5 | Himanshu waiting on Pramod (IntelliRAG engineer) to finish global DB implementation. Then starts. |

## Ready to Build

| Card | What | Owner | Due | Status |
|---|---|---|---|---|
| #95 | WhatsApp connector (WABA) | Nomaan | Mar 19 | Blocked — WABA not provisioned (#108) |
| #97 | Pulse Voice AI enrichment — operational metrics pipeline | **TBD** | Mar 14 | Unassigned. Blocked by #96. |
| #99 | IntelliRAG config UI per agent | **TBD** | — | Unassigned. Waiting on #98 review + Agam wireframes. |
| #104 | Bug: Arabic hardcoded greeting (Azure-realtime model) | **TBD** | — | Unassigned. No due date. Quick fix. |
| #108 | WABA Provisioning | Ravinder (not on Trello) | Mar 10 | Agam coordinating with Ravinder. Blocks #95. |
| #113 | PRISM Hotels — WhatsApp concierge | **TBD** | Mar 10 | Unassigned. Depends on #94 + #95 + #108. |
| #122 | Java service routing fixes — gateway port mismatch + hardcoded tool URLs | Upender + Himanshu | Mar 10 | 3 bugs from architecture deep-dive. Blocks #96. |

## Critical Production Gaps (from system architecture review)

These were surfaced during the full 13-repo architecture deep-dive (2026-03-06). **Not yet communicated to engineering — escalate Monday.**

1. **TOOL_BASE_URL broken in Helm** — `livekit-agent` Helm config sets `TOOL_BASE_URL: "https://"`. All HTTP tool calls from the agent to config-backend tool endpoints fail silently in production.
2. **Helm secrets in plaintext** — `intelliconverse-helm/values.yaml` contains API keys, passwords, tokens in plain text (committed to repo). Must migrate to Kubernetes Secrets or external secrets manager.
3. **audit-publisher MongoDB DB mismatch** — audit-publisher writes to `audit-analytics` DB. audit-listener reads from `audit-analytics-uniweave`. Post-call data is silently dropped. No audit trail for any call.

Reference: [system-architecture.md](system-architecture.md)

## Key Architecture Notes

- **MCP = universal tool interface** (Arjun directive, 2026-03-06). Every new capability = MCP server first. IntelliRAG is the first implementation.
- **Chat module uses Azure OpenAI** (not Ultravox). Channel-agnostic Chat API → WhatsApp connector.
- **UniWeave is master for auth/RBAC** — all services (RAG, Pulse, future modules) inherit orgs + users + roles from UniWeave.
- **IntelliPulse integration**: Upender built cron-based approach (ingestion every 10 min + processing cron). Architecture note: add force-refresh trigger alongside cron for immediate post-call flush.

## Upcoming

- **CX Ops floor session** — March 9. Required for roadmap finalization.
- **Monday actions**: Demo #96 with Upender, review #98 with Himanshu, check #94 remaining items with Nomaan, escalate 3 Helm gaps to Harsh.
