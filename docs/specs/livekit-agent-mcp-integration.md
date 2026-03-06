# livekit-agent MCP Integration

**Status**: Ready to Build
**Priority**: P0 — enables IntelliRAG integration (#98) and all future MCP tools
**Owner**: Harsh Vardhan Mishra (engineering lead) + Himanshu Arora (IntelliRAG)
**Trello**: #98 IntelliRAG backend (`69a9295e4063c49485e9fc1b`)
**Repo**: `UNIWEAVE-AIONOS/livekit-agent`
**Estimated effort**: 2–3 days for a senior Python engineer

---

## Why This Exists

Arjun's directive (2026-03-06): **all tools = native MCP**. IntelliRAG will expose a native MCP server. The `livekit-agent` must be able to discover and call it.

This spec explains:
1. What the current tool architecture looks like
2. What needs to change to support MCP
3. Exactly what to build and where

Read this before touching any code in `livekit-agent`.

---

## Current Tool Architecture (What You're Inheriting)

### How tools reach the agent today

```
Platform UI → config-backend (Node.js :5000)
    → MongoDB: stores ConfigTool { name, description, baseUrl, endpoint, method, headers, params }
    → Redis: pushed on config activation (inMemoryConfigService)

livekit-agent at call start:
    gen_config.py → reads from Redis → returns agent config dict
    tool_helper.py → for each tool in config:
        generates Python code string:
            @function_tool(name="tool_name", description="...")
            async def tool_fn(ctx, **kwargs):
                response = await httpx.post(baseUrl + endpoint, json=kwargs)
                return response.json()
        exec(generated_code, globals())  ← runs at startup

agent.py:
    tools = [list of generated @function_tool() callables]
    agent = MyAgent(tools=tools, instructions=systemPrompt)
    session = AgentSession(llm=UltravoxRealtimeModel(...))
    await session.start(agent=agent, room=room)
```

### Key files in livekit-agent

| File | What it does |
|---|---|
| `agent.py` | Main entrypoint. Connects to LiveKit, loads config, assembles agent, starts session. |
| `gen_config.py` | Reads agent config from Redis. Returns dict with systemPrompt, tools, modelConfig, greetingMessage. |
| `tool_helper.py` | Generates Python @function_tool() code strings. Executes them via exec(). |
| `logs_payload.py` | Post-call: sends audit data to analytics-user-mgmt. |
| `requirements.txt` | Python dependencies. |

---

## What MCP Changes and Why

**MCP (Model Context Protocol)** is a standard protocol for AI agents to discover and call tools exposed by external servers. Instead of UniWeave defining what IntelliRAG does (HTTP endpoint, params), IntelliRAG itself declares its tools via MCP. The agent discovers them at startup and calls them natively.

**Benefits:**
- IntelliRAG team owns their tool definitions — no config-backend changes needed
- Adding a new capability = deploy a new MCP server, not a code change in livekit-agent
- Cleaner, standardized interface for all future tool integrations
- Scales to any MCP-compliant server (IntelliRAG, IntelliPulse, external APIs)

**What does NOT change:**
- The existing HTTP tool path (exec'd @function_tool functions) stays exactly as-is
- Existing agent config, Redis, MongoDB — unchanged
- AgentSession, Ultravox, Azure Realtime — unchanged

**What is new:**
- `mcp_client.py` — new file: discovers MCP server tools, wraps them as @function_tool callables
- `agent.py` — small addition: after HTTP tools are loaded, also load MCP tools
- `requirements.txt` — add `mcp` package

---

## Implementation

### Step 1: Add `mcp` to requirements.txt

```
mcp>=1.0.0
```

The official Python MCP SDK from Anthropic. Install: `pip install mcp`.

---

### Step 2: Create `mcp_client.py`

New file: `UNIWEAVE-AIONOS/livekit-agent/mcp_client.py`

```python
"""
MCP client for livekit-agent.
Discovers tools from MCP servers and wraps them as livekit @function_tool callables.
"""
import asyncio
import logging
from typing import Any
from mcp import ClientSession
from mcp.client.sse import sse_client
from mcp.client.stdio import stdio_client, StdioServerParameters
from livekit.agents.llm import function_tool

logger = logging.getLogger(__name__)


async def discover_mcp_tools(server_configs: list[dict]) -> list:
    """
    Discover tools from one or more MCP servers.

    server_configs: list of dicts, each with:
        - type: "sse" | "stdio"
        - url: str (for SSE servers, e.g. "http://intellirag-service/mcp")
        - name: str (human label for logging)

    Returns: list of @function_tool() callables ready for AgentSession.
    """
    all_tools = []
    for server_config in server_configs:
        try:
            tools = await _load_tools_from_server(server_config)
            all_tools.extend(tools)
            logger.info(f"MCP: loaded {len(tools)} tools from {server_config.get('name', server_config.get('url'))}")
        except Exception as e:
            logger.error(f"MCP: failed to load tools from {server_config}: {e}")
            # Non-fatal: agent starts without MCP tools if server is unavailable
    return all_tools


async def _load_tools_from_server(server_config: dict) -> list:
    """Connect to one MCP server, list its tools, return wrapped callables."""
    server_type = server_config.get("type", "sse")

    if server_type == "sse":
        return await _load_sse_tools(server_config["url"])
    elif server_type == "stdio":
        return await _load_stdio_tools(server_config)
    else:
        raise ValueError(f"Unknown MCP server type: {server_type}")


async def _load_sse_tools(url: str) -> list:
    """Load tools from an HTTP SSE MCP server."""
    async with sse_client(url) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tool_list = await session.list_tools()
            return [_wrap_sse_tool(url, tool) for tool in tool_list.tools]


def _wrap_sse_tool(server_url: str, tool_def) -> Any:
    """Wrap a single MCP tool definition as a livekit @function_tool callable."""
    tool_name = tool_def.name
    tool_description = tool_def.description or f"MCP tool: {tool_name}"

    @function_tool(name=tool_name, description=tool_description)
    async def mcp_tool_fn(ctx, **kwargs):
        """Dynamically created MCP tool wrapper."""
        try:
            async with sse_client(server_url) as (read, write):
                async with ClientSession(read, write) as session:
                    await session.initialize()
                    result = await session.call_tool(tool_name, kwargs)
                    # MCP returns content list; extract text
                    if result.content:
                        parts = [c.text for c in result.content if hasattr(c, "text")]
                        return "\n".join(parts)
                    return ""
        except Exception as e:
            logger.error(f"MCP tool call failed [{tool_name}]: {e}")
            return f"[Tool unavailable: {tool_name}]"

    return mcp_tool_fn


async def _load_stdio_tools(server_config: dict) -> list:
    """Load tools from a stdio MCP server (for local/subprocess servers)."""
    params = StdioServerParameters(
        command=server_config["command"],
        args=server_config.get("args", []),
        env=server_config.get("env"),
    )
    async with stdio_client(params) as (read, write):
        async with ClientSession(read, write) as session:
            await session.initialize()
            tool_list = await session.list_tools()
            return [_wrap_sse_tool(None, tool) for tool in tool_list.tools]
```

**Design decisions:**
- Each tool call re-establishes the MCP session. Intentional: avoids stale sessions across long calls. For production, IntelliRAG should expose SSE (HTTP), not stdio.
- Failures are non-fatal: if IntelliRAG is unavailable, the agent starts without MCP tools. Existing HTTP tools still work.
- The wrapper uses `**kwargs` to accept arbitrary tool parameters without knowing them at definition time.

---

### Step 3: Modify `agent.py`

Add MCP tool discovery after HTTP tools are generated. Minimal changes — add 5–10 lines.

```python
# At the top of agent.py, add:
import os
from mcp_client import discover_mcp_tools

# In the main() async function, after gen_config() and tool_helper.generate_tools():

async def main():
    room = await connect_to_livekit()
    config = await gen_config(phone_number)

    # Existing HTTP tools (unchanged)
    http_tools = tool_helper.generate_tools(config.tools)

    # NEW: MCP tools
    mcp_server_configs = _get_mcp_server_configs(config)
    mcp_tools = []
    if mcp_server_configs:
        mcp_tools = await discover_mcp_tools(mcp_server_configs)
        logger.info(f"MCP tools loaded: {[t.__name__ for t in mcp_tools]}")

    all_tools = http_tools + mcp_tools

    agent = MyAgent(
        templates=config.templates,
        instructions=config.systemPrompt,
        tools=all_tools,  # match existing pattern
    )
    # ... rest unchanged


def _get_mcp_server_configs(config) -> list[dict]:
    """
    Get MCP server configs from:
    1. Agent config (from Redis/config-backend) — per-agent MCP servers
    2. Environment variables — platform-wide MCP servers
    """
    servers = []

    # Option A: from Redis config (per-agent, preferred long-term)
    if hasattr(config, 'mcpServers') and config.mcpServers:
        servers.extend(config.mcpServers)

    # Option B: from environment variables (platform-wide, faster to ship)
    mcp_urls_env = os.environ.get("MCP_SERVER_URLS", "")
    if mcp_urls_env:
        for url in mcp_urls_env.split(","):
            url = url.strip()
            if url:
                servers.append({"type": "sse", "url": url, "name": url})

    return servers
```

---

### Step 4: Configure MCP Server URLs

For the initial integration (fastest path), use environment variable injection.

**Environment variable:** `MCP_SERVER_URLS`

```bash
# Single MCP server:
MCP_SERVER_URLS=http://intellirag-service:8090/mcp

# Multiple MCP servers (comma-separated):
MCP_SERVER_URLS=http://intellirag-service:8090/mcp,http://other-service:8091/mcp
```

Set this in:
- `intelliconverse-helm` — Helm values for the livekit-agent deployment
- `.env` — local development

**Long-term (Phase 2):** Add `mcpServers` field to the agent config in `config-backend` / Redis, so IntelliRAG can be enabled/disabled per agent from the UI. Not needed for v1.

---

### Step 5: IntelliRAG MCP Server (Himanshu's side)

The contract that livekit-agent expects from IntelliRAG:

**Required:**
- MCP server accessible over HTTP SSE at a stable internal URL (e.g. `http://intellirag-service:PORT/mcp`)
- `list_tools` returns at least one tool: `query_knowledge_base`
- `call_tool("query_knowledge_base", {"query": "...", "namespace": "..."})` returns text content

**Suggested MCP tool definition:**
```json
{
  "name": "query_knowledge_base",
  "description": "Search the knowledge base for information relevant to the user's question. Returns the most relevant document chunks.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "The user's question or search query"
      },
      "namespace": {
        "type": "string",
        "description": "Knowledge base namespace or collection to search (optional — defaults to agent's configured KB)"
      }
    },
    "required": ["query"]
  }
}
```

**IntelliRAG MCP server framework recommendation:** Use `mcp` Python SDK with FastAPI transport, or the `fastmcp` library for quick setup.

```python
# IntelliRAG MCP server (example — Himanshu implements this)
from mcp.server import Server
from mcp import types

server = Server("intellirag")

@server.list_tools()
async def list_tools():
    return [
        types.Tool(
            name="query_knowledge_base",
            description="Search the knowledge base for relevant information.",
            inputSchema={
                "type": "object",
                "properties": {
                    "query": {"type": "string"},
                    "namespace": {"type": "string"}
                },
                "required": ["query"]
            }
        )
    ]

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    if name == "query_knowledge_base":
        results = await intellirag_search(arguments["query"], arguments.get("namespace"))
        return [types.TextContent(type="text", text=results)]
    raise ValueError(f"Unknown tool: {name}")
```

---

## Implementation Order

1. **Himanshu: Set up IntelliRAG MCP server** (Day 1)
   - Stand up MCP server with `query_knowledge_base` tool
   - Test: `curl http://intellirag-service:PORT/mcp` → tool list returns
   - Share URL with Harsh for env var config

2. **Harsh: Add `mcp` to requirements.txt** (Day 1, parallel)
   - `pip install mcp` → confirm no conflicts with Python 3.11

3. **Harsh: Create `mcp_client.py`** (Day 1–2)
   - Use code skeleton above
   - Unit test: connect to IntelliRAG local instance, `discover_mcp_tools()` returns callables

4. **Harsh: Modify `agent.py`** (Day 2)
   - Add `_get_mcp_server_configs()` + `discover_mcp_tools()` call
   - Add `MCP_SERVER_URLS` to `.env` pointing to IntelliRAG MCP server
   - Test: start agent locally, confirm MCP tools appear in tool list at startup

5. **Integration test end-to-end** (Day 2–3)
   - Start a call
   - Ask a question that requires knowledge base lookup
   - Confirm: LLM calls `query_knowledge_base` → IntelliRAG MCP responds → agent answers correctly
   - Confirm: if IntelliRAG is down, agent still starts and handles call (graceful degradation)

6. **Helm update** (Day 3)
   - Add `MCP_SERVER_URLS` env var to livekit-agent deployment in `intelliconverse-helm`
   - Deploy to dev → test → deploy to prod

---

## Testing Approach

### Unit tests (mcp_client.py)

```python
# Test 1: discover_mcp_tools returns callables
async def test_discover_tools():
    tools = await discover_mcp_tools([{"type": "sse", "url": "http://localhost:8090/mcp"}])
    assert len(tools) > 0
    assert callable(tools[0])
    assert tools[0].__name__ == "query_knowledge_base"

# Test 2: graceful failure if server unavailable
async def test_server_unavailable():
    tools = await discover_mcp_tools([{"type": "sse", "url": "http://nonexistent:9999/mcp"}])
    assert tools == []  # No crash, empty list

# Test 3: tool call returns text
async def test_tool_call():
    tools = await discover_mcp_tools([{"type": "sse", "url": "http://localhost:8090/mcp"}])
    result = await tools[0](ctx=None, query="What is the check-in time?")
    assert isinstance(result, str)
    assert len(result) > 0
```

### End-to-end test checklist

- [ ] Agent starts with `MCP_SERVER_URLS` set → MCP tools in log output
- [ ] Agent starts with `MCP_SERVER_URLS` unset → no error, existing tools only
- [ ] Agent starts with invalid MCP URL → logs error, continues with HTTP tools
- [ ] Call: user asks KB question → `query_knowledge_base` invoked → response uses retrieved info
- [ ] Call: IntelliRAG down mid-call → tool returns `[Tool unavailable: query_knowledge_base]` → agent handles gracefully
- [ ] Latency: MCP discovery at startup adds <500ms to agent initialization
- [ ] Latency: per-call MCP tool invocation adds <1000ms to response time

---

## Risks and Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Python MCP SDK incompatible with Python 3.11 | Low | Test with `pip install mcp` immediately (Day 1) |
| IntelliRAG MCP server not ready when livekit-agent integration starts | Medium | Use env var flag — if URL not set, skip MCP entirely. Code doesn't block. |
| MCP session re-establishment adds too much latency per call | Medium | Profile. If >500ms: implement session pooling or keep-alive connection. |
| `exec()` generated tools conflict with MCP tool names | Low | Namespace: HTTP tools keep original names; MCP tools prefixed `mcp_` if needed |
| Python skill gap on team | High | Harsh to pair with Himanshu on mcp_client.py. MCP SDK is well-documented. |

---

## Architecture Precedent

This integration establishes the pattern for **all future tool integrations in UniWeave:**

```
New capability → MCP server (owns its tool definitions)
    → registered via MCP_SERVER_URLS env var (or future UI)
    → livekit-agent discovers + calls automatically
    → no changes needed to config-backend, tool registry, or tool_helper.py
```

IntelliRAG is the first. IntelliPulse (read queries), external APIs, and any future service follow the same pattern.

---

## Related

- [IntelliRAG Integration Spec](intellirag-integration.md) — P0 requirements, latency budget, fallback
- [System Architecture](../system-architecture.md) — full platform reference
- [Analytics & User Mgmt Explainer](../analytics-user-mgmt-explainer.md) — current IntelliRAG path via tool-service (legacy)
- Trello #98: `69a9295e4063c49485e9fc1b`
