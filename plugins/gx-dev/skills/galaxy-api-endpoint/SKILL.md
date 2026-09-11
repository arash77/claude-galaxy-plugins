---
name: galaxy-api-endpoint
description: >
  Create Galaxy REST API endpoints with FastAPI routers, Pydantic schemas, and manager pattern.
  Use for: new API routes, FastAPI endpoints, REST resources, Pydantic request/response models,
  lib/galaxy/webapps/galaxy/api routers, lib/galaxy/schema definitions, API controller creation.
argument-hint: "[resource-name]"
---

Persona: You are a senior Galaxy backend developer specializing in FastAPI and the manager pattern.

Arguments:
- $ARGUMENTS - Optional resource name (e.g., "credentials", "workflows", "histories")
  If provided, use this as the resource name throughout the workflow

---

## Creating a New Galaxy API Endpoint

This guide walks you through creating a new REST API endpoint following Galaxy's architecture patterns.

### Step 0: Understand the Request

If $ARGUMENTS is empty, ask the user:
1. What resource are they creating an endpoint for? (e.g., "credentials", "user preferences")
2. What operation(s) are needed? (create, read, update, delete, list, custom action)

Use their answers to guide the rest of the workflow.

---

## Step 1: Find Similar Endpoint as Reference

Before starting, find the most recent similar endpoint to use as a pattern:

```bash
# Find recently modified API routers
ls -t lib/galaxy/webapps/galaxy/api/*.py | head -5
```

Read one of these files to understand current patterns. Good examples:
- `lib/galaxy/webapps/galaxy/api/job_lock.py` - Smallest router (~24 lines): plain function routes
- `lib/galaxy/webapps/galaxy/api/tags.py` - Smallest `@router.cbv` class-based view (~53 lines)
- `lib/galaxy/webapps/galaxy/api/workflows.py` - Complex resource with many operations
- `lib/galaxy/webapps/galaxy/api/histories.py` - RESTful resource with nested routes

Note: `api/job_files.py` is a **legacy WSGI controller** (`BaseGalaxyAPIController`), not a
FastAPI router. Do not use it as a template for new endpoints.

**Key patterns to observe:**
- Router setup: `router = Router(tags=["resource_name"])` - Galaxy's `Router`, never FastAPI's `APIRouter`
- Dependency injection: `depends(SomeManager)` for managers, `DependsOnTrans` for request context
- Request/response models: Pydantic schemas from `galaxy.schema`
- Error handling: Raising appropriate exceptions from `galaxy.exceptions`
- Documentation: Docstrings and OpenAPI metadata

---

## Step 2: Define Pydantic Schemas

Create request/response models in `lib/galaxy/schema/` (or update existing schema file if one exists for this domain).

**Location:** `lib/galaxy/schema/schema.py` (or domain-specific file like `lib/galaxy/schema/workflows.py`)

**Common imports:**
```python
from datetime import datetime
from typing import Optional, List

from pydantic import Field

from galaxy.schema.fields import EncodedDatabaseIdField
from galaxy.schema.schema import Model
```

**Example schema definitions:**
```python
class MyResourceCreateRequest(Model):
    """Request model for creating a new resource."""
    name: str = Field(..., description="Resource name")
    description: Optional[str] = Field(None, description="Optional description")

class MyResourceResponse(Model):
    """Response model for resource operations."""
    id: EncodedDatabaseIdField = Field(..., description="Encoded resource ID")
    name: str
    description: Optional[str]
    create_time: datetime
    update_time: datetime

class MyResourceListResponse(Model):
    """Response model for listing resources."""
    items: List[MyResourceResponse]
    total_count: int
```

**Field types commonly used:**
- `EncodedDatabaseIdField` - For Galaxy's encoded IDs
- `DecodedDatabaseIdField` - For decoded integer IDs (internal use)
- `str`, `int`, `bool`, `float` - Standard types
- `Optional[T]` - For nullable fields
- `List[T]` - For arrays
- `datetime` - For timestamps

**Best practices:**
- Use descriptive field names matching database column names
- Add `description` to all fields for OpenAPI documentation
- Use `...` for required fields, defaults for optional
- Keep request models separate from response models
- Use `Field(alias="...")` if API name differs from Python name

---

## Step 3: Add Manager Method

Business logic belongs in manager classes in `lib/galaxy/managers/`.

**Location:**
- If manager exists for this domain: Update `lib/galaxy/managers/<resource>s.py`
- If new domain: Create `lib/galaxy/managers/<resource>s.py`

**Manager pattern structure:**
```python
from typing import (
    List,
    Optional,
)

from sqlalchemy import select

from galaxy import (
    exceptions,
    model,
)
from galaxy.managers.context import ProvidesUserContext
from galaxy.model.scoped_session import galaxy_scoped_session


class MyResourceManager:
    """Manager for MyResource operations."""

    def __init__(self, sa_session: galaxy_scoped_session):
        self.sa_session = sa_session

    def create(
        self,
        trans: ProvidesUserContext,
        name: str,
        description: Optional[str] = None,
    ) -> model.MyResource:
        """Create a new resource."""
        resource = model.MyResource(
            user=trans.user,
            name=name,
            description=description,
        )
        self.sa_session.add(resource)
        self.sa_session.commit()
        return resource

    def get(self, trans: ProvidesUserContext, resource_id: int) -> model.MyResource:
        """Get resource by ID."""
        resource = self.sa_session.get(model.MyResource, resource_id)
        if not resource:
            raise exceptions.ObjectNotFound("Resource not found")
        if not self.is_accessible(resource, trans.user):
            raise exceptions.ItemAccessibilityException("Access denied")
        return resource

    def is_accessible(self, resource: model.MyResource, user: Optional[model.User]) -> bool:
        """Check if user can access this resource."""
        if not user:
            return False
        return resource.user_id == user.id

    def list_for_user(self, trans: ProvidesUserContext) -> List[model.MyResource]:
        """List all resources for the current user."""
        stmt = select(model.MyResource).where(model.MyResource.user_id == trans.user.id)
        return list(self.sa_session.scalars(stmt))
```

**Manager best practices:**
- Declare constructor dependencies with **type annotations** (`sa_session: galaxy_scoped_session`).
  Galaxy's DI container resolves them by type, which is what makes `depends(MyResourceManager)` work
  in the router. A constructor that takes an untyped `app` cannot be resolved this way.
- Methods take `trans` (request context) as first parameter
- Use `self.sa_session` for database operations
- **Call `self.sa_session.commit()` after mutating** - see the session gotcha below
- Raise appropriate exceptions from `galaxy.exceptions`
- Implement access control checks in separate methods
- Use SQLAlchemy 2.0 `select()` syntax for queries

---

## Step 4: Create FastAPI Router

Create or update the API router in `lib/galaxy/webapps/galaxy/api/`.

**Location:** `lib/galaxy/webapps/galaxy/api/<resource>s.py`

**Router template:**
```python
"""
API endpoints for MyResource operations.
"""
import logging
from typing import (
    Annotated,
    Optional,
)

from fastapi import (
    Body,
    Path,
    status,
)

from galaxy.managers.context import ProvidesUserContext
from galaxy.managers.myresources import MyResourceManager
from galaxy.schema.fields import DecodedDatabaseIdField
from galaxy.schema.schema import (
    MyResourceCreateRequest,
    MyResourceListResponse,
    MyResourceResponse,
)
from . import (
    depends,
    DependsOnTrans,
    Router,
)

log = logging.getLogger(__name__)

router = Router(tags=["myresources"])

# Path param alias: DecodedDatabaseIdField decodes the client's encoded id for you.
# Declare aliases like this next to the router, or reuse one from `api/common.py`.
MyResourceIdPathParam = Annotated[
    DecodedDatabaseIdField, Path(..., title="MyResource ID", description="The encoded database identifier.")
]


@router.cbv
class FastAPIMyResources:
    manager: MyResourceManager = depends(MyResourceManager)

    @router.get(
        "/api/myresources",
        summary="List all resources for current user",
        response_model=MyResourceListResponse,
    )
    def index(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> MyResourceListResponse:
        """List all resources owned by the current user."""
        items = self.manager.list_for_user(trans)
        return MyResourceListResponse(
            items=[self._serialize(trans, item) for item in items],
            total_count=len(items),
        )

    @router.post(
        "/api/myresources",
        summary="Create a new resource",
        status_code=status.HTTP_201_CREATED,
        response_model=MyResourceResponse,
    )
    def create(
        self,
        payload: MyResourceCreateRequest = Body(...),
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> MyResourceResponse:
        """Create a new resource."""
        resource = self.manager.create(
            trans,
            name=payload.name,
            description=payload.description,
        )
        return self._serialize(trans, resource)

    @router.get(
        "/api/myresources/{id}",
        summary="Get resource by ID",
        response_model=MyResourceResponse,
    )
    def show(
        self,
        id: MyResourceIdPathParam,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> MyResourceResponse:
        """Get a specific resource by ID."""
        # `id` already arrives decoded - do NOT call trans.security.decode_id() on it.
        resource = self.manager.get(trans, id)
        return self._serialize(trans, resource)

    def _serialize(self, trans: ProvidesUserContext, resource) -> MyResourceResponse:
        """Convert model object to response schema."""
        return MyResourceResponse(
            id=trans.security.encode_id(resource.id),
            name=resource.name,
            description=resource.description,
            create_time=resource.create_time,
            update_time=resource.update_time,
        )
```

**Router best practices:**
- Use `Router` (capital R) from `galaxy.webapps.galaxy.api`, **never** FastAPI's `APIRouter`.
  `Router` subclasses `FrameworkRouter`, which is what provides `.cbv`, `require_admin=`, and
  Galaxy's standard error responses. A bare `APIRouter` has no `.cbv` and fails at import time.
- Use `@router.cbv` class-based views for grouping related endpoints
- Inject managers with Galaxy's container helper: `manager: Manager = depends(Manager)`.
  **Never** `Depends(Manager)` - FastAPI would treat the manager's constructor arguments as
  request parameters and the route would 422 on every call.
- Use `DependsOnTrans` for request context; `require_admin=True` on the route decorator for admin-only
- Path parameters: use an `Annotated[DecodedDatabaseIdField, Path(...)]` alias so the id is decoded
  for you. `api/common.py` already defines many (`HistoryIDPathParam`, `UserIdPathParam`, ...)
- Query parameters use `Query(...)` with defaults
- Set appropriate HTTP status codes (`status_code=status.HTTP_201_CREATED` for creates)
- Add `summary` to all endpoints for OpenAPI docs
- The manager works with integer IDs; the `DecodedDatabaseIdField` alias performs the decode

---

## Step 5: Router Registration - Nothing To Do

**Galaxy discovers routers automatically. Do not register anything.**

`include_all_package_routers(app, "galaxy.webapps.galaxy.api")`
(`lib/galaxy/webapps/galaxy/fast_app.py:232`) walks every module in the `api` package and
includes any module-level attribute named `router`
(`lib/galaxy/webapps/base/api.py:385-399`).

So the only requirements are:
1. The file lives in `lib/galaxy/webapps/galaxy/api/`
2. It defines a module-level `router = Router(...)`

There are **no** `app.include_router()` calls in `buildapp.py` - that file builds the legacy
WSGI app. Adding one there will not register your route, and referencing the FastAPI `app`
object from it raises `AttributeError` at startup.

> **Warning:** that same auto-discovery walk imports *every* module in the `api` package with no
> `ImportError` guard. A bad import in your new router does not skip one route - **it stops
> Galaxy from booting.** Verify your imports resolve before starting the server.

---

## Step 6: Write API Tests

Create tests in `lib/galaxy_test/api/`.

**Location:** `lib/galaxy_test/api/test_<resource>s.py`

**Test template:**
```python
"""
API tests for MyResource endpoints.
"""
from galaxy_test.base.populators import DatasetPopulator
from ._framework import ApiTestCase


class TestMyResourcesApi(ApiTestCase):
    """Tests for /api/myresources endpoints."""

    def setUp(self):
        super().setUp()
        self.dataset_populator = DatasetPopulator(self.galaxy_interactor)

    def test_create_myresource(self):
        """Test creating a new resource."""
        payload = {
            "name": "Test Resource",
            "description": "Test description",
        }
        response = self._post("myresources", data=payload, json=True)
        self._assert_status_code_is(response, 201)
        resource = response.json()
        self._assert_has_keys(resource, "id", "name", "description", "create_time")
        assert resource["name"] == "Test Resource"

    def test_list_myresources(self):
        """Test listing resources."""
        # Create some test data
        self._create_myresource("Resource 1")
        self._create_myresource("Resource 2")

        # List resources
        response = self._get("myresources")
        self._assert_status_code_is_ok(response)
        data = response.json()
        assert data["total_count"] >= 2
        assert len(data["items"]) >= 2

    def test_get_myresource(self):
        """Test getting a specific resource."""
        resource_id = self._create_myresource("Test Resource")
        response = self._get(f"myresources/{resource_id}")
        self._assert_status_code_is_ok(response)
        resource = response.json()
        assert resource["id"] == resource_id
        assert resource["name"] == "Test Resource"

    def test_get_nonexistent_myresource(self):
        """Test getting a resource that doesn't exist."""
        response = self._get("myresources/invalid_id")
        self._assert_status_code_is(response, 404)

    def test_create_myresource_as_different_user(self):
        """Test that users can only see their own resources."""
        # Create as first user
        resource_id = self._create_myresource("User 1 Resource")

        # Switch to different user
        with self._different_user():
            # Should not be able to access
            response = self._get(f"myresources/{resource_id}")
            self._assert_status_code_is(response, 403)

    def _create_myresource(self, name: str) -> str:
        """Helper to create a resource and return its ID."""
        payload = {"name": name, "description": f"Description for {name}"}
        response = self._post("myresources", data=payload, json=True)
        self._assert_status_code_is(response, 201)
        return response.json()["id"]
```

**Test patterns:**
- Extend `ApiTestCase` from `lib/galaxy_test/api/_framework.py`
- Use `self._get()`, `self._post()`, `self._put()`, `self._delete()` (paths relative to `/api/`)
- Use `self._assert_status_code_is(response, 200)` for status checks
- Use `self._assert_status_code_is_ok(response)` for 2xx status
- Use `self._assert_has_keys(obj, "key1", "key2")` to verify response structure
- Use `self._different_user()` context manager to test as different user
- Create helper methods like `_create_myresource()` for test data setup
- Test both success and error cases (404, 403, 400, etc.)

---

## Step 7: Run Tests

Run your new tests using the Galaxy test runner:

```bash
# Run all tests for your new API
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py

# Run a specific test
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py::TestMyResourcesApi::test_create_myresource

# Run with verbose output
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py --verbose_errors
```

You can also run these with `pytest` directly:

```bash
pytest lib/galaxy_test/api/test_myresources.py
```

`run_tests.sh` is a documented convenience wrapper, not a requirement - it applies the output
options defined in that script and lets the whole selected suite share one Galaxy instance.
Invoking `pytest` directly skips those options and starts a **new Galaxy instance per test
class**, which is fine (often faster) while iterating on a single test and slow for a full run.

---

## Step 8: Verify and Manual Test

1. **Start Galaxy dev server:**
   ```bash
   ./run.sh
   ```

2. **Check OpenAPI docs:**
   Navigate to `http://localhost:8080/api/docs` and verify your endpoints appear

3. **Manual test with curl:**
   ```bash
   # Create
   curl -X POST http://localhost:8080/api/myresources \
     -H "Content-Type: application/json" \
     -d '{"name": "Test", "description": "Test resource"}'

   # List
   curl http://localhost:8080/api/myresources

   # Get specific
   curl http://localhost:8080/api/myresources/{id}
   ```

4. **Regenerate the TypeScript client types:**
   Frontend types live in `client/packages/api-client/src/schema/schema.ts`. They are **not**
   regenerated automatically - run the make target after changing any Pydantic schema:

   ```bash
   make update-client-api-schema
   ```

---

## Reference Files to Check

When implementing your endpoint, reference these files:

**Recent API examples:**
```bash
ls -t lib/galaxy/webapps/galaxy/api/*.py | head -5
```

**Schema patterns:**
- `lib/galaxy/schema/schema.py` - Main schema definitions
- `lib/galaxy/schema/fields.py` - Custom field types

**Manager patterns:**
```bash
ls lib/galaxy/managers/*.py
```

**Test examples:**
```bash
ls lib/galaxy_test/api/test_*.py
```

---

## Common Gotchas

1. **ID encoding:** Responses carry encoded IDs (`EncodedDatabaseIdField` on the schema, or
   `trans.security.encode_id()`). For *incoming* path/query params use a
   `DecodedDatabaseIdField` alias, which decodes automatically.
   **Never type an inbound param `EncodedDatabaseIdField`** - that annotation carries
   `BeforeValidator(encode_id)`, an outbound `int -> str` encoder, so it would encode the
   client's already-encoded id a second time and every valid ID would 404.
2. **Transaction context:** Manager methods should take `trans` as first parameter
3. **Database session:** Call `self.sa_session.commit()` after adding or mutating objects.
   `flush()` alone writes nothing durable - the request-scoped session is discarded at request
   teardown (`app.model.unset_request_id`), so the row silently disappears and a follow-up GET
   404s. Galaxy's managers commit; see `doc/source/dev/database_session_management.md`.
4. **Access control:** Always check if user can access resource before returning it
5. **Error handling:** Raise exceptions from `galaxy.exceptions`, not generic ones
6. **Router registration:** Nothing to register - routers are auto-discovered (Step 5).
   Do not touch `buildapp.py`.
7. **Dependency injection:** `depends(Manager)`, not FastAPI's `Depends(Manager)`
8. **Broken imports break the whole app:** the router auto-discovery walk has no `ImportError`
   guard, so an unresolvable import in your module stops Galaxy from starting

---

## Next Steps

After creating your endpoint:

1. Test thoroughly with automated tests
2. Manual test through browser and curl
3. Check OpenAPI documentation at `/api/docs`
4. Consider adding frontend integration (Vue components)
5. Update any relevant documentation

For more details, see:
- `reference.md` in this skill directory for concrete code examples
- Galaxy's CLAUDE.md for architecture overview
- Existing API implementations for patterns
