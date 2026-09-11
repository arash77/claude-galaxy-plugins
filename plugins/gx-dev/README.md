# gx-dev

A Claude Code plugin providing skills and an exploration agent for writing Galaxy code — database migrations, API endpoints, tests, and linting.

Where `gx-arch-review` reviews a change after it exists, `gx-dev` guides the writing of it.

## Installation

This plugin is distributed via the `claude-galaxy-plugins` marketplace.

### Via Marketplace (Recommended)

```bash
# Add marketplace
/plugin marketplace add galaxyproject/claude-galaxy-plugins

# Install plugin
/plugin install gx-dev@claude-galaxy-plugins
```

### Development/Local Testing

```bash
claude --plugin-dir /path/to/galaxy-plugins
```

## Usage

Skills are namespaced under `gx-dev:`. Most take an optional argument selecting a sub-topic;
invoked with no argument they present the available options.

`galaxy-context` is not user-invocable — it loads automatically in a Galaxy checkout and routes
to the other skills based on what you ask for.

### Skills

| Skill | Argument | Purpose |
|-------|----------|---------|
| `gx-dev:galaxy-context` | — | Conventions and routing; loads automatically |
| `gx-dev:galaxy-db-migration` | `[create\|upgrade\|downgrade\|status\|troubleshoot]` | Alembic migrations against the `gxy` and `tsi` branches |
| `gx-dev:galaxy-api-endpoint` | `[resource-name]` | FastAPI routers, Pydantic schemas, manager pattern |
| `gx-dev:galaxy-test-writing` | `[unit\|api\|integration]` | Writing unit, API, and integration tests |
| `gx-dev:galaxy-linting` | `[check\|fix\|python\|client\|mypy\|full]` | ruff, black, isort, darker, mypy, ESLint, Prettier |

### Agent

| Agent | Purpose |
|-------|---------|
| `gx-dev:galaxy-explorer` | Read-only codebase exploration — architecture questions, locating patterns, file:line citations |

## Skill Details

### galaxy-context

Loads automatically when working in a Galaxy codebase. Routes to the other skills based on
detected intent, and carries the conventions that apply regardless of task: always use
`./run_tests.sh`, never read `client/src/api/schema/schema.ts` or `lib/galaxy/model/__init__.py`
in full, the manager pattern, and FastAPI router structure.

### galaxy-db-migration

Walks through creating a revision against Galaxy's `gxy` and `tsi` Alembic branches: update the
model, create the revision file, fill in `upgrade()`/`downgrade()`, run, verify. Also covers
upgrade/downgrade/status commands and six named failure modes — deadlock, `IncorrectVersionError`,
startup version mismatch, table-already-exists, missing revision file, and foreign key violations.

### galaxy-api-endpoint

An eight-step path from an empty file to a tested endpoint: find a similar router to model on,
define Pydantic schemas, add the manager method, write the FastAPI router, register it, write
tests, run them, verify against `/api/docs`. `reference.md` carries the longer code examples.

### galaxy-test-writing

Covers *writing* tests: `BaseTestCase` structure for unit tests, `ApiTestCase` and the populators
(`DatasetPopulator`, `WorkflowPopulator`, `LibraryPopulator`) for API tests, and
`IntegrationTestCase` with configuration mixins and skip decorators for integration tests.
`reference.md` carries base-class API references, common test patterns, and per-type checklists.

For *running* tests, install the `gx-test-runner` plugin from this marketplace — it owns the
`./run_tests.sh` reference. Each guide here still ends with the command to execute what you wrote.

### galaxy-linting

Dispatches across `check`, `fix`, `python`, `client`, `mypy`, and `full`. Covers the incremental
path (`make diff-format`, darker) versus the full path (`make format`), per-tool configuration
locations, and what CI actually runs.

### galaxy-explorer

A read-only subagent (`Read`, `Glob`, `Grep`, `Bash`; `Write` and `Edit` disallowed) for
architecture questions. Carries a directory map, grep recipes, and an explicit list of files too
large to read whole. Answers with `file:line` citations.

## Plugin Structure

```
claude-galaxy-plugins/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── gx-dev/
│       ├── agents/
│       │   └── galaxy-explorer.md
│       ├── skills/
│       │   ├── galaxy-context/
│       │   ├── galaxy-db-migration/
│       │   ├── galaxy-api-endpoint/
│       │   ├── galaxy-test-writing/
│       │   └── galaxy-linting/
│       └── README.md
└── README.md
```

## Relationship to Other Plugins

- **`gx-arch-review`** — reviews changes after they are written. Complementary: several topics
  appear in both, from opposite directions (this plugin builds a migration, `gx-review-migration`
  checks one).
- **`gx-test-runner`** — owns running tests. `gx-dev:galaxy-test-writing` deliberately does not
  duplicate its `./run_tests.sh` reference.

## License

MIT License - see [LICENSE](../../LICENSE) for details.
