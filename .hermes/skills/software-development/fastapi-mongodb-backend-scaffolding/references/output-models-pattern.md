# Output Models Pattern for FastAPI + MongoDB

## Motivation

Returning raw `dict` from services and routers creates several problems:

1. **No validation at the boundary** — A missing field or wrong type only surfaces at serialization time, often as a 500 error in production.
2. **No IDE support** — Callers work with opaque dicts; autocomplete and type-checking are impossible.
3. **Manual serialization boilerplate** — Routers end up littered with `.model_dump(by_alias=True)` calls and nested wrapper dicts (`{"entity": entity}`).
4. **Test fragility** — Tests assert on string keys (`result["name"]`) instead of typed attributes (`result.name`).

The Output Models pattern solves all of these by having services return validated Pydantic models and routers declare `response_model` so FastAPI handles serialization automatically.

## The Pattern

### Service Layer

```python
from typing import List
from teajuda.core.models.entity import EntityCreate, EntityOut, EntityUpdate
from teajuda.core.repositories.entity_repository import EntityRepository


class EntityService:
    def __init__(self, repository: EntityRepository):
        self.repository = repository

    async def create(self, data: EntityCreate) -> EntityOut:
        row = await self.repository.create(data.model_dump())
        return EntityOut(**row)

    async def get_by_id(self, entity_id: str) -> EntityOut:
        row = await self.repository.get_by_id(entity_id)
        if not row:
            raise EntityNotFoundError()
        return EntityOut(**row)

    async def list_active(self) -> List[EntityOut]:
        rows = await self.repository.list_active()
        return [EntityOut(**r) for r in rows]

    async def update(self, entity_id: str, data: EntityUpdate) -> EntityOut:
        row = await self.repository.update(entity_id, data.model_dump(exclude_unset=True))
        return EntityOut(**row)
```

Rules:
- Every public read/write method returns `EntityOut` (or `List[EntityOut]`).
- The service wraps the repository dict with `EntityOut(**row)` immediately.
- Custom/ad-hoc payloads (e.g., availability calendars, webhook responses) stay as `dict` and live in the router or a dedicated utility.

### Router Layer

```python
from typing import List
from fastapi import APIRouter, Depends, status
from teajuda.api.deps import get_db
from teajuda.core.auth.dependencies import require_auth
from teajuda.core.models.entity import EntityCreate, EntityOut, EntityUpdate
from teajuda.core.repositories.entity_repository import EntityRepository
from teajuda.core.services.entity_service import EntityService

router = APIRouter(prefix="/entities", tags=["entities"])


@router.get("", response_model=List[EntityOut])
async def list_entities(db=Depends(get_db)):
    service = EntityService(EntityRepository(db))
    return await service.list_active()


@router.get("/{entity_id}", response_model=EntityOut)
async def get_entity(entity_id: str, db=Depends(get_db)):
    service = EntityService(EntityRepository(db))
    return await service.get_by_id(entity_id)


@router.post("", status_code=status.HTTP_201_CREATED, response_model=EntityOut)
async def create_entity(payload: EntityCreate, db=Depends(get_db), identity=Depends(require_auth)):
    service = EntityService(EntityRepository(db))
    return await service.create(payload)


@router.patch("/{entity_id}", response_model=EntityOut)
async def update_entity(entity_id: str, payload: EntityUpdate, db=Depends(get_db)):
    service = EntityService(EntityRepository(db))
    return await service.update(entity_id, payload)
```

Rules:
- Declare `response_model=EntityOut` (or `List[EntityOut]`) on standard CRUD endpoints.
- Return the model directly. Do NOT call `.model_dump()` or wrap it in a custom dict.
- Endpoints with custom payloads (health checks, webhook callbacks, aggregated stats) omit `response_model` and return plain `dict`.

## Migrating an Existing Codebase

When converting an existing API from raw-dict to Output Models, follow this order to minimize breakage:

1. **Ensure `EntityOut` exists and is correct** — All required fields must be present, including system fields like `is_active`, `created_at`, `updated_at`.
2. **Refactor the service first** — Change return types from `dict` / `List[dict]` to `EntityOut` / `List[EntityOut]`, wrap repository returns with `EntityOut(**row)`.
3. **Run unit tests for the service** — They will likely fail because mocks now face Pydantic validation. Fix mocks (see Pitfalls below).
4. **Refactor the router** — Add `response_model`, remove manual `.model_dump(by_alias=True)` calls, remove wrapper dicts (`{"entity": entity}`), return the model directly.
5. **Run integration tests** — Assertions that used `data["entity"]["name"]` must change to `data["name"]` because the response is no longer nested.
6. **Repeat per domain** — Do one entity at a time (segment, professional, product, etc.) to keep PRs reviewable.

## Pitfalls

### 1. Unit-test mocks must satisfy Pydantic validation

When a service returns `EntityOut(**row)`, any mock returning an incomplete dict will raise `ValidationError`.

**Wrong:**
```python
mock_repo.get_by_id.return_value = {"_id": "prof123", "name": "Test"}
```

**Right:**
```python
mock_repo.get_by_id.return_value = {
    "_id": "65f1a2b3c4d5e6f7a8b9c0d1",  # valid 24-char hex ObjectId
    "name": "Test",
    "slug": "test",
    "is_active": True,
    "created_at": datetime.now(timezone.utc),
    "updated_at": datetime.now(timezone.utc),
}
```

Requirements:
- `id` / `_id` must be a valid MongoDB ObjectId string (24 hex characters) or an actual `bson.ObjectId` instance.
- All fields marked `required` or without `default` / `default_factory` in `EntityOut` must be present.
- Update test assertions from `result["name"]` to `result.name` (attribute access on the Pydantic model).

### 2. Integration-test response shape changes

When a router switches from returning `{"entity": entity}` to returning the model directly, integration tests that parsed `response.json()["entity"]["name"]` must change to `response.json()["name"]`.

Also watch for aliased fields. If `EntityOut` uses `id` (aliased from `_id`), tests that previously looked for `professional_id` must use `id`.

### 3. Endpoints with custom payloads

Do NOT force `response_model` onto endpoints that return ad-hoc structures:

```python
# KEEP as plain dict
@router.get("/health")
async def health() -> dict:
    return {"status": "ok", "checks": {"db": "connected", "queue": 3}}

# KEEP as plain dict
@router.post("/webhook/whatsapp")
async def whatsapp_webhook(payload: dict) -> dict:
    return {"received": True, "messages_processed": 2}
```

Forcing a Pydantic model onto these endpoints creates unnecessary coupling and often breaks third-party consumers that expect a specific shape.

### 4. `datetime.utcnow()` deprecation

Many existing services use `datetime.utcnow()`. When Pydantic v2 strict mode or newer Python versions are in play, this triggers deprecation warnings. Replace with:

```python
from datetime import datetime, timezone
now = datetime.now(timezone.utc)
```

### 5. Large-scale refactor timing

Converting an entire codebase at once is risky. If using subagents for parallel refactors, split by domain (one entity per subagent) rather than by layer (all services in one subagent, all routers in another). Services and routers for a single entity are tightly coupled; changing one without the other leaves the codebase in an inconsistent state.
