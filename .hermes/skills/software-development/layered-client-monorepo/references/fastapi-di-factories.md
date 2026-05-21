# FastAPI Repository + Service DI Factories

Pattern for wiring layered backend architectures (repository → service → router) using FastAPI's dependency injection system.

## Core Pattern

Create factory functions in `deps.py` that take the database dependency and return repository instances. Then inject repository instances directly into service factories, and services directly into router endpoints.

```python
# deps.py
from typing import Annotated
from fastapi import Depends

from myapp.core.repositories.customer_repository import CustomerRepository
from myapp.core.repositories.product_repository import ProductRepository
from myapp.core.services.customer_service import CustomerService
from myapp.core.services.product_service import ProductService


def get_db():
    db = DBSession()
    try:
        yield db
    finally:
        db.close()


DBDep = Annotated[DBSession, Depends(get_db)]


def get_customer_repo(db: DBDep) -> CustomerRepository:
    return CustomerRepository(db)


def get_product_repo(db: DBDep) -> ProductRepository:
    return ProductRepository(db)


def get_customer_service(
    repo: Annotated[CustomerRepository, Depends(get_customer_repo)],
) -> CustomerService:
    return CustomerService(repo)


def get_product_service(
    repo: Annotated[ProductRepository, Depends(get_product_repo)],
) -> ProductService:
    return ProductService(repo)
```

```python
# routers/customers.py
from typing import Annotated
from fastapi import APIRouter, Depends

from myapp.api.deps import get_customer_service
from myapp.core.services.customer_service import CustomerService

router = APIRouter()

CustomerServiceDep = Annotated[CustomerService, Depends(get_customer_service)]


@router.get("/customers")
async def list_customers(service: CustomerServiceDep):
    return await service.list()
```

## Why This Pattern Wins

- **Eliminates boilerplate**: No more `RepoName(db)` in every endpoint
- **Router purity**: Endpoints focus only on HTTP concerns
- **Testability**: Override `get_customer_service` in `app.dependency_overrides` for mocks
- **Per-request lifecycle**: FastAPI resolves the dependency tree fresh each request

## Anti-pattern to Avoid

```python
# WRONG — manual instantiation inside endpoints
@router.get("/customers")
async def list_customers(db: DBDep):
    repo = CustomerRepository(db)
    service = CustomerService(repo)
    return await service.list()
```

## Migration Path from Manual Instantiation

1. Create `get_X_repo(db)` factories for each repository used in routers
2. Create `get_X_service(repo)` factories for each service
3. Update router imports to use `from myapp.api.deps import get_X_service`
4. Replace `service = XService(repo)` with `service: XServiceDep` parameter
5. Run tests to verify `app.dependency_overrides` still work

## Pitfall: Missing Service Factories

When adding a new service, remember to also add its factory in `deps.py`. If you import the service class directly into a router and instantiate it manually, you lose DI benefits and testability.
