---
name: galaxy-test-writing
description: >
  Write Galaxy tests - unit, API, and integration. Use for: authoring new tests,
  ApiTestCase and IntegrationTestCase patterns, BaseTestCase structure, test fixtures,
  populators (DatasetPopulator, WorkflowPopulator), configuration mixins, skip decorators,
  test/unit, lib/galaxy_test/api, test/integration.
  For *running* tests, see the galaxy-test-runner skill in the gx-test-runner plugin.
argument-hint: "[unit|api|integration]"
---

Persona: You are a senior Galaxy QA engineer specializing in pytest and Galaxy's test infrastructure.

Arguments:
- $ARGUMENTS - Optional: "unit", "api", "integration"
  Examples: "", "unit", "api", "integration"

Parse $ARGUMENTS to determine which guidance to provide.

---

## Galaxy Test Writing Guide

This skill covers *writing* Galaxy tests. Each guide ends with the one command needed to
execute what you just wrote; for the full runner reference -- every test type, flag,
and selector -- install the `gx-test-runner` plugin from this marketplace.

**CRITICAL:** Always use `./run_tests.sh`, never run `pytest` directly. It handles virtualenv
setup, dependency installation, and common startup that raw pytest would miss.

See `reference.md` in this skill directory for base-class API references, populator usage,
and per-test-type checklists.

---

## If $ARGUMENTS is empty: Ask Which Test Type

Ask the user what type of test they want to write:

1. **Unit tests** - Fast, isolated tests with mocked dependencies
2. **API tests** - Test API endpoints with Galaxy server
3. **Integration tests** - Full system tests with real database

Then provide guidance based on their choice (see sections below).

---

## If $ARGUMENTS contains "unit": Unit Test Writing Guide

### What Are Unit Tests?

Unit tests are fast, isolated tests that:
- Run without starting Galaxy server
- Use in-memory SQLite database
- Mock external dependencies
- Test individual manager/service methods
- Are located in `test/unit/`

### Unit Test Structure

**Location:** `test/unit/<module>/test_<class>.py`

**Base class:** `BaseTestCase` from `test.unit.app.managers.base`

**Example unit test:**

```python
"""
Unit tests for MyResourceManager.
"""
from galaxy import model
from galaxy.managers.myresources import MyResourceManager
from test.unit.app.managers.base import BaseTestCase


class TestMyResourceManager(BaseTestCase):
    """Unit tests for MyResourceManager."""

    def setUp(self):
        super().setUp()
        self.set_up_managers()

    def set_up_managers(self):
        """Set up managers under test."""
        self.manager = MyResourceManager(self.app)

    def test_create_myresource(self):
        """Test creating a resource."""
        # Arrange
        trans = self.trans  # MockTrans from BaseTestCase
        name = "Test Resource"

        # Act
        resource = self.manager.create(trans, name=name)
        self.session.flush()

        # Assert
        assert resource.name == name
        assert resource.user_id == trans.user.id
        assert resource.id is not None

    def test_get_myresource(self):
        """Test getting a resource by ID."""
        # Arrange
        resource = self._create_resource("Test Resource")

        # Act
        retrieved = self.manager.get(self.trans, resource.id)

        # Assert
        assert retrieved.id == resource.id
        assert retrieved.name == resource.name

    def test_get_nonexistent_myresource_raises_not_found(self):
        """Test that getting nonexistent resource raises exception."""
        from galaxy.exceptions import ObjectNotFound

        with self.assertRaises(ObjectNotFound):
            self.manager.get(self.trans, 99999)

    def test_list_myresources_for_user(self):
        """Test listing resources for current user."""
        # Arrange
        self._create_resource("Resource 1")
        self._create_resource("Resource 2")

        # Act
        resources = self.manager.list_for_user(self.trans)

        # Assert
        assert len(resources) >= 2
        names = [r.name for r in resources]
        assert "Resource 1" in names
        assert "Resource 2" in names

    def test_update_myresource(self):
        """Test updating a resource."""
        # Arrange
        resource = self._create_resource("Original Name")
        new_name = "Updated Name"

        # Act
        updated = self.manager.update(self.trans, resource.id, name=new_name)
        self.session.flush()

        # Assert
        assert updated.id == resource.id
        assert updated.name == new_name

    def test_delete_myresource(self):
        """Test soft-deleting a resource."""
        # Arrange
        resource = self._create_resource("To Delete")

        # Act
        self.manager.delete(self.trans, resource.id)
        self.session.flush()

        # Assert
        assert resource.deleted is True

    def test_cannot_access_other_user_resource(self):
        """Test access control for other users' resources."""
        from galaxy.exceptions import ItemAccessibilityException

        # Arrange
        other_user = self._create_user("other@example.com")
        other_trans = self._create_trans(user=other_user)
        resource = self.manager.create(other_trans, name="Other User Resource")
        self.session.flush()

        # Act & Assert
        with self.assertRaises(ItemAccessibilityException):
            self.manager.get(self.trans, resource.id)

    def _create_resource(self, name: str, **kwargs):
        """Helper to create a test resource."""
        resource = self.manager.create(self.trans, name=name, **kwargs)
        self.session.flush()
        return resource

    def _create_user(self, email: str):
        """Helper to create a test user."""
        user = model.User(email=email, username=email.split("@")[0])
        self.session.add(user)
        self.session.flush()
        return user

    def _create_trans(self, user=None):
        """Helper to create a transaction context for a user."""
        from galaxy_mock import MockTrans
        return MockTrans(app=self.app, user=user or self.user)
```

### Key Points for Unit Tests

- **Extend `BaseTestCase`** from `test.unit.app.managers.base`
- **Use `self.trans`** - Pre-configured MockTrans with test user
- **Use `self.session`** - SQLAlchemy session (in-memory SQLite)
- **Call `self.session.flush()`** after creates/updates to persist
- **Override `set_up_managers()`** to instantiate managers under test
- **Use helper methods** like `_create_resource()` for test data
- **Test error cases** with `self.assertRaises()`
- **Follow AAA pattern** - Arrange, Act, Assert

### Available from BaseTestCase

```python
self.app          # Galaxy application mock
self.trans        # MockTrans with test user
self.user         # Test user (admin)
self.session      # SQLAlchemy session
self.history      # Default test history
```

### Running Unit Tests

```bash
# Run all unit tests for a manager
./run_tests.sh -unit test/unit/managers/test_myresources.py

# Run specific test
./run_tests.sh -unit test/unit/managers/test_myresources.py::TestMyResourceManager::test_create_myresource

# Run with coverage
./run_tests.sh --coverage -unit test/unit/managers/test_myresources.py
```

---

## If $ARGUMENTS contains "api": API Test Writing Guide

### What Are API Tests?

API tests:
- Start a Galaxy server
- Make HTTP requests to API endpoints
- Test request/response handling
- Verify status codes and response schemas
- Are located in `lib/galaxy_test/api/`

### API Test Structure

**Location:** `lib/galaxy_test/api/test_<resource>s.py`

**Base class:** `ApiTestCase` from `lib/galaxy_test/api/_framework`

**Example API test:**

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
        """Test POST /api/myresources."""
        payload = {
            "name": "Test Resource",
            "description": "Test description",
        }
        response = self._post("myresources", data=payload, json=True)
        self._assert_status_code_is(response, 201)

        resource = response.json()
        self._assert_has_keys(resource, "id", "name", "description", "create_time")
        assert resource["name"] == "Test Resource"
        assert resource["description"] == "Test description"

    def test_list_myresources(self):
        """Test GET /api/myresources."""
        # Create test data
        self._create_myresource("Resource 1")
        self._create_myresource("Resource 2")

        # List
        response = self._get("myresources")
        self._assert_status_code_is_ok(response)

        data = response.json()
        assert "items" in data
        assert "total_count" in data
        assert data["total_count"] >= 2
        assert len(data["items"]) >= 2

    def test_get_myresource(self):
        """Test GET /api/myresources/{id}."""
        resource_id = self._create_myresource("Test Resource")

        response = self._get(f"myresources/{resource_id}")
        self._assert_status_code_is_ok(response)

        resource = response.json()
        assert resource["id"] == resource_id
        assert resource["name"] == "Test Resource"

    def test_update_myresource(self):
        """Test PUT /api/myresources/{id}."""
        resource_id = self._create_myresource("Original Name")

        payload = {"name": "Updated Name"}
        response = self._put(f"myresources/{resource_id}", data=payload, json=True)
        self._assert_status_code_is_ok(response)

        updated = response.json()
        assert updated["name"] == "Updated Name"

    def test_delete_myresource(self):
        """Test DELETE /api/myresources/{id}."""
        resource_id = self._create_myresource("To Delete")

        response = self._delete(f"myresources/{resource_id}")
        self._assert_status_code_is(response, 204)

        # Verify deletion
        response = self._get(f"myresources/{resource_id}")
        self._assert_status_code_is(response, 404)

    def test_get_nonexistent_myresource_returns_404(self):
        """Test that getting nonexistent resource returns 404."""
        response = self._get("myresources/invalid_id")
        self._assert_status_code_is(response, 404)

    def test_create_with_invalid_data_returns_422(self):
        """Test validation error handling."""
        payload = {}  # Missing required 'name'
        response = self._post("myresources", data=payload, json=True)
        self._assert_status_code_is(response, 422)

    def test_access_control_prevents_viewing_other_user_resource(self):
        """Test that users cannot access other users' resources."""
        # Create as first user
        resource_id = self._create_myresource("User 1 Resource")

        # Switch to different user
        with self._different_user():
            response = self._get(f"myresources/{resource_id}")
            self._assert_status_code_is(response, 403)

    def test_admin_can_access_all_resources(self):
        """Test that admin users have broader access."""
        # Create as regular user
        resource_id = self._create_myresource("User Resource")

        # Access as admin
        response = self._get(f"myresources/{resource_id}", admin=True)
        self._assert_status_code_is_ok(response)

    def _create_myresource(self, name: str, **kwargs) -> str:
        """Helper to create a resource and return its ID."""
        payload = {
            "name": name,
            "description": kwargs.get("description", f"Description for {name}"),
        }
        response = self._post("myresources", data=payload, json=True)
        self._assert_status_code_is(response, 201)
        return response.json()["id"]
```

### Key Points for API Tests

- **Extend `ApiTestCase`** from `lib/galaxy_test/api/_framework`
- **HTTP methods:**
  - `self._get(path)` - GET request
  - `self._post(path, data=..., json=True)` - POST request
  - `self._put(path, data=..., json=True)` - PUT request
  - `self._delete(path)` - DELETE request
  - Paths are relative to `/api/` (e.g., `"myresources"` → `/api/myresources`)
- **Assertions:**
  - `self._assert_status_code_is(response, 200)` - Check specific status
  - `self._assert_status_code_is_ok(response)` - Check 2xx status
  - `self._assert_has_keys(obj, "key1", "key2")` - Verify response structure
- **User context:**
  - Default: Regular user
  - `admin=True` parameter: Make request as admin
  - `self._different_user()` context manager: Switch to different user
- **Test data:**
  - Use helper methods like `_create_myresource()`
  - Use `DatasetPopulator` for creating datasets/histories

### Additional ApiTestCase Features

**Create test datasets:**
```python
def setUp(self):
    super().setUp()
    self.dataset_populator = DatasetPopulator(self.galaxy_interactor)

def test_with_dataset(self):
    history_id = self.dataset_populator.new_history()
    dataset = self.dataset_populator.new_dataset(history_id, content="test data")
    # Use dataset["id"] in your test
```

**Test as different user:**
```python
with self._different_user():
    response = self._get("myresources")
    # This request is made as a different user
```

**Test with admin privileges:**
```python
response = self._get("myresources/admin/all", admin=True)
```

### Running API Tests

```bash
# Run all API tests for an endpoint
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py

# Run specific test
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py::TestMyResourcesApi::test_create_myresource

# Run with verbose output
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py --verbose_errors
```

---

## If $ARGUMENTS contains "integration": Integration Test Writing Guide

### What Are Integration Tests?

Integration tests:
- Test full system integration
- Use real database (PostgreSQL optional)
- Test complex workflows and interactions
- Can customize Galaxy configuration
- Are located in `test/integration/`

### Integration Test Structure

**Location:** `test/integration/test_<feature>.py`

**Base class:** `IntegrationTestCase` from `lib/galaxy_test/driver/integration_util`

**Example integration test:**

```python
"""
Integration tests for MyResource with vault integration.
"""
from galaxy_test.driver import integration_util


class TestMyResourceIntegration(integration_util.IntegrationTestCase):
    """Integration tests for MyResource."""

    @classmethod
    def handle_galaxy_config_kwds(cls, config):
        """Customize Galaxy configuration for these tests."""
        super().handle_galaxy_config_kwds(config)
        config["vault_config_file"] = cls.vault_config_file
        config["enable_vault"] = True

    def setUp(self):
        super().setUp()

    def test_myresource_with_vault(self):
        """Test creating resource with vault backend."""
        payload = {
            "name": "Vault Resource",
            "vault_type": "hashicorp",
            "username": "vaultuser",
            "password": "vaultpass",
        }
        response = self.galaxy_interactor.post("myresources", data=payload)
        response.raise_for_status()

        resource = response.json()
        assert resource["vault_type"] == "hashicorp"

        # Verify stored in vault
        vault_data = self._get_from_vault(resource["id"])
        assert vault_data["username"] == "vaultuser"

    def test_myresource_workflow_integration(self):
        """Test resource used in workflow."""
        # Create resource
        resource_id = self._create_myresource("Workflow Resource")

        # Create workflow that uses resource
        workflow_id = self._create_workflow_with_resource(resource_id)

        # Execute workflow
        history_id = self.dataset_populator.new_history()
        response = self.galaxy_interactor.post(
            "workflows",
            data={
                "workflow_id": workflow_id,
                "history_id": history_id,
                "resource_id": resource_id,
            }
        )
        response.raise_for_status()

        # Wait for workflow completion
        self.dataset_populator.wait_for_history(history_id)

        # Verify results
        datasets = self.dataset_populator.get_history_datasets(history_id)
        assert len(datasets) > 0

    def _create_myresource(self, name: str) -> str:
        """Helper to create a resource."""
        response = self.galaxy_interactor.post(
            "myresources",
            data={"name": name, "vault_type": "database"}
        )
        response.raise_for_status()
        return response.json()["id"]

    def _get_from_vault(self, resource_id: str):
        """Helper to retrieve data from vault."""
        # Access app internals for verification
        vault = self._app.vault
        return vault.read_secret(f"myresources/{resource_id}")
```

### Key Points for Integration Tests

- **Extend `IntegrationTestCase`** from `lib/galaxy_test/driver.integration_util`
- **Customize config:** Override `handle_galaxy_config_kwds()` to set Galaxy config
- **HTTP requests:** Use `self.galaxy_interactor.get()`, `.post()`, etc.
- **Direct app access:** `self._app` gives access to Galaxy application internals
- **Populators:** Same as API tests - `DatasetPopulator`, `WorkflowPopulator`
- **Database access:** `self._app.model.context` for SQLAlchemy session

### Configuration Mixins

Use mixins to add common configuration:

```python
from galaxy_test.driver.integration_util import (
    IntegrationTestCase,
    ConfiguresDatabaseVault,
)

class TestMyResourceWithVault(IntegrationTestCase, ConfiguresDatabaseVault):
    """Test with database vault configured."""

    @classmethod
    def handle_galaxy_config_kwds(cls, config):
        super().handle_galaxy_config_kwds(config)
        # Additional config here
```

**Available mixins:**
- `ConfiguresDatabaseVault` - Set up database vault
- `ConfiguresObjectStores` - Configure object stores
- `UsesToolshed` - Set up Tool Shed integration

### Skip Decorators

Skip tests based on environment:

```python
from galaxy_test.driver.integration_util import skip_unless_postgres, skip_unless_docker

@skip_unless_postgres()
def test_postgres_specific_feature(self):
    """Test that requires PostgreSQL."""
    pass

@skip_unless_docker()
def test_docker_specific_feature(self):
    """Test that requires Docker."""
    pass
```

### Running Integration Tests

```bash
# Run integration tests
./run_tests.sh -integration test/integration/test_myresources.py

# Run specific test
./run_tests.sh -integration test/integration/test_myresources.py::TestMyResourceIntegration::test_myresource_with_vault

# Run with PostgreSQL
GALAXY_TEST_DBURI=postgresql://user@localhost/galaxytest ./run_tests.sh -integration test/integration/test_myresources.py

# Run with coverage
./run_tests.sh --coverage -integration test/integration/test_myresources.py
```

---

## Test Best Practices

### General Guidelines

1. **Test naming:** Use descriptive names that explain what is being tested
   - Good: `test_create_myresource_with_valid_data`
   - Bad: `test_1`, `test_myresource`

2. **One assertion per test:** Test one thing at a time
   - Good: Separate `test_create`, `test_update`, `test_delete`
   - Bad: One `test_crud` that does everything

3. **Test error cases:** Test both success and failure paths
   - Test 404, 403, 400, 422 responses
   - Test validation errors
   - Test access control

4. **Use helper methods:** Extract common setup into helper methods
   - `_create_myresource()`, `_create_user()`, etc.

5. **Clean test data:** Tests should be independent and repeatable

6. **Follow AAA pattern:**
   - **Arrange:** Set up test data
   - **Act:** Perform the operation
   - **Assert:** Verify the result

### Common Patterns

**Testing lists:**
```python
resources = response.json()["items"]
assert len(resources) >= 2
names = [r["name"] for r in resources]
assert "Resource 1" in names
```

**Testing timestamps:**
```python
from datetime import datetime
resource = response.json()
assert resource["create_time"] is not None
create_time = datetime.fromisoformat(resource["create_time"])
assert create_time < datetime.now()
```

**Testing pagination:**
```python
response = self._get("myresources?limit=10&offset=0")
data = response.json()
assert len(data["items"]) <= 10
assert data["total_count"] >= len(data["items"])
```

---

## Additional Resources

**Key test infrastructure files:**
- `lib/galaxy_test/api/_framework.py` - ApiTestCase base class
- `lib/galaxy_test/driver/integration_util.py` - IntegrationTestCase base class
- `test/unit/app/managers/base.py` - Unit test base class
- `galaxy_test/base/populators.py` - Test data populators

**Example test files to reference:**
```bash
# Find recent API tests
ls -t lib/galaxy_test/api/test_*.py | head -5

# Find recent integration tests
ls -t test/integration/test_*.py | head -5

# Find unit tests
ls test/unit/managers/test_*.py
```

**Running test suites:**
```bash
# All unit tests
./run_tests.sh -unit test/unit/

# All API tests (slow)
./run_tests.sh -api lib/galaxy_test/api/

# All integration tests (very slow)
./run_tests.sh -integration test/integration/
```

---

## Troubleshooting Tests

### Test fails with "database locked"
- Cause: Multiple tests accessing SQLite concurrently
- Solution: Use `pytest-xdist` with `-n` flag or run serially

### Test fails with "port already in use"
- Cause: Previous test server didn't shut down
- Solution: Kill Galaxy processes: `pkill -f 'python.*galaxy'`

### Test fails with "fixture not found"
- Cause: Missing test dependency
- Solution: Check imports and base class

### Integration test timeout
- Cause: Test waiting for long-running job
- Solution: Use `wait_for_history()` with longer timeout

### Cannot import test module
- Cause: Python path not set correctly
- Solution: Always use `./run_tests.sh`, not direct pytest

---

## Quick Reference

| Test Type | Location | Base Class | Use When |
|-----------|----------|------------|----------|
| Unit | `test/unit/` | `BaseTestCase` | Testing manager/service logic |
| API | `lib/galaxy_test/api/` | `ApiTestCase` | Testing API endpoints |
| Integration | `test/integration/` | `IntegrationTestCase` | Testing full system integration |
| Selenium | `lib/galaxy_test/selenium/` | `SeleniumTestCase` | Testing browser UI |

**Running tests:**
- Unit: `./run_tests.sh -unit test/unit/...`
- API: `./run_tests.sh -api lib/galaxy_test/api/...`
- Integration: `./run_tests.sh -integration test/integration/...`

**Common assertions:**
- `self._assert_status_code_is(response, 200)`
- `self._assert_status_code_is_ok(response)`
- `self._assert_has_keys(obj, "key1", "key2")`
- `self.assertRaises(ExceptionType)`

**Common helpers:**
- `self._get(path)`, `self._post(path, data=...)`, `self._put(...)`, `self._delete(...)`
- `self._different_user()` - Context manager for different user
- `DatasetPopulator(self.galaxy_interactor)` - Create test datasets
