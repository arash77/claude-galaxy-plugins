---
name: galaxy-context
description: >
  Galaxy project development conventions and skill routing guide.
  ALWAYS load this skill when working in a Galaxy codebase.
  Routes to appropriate skills: use /gx-dev:galaxy-db-migration for database/Alembic/schema changes,
  /gx-dev:galaxy-api-endpoint for creating REST API endpoints/FastAPI routers,
  /gx-dev:galaxy-test-writing for writing unit, API, and integration tests,
  /gx-dev:galaxy-linting for code formatting/linting/type checking.
  Use galaxy-explorer agent for codebase architecture questions.
user-invocable: false
---

# Galaxy Development Context & Skill Routing

This skill provides essential Galaxy conventions and routing guidance to help you proactively use the right Galaxy skills and agents.

## Automatic Skill Invocation - CRITICAL

**You should PROACTIVELY invoke Galaxy skills when you detect relevant tasks**, without waiting for explicit user requests. Match user intent to skills below.

---

## Skill Routing Guide

### 1. Database Operations → /gx-dev:galaxy-db-migration

**Invoke when user mentions:**
- "database", "schema", "migration", "Alembic"
- "add column", "create table", "modify database", "database version"
- "upgrade database", "downgrade database", "migration error"
- SQL model changes in `lib/galaxy/model/__init__.py`
- Alembic revision files in `lib/galaxy/model/migrations/alembic/versions_gxy/`

**Examples:**
- "I need to add a new table for credentials" → `/gx-dev:galaxy-db-migration create`
- "How do I add a column to the workflow table?" → `/gx-dev:galaxy-db-migration create`
- "The database won't upgrade" → `/gx-dev:galaxy-db-migration troubleshoot`
- "Check if migration is needed" → `/gx-dev:galaxy-db-migration status`

**Actions:**
- `create` - Creating new migrations after model changes
- `upgrade` - Upgrading database to latest version
- `downgrade` - Rolling back migrations
- `status` - Checking database version vs codebase
- `troubleshoot` - Diagnosing migration errors

---

### 2. API Development → /gx-dev:galaxy-api-endpoint

**Invoke when user mentions:**
- "API", "endpoint", "REST", "route", "router"
- "FastAPI", "Pydantic", "schema", "request/response model"
- "create endpoint for...", "add API for...", "new endpoint"
- Files in `lib/galaxy/webapps/galaxy/api/`
- Files in `lib/galaxy/schema/`

**Examples:**
- "Create an API endpoint for managing credentials" → `/gx-dev:galaxy-api-endpoint credentials`
- "I need a REST API for workflows" → `/gx-dev:galaxy-api-endpoint workflows`
- "Add a new route to handle..." → `/gx-dev:galaxy-api-endpoint [resource-name]`

**Action:**
- Always pass the resource name as argument (e.g., `/gx-dev:galaxy-api-endpoint credentials`)
- Guides through: Pydantic schemas → Manager logic → FastAPI router → Tests

---

### 3. Writing Tests → /gx-dev:galaxy-test-writing

**Invoke when user mentions:**
- "write tests", "add test", "test this feature"
- "ApiTestCase", "IntegrationTestCase", "BaseTestCase"
- "test fixtures", "populators", "DatasetPopulator"
- New test files under `test/unit/`, `lib/galaxy_test/api/`, `test/integration/`

**Examples:**
- "Write tests for this new API" → `/gx-dev:galaxy-test-writing api`
- "How do I structure a unit test for this manager?" → `/gx-dev:galaxy-test-writing unit`
- "Add an integration test with a custom config" → `/gx-dev:galaxy-test-writing integration`

**Actions:**
- `unit` - Unit test patterns (BaseTestCase, mocked dependencies)
- `api` - API test patterns (ApiTestCase, populators, fixtures)
- `integration` - Integration test patterns (config mixins, skip decorators)

**For *running* tests**, this plugin defers to the `galaxy-test-runner` skill in the
`gx-test-runner` plugin -- install it from this marketplace for the full `./run_tests.sh`
reference (test types, flags, selectors). Each writing guide here still ends with the one
command needed to execute the test you just wrote.

---

### 4. Linting & Formatting → /gx-dev:galaxy-linting

**Invoke when user mentions:**
- "lint", "linting", "format", "formatting", "code style"
- "ruff", "black", "isort", "flake8", "mypy", "darker", "autoflake", "pyupgrade"
- "eslint", "prettier", "type check", "type error"
- "tox -e lint", "tox -e format", "make format", "make pyupgrade"
- "CI lint", "lint failure", "lint error", "formatting error"
- "fix formatting", "auto-fix", "clean up code style"
- "unused imports", "modernize Python", "remove imports"
- "API schema lint", "config lint", "XSD", "codespell", "redocly"

**Examples:**
- "Run lint checks" → `/gx-dev:galaxy-linting check`
- "Fix formatting issues" → `/gx-dev:galaxy-linting fix`
- "How do I format Python code?" → `/gx-dev:galaxy-linting python`
- "Type checking errors" → `/gx-dev:galaxy-linting mypy`
- "Run all CI checks" → `/gx-dev:galaxy-linting full`
- "Client-side linting" → `/gx-dev:galaxy-linting client`
- "Remove unused imports" → `/gx-dev:galaxy-linting fix`
- "Modernize Python syntax" → `/gx-dev:galaxy-linting fix`
- "Lint API schema" → `/gx-dev:galaxy-linting` (for specialized targets)

**Actions:**
- `check` - Quick lint check (format + lint, fastest feedback)
- `fix` - Auto-fix formatting (make diff-format, make format, make pyupgrade, autoflake)
- `python` - Python linting details (ruff, black, isort, flake8, darker, pyupgrade)
- `client` - Client-side linting (ESLint, Prettier, granular targets)
- `mypy` - Type checking with mypy
- `full` - Complete lint suite (all CI checks)
- No argument - For specialized targets (API schema, XSD, config files)

---

### 5. Codebase Architecture Questions → galaxy-explorer Agent

**Use Task tool with `subagent_type="gx-dev:galaxy-explorer"` when user asks:**
- "Where is X implemented?"
- "How does Y work?"
- "What's the architecture of Z?"
- "Find the component that handles..."
- "Show me examples of..."
- Broad exploration questions about code structure

**Examples:**
- "Where is authentication handled?" → Spawn galaxy-explorer
- "How do workflows execute?" → Spawn galaxy-explorer
- "What's the testing structure?" → Spawn galaxy-explorer

**Why use galaxy-explorer:**
- Knows Galaxy architecture deeply
- Avoids reading large files completely
- Can answer multi-file architectural questions efficiently
- Understands Galaxy conventions and patterns

**Do NOT use galaxy-explorer for:**
- Specific needle queries (use Grep/Glob directly)
- Reading known file paths (use Read directly)
- Simple file searches (use Glob directly)

---

## Critical Galaxy Conventions

### Testing: ALWAYS use run_tests.sh

**NEVER run pytest directly** - Galaxy's test suite requires special setup:

```bash
# Correct
./run_tests.sh -integration test/integration/test_credentials.py

# Wrong - will fail or miss fixtures
pytest test/integration/test_credentials.py
```

**Why:**
- Sets up Galaxy test database
- Configures Galaxy-specific fixtures
- Manages test isolation
- Handles Galaxy configuration

### Large Files: NEVER Read Completely

**These files will exhaust your token budget if read entirely:**
- `client/src/api/schema/schema.ts` (46,529 lines) - Auto-generated
- `lib/galaxy/model/__init__.py` (12,677 lines) - Core models
- `lib/galaxy/tools/__init__.py` (4,857 lines) - Tool framework
- `lib/galaxy/schema/schema.py` (4,184 lines) - API schemas

**Instead:**
- Use Grep with specific patterns
- Read with offset/limit to get relevant sections
- Use galaxy-explorer agent for architectural understanding

### Manager Pattern

Business logic lives in manager classes:
- Location: `lib/galaxy/managers/`
- Pattern: `{Resource}Manager` (e.g., `WorkflowsManager`)
- Purpose: Separate business logic from API layer

**Flow:** API Router → Manager → Model

### FastAPI Structure

Modern Galaxy API follows FastAPI patterns:
- Routers: `lib/galaxy/webapps/galaxy/api/*.py`
- Schemas: `lib/galaxy/schema/*.py` (Pydantic models)
- Tests: `lib/galaxy_test/api/test_*.py`

---

## Quick Architecture Reference

### Backend (Python)

```
lib/galaxy/
├── model/                 # SQLAlchemy models
│   └── migrations/       # Alembic migrations
├── managers/             # Business logic (Manager pattern)
├── schema/               # Pydantic API schemas
├── webapps/galaxy/api/   # FastAPI routers
├── tools/                # Tool execution engine
└── workflow/             # Workflow engine
```

### Frontend (Vue.js)

```
client/src/
├── components/           # Vue components
├── stores/               # Pinia stores
├── composables/          # Composition functions
└── api/                  # API client + generated schemas
```

### Tests

```
test/
├── unit/                 # Fast unit tests
├── integration/          # Integration tests
└── integration_selenium/ # E2E browser tests

lib/galaxy_test/
└── api/                  # API endpoint tests
```

---

## When to Use Each Approach

### Parallel Tool Calls
When operations are independent:
```
Read: lib/galaxy/managers/workflows.py
Read: client/src/stores/workflowStore.ts
Read: test/unit/test_workflows.py
```

### Targeted Searches
For specific patterns in large files:
```
Grep: pattern="class Workflow\(" path="lib/galaxy/model/__init__.py" output_mode="content" -A=20
```

### Galaxy-Explorer Agent
For understanding architecture or locating functionality across multiple files.

### Direct Tool Use
For known file paths or simple operations.

---

## Proactive Skill Usage Summary

**Before starting implementation:**
1. Detect user intent from their message
2. Match to skill routing guide above
3. Invoke appropriate skill BEFORE writing code
4. Follow skill's guided workflow

**Key principle:** Skills prevent mistakes by enforcing Galaxy conventions and best practices. Use them proactively rather than reactively.

---

## Additional Notes

- Galaxy uses Python 3.9+ with FastAPI, SQLAlchemy 2.0, Celery, Pydantic
- Frontend: Vue.js 2.7 with TypeScript, Pinia, Vite
- Main branch: `dev` (not `main`)
- Code style: Black (120 chars), isort, Ruff, mypy with strict mode
- Always use type hints in Python
- Prefer TypeScript over JavaScript for new frontend code
- Linting tools: ruff (lint + format), black, isort, flake8, mypy, autoflake, pyupgrade, ESLint, Prettier
- Specialized linting: codespell, redocly (API schema), xmllint (XSD), config validators
- Quick formatting tip: Use `make diff-format` during development for fast incremental formatting

This context is optimized for the Galaxy codebase as of January 2026.
