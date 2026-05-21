---
name: fastapi-mongodb-backend-scaffolding
description: Scaffold a new domain entity in a FastAPI + MongoDB layered backend (models, repository, service, router, indexes, seed, registration)
---

# FastAPI + MongoDB Backend Entity Scaffolding

## When to Use

Adding a new domain entity to a FastAPI backend that uses:
- Pydantic v2 models
- Async Motor MongoDB driver
- Three-layer architecture: `api` (routers) → `core` (models, repos, services) → `mcp` (tools)
- Seed data via startup lifespan

## The 8-Step Pattern

### Step 1 — Model (`src/core/models/<entity>.py`)

Create Pydantic models following this exact shape:

```python
from datetime import datetime
from typing import Any, Dict, List, Optional

from pydantic import BaseModel, ConfigDict, Field

from teajuda.core.models.pyobjectid import PyObjectId


class EntityCreate(BaseModel):
    # required fields with Field(..., json_schema_extra={"example": "..."})
    # optional fields with Field(default=None)

class EntityUpdate(BaseModel):
    # same fields as Create but all Optional with default=None

class EntityOut(BaseModel):
    id: PyObjectId = Field(..., alias="id")
    # all fields from Create
    created_at: datetime
    updated_at: datetime

    model_config = ConfigDict(from_attributes=True, populate_by_name=True)
```

**Pitfall:** Do NOT forget `model_config = ConfigDict(...)` on `EntityOut` or MongoDB documents won't serialize.

### Step 2 — Repository (`src/core/repositories/<entity>_repository.py`)

```python
from typing import List, Optional

from motor.motor_asyncio import AsyncIOMotorDatabase

from teajuda.core.repositories.base import BaseRepository


class EntityRepository(BaseRepository):
    def __init__(self, db: AsyncIOMotorDatabase):
        super().__init__(db, "entities")  # collection name

    async def get_by_slug(self, slug: str) -> Optional[dict]:
        doc = await self.collection.find_one({"slug": slug})
        return self._map_doc(doc)

    async def list_active(self) -> List[dict]:
        cursor = self.collection.find({"is_active": True}).sort("name", 1)
        docs = await cursor.to_list(length=None)
        return [self._map_doc(doc) for doc in docs if doc is not None]
```

**Pitfall:** Always return `self._map_doc(doc)` (not raw doc) so `_id` becomes `id` as string.

### Step 3 — Service (`src/core/services/<entity>_service.py`)

```python
from typing import List, Optional

from teajuda.core.exceptions.base import ServiceError
from teajuda.core.models.entity import EntityCreate, EntityOut, EntityUpdate
from teajuda.core.repositories.entity_repository import EntityRepository


class EntityServiceError(ServiceError):
    pass


class EntityService:
    def __init__(self, repository: EntityRepository):
        self.repository = repository

    async def create_entity(self, entity: EntityCreate) -> EntityOut:
        # uniqueness checks first
        created = await self.repository.create(entity.model_dump())
        return EntityOut(**created)

    async def update_entity(self, entity_id: str, update: EntityUpdate) -> EntityOut:
        # validate exists, apply update, return updated
        updated = await self.repository.update(entity_id, update.model_dump(exclude_unset=True))
        return EntityOut(**updated)

    async def get_entity(self, identifier: str, by: str = "id") -> EntityOut:
        # resolve by id/slug/etc, throw EntityServiceError if not found
        row = await self.repository.get_by_id(identifier) if by == "id" else await self.repository.get_by_slug(identifier)
        if not row:
            raise EntityServiceError("Entity not found")
        return EntityOut(**row)

    async def list_active_entities(self) -> List[EntityOut]:
        rows = await self.repository.list_active()
        return [EntityOut(**r) for r in rows]

    async def delete_entity(self, entity_id: str) -> bool:
        # validate exists, then delete
        return await self.repository.delete(entity_id)
```

**Why `EntityOut` instead of `dict`:** Returning Pydantic models from services guarantees validation at the boundary, enables IDE autocomplete, and lets FastAPI routers declare `response_model` without manual serialization. See [references/output-models-pattern.md](references/output-models-pattern.md) for the full migration guide.

**Pitfall:** Always use `update.model_dump(exclude_unset=True)` in update methods; never pass unset fields.

### Step 4 — Router (`src/api/routers/<entities>.py`)

```python
from typing import List

from fastapi import APIRouter, Depends, status
from motor.motor_asyncio import AsyncIOMotorDatabase

from teajuda.api.deps import get_db
from teajuda.core.auth.dependencies import require_auth
from teajuda.core.infrastructure.logging import get_logger
from teajuda.core.models.entity import EntityCreate, EntityOut, EntityUpdate
from teajuda.core.repositories.entity_repository import EntityRepository
from teajuda.core.services.entity_service import EntityService

logger = get_logger(__name__)
router = APIRouter(prefix="/entities", tags=["entities"])


@router.get("", response_model=List[EntityOut])
async def list_entities(
    db: AsyncIOMotorDatabase = Depends(get_db),  # noqa: B008
) -> List[EntityOut]:
    repo = EntityRepository(db)
    service = EntityService(repo)
    return await service.list_active_entities()


@router.get("/{entity_id}", response_model=EntityOut)
async def get_entity_by_id(
    entity_id: str,
    db: AsyncIOMotorDatabase = Depends(get_db),  # noqa: B008
) -> EntityOut:
    repo = EntityRepository(db)
    service = EntityService(repo)
    return await service.get_entity(entity_id, by="id")


@router.post("", status_code=status.HTTP_201_CREATED, response_model=EntityOut)
async def create_entity(
    payload: EntityCreate,
    db: AsyncIOMotorDatabase = Depends(get_db),  # noqa: B008
    identity: dict = Depends(require_auth),  # noqa: B008
) -> EntityOut:
    repo = EntityRepository(db)
    service = EntityService(repo)
    return await service.create_entity(payload)
```

**Key changes from raw-dict style:**
- Use `response_model=EntityOut` (or `List[EntityOut]`) on every endpoint that returns a standard entity.
- Return the Pydantic model directly; FastAPI serializes it. Do NOT call `.model_dump(by_alias=True)` manually.
- Keep services free of HTTP concerns; routers handle HTTP status codes.

**Exception — custom payloads:** Endpoints that return ad-hoc structures (webhook callbacks, health checks, availability calendars, aggregated stats) should remain as plain `dict` and omit `response_model`:

```python
@router.get("/health")
async def health_check() -> dict:
    return {"status": "ok", "version": "1.2.3"}
```

### Step 5 — Register in `main.py`

```python
from teajuda.api.routers.entities import router as entities_router

# Add to exempt_paths in UnifiedAuthMiddleware if public endpoints exist
exempt_paths = [
    # ... existing paths ...
    "/entities",
    "/entities/",
    "/entities/{slug}",
]

# Add to router includes
app.include_router(entities_router)
```

### Step 6 — MongoDB Indexes (`src/core/infrastructure/indexes.py`)

```python
# Inside ensure_indexes()
await db.entities.create_index("slug", unique=True)
await db.entities.create_index("is_active")
```

### Step 7 — Seed Data (`src/api/startup.py`)

If seed data is needed:

```python
async def seed_main_entities(db: AsyncIOMotorDatabase) -> None:
    from teajuda.core.repositories.entity_repository import EntityRepository

    repo = EntityRepository(db)
    for data in DEFAULT_ENTITIES:
        existing = await repo.get_by_slug(data["slug"])
        if existing:
            continue
        await repo.create(data)
```

Then call it from `app_lifespan` after `seed_main_tenants`.

### Step 8 — Update Progress Tracker

Mark the corresponding tasks as complete in `STATUS.md` or equivalent tracker.

## Evolving an Existing Entity

When adding fields to an already-scaffolded entity, follow this 5-step pattern:

1. **Update the Pydantic models** (`Create`, `Update`, `Out`) — add new fields with `Optional` and sensible defaults. Keep `exclude_unset=True` behavior intact.
2. **Update the Repository** — add query methods for the new fields (e.g., `get_by_slug`, `find_conflicts`).
3. **Update the Service** — add business logic that uses the new fields. If the new field references another entity, inject the related repository and validate the foreign key.
4. **Update the Router** — enrich payloads (e.g., auto-fill `created_by` from `request.state.user`) and wire new endpoints.
5. **Update indexes and seed** — add MongoDB indexes for new query patterns; update seed data if the field is required or has uniqueness constraints.

### Conflict Validation Pattern
When an entity has time-based or resource-based conflicts (e.g., appointments sharing a professional):

- Add `find_conflicts(tenant_id, professional_id, start, end, exclude_id)` to the repository.
- Use MongoDB `$expr` with `$add` / `$multiply` to compare datetime ranges against `duration_minutes`.
- Call `_check_conflict()` in both `create_*` and `update_*` service methods.
- In updates, fetch the existing document first to compute the "new" values vs. unchanged ones.

## Cross-Cutting Concerns

### Linking to Existing Entities
When a new entity references an existing one (e.g., Tenant references Segment):

1. Add `segment_id: Optional[str]` to the Create/Update/Out models.
2. Inject the related repository into the service constructor:
   `def __init__(self, repository: EntityRepository, segment_repository: Optional[SegmentRepository] = None)`
3. Validate the foreign key exists in create/update methods.
4. Update the router to pass both repositories to the service.

### Enriching Responses
When an endpoint needs to return nested data (e.g., Tenant + Segment info):

Do it in the **router**, not in the service. The service stays pure; the router composes:

```python
# router
segment = await segment_repo.get_by_id(tenant["segment_id"])
tenant["segment"] = {"id": segment["id"], "slug": segment["slug"], ...}
```

## Remember
- Always follow the 8-step order; skipping a step causes integration errors.
- Never duplicate business logic between router and service — router delegates.
- Always map documents with `_map_doc()` before returning from repository.
- Always add `# noqa: B008` on `Depends()` calls.
- Keep services free of HTTP concerns; they raise `ServiceError`, routers handle HTTP mapping.

## Testing

See [references/testing-patterns.md](references/testing-patterns.md) for comprehensive patterns on testing this architecture, including:
- InMemoryDatabase limitations and workarounds (`$expr`, ObjectId)
- Multi-auth test fixtures (customer, admin, service tokens)
- Middleware parity between production and tests
- Python modernization traps (`utcnow()`, `httpx.AsyncClient(app=...)`, `passlib`/`bcrypt`)
- Smoke test patterns using `ASGITransport`
- Pydantic mock requirements when services return `*Out` models instead of raw dicts

See [references/output-models-pattern.md](references/output-models-pattern.md) for the Output Models migration guide, mock pitfalls, and per-domain refactor strategy.
