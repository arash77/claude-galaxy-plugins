# Galaxy API Endpoint Reference

Patterns for creating Galaxy API endpoints.

**Two kinds of code appear below.** Keep them straight:

- **"Complete Example 1/2"** are *verbatim copies* of real files in the Galaxy tree
  (`api/job_lock.py`, `api/tags.py`). Trust them, and re-read the originals if in doubt.
- **Everything else** is an *illustrative* walkthrough of a hypothetical `credential` resource,
  written to show the shape of each layer. The `model.Credential` table, `CredentialManager`
  and the `Credential*` schemas in these snippets **do not exist** - you write them.

> Galaxy *does* ship a real, unrelated credentials feature
> (`api/credentials.py`, `managers/credentials.py` with `CredentialsManager`, `schema/credentials.py`).
> It is a genuinely good, current example to read - but it is **not** the code below, and the
> names differ. Do not import from it expecting these snippets' symbols.

## Complete Example 1: Function-Based Routes

The whole of `lib/galaxy/webapps/galaxy/api/job_lock.py` - Galaxy's smallest router, verbatim:

```python
from fastapi import Body

from galaxy.managers.jobs import (
    JobLock,
    JobManager,
)
from . import (
    depends,
    Router,
)

router = Router(tags=["job_lock"])


@router.get("/api/job_lock", require_admin=True)
def job_lock_status(job_manager: JobManager = depends(JobManager)) -> JobLock:
    """Get job lock status."""
    return job_manager.job_lock()


@router.put("/api/job_lock", require_admin=True)
def update_job_lock(job_manager: JobManager = depends(JobManager), job_lock: JobLock = Body(...)) -> JobLock:
    """Set job lock status."""
    return job_manager.update_job_lock(job_lock)
```

**Key observations:**
- `router = Router(...)` at module level - this attribute name is what makes Galaxy find the router
- No registration step anywhere; nothing imports this module explicitly
- Managers injected with `depends(JobManager)` - Galaxy's container helper, **not** `Depends(...)`
- `require_admin=True` is a Galaxy `Router` extension, unavailable on a plain `APIRouter`
- Return type annotation (`-> JobLock`) drives the OpenAPI response model
- Routes carry the **full** path including the `/api` prefix

---

## Complete Example 2: Class-Based View

The whole of `lib/galaxy/webapps/galaxy/api/tags.py` - the smallest `@router.cbv`, verbatim:

```python
"""
API Controller providing Galaxy Tags
"""

import logging

from fastapi import (
    Body,
    Response,
    status,
)

from galaxy.managers.context import ProvidesUserContext
from galaxy.managers.tags import (
    ItemTagsPayload,
    TagsManager,
)
from . import (
    depends,
    DependsOnTrans,
    Router,
)

log = logging.getLogger(__name__)

router = Router(tags=["tags"])


@router.cbv
class FastAPITags:
    manager: TagsManager = depends(TagsManager)

    @router.put(
        "/api/tags",
        summary="Apply a new set of tags to an item.",
        status_code=status.HTTP_204_NO_CONTENT,
    )
    def update(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
        payload: ItemTagsPayload = Body(
            ...,  # Required
            title="Payload",
            description="Request body containing the item and the tags to be assigned.",
        ),
    ):
        """Replaces the tags associated with an item with the new ones specified in the payload.

        - The previous tags will be __deleted__.
        - If no tags are provided in the request body, the currently associated tags will also be __deleted__.
        """
        self.manager.update(trans, payload)
        return Response(status_code=status.HTTP_204_NO_CONTENT)
```

**Key observations:**
- `@router.cbv` groups endpoints that share dependencies
- The manager is a **class attribute**: `manager: TagsManager = depends(TagsManager)`
- Request context via `trans: ProvidesUserContext = DependsOnTrans`
- Helpers are imported from the package itself: `from . import depends, DependsOnTrans, Router`
- `status_code=` on the decorator sets the documented success status

---

---

## Schema Patterns

### Basic Response Model

```python
from datetime import datetime

from pydantic import Field

from galaxy.schema.fields import EncodedDatabaseIdField
from galaxy.schema.schema import Model

class CredentialResponse(Model):
    """Response model for credential operations."""
    id: EncodedDatabaseIdField = Field(
        ...,
        title="ID",
        description="Encoded ID of the credential"
    )
    name: str = Field(
        ...,
        title="Name",
        description="Name of the credential"
    )
    vault_type: str = Field(
        ...,
        title="Vault Type",
        description="Type of vault (e.g., 'database', 'hashicorp')"
    )
    create_time: datetime
    update_time: Optional[datetime] = None
```

### Request Models

```python
class CreateCredentialRequest(Model):
    """Request to create a new credential."""
    name: str = Field(..., min_length=1, description="Credential name")
    vault_type: str = Field("database", description="Vault type to use")
    username: Optional[str] = Field(None, description="Username for authentication")
    password: Optional[str] = Field(None, description="Password for authentication")

class UpdateCredentialRequest(Model):
    """Request to update existing credential."""
    name: Optional[str] = Field(None, min_length=1, description="New credential name")
    username: Optional[str] = Field(None, description="New username")
    password: Optional[str] = Field(None, description="New password")
```

### List Response with Metadata

```python
class CredentialListResponse(Model):
    """Response for listing credentials."""
    items: List[CredentialResponse] = Field(..., description="List of credentials")
    total_count: int = Field(..., description="Total number of credentials")
```

---

## Manager Patterns

### Basic Manager Structure

```python
from typing import List, Optional
from sqlalchemy import select
from galaxy import model, exceptions
from galaxy.managers.context import ProvidesUserContext
from galaxy.model import Session

class CredentialManager:
    """Manager for credential operations."""

    def __init__(self, app):
        self.app = app
        self.sa_session: Session = app.model.context

    def create(
        self,
        trans: ProvidesUserContext,
        name: str,
        vault_type: str = "database",
        username: Optional[str] = None,
        password: Optional[str] = None,
    ) -> model.Credential:
        """Create a new credential."""
        credential = model.Credential(
            user=trans.user,
            name=name,
            vault_type=vault_type,
            username=username,
            password=password,
        )
        self.sa_session.add(credential)
        self.sa_session.commit()
        return credential

    def get(self, trans: ProvidesUserContext, credential_id: int) -> model.Credential:
        """Get credential by ID with access check."""
        credential = self.sa_session.get(model.Credential, credential_id)
        if not credential:
            raise exceptions.ObjectNotFound(f"Credential with id {credential_id} not found")
        if not self.is_accessible(credential, trans.user):
            raise exceptions.ItemAccessibilityException("You do not have access to this credential")
        return credential

    def list_for_user(self, trans: ProvidesUserContext) -> List[model.Credential]:
        """List all credentials for current user."""
        stmt = select(model.Credential).where(
            model.Credential.user_id == trans.user.id,
            model.Credential.deleted == False,
        ).order_by(model.Credential.name)
        return self.sa_session.scalars(stmt).all()

    def update(
        self,
        trans: ProvidesUserContext,
        credential_id: int,
        name: Optional[str] = None,
        username: Optional[str] = None,
        password: Optional[str] = None,
    ) -> model.Credential:
        """Update credential fields."""
        credential = self.get(trans, credential_id)
        if name is not None:
            credential.name = name
        if username is not None:
            credential.username = username
        if password is not None:
            credential.password = password
        self.sa_session.commit()
        return credential

    def delete(self, trans: ProvidesUserContext, credential_id: int) -> None:
        """Soft-delete a credential."""
        credential = self.get(trans, credential_id)
        credential.deleted = True
        self.sa_session.commit()

    def is_accessible(self, credential: model.Credential, user: Optional[model.User]) -> bool:
        """Check if user can access this credential."""
        if not user:
            return False
        return credential.user_id == user.id
```

---

## Router Patterns

### Full CRUD Router

```python
from typing import Annotated

from fastapi import (
    Body,
    Path,
    Query,
    status,
)

from galaxy.managers.context import ProvidesUserContext
from galaxy.schema.fields import DecodedDatabaseIdField
# These four live in whichever module you create for this resource:
from galaxy.managers.mycredentials import CredentialManager
from galaxy.schema.mycredentials import (
    CreateCredentialRequest,
    UpdateCredentialRequest,
    CredentialResponse,
    CredentialListResponse,
)
from . import (
    depends,
    DependsOnTrans,
    Router,
)

router = Router(tags=["credentials"])

CredentialIdPathParam = Annotated[
    DecodedDatabaseIdField, Path(..., title="Credential ID", description="The encoded database identifier.")
]


@router.cbv
class FastAPICredentials:
    manager: CredentialManager = depends(CredentialManager)

    @router.get(
        "/api/credentials",
        summary="List user credentials",
        response_model=CredentialListResponse,
    )
    def index(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> CredentialListResponse:
        """List all credentials for the current user."""
        items = self.manager.list_for_user(trans)
        return CredentialListResponse(
            items=[self._serialize(trans, item) for item in items],
            total_count=len(items),
        )

    @router.post(
        "/api/credentials",
        summary="Create new credential",
        status_code=status.HTTP_201_CREATED,
        response_model=CredentialResponse,
    )
    def create(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
        request: CreateCredentialRequest = ...,
    ) -> CredentialResponse:
        """Create a new credential."""
        credential = self.manager.create(
            trans,
            name=request.name,
            vault_type=request.vault_type,
            username=request.username,
            password=request.password,
        )
        return self._serialize(trans, credential)

    @router.get(
        "/api/credentials/{id}",
        summary="Get credential by ID",
        response_model=CredentialResponse,
    )
    def show(
        self,
        id: CredentialIdPathParam,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> CredentialResponse:
        """Get a specific credential by ID."""
        # `id` arrives already decoded - never call decode_id() on it again.
        credential = self.manager.get(trans, id)
        return self._serialize(trans, credential)

    @router.put(
        "/api/credentials/{id}",
        summary="Update credential",
        response_model=CredentialResponse,
    )
    def update(
        self,
        id: CredentialIdPathParam,
        payload: UpdateCredentialRequest = Body(...),
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> CredentialResponse:
        """Update an existing credential."""
        credential = self.manager.update(
            trans,
            id,
            name=payload.name,
            username=payload.username,
            password=payload.password,
        )
        return self._serialize(trans, credential)

    @router.delete(
        "/api/credentials/{id}",
        summary="Delete credential",
        status_code=status.HTTP_204_NO_CONTENT,
    )
    def delete(
        self,
        id: CredentialIdPathParam,
        trans: ProvidesUserContext = DependsOnTrans,
    ) -> None:
        """Delete a credential."""
        self.manager.delete(trans, id)

    def _serialize(self, trans: ProvidesUserContext, credential) -> CredentialResponse:
        """Convert model to response schema."""
        return CredentialResponse(
            id=trans.security.encode_id(credential.id),
            name=credential.name,
            vault_type=credential.vault_type,
            create_time=credential.create_time,
            update_time=credential.update_time,
        )
```

### Router with Query Parameters

```python
@router.get(
    "/api/workflows",
    summary="List workflows",
)
def index(
    self,
    trans: ProvidesUserContext = DependsOnTrans,
    show_published: bool = Query(False, description="Include published workflows"),
    show_shared: bool = Query(False, description="Include shared workflows"),
    sort_by: Optional[str] = Query(None, description="Sort field"),
    sort_desc: bool = Query(False, description="Sort descending"),
    limit: int = Query(100, ge=1, le=1000, description="Maximum results"),
    offset: int = Query(0, ge=0, description="Offset for pagination"),
) -> WorkflowListResponse:
    """List workflows with filtering and pagination."""
    workflows = self.manager.list_workflows(
        trans,
        show_published=show_published,
        show_shared=show_shared,
        sort_by=sort_by,
        sort_desc=sort_desc,
        limit=limit,
        offset=offset,
    )
    return WorkflowListResponse(items=workflows)
```

---

## Test Patterns

### Basic API Test Structure

```python
from galaxy_test.base.populators import DatasetPopulator
from ._framework import ApiTestCase


class TestCredentialsApi(ApiTestCase):
    """Tests for /api/credentials endpoints."""

    def setUp(self):
        super().setUp()
        self.dataset_populator = DatasetPopulator(self.galaxy_interactor)

    def test_create_credential(self):
        """Test creating a credential."""
        payload = {
            "name": "My Credential",
            "vault_type": "database",
            "username": "testuser",
            "password": "testpass",
        }
        response = self._post("credentials", data=payload, json=True)
        self._assert_status_code_is(response, 201)
        credential = response.json()
        self._assert_has_keys(credential, "id", "name", "vault_type", "create_time")
        assert credential["name"] == "My Credential"
        assert credential["vault_type"] == "database"

    def test_list_credentials(self):
        """Test listing credentials."""
        # Create test data
        self._create_credential("Credential 1")
        self._create_credential("Credential 2")

        # List
        response = self._get("credentials")
        self._assert_status_code_is_ok(response)
        data = response.json()
        assert "items" in data
        assert "total_count" in data
        assert len(data["items"]) >= 2

    def test_get_credential(self):
        """Test getting a specific credential."""
        credential_id = self._create_credential("Test Credential")
        response = self._get(f"credentials/{credential_id}")
        self._assert_status_code_is_ok(response)
        credential = response.json()
        assert credential["id"] == credential_id

    def test_update_credential(self):
        """Test updating a credential."""
        credential_id = self._create_credential("Original Name")

        payload = {"name": "Updated Name", "username": "newuser"}
        response = self._put(f"credentials/{credential_id}", data=payload, json=True)
        self._assert_status_code_is_ok(response)

        updated = response.json()
        assert updated["name"] == "Updated Name"

    def test_delete_credential(self):
        """Test deleting a credential."""
        credential_id = self._create_credential("To Delete")

        response = self._delete(f"credentials/{credential_id}")
        self._assert_status_code_is(response, 204)

        # Verify it's gone
        response = self._get(f"credentials/{credential_id}")
        self._assert_status_code_is(response, 404)

    def test_cannot_access_other_user_credential(self):
        """Test that users cannot access other users' credentials."""
        # Create as first user
        credential_id = self._create_credential("User 1 Credential")

        # Try to access as different user
        with self._different_user():
            response = self._get(f"credentials/{credential_id}")
            self._assert_status_code_is(response, 403)

    def test_create_with_missing_required_field(self):
        """Test validation error when required field is missing."""
        payload = {"vault_type": "database"}  # Missing 'name'
        response = self._post("credentials", data=payload, json=True)
        self._assert_status_code_is(response, 422)  # Validation error

    def _create_credential(self, name: str, **kwargs) -> str:
        """Helper to create a credential and return its ID."""
        payload = {
            "name": name,
            "vault_type": kwargs.get("vault_type", "database"),
            "username": kwargs.get("username", "testuser"),
            "password": kwargs.get("password", "testpass"),
        }
        response = self._post("credentials", data=payload, json=True)
        self._assert_status_code_is(response, 201)
        return response.json()["id"]
```

### Test with Admin Requirements

```python
from galaxy_test.base.decorators import requires_admin

class TestAdminCredentialsApi(ApiTestCase):

    @requires_admin
    def test_admin_list_all_credentials(self):
        """Test that admins can list all credentials."""
        response = self._get("credentials/admin/all", admin=True)
        self._assert_status_code_is_ok(response)
```

### Test with Dataset Population

```python
def test_workflow_with_dataset(self):
    """Test workflow execution with datasets."""
    history_id = self.dataset_populator.new_history()
    dataset = self.dataset_populator.new_dataset(history_id, content="test data")

    workflow_id = self._create_workflow("Test Workflow")

    payload = {
        "workflow_id": workflow_id,
        "history_id": history_id,
        "inputs": {"input1": {"id": dataset["id"], "src": "hda"}},
    }
    response = self._post("workflows/execute", data=payload, json=True)
    self._assert_status_code_is_ok(response)
```

---

## Common Import Patterns

### Router File Imports

```python
import logging
from typing import Optional, List

from fastapi import (
    Depends,
    Path,
    Query,
    status,
)

from galaxy.managers.context import ProvidesUserContext
from galaxy.managers.myresource import MyResourceManager
from galaxy.schema.fields import DecodedDatabaseIdField
from galaxy.schema.schema import (
    MyResourceRequest,
    MyResourceResponse,
    MyResourceListResponse,
)
from . import (
    depends,
    DependsOnTrans,
    Router,
)
```

> `galaxy.webapps.galaxy.api.depends` **does not exist**. `depends`, `DependsOnTrans` and `Router`
> all come from the `galaxy.webapps.galaxy.api` package itself - inside the package, import them
> with `from . import ...`. Use `EncodedDatabaseIdField` only on **response** schemas, never on an
> inbound path or query parameter.

### Manager File Imports

```python
from typing import List, Optional
from sqlalchemy import select, and_, or_
from galaxy import model, exceptions
from galaxy.managers.context import ProvidesUserContext
from galaxy.model.scoped_session import galaxy_scoped_session
```

### Test File Imports

```python
from galaxy_test.base.populators import (
    DatasetPopulator,
    WorkflowPopulator,
)
from galaxy_test.base.decorators import (
    requires_admin,
    requires_new_user,
)
from ._framework import ApiTestCase
```

---

## Error Handling Patterns

### Manager Error Handling

```python
from galaxy import exceptions

def get(self, trans: ProvidesUserContext, resource_id: int):
    """Get resource with proper error handling."""
    resource = self.sa_session.get(model.MyResource, resource_id)

    if not resource:
        raise exceptions.ObjectNotFound(f"Resource {resource_id} not found")

    if resource.deleted:
        raise exceptions.ObjectNotFound("Resource has been deleted")

    if not self.is_accessible(resource, trans.user):
        raise exceptions.ItemAccessibilityException(
            "You do not have permission to access this resource"
        )

    return resource
```

### Common Galaxy Exceptions

```python
from galaxy import exceptions

# 404 Not Found
raise exceptions.ObjectNotFound("Resource not found")

# 403 Forbidden
raise exceptions.ItemAccessibilityException("Access denied")

# 400 Bad Request
raise exceptions.RequestParameterInvalidException("Invalid parameter value")

# 409 Conflict
raise exceptions.Conflict("Resource already exists")

# 401 Unauthorized
raise exceptions.AuthenticationRequired("Authentication required")
```

---

## Summary Checklist

When creating a new API endpoint, ensure:

- [ ] Pydantic schemas defined in `lib/galaxy/schema/`
- [ ] Manager class created/updated in `lib/galaxy/managers/`
- [ ] FastAPI router created in `lib/galaxy/webapps/galaxy/api/`
- [ ] Router exposed as a module-level `router = Router(...)` (auto-discovered; nothing to register)
- [ ] Manager injected with `depends(Manager)`, not `Depends(...)`
- [ ] Mutating manager methods call `sa_session.commit()`
- [ ] Tests written in `lib/galaxy_test/api/`
- [ ] All tests pass with `./run_tests.sh -api` (or `pytest` directly)
- [ ] OpenAPI docs show correctly at `/api/docs`
- [ ] Manual testing completed
- [ ] Error cases handled properly
- [ ] Access control implemented
- [ ] IDs encoded/decoded correctly
