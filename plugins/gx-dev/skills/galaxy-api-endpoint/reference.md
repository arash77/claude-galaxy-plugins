# Galaxy API Endpoint Reference

This file contains concrete examples and patterns from the Galaxy codebase for creating API endpoints.

## Complete Example: Job Files API

A real, simple example from `lib/galaxy/webapps/galaxy/api/job_files.py`:

```python
"""
API operations on Job files.
"""
import logging
from typing import Optional

from fastapi import (
    Depends,
    Path,
)

from galaxy.managers.context import ProvidesUserContext
from galaxy.managers.jobs import (
    JobManager,
    summarize_job_files,
)
from galaxy.schema.fields import DecodedDatabaseIdField
from galaxy.schema.schema import JobFile
from galaxy.webapps.galaxy.api import (
    DependsOnTrans,
    Router,
)

log = logging.getLogger(__name__)

router = Router(tags=["jobs"])


@router.cbv
class FastAPIJobFiles:
    job_manager: JobManager = Depends(JobManager)

    @router.get(
        "/api/jobs/{job_id}/files",
        summary="Get a list of files associated with a job",
    )
    def index(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
        job_id: DecodedDatabaseIdField = Path(..., title="Job ID", description="The encoded ID of the job"),
    ) -> list[JobFile]:
        """
        Get a list of files associated with a job.
        """
        job = self.job_manager.get_accessible_job(trans, job_id)
        return summarize_job_files(job)
```

**Key observations:**
- Simple router with one endpoint
- Uses `@router.cbv` class-based view
- Manager injected via `Depends()`
- Transaction context via `DependsOnTrans`
- Path parameter with `DecodedDatabaseIdField` type
- Returns Pydantic model directly (`list[JobFile]`)

---

## Schema Patterns

### Basic Response Model

```python
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
        self.sa_session.flush()
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
        self.sa_session.flush()
        return credential

    def delete(self, trans: ProvidesUserContext, credential_id: int) -> None:
        """Soft-delete a credential."""
        credential = self.get(trans, credential_id)
        credential.deleted = True
        self.sa_session.flush()

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
from fastapi import (
    Depends,
    Path,
    Query,
    status,
)
from galaxy.managers.context import ProvidesUserContext
from galaxy.managers.credentials import CredentialManager
from galaxy.schema.fields import EncodedDatabaseIdField
from galaxy.schema.schema import (
    CreateCredentialRequest,
    UpdateCredentialRequest,
    CredentialResponse,
    CredentialListResponse,
)
from galaxy.webapps.galaxy.api import (
    DependsOnTrans,
    Router,
)
from galaxy.webapps.galaxy.api.depends import get_app

router = Router(tags=["credentials"])

def get_credential_manager(app=Depends(get_app)) -> CredentialManager:
    return CredentialManager(app)

@router.cbv
class FastAPICredentials:
    manager: CredentialManager = Depends(get_credential_manager)

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
        trans: ProvidesUserContext = DependsOnTrans,
        id: EncodedDatabaseIdField = Path(..., description="Credential ID"),
    ) -> CredentialResponse:
        """Get a specific credential by ID."""
        decoded_id = trans.security.decode_id(id)
        credential = self.manager.get(trans, decoded_id)
        return self._serialize(trans, credential)

    @router.put(
        "/api/credentials/{id}",
        summary="Update credential",
        response_model=CredentialResponse,
    )
    def update(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
        id: EncodedDatabaseIdField = Path(..., description="Credential ID"),
        request: UpdateCredentialRequest = ...,
    ) -> CredentialResponse:
        """Update an existing credential."""
        decoded_id = trans.security.decode_id(id)
        credential = self.manager.update(
            trans,
            decoded_id,
            name=request.name,
            username=request.username,
            password=request.password,
        )
        return self._serialize(trans, credential)

    @router.delete(
        "/api/credentials/{id}",
        summary="Delete credential",
        status_code=status.HTTP_204_NO_CONTENT,
    )
    def delete(
        self,
        trans: ProvidesUserContext = DependsOnTrans,
        id: EncodedDatabaseIdField = Path(..., description="Credential ID"),
    ) -> None:
        """Delete a credential."""
        decoded_id = trans.security.decode_id(id)
        self.manager.delete(trans, decoded_id)

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
from galaxy.schema.fields import EncodedDatabaseIdField, DecodedDatabaseIdField
from galaxy.schema.schema import (
    MyResourceRequest,
    MyResourceResponse,
    MyResourceListResponse,
)
from galaxy.webapps.galaxy.api import (
    DependsOnTrans,
    Router,
)
from galaxy.webapps.galaxy.api.depends import get_app
```

### Manager File Imports

```python
from typing import List, Optional
from sqlalchemy import select, and_, or_
from galaxy import model, exceptions
from galaxy.managers.context import ProvidesUserContext
from galaxy.model import Session
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
- [ ] Router registered in `lib/galaxy/webapps/galaxy/buildapp.py`
- [ ] Tests written in `lib/galaxy_test/api/`
- [ ] All tests pass with `./run_tests.sh -api`
- [ ] OpenAPI docs show correctly at `/api/docs`
- [ ] Manual testing completed
- [ ] Error cases handled properly
- [ ] Access control implemented
- [ ] IDs encoded/decoded correctly
