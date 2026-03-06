# UniWeave Platform — System Architecture

Reverse-engineered from all repos in UNIWEAVE-AIONOS and AIONOS-UNIWEAVE-PLATFORM (March 2026). Permanent reference for engineers, AI agents, and PMs. Last updated: 2026-03-06.

---

## Platform Overview

UniWeave is a real-time conversational execution platform for enterprise voice agents. The platform orchestrates LLM-driven voice sessions (via LiveKit + Ultravox or Azure Realtime), manages agent configuration, executes tools via HTTP during conversations, captures post-call data for analytics (IntelliPulse/Pulse), and is adding a Chat module (Azure OpenAI) alongside its Voice module.

The control point: **real-time orchestration, tool execution, latency, reliability.** Not telephony carrier, not CRM, not CCaaS routing.

---

## All Repositories

### UNIWEAVE-AIONOS (primary engineering repos)

| Repo | Language | Role | Port / URL |
|---|---|---|---|
| `livekit-agent` | Python 3.11 | **THE voice agent runtime.** Connects to LiveKit, loads config from Redis, registers tools, runs Ultravox or Azure Realtime session. MCP integration goes here. | — (spawned as process) |
| `intelliconverse-config-backend` | Node.js / Express | **Config API.** MongoDB + Redis. CRUD for agents, versions, tools, call stages, active configs. Source of truth for all agent definitions. | 5000 |
| `intelli-converse-tools-service` | Node.js / Express | Omnichannel layer: WhatsApp (Twilio), FAQ/RAG (Pinecone), WebRTC + PSTN call init. NOT a tool executor during calls — a separate service. | — |
| `intelliconverse-livekit-frontend` | React | Platform UI (agent builder, tool registry, live rooms, analytics). | — |
| `analytics-user-mgmt` | Java 17, Spring Boot | Audit pipeline, user management, ragQuery tool, external API, API gateway. 8 modules. Full reference: [analytics-user-mgmt-explainer.md](analytics-user-mgmt-explainer.md). | 8080 (gateway), 7000 (audit-publisher), 8084 (tool-service), 8089 (user-mgmt) |
| `livekit-dispatcher` | Python, FastAPI | Webhook listener. On `participant_joined` event from LiveKit → auto-spawns `indigo-voice-bot` agent into the room. | — |
| `livekit-tokenizer` | Python, FastAPI | Generates LiveKit access tokens for clients joining rooms. | — |
| `livekit-room-avail` | Python, FastAPI + Redis | SIP call concurrency monitor. Tracks active sessions, enforces capacity limits. | — |
| `livekit-server` | Config | LiveKit OSS server configuration and deployment. | — |
| `livekit-sip` | Config | SIP bridge for telephony (PSTN integration). | — |
| `czentrix-audiohook-livekit` | Config | C-Zentrix CCaaS audiohook integration point. | — |
| `intelliconverse-helm` | Helm charts | Kubernetes deployment manifests for all services. | — |
| `intelliconverse-evals` | Python | Evaluation framework for agent quality testing. | — |

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
  → livekit-tokenizer: request token
  → LiveKit Server: join room (with token)
  → livekit-dispatcher: webhook participant_joined
  → livekit-agent: spawn indigo-voice-bot

livekit-agent startup:
  1. Connect to LiveKit room
  2. Extract phone/email from SIP attributes
  3. gen_config.py → Redis lookup → systemPrompt + tools + modelConfig
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

---

## Deployment Topology

What runs where (as of March 2026):

| Component | Deployment | Notes |
|---|---|---|
| LiveKit server | Self-hosted / cloud | Core WebRTC infrastructure |
| livekit-agent | Container (spawned per call) | Python 3.11 process |
| livekit-dispatcher | Container (always-on) | Webhook listener |
| livekit-tokenizer | Container (always-on) | Token service |
| livekit-room-avail | Container (always-on) | Concurrency monitor |
| config-backend | Container | Node.js API :5000 |
| analytics-user-mgmt | Container | 8 Spring Boot modules |
| MongoDB | Managed (Atlas or self-hosted) | `voicebotdb-uniweave`, `audit_store` |
| Redis | Managed | Config cache |
| Kafka | Managed | `audit-queue-voicebot-dev` |
| ai-gateway | Azure Container Apps (South India) | `gateway.uniweave.com` |
| Platform UI | Static hosting | React app |
| intelli-converse-tools-service | Container | WhatsApp / WebRTC / FAQ |
| Kubernetes | All services via Helm | `intelliconverse-helm` |

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

### What is the livekit-dispatcher?

A FastAPI service that listens to LiveKit webhook events. When a participant joins a room, it spawns the `indigo-voice-bot` agent into that room. This is the trigger that starts every voice call.

### What is C-Zentrix / czentrix-audiohook-livekit?

An integration with C-Zentrix's CCaaS platform. Their audiohook connects to LiveKit so UniWeave voice agents can handle calls routed through C-Zentrix. Config-only repo.

### How does IndiGo (the airline) use UniWeave?

IndiGo uses Single Prompt Mode (Call Stages feature exists but is not used). One agent handles the full call. The `indigo-voice-bot` is the specific agent that livekit-dispatcher spawns for their calls.

---

## Architecture Conventions (Arjun Directive, 2026-03-06)

**Every new capability = MCP server first.**

Agents consume tools via MCP. IntelliRAG already has MCP. All future tools follow this pattern. The `livekit-agent` is the MCP client — it discovers and calls MCP servers at call start, alongside the existing HTTP tool path.

This sets the integration pattern for: IntelliRAG, IntelliPulse data read, any new tool integration.
