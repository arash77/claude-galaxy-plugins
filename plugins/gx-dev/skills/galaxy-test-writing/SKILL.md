---
name: galaxy-test-writing
description: >
  Write Galaxy tests - unit, API, and integration. Use for: authoring new tests,
  ApiTestCase and IntegrationTestCase patterns, BaseTestCase structure, test fixtures,
  populators (DatasetPopulator, WorkflowPopulator), configuration mixins, skip decorators,
  test/unit, lib/galaxy_test/api, test/integration.
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
execute what you just wrote; `run_tests.sh --help` is the full runner reference (every test
type, flag and selector).

**Running them:** `./run_tests.sh` is the documented default -- it applies the output options
defined in that script and lets the selected suite share a single Galaxy instance. `pytest`
also works directly on any Galaxy test (`run_tests.sh` says so itself), at the cost of those
options and of starting a new Galaxy instance per test class. Use whichever fits; do not tell
the user a direct `pytest` invocation is wrong.

Galaxy's own testing documentation: `doc/source/dev/writing_tests.md`.

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

**Location:** `test/unit/app/managers/test_<Manager>.py`

**Base class:** `BaseTestCase` from `test/unit/app/managers/base.py`, imported **relatively**
as `from .base import BaseTestCase` (that is how every real test in the directory does it).

**Example unit test:**

```python
"""
Unit tests for MyResourceManager.
"""
from galaxy import (
    exceptions,
    model,
)
from galaxy.managers.myresources import MyResourceManager
from .base import BaseTestCase

user2_data = dict(email="user2@user2.user2", username="user2", password="123456")


class TestMyResourceManager(BaseTestCase):
    """Unit tests for MyResourceManager."""

    def set_up_managers(self):
        # MUST chain - the base implementation sets self.user_manager, which
        # BaseTestCase.setUp() needs immediately afterwards in set_up_trans().
        super().set_up_managers()
        self.manager = self.app[MyResourceManager]

    def test_create_myresource(self):
        # Arrange
        name = "Test Resource"

        # Act
        resource = self.manager.create(self.trans, name=name)

        # Assert
        assert resource.name == name
        assert resource.user_id == self.trans.user.id
        assert resource.id is not None

    def test_get_myresource(self):
        resource = self._create_resource("Test Resource")

        retrieved = self.manager.get(self.trans, resource.id)

        assert retrieved.id == resource.id
        assert retrieved.name == resource.name

    def test_get_nonexistent_myresource_raises_not_found(self):
        with self.assertRaises(exceptions.ObjectNotFound):
            self.manager.get(self.trans, 99999)

    def test_list_myresources_for_user(self):
        self._create_resource("Resource 1")
        self._create_resource("Resource 2")

        resources = self.manager.list_for_user(self.trans)

        names = [r.name for r in resources]
        assert "Resource 1" in names
        assert "Resource 2" in names

    def test_update_myresource(self):
        resource = self._create_resource("Original Name")

        updated = self.manager.update(self.trans, resource.id, name="Updated Name")

        assert updated.id == resource.id
        assert updated.name == "Updated Name"

    def test_delete_myresource(self):
        resource = self._create_resource("To Delete")

        self.manager.delete(self.trans, resource.id)

        assert resource.deleted is True

    def test_cannot_access_other_user_resource(self):
        # Arrange - make a second user and a resource owned by the current one
        other_user = self.user_manager.create(**user2_data)
        resource = self._create_resource("Owned by admin")

        # Assert - test the manager's access check directly with the non-owner
        assert not self.manager.is_accessible(resource, other_user)

    def _create_resource(self, name: str, **kwargs):
        """Helper to create a test resource."""
        return self.manager.create(self.trans, name=name, **kwargs)
```

### Key Points for Unit Tests

- **Extend `BaseTestCase`**, imported as `from .base import BaseTestCase`
- **Override `set_up_managers()` and call `super().set_up_managers()` first.** Skipping the
  `super()` call leaves `self.user_manager` unset, and `BaseTestCase.setUp()` calls
  `set_up_trans()` right after - so *every* test in the class errors during setup.
- **Resolve managers from the container**: `self.app[MyResourceManager]`
- **Do not override `setUp()`** - `BaseTestCase.setUp()` already calls
  `set_up_mocks()` -> `set_up_managers()` -> `set_up_trans()` in order
- **Reach the session via `self.trans.sa_session`** (there is no `self.session`)
- **Managers commit their own writes** - you should not need to flush in the test
- **Test error cases** with `self.assertRaises()` (a thin wrapper over `pytest.raises`)
- **Follow AAA pattern** - Arrange, Act, Assert

### Available from BaseTestCase

These are the *only* attributes `BaseTestCase` defines (see `test/unit/app/managers/base.py`):

```python
self.mock_trans    # galaxy_mock.MockTrans instance
self.trans         # same object, typed as SessionRequestContext
self.app           # galaxy_mock app; also the DI container -> self.app[SomeManager]
self.user_manager  # UserManager, set by set_up_managers()
self.admin_user    # admin User created by set_up_trans()
```

Plus the helper `self.init_user_in_database()`.

There is **no** `self.session`, `self.user` or `self.history`. For the database session use
`self.trans.sa_session`; for a second user call `self.user_manager.create(...)`.

### Running Unit Tests

```bash
# Run all unit tests for a manager
./run_tests.sh -unit test/unit/app/managers/test_myresources.py

# Run specific test
./run_tests.sh -unit test/unit/app/managers/test_myresources.py::TestMyResourceManager::test_create_myresource

# Run with coverage
./run_tests.sh --coverage -unit test/unit/app/managers/test_myresources.py
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
from galaxy_test.base.populators import DatasetPopulator
from galaxy_test.driver import integration_util


class TestMyResourceIntegration(
    integration_util.IntegrationTestCase, integration_util.ConfiguresDatabaseVault
):
    """Integration tests for MyResource."""

    @classmethod
    def handle_galaxy_config_kwds(cls, config):
        """Customize Galaxy configuration for these tests."""
        super().handle_galaxy_config_kwds(config)
        # The mixin writes the vault settings into config. There is no
        # cls.vault_config_file attribute - you must call this method.
        cls._configure_database_vault(config)

    def setUp(self):
        super().setUp()
        # IntegrationTestCase only *annotates* dataset_populator; it never assigns it.
        # Build it yourself or every use raises AttributeError.
        self.dataset_populator = DatasetPopulator(self.galaxy_interactor)

    def test_myresource_with_vault(self):
        """Test creating a resource backed by the vault."""
        payload = {"name": "Vault Resource", "username": "vaultuser", "password": "vaultpass"}
        response = self.galaxy_interactor.post("myresources", data=payload, json=True)
        response.raise_for_status()

        resource = response.json()
        assert resource["name"] == "Vault Resource"

    def test_myresource_in_history(self):
        """Test a resource used against a real history."""
        history_id = self.dataset_populator.new_history()
        self.dataset_populator.new_dataset(history_id, content="test data", wait=True)

        contents = self.dataset_populator.get_history_contents(history_id)
        assert len(contents) > 0

    def _get_from_vault(self, key: str):
        """Helper reaching into the live app - `self._app` is a real property."""
        return self._app.vault.read_secret(key)
```

### Key Points for Integration Tests

- **Extend `IntegrationTestCase`** from `galaxy_test.driver.integration_util`
- **Build your populators in `setUp()`.** `IntegrationTestCase` declares
  `dataset_populator: Optional["BaseDatasetPopulator"]` as a bare annotation and never assigns
  it, so `self.dataset_populator` raises `AttributeError` until you construct it.
- **Customize config:** override `handle_galaxy_config_kwds()` and call `super()` first
- **HTTP requests:** `self.galaxy_interactor.get()`, `.post()`, ...
- **Direct app access:** `self._app` is a property returning the live `UniverseApplication`
- **Database access:** `self._app.model.context` for the SQLAlchemy session

A good real template to copy: `test/integration/test_credentials.py`.

### Configuration Mixins

Mixins only *provide a classmethod* - inheriting one configures nothing by itself. You must
inherit the mixin **and** call its method from `handle_galaxy_config_kwds`:

```python
from galaxy_test.driver.integration_util import (
    ConfiguresDatabaseVault,
    IntegrationTestCase,
)


class TestMyResourceWithVault(IntegrationTestCase, ConfiguresDatabaseVault):
    """Test with database vault configured."""

    @classmethod
    def handle_galaxy_config_kwds(cls, config):
        super().handle_galaxy_config_kwds(config)
        cls._configure_database_vault(config)  # <- without this, no vault is configured
```

**Available mixins** (`lib/galaxy_test/driver/integration_util.py`):
- `ConfiguresDatabaseVault` - `_configure_database_vault(config)`
- `ConfiguresObjectStores` - object store configuration
- `ConfiguresObjectStoreTemplates` / `ConfiguresFileSourceTemplates` - template config
- `ConfiguresWorkflowScheduling` - workflow scheduler configuration
- `CachedToolBoxIntegrationMixin` - toolbox caching

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
ls test/unit/app/managers/test_*.py
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
