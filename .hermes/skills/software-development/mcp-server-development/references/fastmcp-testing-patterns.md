# Session: Testing fastmcp tools with MongoDB-mapped docs

## Problem

Attempting to test fastmcp tools by accessing `mcp_server._tools` fails:

```python
tool = mcp_server._tools["create_schedule"]  # AttributeError: 'FastMCP' object has no attribute '_tools'
```

## Solution: Use `call_tool`

```python
result = await mcp_server.call_tool("create_schedule", {
    "tenant_id": "tenant_123",
    "user_id": "user_123",
    "product_id": "prod_123",
    "scheduled_at": "2026-03-20T14:00:00",
})
text = result.content[0].text
assert "Schedule created successfully" in text
```

`result` is a `ToolResult` (or `CallToolResult`) with `content: list[TextContent]`.

## Pitfall: `_id` vs `id` in mock data

Professional tool `check_availability` iterates over professionals and accesses `prof["id"]`.

If mock data uses raw MongoDB `_id`:

```python
mock_profs = [{"_id": ObjectId("..."), "name": "Joao"}]  # WRONG
```

Error: `KeyError: 'id'` at line `prof["id"]`.

Fix: Use the mapped `id` field (as returned by repository `_map_doc`):

```python
mock_profs = [{"id": str(ObjectId("...")), "name": "Joao"}]  # CORRECT
```

## Verifying available methods on FastMCP instance

```python
[a for a in dir(mcp) if 'tool' in a.lower()]
# ['_call_tool_mcp', '_get_tool', '_list_tools', '_list_tools_mcp',
#  'add_tool', 'add_tool_transformation', 'call_tool', 'get_app_tool',
#  'get_tool', 'get_tool_by_hash', 'list_tools', 'remove_tool', ...]
```

## Testing tool that returns empty results

```python
with patch("...ProductRepository.list_by_tenant", new=AsyncMock(return_value=[])):
    result = await mcp_server.call_tool("get_services", {"tenant_id": "tenant_123"})
    text = result.content[0].text
    assert "Nenhum servico encontrado" in text
```

## Testing `check_availability` with specific date

```python
with patch("...get_database", return_value=mock_db), \
     patch("...list_by_tenant", new=AsyncMock(return_value=mock_profs)), \
     patch("...get_availability", new=AsyncMock(return_value=["09:00", "10:00", "11:00"])):
    result = await mcp_server.call_tool("check_availability", {
        "tenant_id": "tenant_123",
        "date": "2026-03-20"
    })
    text = result.content[0].text
    assert "Disponibilidade" in text
```

## Key imports for tests

```python
from unittest.mock import AsyncMock, MagicMock, patch
import pytest
from bson import ObjectId
from teajuda.mcp.app import mcp_server
```

## Test count after fix

- `tests/mcp/test_professional_tools.py`: 4 tests (get_services, get_services_empty, check_availability, list_professionals)
- `tests/mcp/test_schedule_tools.py`: 4 tests (create_schedule, cancel_schedule, list_user_schedules, list_user_schedules_empty)
- Total MCP tests: 20 (including pre-existing text_tools, agent_tools, etc.)
