---
name: mcp-server-development
description: |
  Build and test MCP (Model Context Protocol) servers using the fastmcp Python library,
  including tool registration, HTTP mounting in FastAPI, and unit-testing patterns
  for tools that touch MongoDB and external services.
category: software-development
trigger:
  - "create MCP server"
  - "fastmcp tool"
  - "MCP tool registration"
  - "test MCP tools"
  - "register tools on FastMCP"
  - "mcp server with mongodb"
---

# MCP Server Development with fastmcp

## When to use

Building an MCP server in Python (typically alongside a FastAPI backend) that exposes tools for AI agents to call. Common scenarios:
- Agent can query business data (services, availability, schedules)
- Agent can mutate state (create bookings, send messages)
- Agent needs tools that interact with MongoDB repositories and external APIs

## Architecture Pattern

```
FastAPI app
    |
    +-- /mcp  (mounted SSE/HTTP MCP endpoint)
    |       |
    |       +-- FastMCP server
    |               |
    |               +-- register_schedule_tools()
    |               +-- register_professional_tools()
    |               +-- register_whatsapp_tools()
    |
    +-- /api routers (HTTP REST)
```

## 1. Register Tools

```python
# mcp/tools/schedule_tools.py
from fastmcp import FastMCP
from teajuda.core.infrastructure.mongo import get_database
from teajuda.core.services.schedule_service import ScheduleService

def register_schedule_tools(mcp: FastMCP) -> None:
    @mcp.tool()
    async def create_schedule(tenant_id: str, user_id: str, ...) -> str:
        """Docstring becomes the tool description."""
        db = get_database()
        service = ScheduleService(ScheduleRepository(db))
        result = await service.create_schedule(...)
        return f"Created: {result}"
```

```python
# mcp/app.py
from fastmcp import FastMCP
from teajuda.mcp.tools.schedule_tools import register_schedule_tools

mcp_server = FastMCP("My MCP Server")
register_schedule_tools(mcp_server)
# ... other registrations

mcp_app = mcp_server.http_app()
```

## 2. Mount in FastAPI

```python
# main.py
from teajuda.mcp.app import mcp_app
app.mount("/mcp", mcp_app)
```

## 3. Testing Tools

### Calling tools programmatically

**Do NOT access `mcp_server._tools` directly** — newer versions of fastmcp removed this attribute. Use `call_tool` instead.

```python
import pytest
from teajuda.mcp.app import mcp_server

@pytest.mark.asyncio
async def test_create_schedule(mock_db):
    with patch("teajuda.mcp.tools.schedule_tools.get_database", return_value=mock_db), \
         patch("teajuda.core.services.schedule_service.ScheduleService.create_schedule", new_callable=AsyncMock) as mock_create:

        mock_create.return_value = {"_id": ObjectId("..."), "status": "pending"}

        result = await mcp_server.call_tool("create_schedule", {
            "tenant_id": "t1",
            "user_id": "u1",
            "product_id": "p1",
            "scheduled_at": "2026-03-20T14:00:00",
        })

        # result is a ToolResult with a content list
        text = result.content[0].text
        assert "Created" in text
```

### Mock data pitfalls

When mocking repository responses that return mapped MongoDB documents:

| Wrong | Right |
|-------|-------|
| `{"_id": ObjectId("..."), "name": "Joao"}` | `{"id": str(ObjectId("...")), "name": "Joao"}` |

Tool implementations typically access `prof["id"]` (the mapped field), not `prof["_id"]` (the raw MongoDB field). If your mock uses `_id`, the tool will raise `KeyError: 'id'`.

### Testing tools that call other tools

FastMCP isolates tools — you cannot call one tool from another via `mcp_server.call_tool` inside a tool body (that creates a middleware loop). Instead, extract shared logic into a plain async function and call it from both tools.

## 4. Tool Design Guidelines

- **Return strings, not objects** — MCP tools communicate with LLMs via text. Serialize structured data into a readable string.
- **Use try/except around DB calls** — raise only for unrecoverable errors; return error messages as strings for recoverable ones.
- **Keep tenant_id as first param** — makes it easy for the agent to scope all operations.
- **Add docstrings** — the docstring becomes the tool description the LLM sees.

## References

- [references/fastmcp-testing-patterns.md](references/fastmcp-testing-patterns.md) — Full session transcript with real test files, error traces, and fixes
