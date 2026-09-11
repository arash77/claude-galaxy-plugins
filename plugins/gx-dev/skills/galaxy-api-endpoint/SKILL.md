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
- `lib/galaxy/webapps/galaxy/api/job_files.py` - Simple CRUD operations
- `lib/galaxy/webapps/galaxy/api/workflows.py` - Complex resource with many operations
- `lib/galaxy/webapps/galaxy/api/histories.py` - RESTful resource with nested routes

**Key patterns to observe:**
- Router setup: `router = APIRouter(tags=["resource_name"])`
- Dependency injection: `DependsOnTrans`, custom dependency functions
- Request/response models: Pydantic schemas from `galaxy.schema`
- Error handling: Raising appropriate HTTP exceptions
- Documentation: Docstrings and OpenAPI metadata

---

## Step 2: Define Pydantic Schemas

Create request/response models in `lib/galaxy/schema/` (or update existing schema file if one exists for this domain).

**Location:** `lib/galaxy/schema/schema.py` (or domain-specific file like `lib/galaxy/schema/workflows.py`)

**Common imports:**
```python
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
from typing import Optional
from galaxy import model
from galaxy.managers.context import ProvidesUserContext
from galaxy.model import Session

class MyResourceManager:
    """Manager for MyResource operations."""

    def __init__(self, app):
        self.app = app
        self.sa_session: Session = app.model.context

    def create(
        self,
        trans: ProvidesUserContext,
        name: str,
        description: Optional[str] = None
    ) -> model.MyResource:
        """Create a new resource."""
        resource = model.MyResource(
            user=trans.user,
            name=name,
            description=description
        )
        self.sa_session.add(resource)
        self.sa_session.flush()
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
        stmt = select(model.MyResource).where(
            model.MyResource.user_id == trans.user.id
        )
        return self.sa_session.scalars(stmt).all()
```

**Manager best practices:**
- Constructor takes `app` (the Galaxy application object)
- Methods take `trans` (transaction/request context) as first parameter
- Use `self.sa_session` for database operations
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
from typing import Optional

from fastapi import (
    APIRouter,
    Depends,
    Path,
    Query,
    status,
)

from galaxy.managers.context import ProvidesUserContext
from galaxy.managers.myresources import MyResourceManager
from galaxy.schema.schema import (
    MyResourceCreateRequest,
    MyResourceResponse,
    MyResourceListResponse,
)
from galaxy.webapps.galaxy.api import (
    DependsOnTrans,
    Router,
)
from galaxy.webapps.galaxy.api.depends import get_app

log = logging.getLogger(__name__)

router = Router(tags=["myresources"])

# Dependency for manager
def get_myresource_manager(app=Depends(get_app)) -> MyResourceManager:
    return MyResourceManager(app)

@router.cbv
class FastAPIMyResources:
    manager: MyResourceManager = Depends(get_myresource_manager)

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
            items=[self._serialize(item) for item in items],
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
        trans: ProvidesUserContext = DependsOnTrans,
        request: MyResourceCreateRequest = ...,
    ) -> MyResourceResponse:
        """Create a new resource."""
        resource = self.manager.create(
            trans,
            name=request.name,
            description=request.description,
        )
        return self._serialize(resource)

    @router.get(
        "/api/myresources/{id}",
        summary="Get resource by ID",
        response_model=MyResourceResponse,
    )
    def show(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
        id: EncodedDatabaseIdField = Path(..., description="Resource ID"),
    ) -> MyResourceResponse:
        """Get a specific resource by ID."""
        decoded_id = trans.security.decode_id(id)
        resource = self.manager.get(trans, decoded_id)
        return self._serialize(resource)

    def _serialize(self, resource) -> MyResourceResponse:
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
- Use `Router` (capital R) from `galaxy.webapps.galaxy.api` (subclass of FastAPI's APIRouter)
- Use `@router.cbv` class-based views for grouping related endpoints
- Use dependency injection for managers: `manager: Manager = Depends(get_manager)`
- Use `DependsOnTrans` for transaction context
- Path parameters use `Path(...)` with descriptions
- Query parameters use `Query(...)` with defaults
- Set appropriate HTTP status codes (`status_code=status.HTTP_201_CREATED` for creates)
- Add `summary` to all endpoints for OpenAPI docs
- Decode IDs in endpoint, not in manager (manager works with integer IDs)

---

## Step 5: Register Router

The router must be registered in the main application builder.

**Location:** `lib/galaxy/webapps/galaxy/buildapp.py`

**Add import:**
```python
from galaxy.webapps.galaxy.api import myresources
```

**Register router in `app_factory()`:**
```python
app.include_router(myresources.router)
```

**Find the section:** Look for other `app.include_router()` calls and add yours in alphabetical order.

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

**IMPORTANT:** Always use `./run_tests.sh`, not `pytest` directly. The wrapper script sets up the correct environment.

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

4. **Check auto-generated TypeScript types:**
   The frontend types in `client/src/api/schema/schema.ts` will be auto-generated from your Pydantic schemas next time the schema is rebuilt.

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

1. **ID encoding:** Always encode IDs in API responses (`trans.security.encode_id()`) and decode in endpoints
2. **Transaction context:** Manager methods should take `trans` as first parameter
3. **Database session:** Use `self.sa_session.flush()` after adding objects, not `commit()`
4. **Access control:** Always check if user can access resource before returning it
5. **Error handling:** Raise exceptions from `galaxy.exceptions`, not generic ones
6. **Router registration:** Don't forget to register your router in `buildapp.py`
7. **Test runner:** Use `./run_tests.sh -api`, not plain `pytest`

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
