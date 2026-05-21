# Testing Patterns for FastAPI + MongoDB Layered Backend

## Test Infrastructure Overview

The project uses a custom `InMemoryDatabase` that mimics Motor's async MongoDB interface without requiring a real MongoDB instance. This enables fast, deterministic tests but introduces specific constraints.

## Key Fixtures

### Multi-Auth Test Fixtures
Create fixtures for each auth type your middleware supports:

```python
@pytest.fixture
def customer_auth_headers(customer_token: tuple, active_agent: dict) -> dict:
    token, _ = customer_token
    return {
        "Authorization": f"Bearer {token}",
        "X-Agent-Id": active_agent["id"],
    }

@pytest.fixture
def admin_auth_headers(admin_token: str) -> dict:
    return {"Authorization": f"Bearer {admin_token}"}
```

### Seeded Tenant Fixture
For multi-tenant routes, create a fixture that returns both the tenant and pre-built headers:

```python
@pytest.fixture
def seeded_tenant_with_headers(mock_db, customer_token) -> dict:
    tenant = mock_db.seed_tenant(customer_id=customer_id, ...)
    token, _ = customer_token
    return {
        "tenant": tenant,
        "headers": {
            "Authorization": f"Bearer {token}",
            "X-Tenant-Id": str(tenant["_id"]),
        },
    }
```

## Middleware Parity Rule

**Critical:** The test middleware MUST mirror the production middleware behavior exactly. If production only enforces `X-Agent-Id` for `service` tokens, the test middleware must do the same. A common bug is making the test middleware stricter or looser than production.

```python
# Correct: only enforce for service tokens
is_service = isinstance(user, dict) and user.get("type") == "service"
if is_service and not path.startswith("/agents"):
    # require agent_id
else:
    # inject agent if present but don't require
```

## InMemoryDatabase Limitations & Workarounds

### `$expr` Not Supported
The in-memory collection does not support MongoDB `$expr` queries. Move conflict-checking logic to Python:

```python
# DON'T in repository:
# query["$expr"] = {"$lt": [...]}  # fails in tests

# DO: fetch candidates and filter in Python
candidates = await self.collection.find({...}).to_list(None)
conflicts = []
for doc in candidates:
    doc_start = doc["start_time"]
    doc_end = doc["start_time"] + timedelta(minutes=doc.get("duration_minutes", 0))
    if start < doc_end and end > doc_start:
        conflicts.append(self._map_doc(doc))
```

### `ObjectId` Comparisons
Always use `str()` when comparing IDs from different sources in tests, since some may be strings and others `ObjectId` instances.

## Python Modernization Traps

### `datetime.utcnow()` Deprecation
Python 3.12+ deprecates `datetime.utcnow()`. Replace with timezone-aware datetimes:

```python
# OLD (deprecated, generates warnings)
from datetime import datetime
datetime.utcnow()

# NEW (correct)
from datetime import datetime, timezone
datetime.now(timezone.utc)
```

Search the entire codebase for `utcnow()` and replace during any maintenance window.

### `httpx.AsyncClient(app=...)` Deprecation
httpx 0.28+ removed the `app=` parameter. Use `ASGITransport`:

```python
# OLD
from httpx import AsyncClient
async with AsyncClient(app=app, base_url="http://test") as client:

# NEW
from httpx import ASGITransport, AsyncClient
async with AsyncClient(transport=ASGITransport(app=app), base_url="http://test") as client:
```

### `passlib` + `bcrypt` Compatibility
`passlib[bcrypt]` can break with newer bcrypt versions. Use raw `bcrypt` directly:

```python
# OLD
from passlib.context import CryptContext
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
pwd_context.hash(password)

# NEW
import bcrypt
bcrypt.hashpw(password.encode(), bcrypt.gensalt()).decode()
bcrypt.checkpw(password.encode(), hashed_password.encode())
```

## Test Organization

Group integration tests by domain area, not by HTTP method:

```python
@pytest.mark.asyncio
class TestProfessionals:
    async def test_create_professional(self, client, seeded_tenant_with_headers):
        ...

    async def test_get_professional_availability(self, client, seeded_tenant_with_headers):
        ...
```

## Smoke Tests

Always maintain a smoke test that verifies the app boots and key routes respond without errors:

```python
async def test_healthz(test_app):
    async with AsyncClient(transport=ASGITransport(app=test_app), base_url="http://test") as client:
        resp = await client.get("/healthz")
    assert resp.status_code == 200
```

Use `test_app` fixture (the app with in-memory DB) instead of importing `app` directly, to avoid "Database not initialized" errors.
