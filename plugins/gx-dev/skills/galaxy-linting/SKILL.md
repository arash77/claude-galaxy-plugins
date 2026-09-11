---
name: galaxy-linting
description: >
  Galaxy code linting, formatting, and type checking. Run checks, auto-fix formatting,
  Python lint, client lint, mypy type checks. Use for: ruff, flake8, black, isort, darker,
  autoflake, pyupgrade, eslint, prettier, mypy, tox, make format, make diff-format,
  code style, lint failures, CI lint checks, formatting errors, type errors, codespell,
  redocly, api schema, xsd, config lint.
argument-hint: "[check|fix|python|client|mypy|full]"
---

Persona: You are a senior Galaxy developer specializing in code quality, style enforcement, and CI compliance.

Arguments:
- $ARGUMENTS - Optional task specifier: "check", "fix", "python", "client", "mypy", "full"
  Examples: "", "check", "fix", "python", "mypy"

Parse $ARGUMENTS to determine which guidance to provide.

---

## Quick Reference: Galaxy Linting & Formatting

Galaxy uses multiple linting tools enforced through CI:
- **Python:** ruff (lint + format), black, isort, flake8, darker (incremental formatting), mypy (type checking)
- **Client:** ESLint, Prettier
- **CI commands:** `tox -e lint`, `tox -e format`, `tox -e mypy`, `tox -e lint_docstring_include_list`

### Tools Overview

| Tool | Purpose | Config File | Check Command | Fix Command |
|------|---------|-------------|---------------|-------------|
| **ruff** | Fast Python linter + formatter | `pyproject.toml` | `ruff check .` | `ruff check --fix .` |
| **black** | Python code formatter | `pyproject.toml` | `black --check .` | `black .` |
| **isort** | Python import sorter | `pyproject.toml` | `isort --check .` | `isort .` |
| **flake8** | Python linter (legacy) | `.flake8` | `flake8 .` | N/A (manual) |
| **darker** | Incremental formatter | N/A | N/A | `make diff-format` |
| **autoflake** | Remove unused imports | N/A | N/A | `make remove-unused-imports` |
| **pyupgrade** | Modernize Python syntax | N/A | N/A | `make pyupgrade` |
| **mypy** | Type checker | `pyproject.toml` | `tox -e mypy` | N/A (manual) |
| **ESLint** | JavaScript/TypeScript linter | `client/.eslintrc.js` | `make client-lint` | `make client-format` |
| **Prettier** | JS/TS/CSS formatter | `client/prettier.config.js` | `make client-lint` | `make client-format` |

---

## If $ARGUMENTS is empty or "check": Quick Lint Check

**Run the fastest feedback loop to catch most issues:**

```bash
# 1. Check formatting (fast - shows what would change)
tox -e format

# 2. Check linting (catches style violations)
tox -e lint
```

**What these check:**
- **`tox -e format`**: Runs black, isort checks (no changes, just reports)
- **`tox -e lint`**: Runs ruff and flake8 (reports violations)

**If you see errors**, use `/gx-dev:galaxy-linting fix` to auto-fix, or `/gx-dev:galaxy-linting python` for detailed guidance.

**Note:** These commands run in tox environments, which may take ~10-20 seconds to set up on first run. Subsequent runs are faster.

---

## If $ARGUMENTS is "fix": Auto-Fix Formatting

Galaxy provides multiple ways to auto-fix formatting issues.

### Recommended: Incremental Formatting (Faster)

**Fix only changed lines** using darker (via Makefile):

```bash
# Fix formatting for uncommitted changes only
make diff-format
```

**What it does:**
- Runs `darker .`, i.e. black (and isort) only on lines you modified
- **Compares against your last commit, not `origin/dev`.** `Makefile` has plain `darker .`
  with no `--revision`. Once you commit work-in-progress, `make diff-format` stops seeing
  those lines - and CI will still fail on them. Run `make format` (or
  `darker --revision origin/dev .`) before pushing a branch with several commits.
- Much faster than formatting entire codebase
- Recommended for daily development

### Full Formatting (Comprehensive)

**Format entire codebase:**

```bash
# Format all Python files
make format
```

**What it does:**
- Runs black on all Python files
- Runs isort on all Python imports
- Takes longer but ensures full compliance
- Recommended before final commit

### Specific Fixes

**Fix ruff auto-fixable issues:**
```bash
ruff check --fix .
```

**Remove unused imports:**
```bash
make remove-unused-imports
```

**Modernize Python syntax (pyupgrade):**
```bash
make pyupgrade
```
Converts old-style Python patterns to Python 3.8/3.9 idioms (e.g., `typing.List` → `list`, adds walrus operators where appropriate). Skips vendored/generated paths.

**Fix client-side formatting:**
```bash
make client-format
```

### Workflow Recommendation

```bash
# During development (fast):
make diff-format

# Before committing (thorough):
make format
tox -e lint  # Verify no remaining issues
```

---

## If $ARGUMENTS is "python": Python Linting Details

Galaxy's Python linting stack consists of multiple tools, each with a specific purpose.

### Ruff (Primary Linter)

**Fast, modern Python linter** that replaces many tools.

**Check for issues:**
```bash
ruff check .
```

**Auto-fix issues:**
```bash
ruff check --fix .
```

**Configuration:** `pyproject.toml` under `[tool.ruff]` and `[tool.ruff.lint]`

**Enabled rule categories** - the full `select` list, verbatim from `pyproject.toml`:

```toml
select = ["E", "F", "B", "C4", "G", "ISC", "NPY", "UP"]
ignore = ["B008", "B9", "E402", "E501", "G001", "G002", "G004"]
```

- **E**: pycodestyle errors
- **F**: Pyflakes (undefined names, unused imports)
- **B**: bugbear (likely bugs)
- **C4**: comprehensions
- **G**: logging format
- **ISC**: implicit string concatenation
- **NPY**: NumPy rules
- **UP**: pyupgrade / modernization

**Not enabled - do not expect ruff to catch these:**
- **I** (import sorting). `pyproject.toml` says so explicitly: *"We are not selecting 'I' rules
  in ruff yet because the isort option force_grid_wrap=2 is not currently supported."*
  **`ruff check --fix` will not sort your imports** - run `isort .` (or `make format`).
- **W**, **N**, **A** - not in the select list at all.
- **E501** (line length) is in the `ignore` list - delegated to black.

**Common errors and fixes:**
- `F401` - Unused import → Remove import or use `# noqa: F401`
- `F841` - Unused variable → Remove or rename to `_`
- `UP` - Use modern syntax → Let ruff auto-fix with `--fix`

### Black (Code Formatter)

**Opinionated code formatter** - ensures consistent style.

**Check what would change:**
```bash
black --check .
```

**Format files:**
```bash
black .
```

**Configuration:** `pyproject.toml` under `[tool.black]`
- Line length: 120 characters
- Target version: Python 3.9+

**Common scenarios:**
- **"black would reformat"** error in CI → Run `black .` locally
- String formatting → Black enforces double quotes
- Multiline expressions → Black has strong opinions on line breaks

### isort (Import Sorter)

**Sorts and organizes imports** into groups.

**Check import order:**
```bash
isort --check .
```

**Fix import order:**
```bash
isort .
```

**Configuration:** `.isort.cfg` in the repository root - **not** `pyproject.toml`
(there is no `[tool.isort]` section there; the `[tool.ruff.lint.isort]` section in
`pyproject.toml` is a different thing and is inert while "I" rules are unselected).

- `profile = black`
- `line_length = 120`
- `force_grid_wrap = 2` - **Galaxy-specific**: any import of 2+ names gets one name per line.
  This is the setting ruff cannot yet reproduce, and the usual reason `tox -e format` fails.
- `reverse_relative = true`, `force_alphabetical_sort_within_sections = true`

**Import groups (in order):**
1. Standard library imports
2. Third-party imports
3. Local application imports

**Example:**
```python
# Standard library
import os
from typing import Optional

# Third-party
from fastapi import APIRouter
from pydantic import BaseModel

# Local
from galaxy.managers.workflows import WorkflowsManager
from galaxy.schema.schema import StoredWorkflowSummary
```

### Flake8 (Legacy Linter)

**Traditional Python linter** - still used alongside ruff.

**Run flake8 the way CI does** - a bare `flake8 .` is not equivalent:

```bash
bash .ci/flake8_wrapper.sh
```

That wrapper is two passes:

```bash
flake8 --exclude $(paste -sd, .ci/flake8_ignorelist.txt) .
# plus stricter rules for the directories shared with Pulsar
flake8 --ignore=E203,D --max-line-length=150 lib/galaxy/jobs/runners/util/
```

**Configuration:** `.flake8` in the repository root - but note its own `exclude` is only
`lib/tool_shed/test/test_data/repos`. The real exclusions come from
`.ci/flake8_ignorelist.txt` via the wrapper. Running bare `flake8 .` therefore lints
`.venv/` and `node_modules/` while **skipping the second, stricter pass CI enforces**.

**Note:** Ruff covers most flake8 checks. Galaxy maintains flake8 for specific rules not yet in ruff.

### Darker (Incremental Formatter)

**Applies black/isort only to changed lines.**

**Format changed code:**
```bash
make diff-format
```

**How it works:**
- `darker .` - compares the working tree to **the last commit** (not `origin/dev`)
- Runs black and isort only on those modified lines
- Much faster than full formatting
- Ideal for uncommitted work; use `make format` before pushing a multi-commit branch

**Use when:**
- Working on large files with many unchanged lines
- Want fast formatting feedback
- Developing features incrementally

### CI Integration

**GitHub Actions runs these checks:**
- `.github/workflows/lint.yaml` defines CI linting pipeline
- Four tox environments: `format`, `lint`, `mypy`, `lint_docstring_include_list`

**To match CI locally:**
```bash
tox -e format  # Check formatting
tox -e lint    # Check linting
```

### Configuration Files

**Python linting configuration:**
- `pyproject.toml` - ruff, black, isort, mypy config
- `.flake8` - flake8 rules and exclusions
- `tox.ini` - tox environment definitions

**Read these files** to understand specific rules:
```bash
# View ruff config
grep -A 20 '\[tool.ruff\]' pyproject.toml

# View black config
grep -A 10 '\[tool.black\]' pyproject.toml

# View flake8 config
cat .flake8
```

---

## If $ARGUMENTS is "client": Client-Side Linting

Galaxy's client-side code (Vue.js, TypeScript) uses ESLint and Prettier.

### Prerequisites

**Ensure Node.js dependencies are installed:**
```bash
make client-node-deps
```

This installs ESLint, Prettier, and related packages in `client/node_modules/`.

### ESLint (JavaScript/TypeScript Linter)

**Check for linting issues:**
```bash
make client-lint
```

**Configuration:** `client/.eslintrc.js`

**Rules enforced:**
- Vue.js best practices
- TypeScript type safety
- Unused variables
- Console statements (warnings)
- Naming conventions

**Common issues:**
- Unused variables → Remove or prefix with `_`
- Missing types → Add TypeScript type annotations
- Vue template issues → Check component syntax
- Console.log statements → Remove or justify with comments

### Prettier (Code Formatter)

**Format client code:**
```bash
make client-format
```

**Configuration:** `client/prettier.config.js`

**Formats:**
- JavaScript/TypeScript files
- Vue single-file components
- CSS/SCSS files
- JSON files

**Integration with ESLint:**
- Prettier handles formatting (indentation, quotes, line breaks)
- ESLint handles code quality (unused vars, type safety)
- ESLint config includes `prettier` plugin to avoid conflicts

### Granular Makefile Targets

Galaxy's Makefile provides fine-grained control over client linting:

**Run only ESLint (no Prettier check):**
```bash
make client-eslint
```

**Run only Prettier check (no ESLint):**
```bash
make client-format-check
```

**Auto-fix ESLint errors with --fix (no Prettier):**
```bash
make client-lint-autofix
```

**Pre-commit hook linting (for specific file paths):**
```bash
make client-eslint-precommit
```
This is used by Git hooks and operates on specific file paths rather than glob patterns.

**Important distinctions:**
- `make client-lint` = `make client-eslint` + `make client-format-check` (runs both tools)
- `make client-format` = `make client-lint-autofix` + Prettier write (auto-fixes everything)

### Manual Commands

**Run ESLint directly** (the client uses **pnpm**, see `packageManager` in `client/package.json`):
```bash
cd client
pnpm run eslint
```

**Run Prettier directly:**
```bash
cd client
pnpm run format-check   # Check
pnpm run format          # Fix (alias: pnpm run prettier)
```

### Client Lint Workflow

```bash
# 1. Install dependencies (if needed)
make client-node-deps

# 2. Check for issues
make client-lint

# 3. Auto-fix formatting
make client-format

# 4. Manually fix remaining ESLint issues
# (Edit files based on lint output)

# 5. Verify fixed
make client-lint
```

---

## If $ARGUMENTS is "mypy": Type Checking

Galaxy uses **mypy** for static type checking with strict mode enabled.

### Running Type Checks

**Check types across codebase:**
```bash
tox -e mypy
```

**Check specific file or directory:**
```bash
mypy lib/galaxy/managers/workflows.py
mypy lib/galaxy/schema/
```

**Configuration:** `mypy.ini` in the repository root - **not** `pyproject.toml`
(there is no `[tool.mypy]` section there).

**Galaxy is not in strict mode.** The global `[mypy]` section sets moderate options:

- `check_untyped_defs = True`
- `ignore_missing_imports = True`
- `no_implicit_optional = True`, `no_implicit_reexport = True`
- `strict_equality = True`, `warn_redundant_casts = True`
- `warn_unreachable = True`, `warn_unused_ignores = True`

The stricter flags (`disallow_untyped_defs`, `disallow_any_generics`,
`disallow_subclassing_any`, ...) are applied **per module**, under a growing list of
`[mypy-galaxy.some.module]` sections below the comment
`# green list - work on growing these please!`.

So: a missing annotation is only an error if the module you touched is on that green list.
Check `mypy.ini` for your module before assuming a rule applies.

### Common Type Errors

**Error: "Function is missing a type annotation"**
```python
# Bad
def process_workflow(workflow_id):
    ...

# Good
def process_workflow(workflow_id: int) -> dict:
    ...
```

**Error: "Need type annotation for variable"**
```python
# Bad
workflows = []

# Good
workflows: List[Workflow] = []
```

**Error: "Incompatible return value type"**
```python
# Bad
def get_workflow() -> Workflow:
    return None  # Error: None is not Workflow

# Good
from typing import Optional

def get_workflow() -> Optional[Workflow]:
    return None  # OK: Optional[Workflow] allows None
```

**Error: "Incompatible types in assignment"**
```python
# Bad
workflow_id: int = "123"  # Error: str is not int

# Good
workflow_id: int = 123
```

**Error: "Cannot determine type of variable"**
```python
# Bad (mypy can't infer type)
result = get_complex_data()

# Good (explicit annotation)
result: Dict[str, Any] = get_complex_data()
```

### Type Ignoring (Use Sparingly)

When mypy is wrong or you're working with untyped libraries:

```python
# Ignore specific line
result = third_party_function()  # type: ignore[misc]

# Ignore entire function
def legacy_function():  # type: ignore
    ...
```

**Best practice:** Add an issue comment explaining why you're ignoring types.

### Common Typing Patterns

**Optional values:**
```python
from typing import Optional

def find_workflow(workflow_id: int) -> Optional[Workflow]:
    # Returns Workflow or None if not found
    ...
```

**Union types:**
```python
from typing import Union

def process_input(data: Union[str, int]) -> str:
    return str(data)
```

**Generic collections:**
```python
from typing import List, Dict, Set, Tuple

workflows: List[Workflow] = []
metadata: Dict[str, str] = {}
tags: Set[str] = set()
coord: Tuple[int, int] = (10, 20)
```

**Callable types:**
```python
from typing import Callable

def apply_function(func: Callable[[int], str], value: int) -> str:
    return func(value)
```

### Untyped Dependencies

Galaxy does **not** vendor type stubs (there is no `lib/galaxy/type_stubs/`, and no `.pyi`
files under `lib/`). Instead `mypy.ini` sets `ignore_missing_imports = True` globally, and
third-party stub packages are pinned in
`lib/galaxy/dependencies/pinned-typecheck-requirements.txt`.

---

## If $ARGUMENTS is "full": Complete Lint Suite

**Run all CI checks locally** to ensure your code will pass:

### Recommended Order

```bash
# 1. Format check (fast)
tox -e format

# If issues found, fix formatting:
make diff-format  # or make format

# 2. Lint check (fast)
tox -e lint

# If issues found, fix with:
ruff check --fix .

# 3. Type check (medium speed)
tox -e mypy

# Fix type errors manually (see /gx-dev:galaxy-linting mypy)

# 4. Docstring check (fast)
tox -e lint_docstring_include_list

# 5. Client lint (if you changed client code)
make client-lint
make client-format
```

### All-in-One Command

**Run all Python checks:**
```bash
tox -e format && tox -e lint && tox -e mypy && tox -e lint_docstring_include_list
```

**Why this order:**
1. **Formatting first** - Cheapest to fix, may resolve some lint issues
2. **Linting second** - Many auto-fixable issues
3. **Type checking third** - Requires manual fixes, benefits from clean lint
4. **Docstring check last** - Specific to certain files

### Tox Environment Details

**`tox -e format`:**
- Runs: black --check, isort --check
- Purpose: Verify formatting compliance
- Fix with: `make format` or `make diff-format`

**`tox -e lint`:**
- Runs: ruff check, flake8
- Purpose: Catch code style violations
- Fix with: `ruff check --fix .` + manual fixes

**`tox -e mypy`:**
- Runs: mypy on selected directories
- Purpose: Type checking
- Fix with: Add type hints manually

**`tox -e lint_docstring_include_list`:**
- Runs: `bash .ci/flake8_wrapper_docstrings.sh --include` - **flake8**, not ruff
  (ruff selects no `D` rules at all)
- Purpose: Enforce docstring standards on the modules in the include list
- Fix with: Add/improve docstrings

### CI Workflow File

**See the full CI setup:**
```bash
cat .github/workflows/lint.yaml
```

This shows exactly what CI runs. Match these checks locally to avoid CI failures.

---

## Other Lint Targets

Galaxy's Makefile includes additional linting targets for specialized use cases:

### API Schema Linting

**Lint OpenAPI schema:**
```bash
make lint-api-schema
```
Builds the OpenAPI schema, then lints it with Redocly CLI and checks spelling with codespell. Useful when modifying API endpoints or Pydantic schemas.

### XSD Schema Formatting

**Format Galaxy's tool XSD schema:**
```bash
make format-xsd
```
Formats Galaxy's tool XSD schema (`lib/galaxy/tool_util/xsd/galaxy.xsd`) with xmllint. Use when modifying tool schema definitions.

### Configuration File Linting

**Lint Galaxy configuration files:**
```bash
make config-lint              # Lint galaxy.yml
make tool-shed-config-lint    # Lint tool_shed.yml
make reports-config-lint      # Lint reports.yml
```
Validates YAML configuration files for syntax and structure issues. Run these when modifying Galaxy configuration schemas.

### File Sources Validation

**Validate file sources configuration:**
```bash
make files-sources-lint          # Validate file sources config
make files-sources-lint-verbose  # Verbose output with details
```
Use when working with file sources plugins or configuration.

### Update Lint Requirements

**Update pinned lint dependency versions:**
```bash
make update-lint-requirements
```
Updates pinned lint dependency versions in `lib/galaxy/dependencies/`. Run this when upgrading linting tools or resolving dependency conflicts.

---

## Troubleshooting

### "black would reformat" Error

**Symptom:** CI fails with "would reformat X files"

**Solution:**
```bash
# Run black locally
black .

# Or for changed files only
make diff-format

# Commit the formatting changes
git add -u
git commit -m "Apply black formatting"
```

### Ruff Errors Won't Auto-Fix

**Symptom:** `ruff check --fix` doesn't fix all issues

**Cause:** Some ruff rules require manual fixes (e.g., undefined names, logic errors)

**Solution:**
1. Run `ruff check .` to see remaining issues
2. Read error messages carefully
3. Fix manually based on error codes
4. Common manual fixes:
   - `F821` (undefined name) → Import or define the name
   - `E501` (line too long) → Break line manually
   - `B` rules (bugbear) → Logic changes required

### Flake8 Errors After Passing Ruff

**Symptom:** Ruff passes but flake8 fails in CI

**Cause:** Flake8 has some rules ruff doesn't cover yet

**Solution:**
```bash
# Run flake8 locally
flake8 .

# Fix issues manually
# (Common: line length, docstring issues, specific style rules)
```

### Tox Environment Creation Fails

**Symptom:** `tox -e lint` fails to create environment

**Cause:** Dependency conflicts or outdated tox cache

**Solution:**
```bash
# Recreate tox environment
tox -e lint --recreate

# If still failing, clear tox cache
rm -rf .tox/
tox -e lint
```

### Mypy Errors in Unchanged Code

**Symptom:** Type errors in files you didn't modify

**Cause:** Your changes altered types used elsewhere

**Solution:**
1. Check imports - did you change a function signature?
2. Run mypy on specific file to see full context:
   ```bash
   mypy lib/galaxy/managers/your_file.py
   ```
3. Fix type annotations to match your changes
4. Consider if your change needs broader type updates

### Missing Linting Tools

**Symptom:** Command not found (ruff, black, etc.)

**Cause:** Tools not installed in current environment

**Solution:**
```bash
# Install dev dependencies
pip install -r lib/galaxy/dependencies/dev-requirements.txt

# Or use tox (recommended)
tox -e lint  # Tox creates isolated environments with correct dependencies
```

### Conflicting Formatting Between Tools

**Symptom:** Black and isort disagree on formatting

**Cause:** Misconfigured tool settings

**Solution:**
- Galaxy's config already handles this (`pyproject.toml` sets isort profile to "black")
- If you see conflicts, ensure you're using repo's config:
  ```bash
  # Check config is being read
  black --verbose .
  isort --check-only --verbose .
  ```

### Client Lint Failures

**Symptom:** `make client-lint` fails with many errors

**Solution:**
```bash
# 1. Ensure dependencies installed
make client-node-deps

# 2. Auto-fix formatting
make client-format

# 3. Check remaining issues
make client-lint

# 4. Fix ESLint issues manually
# (Usually: remove unused vars, add types, fix Vue template issues)
```

---

## Additional Resources

**Configuration files:**
- `pyproject.toml` - ruff, black, isort, mypy settings
- `.flake8` - flake8 configuration
- `tox.ini` - tox environment definitions
- `client/.eslintrc.js` - ESLint rules
- `client/prettier.config.js` - Prettier settings

**Makefile targets:**
```bash
# View all linting-related targets
grep -A 2 'format\|lint' Makefile
```

**CI workflow:**
```bash
# See what CI runs
cat .github/workflows/lint.yaml
```

**Tool documentation:**
- Ruff: https://docs.astral.sh/ruff/
- Black: https://black.readthedocs.io/
- mypy: https://mypy.readthedocs.io/
- ESLint: https://eslint.org/docs/
- Prettier: https://prettier.io/docs/

**Galaxy-specific patterns:**
- Config files are the authority: `pyproject.toml` (black, ruff), `.isort.cfg`, `mypy.ini`,
  `.flake8`, `client/prettier.config.js`, `client/.eslintrc.js`
- CI entry points: `tox.ini`, `.ci/flake8_wrapper.sh`, `.ci/flake8_wrapper_docstrings.sh`

---

## Quick Command Reference

**Most common workflows:**

```bash
# Quick check before committing
tox -e format && tox -e lint

# Fix formatting issues
make diff-format  # Incremental (fast)
make format       # Full (thorough)

# Fix auto-fixable lint issues
ruff check --fix .

# Modernize Python syntax
make pyupgrade

# Full CI simulation
tox -e format && tox -e lint && tox -e mypy && tox -e lint_docstring_include_list

# Client-side
make client-lint
make client-format

# Client-side (granular)
make client-eslint         # ESLint only
make client-format-check   # Prettier check only
make client-lint-autofix   # Auto-fix ESLint

# Lint API schema (after endpoint changes)
make lint-api-schema

# Lint config files
make config-lint
make tool-shed-config-lint
make reports-config-lint
```

**Remember:** Run `make diff-format` frequently during development to keep code formatted incrementally!
