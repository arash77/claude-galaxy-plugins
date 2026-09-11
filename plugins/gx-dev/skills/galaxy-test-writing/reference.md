# Galaxy Testing Reference

This file contains comprehensive reference information for Galaxy's test infrastructure.

## Test Base Class Hierarchy

```
pytest.TestCase (pytest framework)
    │
    ├── BaseTestCase (test/unit/app/managers/base.py)
    │   └── Used for unit tests with mocked dependencies
    │
    ├── UsesApiTestCaseMixin (lib/galaxy_test/api/_framework.py)
    │   ├── ApiTestCase (lib/galaxy_test/api/_framework.py)
    │   │   └── Used for API endpoint tests
    │   │
    │   └── IntegrationTestCase (lib/galaxy_test/driver/integration_util.py)
    │       └── Used for integration tests with full Galaxy
    │
    └── SeleniumTestCase (lib/galaxy_test/selenium/framework.py)
        └── Used for browser-based E2E tests
```

## Key Imports by Test Type

### Unit Tests

```python
# Base class
from test.unit.app.managers.base import BaseTestCase

# Models
from galaxy import model

# Manager under test
from galaxy.managers.myresource import MyResourceManager

# Exceptions
from galaxy import exceptions

# Mock transaction
from galaxy_mock import MockTrans
```

### API Tests

```python
# Base class
from ._framework import ApiTestCase

# Populators
from galaxy_test.base.populators import (
    DatasetPopulator,
    WorkflowPopulator,
    LibraryPopulator,
)

# Decorators
from galaxy_test.base.decorators import (
    requires_admin,
    requires_new_user,
    skip_without_tool,
)
```

### Integration Tests

```python
# Base class and utilities
from galaxy_test.driver import integration_util
from galaxy_test.driver.integration_util import (
    IntegrationTestCase,
    ConfiguresDatabaseVault,
    ConfiguresObjectStores,
    skip_unless_postgres,
    skip_unless_docker,
)

# Populators
from galaxy_test.base.populators import (
    DatasetPopulator,
    WorkflowPopulator,
)
```

## BaseTestCase (Unit Tests) Reference

### Available Attributes

```python
self.app           # Galaxy application mock
self.trans         # MockTrans with test user (admin)
self.user          # Test user object (admin)
self.session       # SQLAlchemy session (in-memory SQLite)
self.history       # Default test history
```

### Common Methods

```python
# Set up managers
def set_up_managers(self):
    self.manager = MyResourceManager(self.app)

# Create test user
def _create_user(self, email: str):
    user = model.User(email=email, username=email.split("@")[0])
    self.session.add(user)
    self.session.flush()
    return user

# Create transaction for user
def _create_trans(self, user=None):
    from galaxy_mock import MockTrans
    return MockTrans(app=self.app, user=user or self.user)
```

### Example Test Template

```python
from test.unit.app.managers.base import BaseTestCase
from galaxy.managers.myresource import MyResourceManager

class TestMyResourceManager(BaseTestCase):

    def setUp(self):
        super().setUp()
        self.set_up_managers()

    def set_up_managers(self):
        self.manager = MyResourceManager(self.app)

    def test_example(self):
        # Arrange
        name = "Test"

        # Act
        result = self.manager.create(self.trans, name=name)
        self.session.flush()

        # Assert
        assert result.name == name
```

## ApiTestCase Reference

### Available Attributes

```python
self.galaxy_interactor  # HTTP client for Galaxy API
self.history_id         # Default test history ID
```

### HTTP Methods

All methods take paths relative to `/api/`:

```python
# GET request
response = self._get("myresources")
response = self._get(f"myresources/{resource_id}")

# POST request
response = self._post("myresources", data=payload, json=True)

# PUT request
response = self._put(f"myresources/{resource_id}", data=payload, json=True)

# DELETE request
response = self._delete(f"myresources/{resource_id}")

# With admin privileges
response = self._get("admin/users", admin=True)
```

### Assertion Methods

```python
# Status code checks
self._assert_status_code_is(response, 200)
self._assert_status_code_is(response, 201)
self._assert_status_code_is(response, 204)
self._assert_status_code_is(response, 404)
self._assert_status_code_is_ok(response)  # Any 2xx

# Response structure
self._assert_has_keys(obj, "id", "name", "create_time")

# Error responses
self._assert_error_code_is(response, error_code=400001)
```

### User Context Methods

```python
# Switch to different user
with self._different_user():
    response = self._get("myresources")
    # Executes as different user

# Switch to anonymous user
with self._different_user(anon=True):
    response = self._get("myresources")
    # Executes as anonymous
```

### Populators

```python
def setUp(self):
    super().setUp()
    self.dataset_populator = DatasetPopulator(self.galaxy_interactor)
    self.workflow_populator = WorkflowPopulator(self.galaxy_interactor)

def test_with_dataset(self):
    # Create history
    history_id = self.dataset_populator.new_history()

    # Create dataset
    dataset = self.dataset_populator.new_dataset(
        history_id,
        content="test content",
        name="Test Dataset"
    )

    # Wait for dataset
    self.dataset_populator.wait_for_dataset(history_id, dataset["id"])

    # Use dataset
    dataset_id = dataset["id"]
```

### Decorators

```python
from galaxy_test.base.decorators import (
    requires_admin,
    requires_new_user,
    skip_without_tool,
)

class TestMyApi(ApiTestCase):

    @requires_admin
    def test_admin_only_endpoint(self):
        """Test that requires admin user."""
        response = self._get("admin/resources", admin=True)
        self._assert_status_code_is_ok(response)

    @requires_new_user
    def test_with_fresh_user(self):
        """Test that requires new user."""
        pass

    @skip_without_tool("cat1")
    def test_requires_tool(self):
        """Test that requires specific tool."""
        pass
```

## IntegrationTestCase Reference

### Configuration

```python
@classmethod
def handle_galaxy_config_kwds(cls, config):
    """Customize Galaxy configuration."""
    super().handle_galaxy_config_kwds(config)
    config["vault_config_file"] = cls.vault_config_file
    config["enable_vault"] = True
    config["database_connection"] = "postgresql://..."
```

### Available Attributes

```python
self._app                 # Galaxy application instance
self.galaxy_interactor    # HTTP client
self.dataset_populator    # Dataset creation helper
self.workflow_populator   # Workflow creation helper
```

### Direct App Access

```python
def test_with_direct_app_access(self):
    """Test using direct app access."""
    # Access database
    session = self._app.model.context
    stmt = select(model.MyResource).where(model.MyResource.name == "Test")
    resource = session.scalars(stmt).first()

    # Access vault
    vault = self._app.vault
    secret = vault.read_secret("path/to/secret")

    # Access security helper
    encoded_id = self._app.security.encode_id(123)
```

### Configuration Mixins

```python
from galaxy_test.driver.integration_util import (
    IntegrationTestCase,
    ConfiguresDatabaseVault,
    ConfiguresObjectStores,
)

class TestWithVault(IntegrationTestCase, ConfiguresDatabaseVault):
    """Test with database vault enabled."""

    @classmethod
    def handle_galaxy_config_kwds(cls, config):
        super().handle_galaxy_config_kwds(config)
        # Vault is already configured by mixin
        config["additional_option"] = "value"
```

**Available mixins:**
- `ConfiguresDatabaseVault` - Database vault configuration
- `ConfiguresObjectStores` - Object store configuration
- `UsesToolshed` - Tool Shed integration

### Skip Decorators

```python
from galaxy_test.driver.integration_util import (
    skip_unless_postgres,
    skip_unless_docker,
    skip_unless_executable,
)

@skip_unless_postgres()
def test_postgres_feature(self):
    """Only runs with PostgreSQL."""
    pass

@skip_unless_docker()
def test_docker_feature(self):
    """Only runs with Docker available."""
    pass

@skip_unless_executable("singularity")
def test_singularity_feature(self):
    """Only runs if singularity is available."""
    pass
```

## Populators Reference

### DatasetPopulator

```python
populator = DatasetPopulator(self.galaxy_interactor)

# Create history
history_id = populator.new_history(name="Test History")

# Create dataset
dataset = populator.new_dataset(
    history_id,
    content="test data",
    name="Test Dataset",
    file_type="txt"
)

# Wait for dataset
populator.wait_for_dataset(history_id, dataset["id"])

# Get dataset content
content = populator.get_dataset_content(history_id, dataset_id=dataset["id"])

# Create collection
collection = populator.create_list_collection(
    history_id,
    [dataset["id"], dataset2["id"]],
    name="Test Collection"
)

# Upload file
dataset = populator.upload_file(
    history_id,
    content="file content",
    filename="test.txt"
)
```

### WorkflowPopulator

```python
populator = WorkflowPopulator(self.galaxy_interactor)

# Create workflow
workflow_id = populator.create_workflow(workflow_dict)

# Run workflow
invocation = populator.invoke_workflow(
    history_id,
    workflow_id,
    inputs={"input1": {"id": dataset_id, "src": "hda"}}
)

# Wait for workflow
populator.wait_for_workflow(workflow_id, invocation["id"], history_id)

# Get workflow
workflow = populator.get_workflow(workflow_id)

# Import workflow from GA file
workflow_id = populator.import_workflow_from_path(path_to_ga_file)
```

### LibraryPopulator

```python
populator = LibraryPopulator(self.galaxy_interactor)

# Create library
library = populator.new_library("Test Library")

# Create folder
folder = populator.new_folder(library["id"], "Test Folder")

# Upload to library
dataset = populator.new_library_dataset("Test Dataset", file_path)
```

## Test Command Reference

### Basic Commands

```bash
# Unit tests
./run_tests.sh -unit test/unit/managers/test_myresource.py

# API tests
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py

# Integration tests
./run_tests.sh -integration test/integration/test_myresources.py

# Specific test method
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py::TestMyResourcesApi::test_create
```

### Useful Flags

```bash
# Verbose output
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py --verbose_errors

# Show print statements
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py -- -s

# Coverage report
./run_tests.sh --coverage -api lib/galaxy_test/api/test_myresources.py

# Debug mode (drop into pdb on failure)
./run_tests.sh --debug -api lib/galaxy_test/api/test_myresources.py

# Run tests matching pattern
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py -- -k "test_create"

# Parallel execution
./run_tests.sh -integration test/integration/ -- -n 4

# With PostgreSQL
GALAXY_TEST_DBURI=postgresql://user@localhost/galaxytest ./run_tests.sh -integration test/integration/test_myresources.py

# Stop on first failure
./run_tests.sh -api lib/galaxy_test/api/test_myresources.py -- -x
```

### Pytest Markers

When using pytest directly:

```bash
# Run only unit tests
pytest -m "unit" test/unit/

# Run only integration tests
pytest -m "integration" test/integration/

# Skip slow tests
pytest -m "not slow" test/

# Run specific markers
pytest -m "requires_postgres" test/integration/
```

## Common Test Patterns

### Testing CRUD Operations

```python
def test_crud_operations(self):
    # Create
    payload = {"name": "Test"}
    response = self._post("myresources", data=payload, json=True)
    self._assert_status_code_is(response, 201)
    resource_id = response.json()["id"]

    # Read
    response = self._get(f"myresources/{resource_id}")
    self._assert_status_code_is_ok(response)

    # Update
    payload = {"name": "Updated"}
    response = self._put(f"myresources/{resource_id}", data=payload, json=True)
    self._assert_status_code_is_ok(response)

    # Delete
    response = self._delete(f"myresources/{resource_id}")
    self._assert_status_code_is(response, 204)
```

### Testing Pagination

```python
def test_pagination(self):
    # Create test data
    for i in range(25):
        self._create_myresource(f"Resource {i}")

    # Test first page
    response = self._get("myresources?limit=10&offset=0")
    self._assert_status_code_is_ok(response)
    data = response.json()
    assert len(data["items"]) == 10
    assert data["total_count"] >= 25

    # Test second page
    response = self._get("myresources?limit=10&offset=10")
    self._assert_status_code_is_ok(response)
    data = response.json()
    assert len(data["items"]) == 10
```

### Testing Access Control

```python
def test_access_control(self):
    # Create as user 1
    resource_id = self._create_myresource("User 1 Resource")

    # Verify user 1 can access
    response = self._get(f"myresources/{resource_id}")
    self._assert_status_code_is_ok(response)

    # Switch to user 2
    with self._different_user():
        # Verify user 2 cannot access
        response = self._get(f"myresources/{resource_id}")
        self._assert_status_code_is(response, 403)

    # Verify admin can access
    response = self._get(f"myresources/{resource_id}", admin=True)
    self._assert_status_code_is_ok(response)
```

### Testing Validation

```python
def test_validation_errors(self):
    # Missing required field
    payload = {}
    response = self._post("myresources", data=payload, json=True)
    self._assert_status_code_is(response, 422)

    # Invalid field type
    payload = {"name": 123}  # Should be string
    response = self._post("myresources", data=payload, json=True)
    self._assert_status_code_is(response, 422)

    # Invalid field value
    payload = {"name": ""}  # Empty string
    response = self._post("myresources", data=payload, json=True)
    self._assert_status_code_is(response, 422)
```

### Testing Error Cases

```python
def test_error_cases(self):
    # Not found
    response = self._get("myresources/nonexistent_id")
    self._assert_status_code_is(response, 404)

    # Unauthorized
    with self._different_user(anon=True):
        response = self._get("myresources")
        self._assert_status_code_is(response, 401)

    # Forbidden
    resource_id = self._create_myresource("Test")
    with self._different_user():
        response = self._delete(f"myresources/{resource_id}")
        self._assert_status_code_is(response, 403)
```

### Testing with Datasets

```python
def test_with_dataset(self):
    # Create history and dataset
    history_id = self.dataset_populator.new_history()
    dataset = self.dataset_populator.new_dataset(
        history_id,
        content="test data"
    )

    # Wait for dataset to be ready
    self.dataset_populator.wait_for_dataset(history_id, dataset["id"])

    # Use dataset in test
    response = self._post(
        "myresources",
        data={"name": "Test", "dataset_id": dataset["id"]},
        json=True
    )
    self._assert_status_code_is(response, 201)
```

### Testing Workflows

```python
def test_workflow_integration(self):
    # Create workflow
    workflow_dict = {
        "name": "Test Workflow",
        "steps": {...}
    }
    workflow_id = self.workflow_populator.create_workflow(workflow_dict)

    # Create input datasets
    history_id = self.dataset_populator.new_history()
    dataset = self.dataset_populator.new_dataset(history_id)

    # Invoke workflow
    invocation = self.workflow_populator.invoke_workflow(
        history_id,
        workflow_id,
        inputs={"input1": {"id": dataset["id"], "src": "hda"}}
    )

    # Wait for completion
    self.workflow_populator.wait_for_workflow(
        workflow_id,
        invocation["id"],
        history_id
    )

    # Verify outputs
    outputs = self.dataset_populator.get_history_datasets(history_id)
    assert len(outputs) > 1  # Original input + workflow outputs
```

## Troubleshooting Guide

### Database Locked Errors

**Symptom:** `sqlite3.OperationalError: database is locked`

**Cause:** Multiple tests accessing SQLite concurrently

**Solutions:**
1. Run tests serially: `./run_tests.sh -unit test/unit/managers/test_myresource.py`
2. Use pytest-xdist: `./run_tests.sh -unit test/unit/ -n auto`
3. For integration tests against PostgreSQL, set `GALAXY_TEST_DBURI`: `GALAXY_TEST_DBURI=postgresql://user@localhost/galaxytest ./run_tests.sh -integration test/integration/`

### Port Already in Use

**Symptom:** `OSError: [Errno 48] Address already in use`

**Cause:** Previous test server didn't shut down

**Solutions:**
1. Kill Galaxy processes: `pkill -f 'python.*galaxy'`
2. Wait a few seconds and retry
3. Check for zombie processes: `ps aux | grep galaxy`

### Import Errors

**Symptom:** `ModuleNotFoundError: No module named '...'`

**Cause:** Python path not configured correctly

**Solutions:**
1. Always use `./run_tests.sh`, not plain `pytest`
2. Ensure dependencies installed: `make update-dependencies`
3. Check virtual environment activated

### Test Timeout

**Symptom:** Test hangs or times out

**Cause:** Waiting for long-running job/workflow

**Solutions:**
1. Use explicit waits: `self.dataset_populator.wait_for_dataset(history_id, dataset_id, timeout=60)`
2. Check job/workflow status in Galaxy logs
3. Increase timeout if legitimately slow operation

### Fixture Not Found

**Symptom:** `pytest.FixtureNotFound: fixture 'some_fixture' not found`

**Cause:** Missing test dependency or incorrect base class

**Solutions:**
1. Check imports at top of test file
2. Ensure correct base class (ApiTestCase, IntegrationTestCase, etc.)
3. Check if fixture defined in conftest.py

## Summary Checklist

### When Writing Unit Tests

- [ ] Extend `BaseTestCase` from `test.unit.app.managers.base`
- [ ] Override `set_up_managers()` to instantiate manager
- [ ] Use `self.trans` for transaction context
- [ ] Use `self.session.flush()` after creates/updates
- [ ] Test both success and error cases
- [ ] Use helper methods for test data creation

### When Writing API Tests

- [ ] Extend `ApiTestCase` from `lib/galaxy_test/api/_framework`
- [ ] Use `self._get()`, `self._post()`, `self._put()`, `self._delete()`
- [ ] Use `self._assert_status_code_is()` for assertions
- [ ] Test all HTTP methods (GET, POST, PUT, DELETE)
- [ ] Test access control with `self._different_user()`
- [ ] Test validation errors (422 responses)
- [ ] Test error cases (404, 403, 401)

### When Writing Integration Tests

- [ ] Extend `IntegrationTestCase` from `lib/galaxy_test/driver.integration_util`
- [ ] Override `handle_galaxy_config_kwds()` if custom config needed
- [ ] Use `self.galaxy_interactor` for HTTP requests
- [ ] Use `self._app` for direct app access
- [ ] Add skip decorators if environment-specific
- [ ] Test full system integration, not just API

### When Running Tests

- [ ] Use `./run_tests.sh`, not plain `pytest`
- [ ] Use appropriate flag: `-unit`, `-api`, `-integration`
- [ ] Run specific test file, not entire suite
- [ ] Use `--verbose_errors` for debugging
- [ ] Use `--coverage` to check coverage
- [ ] Verify all tests pass before committing
