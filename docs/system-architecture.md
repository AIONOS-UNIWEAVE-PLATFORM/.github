# UniWeave Platform — System Architecture

Reverse-engineered from all repos in UNIWEAVE-AIONOS and AIONOS-UNIWEAVE-PLATFORM (March 2026). Permanent reference for engineers, AI agents, and PMs. Last updated: 2026-03-06 (second pass — all 13 repos explored).

---

## Platform Overview

UniWeave is a real-time conversational execution platform for enterprise voice agents. The platform orchestrates LLM-driven voice sessions (via LiveKit + Ultravox or Azure Realtime), manages agent configuration, executes tools via HTTP during conversations, captures post-call data for analytics (IntelliPulse/Pulse), and is adding a Chat module (Azure OpenAI) alongside its Voice module.

The control point: **real-time orchestration, tool execution, latency, reliability.** Not telephony carrier, not CRM, not CCaaS routing.

---

## All Repositories

### UNIWEAVE-AIONOS (primary engineering repos)

| Repo | Language | Role | Port / URL |
|---|---|---|---|
| `livekit-agent` | Python 3.11 | **THE voice agent runtime.** Connects to LiveKit, loads config from Redis, registers tools, runs Ultravox or Azure Realtime session. MCP integration goes here. | — (worker pool, no inbound) |
| `intelliconverse-config-backend` | Node.js / Express | **Config API.** MongoDB + Redis. CRUD for agents, versions, tools, call stages, active configs. Source of truth for all agent definitions. | 5000 |
| `intelli-converse-tools-service` | Node.js / Express (ESM) | Omnichannel layer: outbound WhatsApp (Twilio), inbound WhatsApp webhook + WebSocket relay, FAQ/RAG (Pinecone + Azure OpenAI), WebRTC + PSTN call init via Ultravox, live-call LLM analytics, human call transfer. Reference implementation for #94/#95 Chat module. | 3001 (server.js) / 3000 (Dockerfile — mismatch) |
| `intelliconverse-livekit-frontend` | React | Platform UI (agent builder, tool registry, live rooms, analytics). | — |
| `analytics-user-mgmt` | Java 17, Spring Boot | Audit pipeline, user management, ragQuery tool, external API, API gateway. 8 modules. See [analytics-user-mgmt-explainer.md](analytics-user-mgmt-explainer.md). | 8080 (gateway), 7000 (audit-publisher), 8084 (tool-service), 8089 (user-mgmt) |
| `livekit-dispatcher` | Python, FastAPI | Webhook listener. On `participant_joined` → calls LiveKit Agent Dispatch API (`CreateAgentDispatchRequest`) to assign a pre-running `indigo-voice-bot` worker to the room. NOT a subprocess spawner. Handles cleanup on `participant_left`. | 3000 (Docker) / 3001 (direct) |
| `livekit-tokenizer` | Python, FastAPI | Generates LiveKit access tokens for clients. `POST /token` → `{token, room, identity}`. Uses bundled `lk` CLI binary (not SDK). No auth on endpoint — must be behind gateway in prod. | 8000 |
| `livekit-room-avail` | Python, FastAPI + Redis | SIP concurrency monitor. Counts rooms with `sip-` prefix by querying LiveKit directly (room existence = active call; participant count is unreliable for SIP). Redis = daily call aggregate only, not live count. | 8000 (internal) / 5555 (Docker example) |
| `livekit-server` | Config | LiveKit OSS server configuration and deployment. | — |
| `livekit-sip` | Config | SIP bridge for telephony (PSTN integration). | — |
| `czentrix-audiohook-livekit` | Config | C-Zentrix CCaaS audiohook integration point. | — |
| `intelliconverse-helm` | Helm charts | Kubernetes deployment manifests for all services. Chart name: `voice-ai` v0.1.0. | — |
| `intelliconverse-evals` | Node.js (NOT Python — mislabelled) | Multi-turn conversation eval framework wrapping [promptfoo](https://promptfoo.dev). REST API (`POST /api/evals/run`) triggers evals against live APIs. Dual assertion: tool-call accuracy (binary) + LLM rubric (judge LLM). Currently IndiGo-only coverage. Results persisted to MongoDB. | 6000 |

### AIONOS-UNIWEAVE-PLATFORM (platform org repos)

| Repo | Role |
|---|---|
| `.github` | Org defaults: CLAUDE.md, PR template, CI templates (Node.js + Python), org profile. |
| `voice-prompt-builder` | UniScript — prompt engineering tool (in progress, owned by Agam). |
| `ai-gateway` | LiteLLM proxy. Production: `gateway.uniweave.com`. Three virtual key teams: `uniweave-internal`, `intellipulse`, `uniscript`. Model: gpt-5-nano. |

---

## Data Flows

### 1. Config Flow

How agent definitions go from UI → agent runtime.

```
Platform UI (React)
  → config-backend (Node.js :5000)
    → MongoDB (voicebotdb-uniweave)
    → [on activation] inMemoryConfigService → Redis
    → livekit-agent reads Redis at call start (gen_config.py)
```

**Key detail:** Config is written to MongoDB on every save, but pushed to Redis only when a version is activated in the UI. The agent reads from Redis at the start of each call — not from MongoDB directly.

**Config payload in Redis contains:**
- `systemPrompt` — the agent's instructions
- `tools[]` — array of ConfigTool objects: `{name, baseUrl, endpoint, method, headers, params}`
- `modelConfig` — which model (Ultravox / Azure Realtime), voice settings
- `greetingMessage` — first thing the agent says
- `callStages` — multi-phase call flow (unused by IndiGo — Single Prompt Mode)

---

### 2. Call Flow (Voice)

Full lifecycle of a voice call.

```
Client
  → livekit-tokenizer (POST /token): get access token + room name
  → LiveKit Server: join room with token

LiveKit Server
  → livekit-dispatcher (POST /webhook): participant_joined event
  → livekit-dispatcher: CreateAgentDispatchRequest(agent_name=indigo-voice-bot, room, metadata)

IMPORTANT: Dispatcher does NOT spawn a subprocess.
indigo-voice-bot workers must be pre-running and connected to LiveKit,
polling for dispatch jobs. Dispatcher signals LiveKit to assign one idle worker.

LiveKit → assigns dispatch job to idle indigo-voice-bot worker

livekit-agent startup:
  1. Connect to LiveKit room
  2. Extract phone/email from SIP attributes
  3. gen_config.py → Redis lookup by phone/email → systemPrompt + tools + modelConfig
  4. tool_helper.py → generate @function_tool() Python code → exec()
  5. Create AgentSession (Ultravox RealtimeModel or Azure Realtime)
  6. session.say(greetingMessage)

Conversation loop:
  User speaks → STT (Ultravox handles natively)
  LLM decides to call tool → @function_tool() runs → httpx POST to external API
  Tool result injected into LLM context → LLM responds → TTS

Call ends:
  logs_payload.py → POST /api/audit-publisher/publish (analytics-user-mgmt:7000)
```

**Metadata as call context bus:** The dispatcher merges `participant.metadata` + `participant.attributes` from the LiveKit webhook into the dispatch metadata field. This is the only mechanism for passing per-call context (caller number, language, config ID) to the agent.

---

### 3. Tool Execution Path (Current — HTTP)

How tools defined in the UI become callable Python functions during a conversation.

```
config-backend stores:
  ConfigTool { name, baseUrl, endpoint, method, headers, params }

On activation → Redis (via inMemoryConfigService)

At call start (livekit-agent):
  gen_config.py → reads tools from Redis
  tool_helper.py → for each tool:
    generates Python code string:
      @function_tool(name="...", description="...")
      async def tool_fn(ctx, param1, param2):
          response = await httpx.post(baseUrl + endpoint, json={...})
          return response.json()
    exec(generated_code, globals())  ← code is executed at startup

During conversation:
  Ultravox LLM decides to call tool
  → Python @function_tool() executes
  → httpx makes HTTP call to external API
  → result returned to LLM as tool output
```

**Important:** The `exec()` approach generates one Python function per tool at agent startup. Tools are HTTP-first — any external API can be registered as a tool in the platform UI.

**Known gap:** `TOOL_BASE_URL` env var in the Helm chart is set to `"https://"` (no hostname). HTTP tool calls from the agent will fail unless this is corrected.

---

### 4. Tool Execution Path (Future — MCP)

After IntelliRAG MCP integration and Arjun's directive (all tools = native MCP):

```
MCP server (IntelliRAG, or any future tool):
  Exposes tool definitions + tool call endpoint via MCP protocol

livekit-agent at startup:
  1. Load HTTP tools from Redis (existing path, unchanged)
  2. Discover MCP tools from configured MCP server URLs
  3. Wrap MCP tool calls as @function_tool() Python functions
  4. Register both HTTP + MCP tools with AgentSession

During conversation:
  LLM calls tool by name
  → if HTTP tool: existing httpx path
  → if MCP tool: mcp_client.call_tool() → MCP server responds
  → result returned to LLM
```

Full MCP integration spec: [specs/livekit-agent-mcp-integration.md](specs/livekit-agent-mcp-integration.md)

---

### 5. Post-Call Data Flow

What happens after a conversation ends.

```
livekit-agent (logs_payload.py)
  → POST /api/audit-publisher/publish (Java :7000) with AuditStoreRequest
  → Kafka topic: audit-queue-voicebot-dev
  → audit-listener (Java :7001): saves to MongoDB (audit_store)
  → audit-analytics (Java :7002): dashboard queries
  → IntelliPulse Dashboard UI
```

**AuditStoreRequest payload includes:** call ID, agent ID, transcript, duration, tool calls made, outcome (contained/escalated), timestamps.

**IntelliPulse enrichment (#97):** Voice AI operational metadata (containment rate, escalation reason, tool invocation count, session duration) flows into Pulse's data model on top of the raw transcript. This is the `audit-analytics` layer — separate from the transport layer (#96).

**Known gap:** `audit-publisher` writes to MongoDB DB `audit-analytics`, but `audit-listener` and `audit-analytics` read from `audit-analytics-uniweave`. Data written by the publisher is not visible to the listener. This is a confirmed bug from both the Helm config and the Java explainer.

---

## Deployment Topology

Kubernetes cluster on **ACE Cloud (OpenStack-based)**. All services deployed via `intelliconverse-helm` Helm chart (chart name: `voice-ai`, v0.1.0). Private registry: `harbor.whilter.ai`. CI/CD: Jenkins (`helm upgrade --install`).

### Kubernetes Namespaces

| Namespace | Services |
|---|---|
| `voice-ai-dev` | api-gateway (8080), audit-analytics (7002), audit-listener (7001), audit-publisher (7000), external-api (8085), tool-service (8084), user-management (8089), intelliconverse-config-backend (5000), intelliconverse-livekit-frontend (3000) |
| `livekit` | livekit-server (7880/7881/7801), livekit-dispatcher (3000), livekit-tokenizer (8000), livekit-egress (8080 health / 9090 metrics), redis StatefulSet (6379) |
| `agent` | livekit-agent (no inbound service — outbound only, worker pool) |
| `redis` | redis StatefulSet (emptyDir — non-persistent) |
| `stunner` | STUNner STUN/TURN gateway |

### Public URLs (confirmed from Helm)

| URL | Routes to | Ingress |
|---|---|---|
| `demo.uniweave.com` | intelliconverse-livekit-frontend | ALB NGINX |
| `routesvr.uniweave.com` | api-gateway (Java) | ALB NGINX |
| `routenode.uniweave.com` | intelliconverse-config-backend (Node.js) | ALB NGINX |
| `lkagentsvr.uniweave.com` | livekit-server | LiveKit NGINX |
| `lkagentsvr.uniweave.com/webhook` | livekit-dispatcher | LiveKit NGINX |
| `lkagentsvr.uniweave.com/token` | livekit-tokenizer | LiveKit NGINX |
| `stunner.uniweave.com:3478` | STUNner STUN/TURN (UDP) | Direct |
| `stunner.uniweave.com:5349` | STUNner STUN/TURN (TLS) | Direct |
| `sip.livekit.uniweave.com` | SIP/room avail API (separate deploy) | External |
| `gateway.uniweave.com` | AI gateway / LiteLLM proxy | Azure Container Apps (South India) |

### External Dependencies (outside K8s cluster)

| Service | Address | Used by |
|---|---|---|
| MongoDB | `154.201.126.202:27017` (bare metal) | All Java services, config-backend |
| Kafka | `43.204.24.97:9092` | audit-publisher, audit-listener |
| Redis (app) | `154.201.126.202:6380` | config-backend, livekit-agent |
| Redis (LiveKit internal) | `45.194.90.57:6379` | livekit-server, livekit-egress |
| STT service | `45.194.2.197:5001` | livekit-agent |
| LLM service | `154.201.127.0:5000` | livekit-agent |
| TTS service | `45.194.2.197:5008` | livekit-agent |
| Azure Blob | `voicestorageuni` / `call-recordings` | livekit-egress (recordings) |

Note: Three distinct Redis addresses serve different purposes. The in-chart Redis StatefulSet (`redis` namespace) uses `emptyDir` — data lost on pod restart.

### Resource Limits

| Service | CPU | Memory | Notes |
|---|---|---|---|
| **livekit-agent** | **4 cores** | **16Gi + 16Gi ephemeral** | By far the largest workload — real-time AI inference |
| Java/Node services (most) | 250m | 256Mi | All equal, likely under-tuned |
| livekit-dispatcher / tokenizer | 100m | 256Mi | Lightweight webhook/token |
| livekit-egress | None set | None set | Unconstrained — runs Chrome for recording |

### Image Versions (as of Helm chart)

| Service | Image Tag |
|---|---|
| intelliconverse-livekit-frontend | 0.0.83 |
| intelliconverse-config-backend | 0.0.19 |
| livekit-agent | 0.0.11 |
| audit-publisher | 0.0.7 |
| audit-analytics | 0.0.10 |
| livekit-server (OSS) | v1.9.0 |
| livekit-egress | v1.9.0 |

---

## Known Production Gaps (from Helm + code review, March 2026)

These were found during the second-pass exploration. Flagged here so engineers don't need to rediscover them.

| # | Gap | Severity | Where |
|---|---|---|---|
| 1 | **Secrets in values.yaml plaintext** — all AI keys, DB passwords, Azure credentials, Google OAuth secret in the Helm repo | Critical | `intelliconverse-helm/values.yaml` |
| 2 | **`TOOL_BASE_URL` is broken** — `livekit-agent` has `TOOL_BASE_URL: "https://"` (no hostname). HTTP tool calls from agent will fail. | Critical | Helm livekit-agent env |
| 3 | **audit-publisher MongoDB DB mismatch** — publisher writes to `audit-analytics`, listener/analytics read from `audit-analytics-uniweave`. Data is dropped. | Critical | Helm + analytics-user-mgmt |
| 4 | **No webhook signature verification** on livekit-dispatcher — any POST triggers a real agent dispatch | High | `livekit-dispatcher/dispatcher_webhook.py` |
| 5 | **No auth on call/LLM endpoints** in tools-service — `POST /api/v1/call/*` and `/api/v1/llm/*` are open | High | `intelli-converse-tools-service` |
| 6 | **No liveness/readiness probes** on Java/Node pods — Kubernetes cannot detect hung processes | High | `intelliconverse-helm` (all but egress) |
| 7 | **Missing voice-call service** — api-gateway routes to `http://voice-call.voice-ai-dev.svc.cluster.local` but no such deployment exists | High | Helm api-gateway config |
| 8 | **livekit-egress has no resource limits** — Chrome browser pod is unconstrained | High | Helm livekit-egress |
| 9 | **livekit-tokenizer: env.example uses wrong var names** — `API_KEY`/`API_SECRET` vs code reads `LIVEKIT_API_KEY`/`LIVEKIT_API_SECRET` | High | `livekit-tokenizer` |
| 10 | **Three Redis instances** with no documentation — 154.201.126.202:6380 (app), 45.194.90.57:6379 (LiveKit/egress), in-chart StatefulSet (emptyDir, apparently unused by LiveKit) | Medium | Helm |
| 11 | **livekit-dispatcher in-memory dispatch set** — service restart loses dedup state, potential double-dispatch on reconnect | Medium | `livekit-dispatcher` |
| 12 | **`CONCURRENCY_LIMIT` default is 1** in livekit-room-avail — any deploy without this env var reports busy after first call | Medium | `livekit-room-avail` |
| 13 | **No daily-count auto-increment in production** livekit-room-avail — only manual/test variant has it | Low | `livekit-room-avail` |
| 14 | **Twilio webhook auth bypassed** (`skipTwilioAuth: true`) in config-backend Helm config | Medium | Helm config-backend env |
| 15 | **livekit-tokenizer: roomRecord:true on all tokens** — every participant token grants recording permission | Low | `livekit-tokenizer` |
| 16 | **Evals call live APIs** — eval runs fail if IntelliConverse backend is down | Low | `intelliconverse-evals` |

---

## Answers to Common Questions

### How does IntelliRAG connect to the agent?

**Current path:**
IntelliRAG exposes a `ragQuery` REST endpoint via `analytics-user-mgmt/tool-service` (port 8084). This endpoint is registered as one of the 13 platform tools in `config-backend`. The `livekit-agent` calls it via the generated `@function_tool()` httpx path — the same way it calls any other tool.

**Future path (MCP):**
IntelliRAG will expose a native MCP server. The `livekit-agent` will discover and call it via `mcp_client.py` (to be built, spec at [specs/livekit-agent-mcp-integration.md](specs/livekit-agent-mcp-integration.md)). This eliminates the indirect `analytics-user-mgmt` route and gives IntelliRAG direct MCP tool registration.

### How do you change the Ultravox (or Azure Realtime) settings?

In the Platform UI → Agent → Model tab. Settings are stored in `config-backend` MongoDB, pushed to Redis on activation, read by `gen_config.py` in `livekit-agent` at call start. Changes take effect on the next call (not mid-call).

The `modelConfig` in Redis specifies: provider (Ultravox / Azure Realtime), model ID, voice ID (ElevenLabs for Ultravox), temperature, language, interruption sensitivity.

### What does the livekit-agent actually do during a call?

1. Connects to the LiveKit room (WebRTC)
2. Reads agent config from Redis
3. Generates Python tool functions via `exec()`
4. Creates an AgentSession with the LLM (Ultravox handles STT + LLM + TTS natively; Azure Realtime is Azure's equivalent)
5. Says the greeting
6. Listens to the user, passes audio to LLM, LLM decides to call tools or respond
7. Tool calls are executed by the Python `@function_tool()` functions (HTTP calls)
8. After call ends: posts audit data to `analytics-user-mgmt`

### Where do tool definitions live?

**Platform tool registry** → `config-backend` MongoDB (`voicebotdb-uniweave`) → `tools` collection. The UI (Platform UI → Tools tab) is the management surface. 13 tools currently registered org-wide. Each tool is: `{ name, description, baseUrl, endpoint, method, headers[], params[] }`.

Tools are attached to agents via the Agent → Workflow tab. At activation, the agent's tool list is pushed to Redis with the rest of the config.

### How does auth work?

- Platform UI → config-backend: session/JWT
- config-backend → MongoDB/Redis: connection string auth
- livekit-agent → Redis: connection string
- livekit-agent → external tool APIs: per-tool headers (API keys defined in the tool definition in config-backend)
- livekit-agent → audit-publisher: no auth (internal service)
- ai-gateway: LiteLLM virtual keys per team (`uniweave-internal`, `intellipulse`, `uniscript`)
- analytics-user-mgmt/user-management module: owns user auth for the platform (port 8089)

**Gap:** Security configs not fully configured in analytics-user-mgmt. Review before production exposure.

### What is UniScript?

A prompt engineering tool (`voice-prompt-builder` repo). In progress, owned by Agam. Uses `ai-gateway` (`gateway.uniweave.com`) with the `uniscript` virtual key. Helps build system prompts for UniWeave agents.

### How does livekit-tokenizer work?

`POST /token` with optional `{"metadata": {}}` body. Returns `{token, room, identity}` where both room and identity are auto-generated per request. The room name is `room-YYYYMMDD-<8-hex>` — a **new room is always created per token**. There is no way to request a token for an existing room.

Implementation: shells out to the bundled `lk` CLI binary (`lk token create`). Token grants: join, update-metadata, `roomRecord: true` (recording enabled on every token — intentional?), 1h validity.

**Prod gaps:** No auth on `/token`; CORS `allow_origins=["*"]`; `env.example` uses wrong var names; no `requirements.txt`; 57MB binary committed to git.

Env vars: `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LK_PATH` (optional), `LOG_LEVEL`.

### What is the livekit-dispatcher?

A FastAPI service (`POST /webhook`) that listens to LiveKit webhook events. When a real participant joins a room, it calls the **LiveKit Agent Dispatch API** (`CreateAgentDispatchRequest`) to assign an idle `indigo-voice-bot` worker to that room.

**Critical detail:** This service does NOT spawn subprocesses or start agents. The `indigo-voice-bot` agent processes must already be running and connected to LiveKit, polling for dispatch jobs. The dispatcher just signals LiveKit to assign one waiting worker to the new room.

**Prod gaps:** No webhook signature verification; in-memory dispatch dedup set (lost on restart); port mismatch (Dockerfile:3000, `__main__`:3001); accepts `participant_joined` from ANY room (not just `sip-` rooms).

Env vars: `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_URL`, `PORT`, `LOG_LEVEL`.

### What is the livekit-room-avail service?

Monitors active SIP call concurrency by querying LiveKit directly. Exposes `GET /get_busy_status` → `{active_call_count, concurrency_limit, is_busy}`. Call routing decisions are made against this.

**Key design point:** For SIP calls, room existence = active call (SIP rooms routinely show 0 participants even with a live call). The service counts rooms whose names start with `sip-`.

**Redis role:** Redis is used only for a daily aggregate call counter (`GET /daily_count`), NOT for live concurrency tracking. Live concurrency is queried fresh from LiveKit on every request.

**Concurrency limit:** `CONCURRENCY_LIMIT` env var (default `1` — any deployment without this set reports busy after the first SIP call). Example file sets `5`.

Env vars: `LIVEKIT_API_KEY`, `LIVEKIT_API_SECRET`, `LIVEKIT_HTTP_URL`, `CONCURRENCY_LIMIT`, `REDIS_HOST/PORT/DB/PASSWORD`.

### What does intelli-converse-tools-service do? Is it the Chat Engine?

No — it is NOT the Chat Engine (#94). It is a standalone omnichannel integration service built as a proof-of-concept/for the IndiGo airline customer. It is the closest working reference implementation for #94/#95.

**What it implements (all working):**
- Outbound WhatsApp via Twilio WABA: `POST /api/v1/whatsapp/messages/send`
- Inbound WhatsApp webhook handling with WebSocket relay to browser/mobile clients
- FAQ/RAG: Azure OpenAI embedding → Pinecone search (topK=3, score thresholds 0.85/0.75) → Azure GPT completion
- Document ingestion: PDF/TXT → LangChain chunk → embed → Pinecone upsert
- WebRTC call creation: `POST /api/v1/call/web` → Ultravox API → browser-joinable URL
- PSTN call creation: `POST /api/v1/call/telephony` → Ultravox API → Twilio dials user → TwiML bridges audio
- Human call transfer: `POST /api/v1/call/transfer` → updates live Twilio call mid-conversation
- Live-call LLM analytics: sentiment, intent, summary from partial transcripts

**Key patterns for #95 (Nomaan):** Twilio webhook validation (`twilioAuth.js`) is production-ready and directly reusable. Outbound WA: `twilioClient.messages.create({body, from:"whatsapp:+SENDER", to:"whatsapp:+RECIPIENT"})`. Inbound: POST webhook → parse → relay to WS client by phone number.

**Prod gaps:** Port mismatch (server.js:3001, Dockerfile:3000); no auth on call/LLM routes; `twilioActiveCalls` Map is in-memory; no database; sandbox WA number in config.

### What is intelliconverse-evals?

A Node.js REST API server (port 6000) wrapping [promptfoo](https://promptfoo.dev) for multi-turn conversation evaluations.

**How it works:** `POST /api/evals/run` with a conversation (array of turns with `query`, `tool_calls_expectation`, `output_expectation`). Two assertion types per turn: tool-call accuracy (binary 0/1) + LLM rubric (judge LLM, 0–1 float). Results stored in MongoDB. Primary LLM under test: Groq Llama-3.3-70B.

**Coverage as of March 2026:** IndiGo Airlines only. 4 test files: `search_flight` (3 conversations) + `flight_status` (1 conversation). No CI/CD integration — manual trigger only. Evals call live production/dev APIs (no mock layer).

**For PRISM Hotels (#113) / Chat module:** separate evals must be built — this repo is IndiGo-specific throughout.

### What is C-Zentrix / czentrix-audiohook-livekit?

An integration with C-Zentrix's CCaaS platform. Their audiohook connects to LiveKit so UniWeave voice agents can handle calls routed through C-Zentrix. Config-only repo — the integration happens at the infrastructure level.

### How does IndiGo (the airline) use UniWeave?

IndiGo uses Single Prompt Mode (Call Stages feature exists but is not used). One agent handles the full call. Their tools include flight lookup, booking modification, and customer identification. The `indigo-voice-bot` is the specific agent that livekit-dispatcher dispatches for their calls.

---

## Architecture Conventions (Arjun Directive, 2026-03-06)

**Every new capability = MCP server first.**

Agents consume tools via MCP. IntelliRAG already has MCP. All future tools follow this pattern. The `livekit-agent` is the MCP client — it discovers and calls MCP servers at call start, alongside the existing HTTP tool path.

This sets the integration pattern for: IntelliRAG, IntelliPulse data read, any new tool integration.

---

## Exploration Coverage

| Repo | Depth | Last explored |
|---|---|---|
| `livekit-agent` | ✅ Deep | 2026-03-06 |
| `intelliconverse-config-backend` | ✅ Deep | 2026-03-06 |
| `analytics-user-mgmt` | ✅ Deep | 2026-03-05 |
| `intelliconverse-livekit-frontend` | ✅ UI-mapped | 2026-03-06 |
| `intelli-converse-tools-service` | ✅ Deep | 2026-03-06 |
| `livekit-dispatcher` | ✅ Deep | 2026-03-06 |
| `livekit-tokenizer` | ✅ Deep | 2026-03-06 |
| `livekit-room-avail` | ✅ Deep | 2026-03-06 |
| `intelliconverse-helm` | ✅ Deep | 2026-03-06 |
| `intelliconverse-evals` | ✅ Deep | 2026-03-06 |
| `livekit-server` | ⚙️ Config-only | 2026-03-06 |
| `livekit-sip` | ⚙️ Config-only | 2026-03-06 |
| `czentrix-audiohook-livekit` | ⚙️ Config-only | 2026-03-06 |
