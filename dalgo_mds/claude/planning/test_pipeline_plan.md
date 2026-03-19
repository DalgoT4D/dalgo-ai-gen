# Test Pipeline Plan — Dalgo Django Backend

**Date**: 2026-03-16
**Status**: Planning
**Scope**: `/Users/siddhant/Documents/Dalgo/DDP_backend/`

---

## Table of Contents

1. [Current State Analysis](#1-current-state-analysis)
2. [Why We Need This](#2-why-we-need-this)
3. [Target Architecture](#3-target-architecture)
4. [Implementation Plan](#4-implementation-plan)
5. [File-by-File Changes](#5-file-by-file-changes)
6. [CI Integration (basic)](#6-ci-integration)
7. [What Improves After This](#7-what-improves-after-this)
8. [Future Enhancements](#8-future-enhancements)
9. [CI/CD — Current State, Gaps, and Improvements](#9-cicd--current-state-gaps-and-improvements)
10. [Updated Implementation Order (with CI)](#10-updated-implementation-order-with-ci)

---

## 1. Current State Analysis

### What exists today

| Area | Current State | Problem |
|------|--------------|---------|
| **Database** | Single `postgres` DB on localhost:5432 | Tests and dev share the same database |
| **Test DB isolation** | None — pytest-django's auto `test_` prefix is the only guard | A misconfigured test can corrupt dev data |
| **Linting** | Black in pre-commit; flake8/pylint commented out | No enforced lint gate before tests run |
| **isort** | Not configured at all | Import ordering is inconsistent |
| **Test runner** | `pytest` with `pytest-django` installed | No single command to run the full pipeline |
| **Seed data** | 7 JSON fixtures in `seed/` loaded via `loaddata` | No `seed` management command; seed and test data are mixed |
| **Makefile / scripts** | `scripts/` directory exists but is empty (only `.gitkeep`) | Developers must remember multiple commands |
| **`.env.test`** | Does not exist | No explicit test environment config |
| **conftest.py** | No root-level conftest; fixtures duplicated across test files | Seed loading is repeated in individual test files |
| **Pre-commit** | Only Black (v23.3.0); flake8 and pylint are commented out | Incomplete code quality gate |
| **Coverage** | Configured in `pyproject.toml` (omits models, migrations, etc.) | Not enforced as part of any pipeline |

### Current developer workflow (manual)

```bash
# Developer has to remember and run each step:
cd DDP_backend
source .venv/bin/activate
black .                                    # format
pytest                                     # test (hopes DB is clean)
python manage.py loaddata seed/*.json      # seed (into dev DB!)
```

Problems:
- No guarantee tests run on a clean database
- No lint/format check before tests
- Seed data loads into dev DB, not test DB
- No single command — easy to skip steps
- New developers have no idea what to run

---

## 2. Why We Need This

### For developers

- **One command** (`make test`) replaces 5+ manual steps
- **Clean database every time** — no flaky tests from leftover state
- **Catch formatting/lint issues before tests** — faster feedback loop
- **Confidence** — if `make test` passes, the code is ready for PR

### For the team

- **Consistent standards** — every developer runs the same pipeline
- **CI parity** — local pipeline mirrors CI exactly
- **Onboarding** — new developers run `make test` on day one
- **Code quality** — enforced formatting + linting prevents style debates in PRs

### For the codebase

- **Test isolation** — tests never touch dev data
- **Reproducibility** — pipeline is deterministic (drop → create → migrate → test)
- **Seed separation** — dev seed and test seed are different concerns

---

## 3. Target Architecture

### Environment separation

```
.env          → Development database (dalgo on localhost:5432)
.env.test     → Test database (test_dalgo on localhost:5432)
```

### Database layout

```
PostgreSQL (localhost:5432)
├── dalgo          ← Development DB (seeded with full data)
└── test_dalgo     ← Test DB (created/destroyed by pipeline)
```

### Pipeline flow

```
make test
│
├── Step 1: Format & Lint
│   ├── black --check .
│   ├── isort --check-only .
│   └── flake8 ddpui/
│
├── Step 2: Reset Test Database
│   ├── DROP DATABASE IF EXISTS test_dalgo
│   ├── CREATE DATABASE test_dalgo
│   └── python manage.py migrate (against test_dalgo)
│
├── Step 3: Run Tests
│   └── pytest --tb=short -q (against test_dalgo)
│
├── Step 4: Seed Development Database
│   └── python manage.py seed (against dalgo, only if needed)
│
└── Step 5: Success message
```

### Folder structure (new/modified files)

```
DDP_backend/
├── Makefile                          ← NEW: Pipeline entry point
├── scripts/
│   └── test.sh                       ← NEW: Shell script pipeline
├── .env                              ← MODIFIED: DBNAME=dalgo (not postgres)
├── .env.test                         ← NEW: Test environment config
├── conftest.py                       ← NEW: Root-level shared fixtures
├── setup.cfg                         ← NEW: flake8 + isort config
├── ddpui/
│   ├── settings.py                   ← MODIFIED: Conditional test DB config
│   └── management/
│       └── commands/
│           └── seed.py               ← NEW: Seed management command
└── pyproject.toml                    ← MODIFIED: isort config added
```

---

## 4. Implementation Plan

### Step 1: Environment files

#### `.env` (modify existing)

Change database name from `postgres` to `dalgo`:

```env
# Database — Development
DBNAME=dalgo
DBHOST=localhost
DBPORT=5432
DBUSER=postgres
DBPASSWORD=postgres
```

**Why**: Using `postgres` as the dev DB name is risky — it's the default superuser database. A named database (`dalgo`) is safer and clearer.

#### `.env.test` (new file)

```env
# Database — Test (used by test pipeline only)
DBNAME=test_dalgo
DBHOST=localhost
DBPORT=5432
DBUSER=postgres
DBPASSWORD=postgres

# Disable external services in tests
AIRBYTE_SERVER_HOST=
PREFECT_PROXY_API_URL=
USE_AWS_SECRETS_MANAGER=
DEV_SECRETS_DIR=/tmp/dalgo-test-secrets

# Django
DEBUG=False
DJANGOSECRET=test-secret-key-for-testing-only
```

**Why**: Explicit test config means tests never accidentally connect to dev services.

---

### Step 2: Django settings update

#### `ddpui/settings.py` — Add test database awareness

The key insight: **pytest-django already handles test DB creation** when configured properly. We just need to ensure:
1. The `DATABASES` setting uses our `.env` values
2. pytest-django knows to create `test_dalgo` (it auto-prefixes `test_` to `DBNAME`)

No changes needed to `DATABASES` dict itself — it already reads from env vars. The `scripts/test.sh` will set the environment.

However, we add a small guard:

```python
# At the bottom of settings.py
import sys

TESTING = "pytest" in sys.modules or "test" in sys.argv

if TESTING:
    # Disable Celery in tests
    CELERY_TASK_ALWAYS_EAGER = True
    CELERY_TASK_EAGER_PROPAGATES = True
    # Use faster password hasher in tests
    PASSWORD_HASHERS = [
        "django.contrib.auth.hashers.MD5PasswordHasher",
    ]
```

**Why**:
- `CELERY_TASK_ALWAYS_EAGER` runs Celery tasks synchronously in tests (no Redis needed)
- `MD5PasswordHasher` speeds up user creation in tests by 10-50x
- The `TESTING` flag is useful for conditional behavior elsewhere

---

### Step 3: isort + flake8 configuration

#### `pyproject.toml` — Add isort config (append to existing)

```toml
[tool.isort]
profile = "black"
line_length = 100
known_django = ["django"]
known_first_party = ["ddpui"]
sections = ["FUTURE", "STDLIB", "THIRDPARTY", "DJANGO", "FIRSTPARTY", "LOCALFOLDER"]
skip_glob = ["*/migrations/*"]
```

**Why**: `profile = "black"` ensures isort and black don't fight each other. Setting `known_django` and `known_first_party` groups imports logically.

#### `setup.cfg` — flake8 config (new file)

```ini
[flake8]
max-line-length = 100
extend-ignore = E203, W503
exclude =
    */migrations/*,
    .venv,
    __pycache__,
    ddpui/dbt_automation/assets/*,
    ddpui/dbt_automation/seeds/*
per-file-ignores =
    __init__.py: F401
```

**Why**: flake8 doesn't support `pyproject.toml` natively. `E203` and `W503` conflict with black's formatting. `F401` in `__init__.py` allows re-exports.

---

### Step 4: Root conftest.py

#### `DDP_backend/conftest.py` (new file)

```python
"""
Root conftest.py — shared fixtures for all tests.

This file is auto-discovered by pytest. Fixtures defined here are
available to every test without importing.
"""

import pytest
from django.core.management import call_command


@pytest.fixture(scope="session")
def django_db_setup(django_test_runner, django_db_blocker):
    """
    Override the default django_db_setup to load seed data once
    per test session into the test database.

    pytest-django creates test_<DBNAME> automatically.
    This fixture runs migrations + loads seed data into it.
    """
    with django_db_blocker.unblock():
        call_command("loaddata", "001_roles.json")
        call_command("loaddata", "002_permissions.json")
        call_command("loaddata", "003_role_permissions.json")


@pytest.fixture
def seed_db(django_db_setup, django_db_blocker):
    """
    Convenience fixture for tests that need seed data.
    Since django_db_setup already loads seeds, this just
    depends on it to ensure ordering.
    """
    pass
```

**Why**:
- Centralizes seed loading — individual test files no longer need their own `seed_db` fixture
- `scope="session"` means seeds load once, not per-test (fast)
- Overriding `django_db_setup` is the official pytest-django way to customize DB setup

---

### Step 5: Seed management command

#### `ddpui/management/commands/seed.py` (new file)

```python
"""
Management command to seed the database with initial data.

Usage:
    python manage.py seed              # Load all seed data
    python manage.py seed --roles      # Load only roles
    python manage.py seed --flush      # Flush DB first, then seed

This is for DEVELOPMENT databases only.
The test pipeline handles test DB seeding via conftest.py.
"""

from django.core.management.base import BaseCommand, CommandError
from django.core.management import call_command
from django.conf import settings


class Command(BaseCommand):
    help = "Seed the database with initial data (roles, permissions, prompts, tasks)"

    SEED_FILES_ORDERED = [
        "001_roles.json",
        "002_permissions.json",
        "003_role_permissions.json",
        "tasks.json",
        "user_prompts.json",
        "assistant_prompts.json",
    ]

    def add_arguments(self, parser):
        parser.add_argument(
            "--flush",
            action="store_true",
            help="Flush the database before seeding (DESTRUCTIVE)",
        )
        parser.add_argument(
            "--roles",
            action="store_true",
            help="Only seed roles and permissions",
        )

    def handle(self, *args, **options):
        db_name = settings.DATABASES["default"]["NAME"]
        self.stdout.write(f"Seeding database: {db_name}")

        if options["flush"]:
            self.stdout.write(self.style.WARNING("Flushing database..."))
            call_command("flush", "--no-input")

        if options["roles"]:
            files = self.SEED_FILES_ORDERED[:3]  # roles + permissions only
        else:
            files = self.SEED_FILES_ORDERED

        for seed_file in files:
            try:
                call_command("loaddata", seed_file)
                self.stdout.write(self.style.SUCCESS(f"  Loaded {seed_file}"))
            except Exception as e:
                raise CommandError(f"Failed to load {seed_file}: {e}")

        self.stdout.write(self.style.SUCCESS("Seeding complete."))
```

**Why**:
- Single command replaces multiple `loaddata` calls
- `--flush` option for clean reseeds
- `--roles` option for minimal seeding (useful in CI)
- Order matters — roles before permissions before role-permissions
- Prints what it's doing — developer-friendly

---

### Step 6: Test shell script

#### `scripts/test.sh` (new file)

```bash
#!/usr/bin/env bash
#
# Full test pipeline for Dalgo Django backend.
# Usage: ./scripts/test.sh [--skip-lint] [--skip-reset] [-- pytest args]
#
# Examples:
#   ./scripts/test.sh                           # Full pipeline
#   ./scripts/test.sh --skip-lint               # Skip formatting/lint checks
#   ./scripts/test.sh --skip-reset              # Skip DB reset (faster reruns)
#   ./scripts/test.sh -- -k "test_auth"         # Pass args to pytest
#   ./scripts/test.sh -- --cov=ddpui            # With coverage

set -euo pipefail

# Colors
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Navigate to project root
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
PROJECT_ROOT="$(dirname "$SCRIPT_DIR")"
cd "$PROJECT_ROOT"

# Parse arguments
SKIP_LINT=false
SKIP_RESET=false
PYTEST_ARGS=""

while [[ $# -gt 0 ]]; do
    case $1 in
        --skip-lint)
            SKIP_LINT=true
            shift
            ;;
        --skip-reset)
            SKIP_RESET=true
            shift
            ;;
        --)
            shift
            PYTEST_ARGS="$*"
            break
            ;;
        *)
            PYTEST_ARGS="$*"
            break
            ;;
    esac
done

# Load test environment
if [ -f .env.test ]; then
    set -a
    source .env.test
    set +a
fi

echo -e "${BLUE}================================================${NC}"
echo -e "${BLUE}  Dalgo Backend — Test Pipeline${NC}"
echo -e "${BLUE}================================================${NC}"
echo ""

# ──────────────────────────────────────────────
# Step 1: Format & Lint
# ──────────────────────────────────────────────
if [ "$SKIP_LINT" = false ]; then
    echo -e "${YELLOW}[1/4] Checking code formatting & lint...${NC}"

    echo "  Running black --check..."
    if ! black --check . 2>&1 | tail -1; then
        echo -e "${RED}  Black found formatting issues. Run 'black .' to fix.${NC}"
        exit 1
    fi
    echo -e "${GREEN}  black: OK${NC}"

    echo "  Running isort --check-only..."
    if ! isort --check-only . 2>&1 | tail -1; then
        echo -e "${RED}  isort found import ordering issues. Run 'isort .' to fix.${NC}"
        exit 1
    fi
    echo -e "${GREEN}  isort: OK${NC}"

    echo "  Running flake8..."
    if ! flake8 ddpui/ --statistics 2>&1 | tail -5; then
        echo -e "${RED}  flake8 found lint errors. Fix them before continuing.${NC}"
        exit 1
    fi
    echo -e "${GREEN}  flake8: OK${NC}"

    echo ""
else
    echo -e "${YELLOW}[1/4] Skipping lint (--skip-lint)${NC}"
    echo ""
fi

# ──────────────────────────────────────────────
# Step 2: Reset Test Database
# ──────────────────────────────────────────────
TEST_DB="${DBNAME:-test_dalgo}"

if [ "$SKIP_RESET" = false ]; then
    echo -e "${YELLOW}[2/4] Resetting test database '${TEST_DB}'...${NC}"

    DBHOST="${DBHOST:-localhost}"
    DBPORT="${DBPORT:-5432}"
    DBUSER="${DBUSER:-postgres}"

    # Drop and recreate
    echo "  Dropping ${TEST_DB} (if exists)..."
    PGPASSWORD="${DBPASSWORD:-postgres}" dropdb \
        -h "$DBHOST" -p "$DBPORT" -U "$DBUSER" \
        --if-exists "$TEST_DB" 2>/dev/null || true

    echo "  Creating ${TEST_DB}..."
    PGPASSWORD="${DBPASSWORD:-postgres}" createdb \
        -h "$DBHOST" -p "$DBPORT" -U "$DBUSER" \
        "$TEST_DB"

    echo "  Running migrations..."
    python manage.py migrate --verbosity=0

    echo "  Loading seed data..."
    python manage.py seed --roles

    echo -e "${GREEN}  Database '${TEST_DB}' is ready.${NC}"
    echo ""
else
    echo -e "${YELLOW}[2/4] Skipping DB reset (--skip-reset)${NC}"
    echo ""
fi

# ──────────────────────────────────────────────
# Step 3: Run Tests
# ──────────────────────────────────────────────
echo -e "${YELLOW}[3/4] Running tests...${NC}"
echo ""

# pytest-django will use DBNAME from env (test_dalgo) and prefix test_ → test_test_dalgo?
# No — we set DBNAME=test_dalgo in .env.test. pytest-django creates test_test_dalgo.
# ALTERNATIVE: We already reset the DB manually. Tell pytest-django to use it directly.
# Use --no-migrations since we already migrated.

if [ -n "$PYTEST_ARGS" ]; then
    pytest $PYTEST_ARGS
else
    pytest --tb=short -q
fi

echo ""

# ──────────────────────────────────────────────
# Step 4: Summary
# ──────────────────────────────────────────────
echo -e "${GREEN}================================================${NC}"
echo -e "${GREEN}  All checks passed!${NC}"
echo -e "${GREEN}================================================${NC}"
echo ""
echo -e "  Format:  ${GREEN}OK${NC}"
echo -e "  Lint:    ${GREEN}OK${NC}"
echo -e "  Tests:   ${GREEN}OK${NC}"
echo ""
echo -e "  Your code is ready for a pull request."
echo ""
```

**Why**:
- `set -euo pipefail` — fail fast on any error
- `--skip-lint` and `--skip-reset` flags for faster iteration
- `--` separator lets you pass arbitrary args to pytest
- Color output for readability
- Loads `.env.test` to isolate from dev environment

---

### Step 7: Makefile

#### `Makefile` (new file at DDP_backend root)

```makefile
.PHONY: test test-quick lint format seed migrate reset-db help

# Default: full pipeline
test: ## Run the full test pipeline (lint + reset DB + tests)
	@./scripts/test.sh

# Quick: skip lint and DB reset (for fast iteration)
test-quick: ## Run tests only (no lint, no DB reset)
	@./scripts/test.sh --skip-lint --skip-reset

# Individual steps
lint: ## Run all linters (black, isort, flake8)
	@black --check .
	@isort --check-only .
	@flake8 ddpui/

format: ## Auto-format code (black + isort)
	@black .
	@isort .

seed: ## Seed the development database
	@python manage.py seed

seed-roles: ## Seed only roles and permissions
	@python manage.py seed --roles

migrate: ## Run Django migrations
	@python manage.py migrate

reset-db: ## Drop and recreate the test database
	@./scripts/test.sh --skip-lint -- --co -q 2>/dev/null || true

# Coverage
test-cov: ## Run tests with coverage report
	@./scripts/test.sh --skip-lint -- --cov=ddpui --cov-report=term-missing

# Help
help: ## Show this help message
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | \
		awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-15s\033[0m %s\n", $$1, $$2}'
```

**Why**:
- `make test` — the one command every developer needs
- `make test-quick` — fast iteration during development
- `make format` — auto-fix formatting (don't just check)
- `make help` — self-documenting

---

### Step 8: pytest configuration update

#### `pyproject.toml` — Update pytest section

```toml
[tool.pytest.ini_options]
pythonpath = ["."]
DJANGO_SETTINGS_MODULE = "ddpui.settings"
testpaths = ["ddpui/tests"]
addopts = [
    "--strict-markers",
    "--tb=short",
    "-q",
]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks integration tests requiring external services",
]
```

**Why**:
- `--strict-markers` catches typos in `@pytest.mark.xxx`
- `--tb=short` keeps error output readable
- Custom markers let you skip slow/integration tests during quick runs

---

## 5. File-by-File Changes

### New files

| File | Purpose |
|------|---------|
| `DDP_backend/.env.test` | Test environment variables |
| `DDP_backend/conftest.py` | Root-level shared test fixtures |
| `DDP_backend/setup.cfg` | flake8 configuration |
| `DDP_backend/Makefile` | Pipeline entry point |
| `DDP_backend/scripts/test.sh` | Full test pipeline script |
| `DDP_backend/ddpui/management/commands/seed.py` | Seed management command |

### Modified files

| File | Change |
|------|--------|
| `DDP_backend/.env` | `DBNAME=postgres` → `DBNAME=dalgo` |
| `DDP_backend/pyproject.toml` | Add `[tool.isort]` section; update `[tool.pytest.ini_options]` |
| `DDP_backend/ddpui/settings.py` | Add `TESTING` flag, `CELERY_TASK_ALWAYS_EAGER`, `MD5PasswordHasher` |

### NOT modified (intentionally)

| File | Reason |
|------|--------|
| Existing test files | They already use `@pytest.mark.django_db` and fixtures correctly |
| `docker-compose.dev.yml` | Docker setup is separate from local dev pipeline |
| `.pre-commit-config.yaml` | Pre-commit stays as-is; the pipeline handles lint separately |

---

## 6. CI Integration

The same `scripts/test.sh` works in CI. Example GitHub Actions workflow:

```yaml
# .github/workflows/test.yml
name: Test Pipeline

on:
  pull_request:
    branches: [main, develop]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_dalgo
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.10"

      - name: Install UV
        run: curl -LsSf https://astral.sh/uv/install.sh | sh

      - name: Install dependencies
        working-directory: DDP_backend
        run: uv sync

      - name: Run test pipeline
        working-directory: DDP_backend
        env:
          DBNAME: test_dalgo
          DBHOST: localhost
          DBPORT: 5432
          DBUSER: postgres
          DBPASSWORD: postgres
          REDIS_HOST: localhost
          REDIS_PORT: 6379
          DJANGOSECRET: ci-secret-key
          JWT_SECRET_KEY: ci-jwt-secret
          DEBUG: "False"
        run: |
          source .venv/bin/activate
          ./scripts/test.sh --skip-reset
          # skip-reset because the DB is already clean (fresh CI container)
```

**Why CI uses `--skip-reset`**: CI starts with a fresh Postgres container every time — no need to drop/recreate. The pipeline still runs migrations and loads seeds via `conftest.py`.

---

## 7. What Improves After This

### Before vs After

| Aspect | Before | After |
|--------|--------|-------|
| **Dev command** | Remember 5 commands | `make test` |
| **DB isolation** | Dev and test share `postgres` | Dev = `dalgo`, Test = `test_dalgo` |
| **Lint enforcement** | Only black (pre-commit) | black + isort + flake8 (pipeline) |
| **Import ordering** | Not configured | isort with black-compatible profile |
| **Test DB state** | Unknown (leftover data) | Clean every run (drop → create → migrate) |
| **Seed data** | `loaddata seed/*.json` manually | `python manage.py seed` or `make seed` |
| **CI parity** | Different from local | Same `scripts/test.sh` |
| **New developer setup** | Trial and error | `make help` shows everything |
| **Fast iteration** | Full pipeline every time | `make test-quick` skips lint + DB reset |
| **Test speed** | bcrypt password hashing | MD5 in tests (10-50x faster) |
| **Celery in tests** | Needs Redis running | `CELERY_TASK_ALWAYS_EAGER` (synchronous) |

### Developer experience

```bash
# Full check before PR
make test

# Quick iteration during development
make test-quick

# Fix formatting
make format

# Seed dev DB after fresh setup
make seed

# Run specific tests fast
make test-quick -- -k "test_auth"
```

---

## 8. Future Enhancements

After the base pipeline is working, consider these improvements:

### Short-term
- **Coverage threshold**: Add `--cov-fail-under=80` to pytest to enforce minimum coverage
- **Type checking**: Add `mypy ddpui/` as a pipeline step
- **Parallel tests**: Add `pytest-xdist` for `pytest -n auto` (parallel test execution)
- **Re-enable pre-commit hooks**: Uncomment flake8 and pylint in `.pre-commit-config.yaml`

### Medium-term
- **Test factories**: Replace manual `User.objects.create_user()` with `factory_boy` or `model_bakery` for cleaner test data
- **Fixtures file**: Create a `ddpui/tests/factories.py` with reusable factories
- **Docker test runner**: `make test-docker` that runs the full pipeline in a container (100% reproducible)
- **Database snapshots**: Use `pytest-django`'s `--reuse-db` with migration tracking for faster runs

### Long-term
- **Mutation testing**: Add `mutmut` to verify test quality (not just coverage)
- **Performance benchmarks**: Add `pytest-benchmark` for critical paths
- **Contract tests**: API schema validation against OpenAPI spec
- **Load testing**: Locust scripts for API endpoint performance

---

## Implementation Order

1. **Create `.env.test`** and modify `.env` (DBNAME=dalgo)
2. **Create `setup.cfg`** (flake8 config)
3. **Add isort config** to `pyproject.toml`
4. **Update `ddpui/settings.py`** (TESTING flag, test optimizations)
5. **Create `conftest.py`** at project root
6. **Create `ddpui/management/commands/seed.py`**
7. **Create `scripts/test.sh`** and `chmod +x`
8. **Create `Makefile`**
9. **Update `pyproject.toml`** pytest section
10. **Test the pipeline**: `make test`
11. **Create dev database**: `createdb dalgo && make migrate && make seed`

---

## Important: pytest-django DB Naming

pytest-django automatically prefixes `test_` to the `DATABASES["default"]["NAME"]`. This means:

- If `.env.test` sets `DBNAME=test_dalgo`, pytest-django creates `test_test_dalgo`
- **Solution**: In `.env.test`, set `DBNAME=dalgo` (same as dev). pytest-django creates `test_dalgo`.
- The `scripts/test.sh` manually resets `test_dalgo`, and pytest-django's auto-creation uses the same name.

Alternatively, override the test DB name in settings:

```python
# settings.py
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.postgresql",
        "NAME": os.getenv("DBNAME"),
        "HOST": os.getenv("DBHOST"),
        "USER": os.getenv("DBUSER"),
        "PASSWORD": os.getenv("DBPASSWORD"),
        "PORT": os.getenv("DBPORT"),
        "TEST": {
            "NAME": "test_dalgo",  # Explicit test DB name
        },
    }
}
```

This approach is cleaner — no matter what `DBNAME` is, the test DB is always `test_dalgo`.

**Recommended approach**: Use the `TEST.NAME` override in settings.py. This way:
- `.env` has `DBNAME=dalgo` (dev)
- `.env.test` is not needed for DB config — it's only for disabling external services
- pytest-django always uses `test_dalgo` regardless of environment
- `scripts/test.sh` resets `test_dalgo` explicitly

---

## 9. CI/CD — Current State, Gaps, and Improvements

### 9.1 Current CI/CD Architecture

The Dalgo project uses GitHub Actions across multiple repos. Here's what exists today:

```
GitHub Actions Workflows
│
├── DDP_backend/.github/workflows/
│   ├── dalgo-ci.yml              ← CI: lint + migrate + pytest + coverage
│   ├── dalgo-cd.yml              ← CD: SSH deploy to EC2
│   ├── dalgo-docker-release.yml  ← Docker: build + push on release
│   └── docs.yml                  ← Docs: deploy to GitHub Pages
│
├── webapp_v2/.github/workflows/
│   ├── dalgo-ci.yml              ← CI: prettier + eslint + jest + coverage
│   └── dalgo-cd.yml              ← CD: SSH deploy to EC2
│
├── webapp/.github/workflows/     (legacy)
│   ├── dalgo-cd.yml              ← CD: SSH deploy to EC2
│   └── dalgo-docker-release.yml  ← Docker build
│
└── prefect-proxy/.github/workflows/
    ├── dalgo-ci.yml              ← CI: pre-commit + pytest + coverage
    ├── dalgo-cd.yml              ← CD: SSH deploy to EC2
    └── docker.yml                ← Docker build
```

### 9.2 Backend CI — What It Does Now (`dalgo-ci.yml`)

```
Trigger: push to main, PR against main

Job: checks
├── Services: PostgreSQL 15, Redis 6
├── Python: 3.10 (matrix, single version)
│
├── Step 1: Checkout
├── Step 2: Setup Python + UV
├── Step 3: Install dependencies (uv sync)
├── Step 4: Pre-commit (black only — flake8/pylint commented out)
├── Step 5: Start Redis
├── Step 6: Create logs directory
├── Step 7: Apply migrations (against CI PostgreSQL)
├── Step 8: Run pytest + coverage (ignores integration_tests, --durations=20)
├── Step 9: Upload coverage XML to Codecov
└── Step 10: Check coverage >= 70%
```

**Key details from the actual workflow:**
- DB user: `ddp` (not `postgres`) — credentials from secrets `CI_DBNAME`, `CI_DBPASSWORD`
- Uses `uv run coverage run -m pytest` (not bare `pytest`)
- Explicitly ignores `ddpui/tests/integration_tests`
- Coverage threshold: 70% (via `coverage report --fail-under=70`)
- UV caching enabled (`astral-sh/setup-uv@v5` with `enable-cache: true`)

### 9.3 Backend CD — What It Does Now (`dalgo-cd.yml`)

```
Trigger: push to main only

Job: deploy
├── SSH into EC2 server (appleboy/ssh-action@v1.2.0)
├── cd /home/ddp/DDP_backend
├── Verify on main branch (fail if not)
├── git pull
├── uv run python manage.py migrate
├── uv run python manage.py loaddata seed/*.json
└── pm2 restart django-celery-worker django-backend-asgi
```

**Problems with current CD:**
- No build step — just `git pull` on the server
- No health check after restart
- No rollback mechanism
- Deploys on every push to main (even if CI fails — no `needs: checks` dependency)
- Loads ALL seed data every deploy (unnecessary after first time)

### 9.4 Docker Release — What It Does Now (`dalgo-docker-release.yml`)

```
Trigger: GitHub release published

Job: push_image_to_registry
├── Multi-arch: linux/amd64, linux/arm64
├── Dockerfile: Docker/Dockerfile.main
├── Tags: tech4dev/dalgo_backend:{version} + :latest
└── Cache: registry-based
```

This is fine as-is.

---

### 9.5 Gap Analysis — What's Missing

| Gap | Severity | Current State | Impact |
|-----|----------|---------------|--------|
| **CD runs without CI passing** | HIGH | `dalgo-cd.yml` triggers on `push` independently | Broken code can deploy to production |
| **No isort in CI** | MEDIUM | Not installed or configured | Inconsistent imports across codebase |
| **No flake8 in CI** | MEDIUM | Commented out in pre-commit | Lint errors ship to production |
| **No integration tests in CI** | LOW | Explicitly `--ignore`d | Integration bugs caught late |
| **No security scanning** | HIGH | No SAST, no dependency audit | Vulnerable dependencies ship silently |
| **No staging environment** | HIGH | main → production directly | No pre-production validation |
| **No E2E tests in CI** | MEDIUM | Playwright available but not wired | UI regressions caught manually |
| **No migration check** | MEDIUM | Migrations run but not validated | Missing migration files can break deploy |
| **No rollback in CD** | HIGH | Deploy is one-way | Failed deploys require manual SSH intervention |
| **CI/CD not linked** | HIGH | CD doesn't depend on CI job | Can deploy with failing tests |
| **No PR checks** | MEDIUM | CI runs on PR but no required status checks | PRs can merge without passing CI |
| **No cache for Python deps** | LOW | UV cache enabled but pip cache not | Acceptable — UV is already fast |
| **Coverage not trending** | LOW | Codecov uploads exist | No PR comments showing coverage delta |

---

### 9.6 Proposed CI Improvements

#### Priority 1: Fix CD depending on CI (Critical)

**Problem**: `dalgo-cd.yml` triggers on `push to main` independently of `dalgo-ci.yml`. A push to main deploys even if tests fail.

**Solution**: Merge CI and CD into one workflow, or make CD depend on CI.

```yaml
# Option A: dalgo-cd.yml — Add dependency on CI
name: Dalgo CD

on:
  workflow_run:
    workflows: ["Dalgo CI"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    steps:
      # ... existing deploy steps
```

```yaml
# Option B: Single workflow with CI → CD stages
name: Dalgo CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  ci:
    runs-on: ubuntu-latest
    # ... existing CI steps

  deploy:
    needs: ci
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    # ... existing CD steps
```

**Recommended**: Option B (single workflow) — simpler, no race conditions.

#### Priority 2: Add isort + flake8 to CI

Replace the pre-commit step with explicit lint steps so failures are clear:

```yaml
      - name: Check formatting (black)
        run: uv run black --check .

      - name: Check import ordering (isort)
        run: uv run isort --check-only .

      - name: Lint (flake8)
        run: uv run flake8 ddpui/ --statistics
```

**Why separate steps instead of pre-commit**: Each tool's failures show as a distinct step in the GitHub Actions UI — easier to diagnose.

#### Priority 3: Add security scanning

```yaml
      - name: Check for vulnerable dependencies
        run: uv run pip-audit

      - name: Check for secrets in code
        uses: trufflesecurity/trufflehog@main
        with:
          extra_args: --only-verified
```

**Minimal setup**: `pip-audit` catches known CVEs in dependencies. `trufflehog` catches accidentally committed secrets.

#### Priority 4: Add migration check

```yaml
      - name: Check for missing migrations
        env:
          # ... DB env vars
        run: |
          uv run python manage.py makemigrations --check --dry-run
```

**Why**: If a developer modifies a model but forgets `makemigrations`, this catches it before merge.

#### Priority 5: Add deploy health check + rollback

```yaml
      - name: Deploy and verify
        uses: appleboy/ssh-action@v1.2.0
        with:
          script: |
            set -e
            cd /home/ddp/DDP_backend

            # Save current commit for rollback
            PREVIOUS_COMMIT=$(git rev-parse HEAD)

            git pull
            /home/ddp/.local/bin/uv run python manage.py migrate
            /home/ddp/.local/bin/uv run python manage.py loaddata seed/*.json
            /home/ddp/.yarn/bin/pm2 restart django-celery-worker django-backend-asgi

            # Health check
            sleep 10
            if ! curl -sf http://localhost:8002/api/docs > /dev/null; then
              echo "Health check failed! Rolling back..."
              git checkout $PREVIOUS_COMMIT
              /home/ddp/.local/bin/uv run python manage.py migrate
              /home/ddp/.yarn/bin/pm2 restart django-celery-worker django-backend-asgi
              exit 1
            fi

            echo "Deploy successful!"
```

---

### 9.7 Proposed New CI Workflow (Complete)

This replaces the existing `dalgo-ci.yml` and `dalgo-cd.yml` with a unified workflow:

```yaml
# .github/workflows/dalgo-ci-cd.yml
name: Dalgo CI/CD

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # Cancel stale PR runs

jobs:
  # ─────────────────────────────────────────────
  # Stage 1: Code Quality (fast, no DB needed)
  # ─────────────────────────────────────────────
  lint:
    name: Lint & Format
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.10
        uses: actions/setup-python@v5
        with:
          python-version: "3.10"

      - name: Install UV
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Install dependencies
        run: uv sync

      - name: Check formatting (black)
        run: uv run black --check .

      - name: Check import ordering (isort)
        run: uv run isort --check-only .

      - name: Lint (flake8)
        run: uv run flake8 ddpui/ --statistics

  # ─────────────────────────────────────────────
  # Stage 2: Security (parallel with lint)
  # ─────────────────────────────────────────────
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.10
        uses: actions/setup-python@v5
        with:
          python-version: "3.10"

      - name: Install UV
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Install dependencies
        run: uv sync

      - name: Audit dependencies for vulnerabilities
        run: uv run pip-audit
        continue-on-error: true  # Advisory initially, make strict later

  # ─────────────────────────────────────────────
  # Stage 3: Tests (needs lint to pass)
  # ─────────────────────────────────────────────
  test:
    name: Test Suite
    needs: lint
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: ddp
          POSTGRES_PASSWORD: ${{ secrets.CI_DBPASSWORD }}
          POSTGRES_DB: ${{ secrets.CI_DBNAME }}
        ports:
          - 5432:5432
        options: >-
          --health-cmd "pg_isready -U ddp"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    strategy:
      matrix:
        python-version: ["3.10"]

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install UV
        uses: astral-sh/setup-uv@v5
        with:
          enable-cache: true

      - name: Install dependencies
        run: uv sync

      - name: Start Redis
        uses: supercharge/redis-github-action@1.8.1
        with:
          redis-version: 6

      - name: Create logs directory
        run: mkdir -p ddpui/logs

      - name: Check for missing migrations
        env:
          DBHOST: localhost
          DBPORT: 5432
          DBNAME: ${{ secrets.CI_DBNAME }}
          DBUSER: ddp
          DBPASSWORD: ${{ secrets.CI_DBPASSWORD }}
          DJANGOSECRET: "ci-secret-key"
        run: uv run python manage.py makemigrations --check --dry-run

      - name: Apply database migrations
        env:
          DBHOST: localhost
          DBPORT: 5432
          DBNAME: ${{ secrets.CI_DBNAME }}
          DBUSER: ddp
          DBPASSWORD: ${{ secrets.CI_DBPASSWORD }}
          DJANGOSECRET: "ci-secret-key"
        run: uv run python manage.py migrate

      - name: Load seed data
        env:
          DBHOST: localhost
          DBPORT: 5432
          DBNAME: ${{ secrets.CI_DBNAME }}
          DBUSER: ddp
          DBPASSWORD: ${{ secrets.CI_DBPASSWORD }}
          DJANGOSECRET: "ci-secret-key"
        run: |
          uv run python manage.py loaddata seed/001_roles.json
          uv run python manage.py loaddata seed/002_permissions.json
          uv run python manage.py loaddata seed/003_role_permissions.json

      - name: Run tests with coverage
        env:
          AIRBYTE_SERVER_HOST: localhost
          AIRBYTE_SERVER_PORT: 8001
          AIRBYTE_SERVER_APIVER: v1
          AIRBYTE_API_TOKEN: ${{ secrets.AIRBYTE_API_TOKEN }}
          DBHOST: localhost
          DBPORT: 5432
          DBNAME: ${{ secrets.CI_DBNAME }}
          DBUSER: ddp
          DBPASSWORD: ${{ secrets.CI_DBPASSWORD }}
          PREFECT_PROXY_API_URL: http://localhost:8080
          DEV_SECRETS_DIR: /tmp/
          CLIENTDBT_ROOT: /tmp
          DJANGOSECRET: "ci-secret-key"
        run: |
          uv run coverage run -m pytest --ignore=ddpui/tests/integration_tests --durations=20
          uv run coverage xml
          uv run coverage report --fail-under=70

      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v5
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true

  # ─────────────────────────────────────────────
  # Stage 4: Deploy (only on main, after tests pass)
  # ─────────────────────────────────────────────
  deploy:
    name: Deploy to Production
    needs: [test, security]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: production  # Requires GitHub environment approval (optional)

    steps:
      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.2.0
        with:
          host: ${{ secrets.SERVERIP }}
          username: ${{ secrets.SSH_USERNAME }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          port: 22
          command_timeout: 500s
          script: |
            set -e
            source ~/.nvm/nvm.sh
            cd /home/ddp/DDP_backend

            current_branch=$(git rev-parse --abbrev-ref HEAD)
            if [ "$current_branch" != "main" ]; then
              echo "Error: Not on main branch. Current: $current_branch"
              exit 1
            fi

            # Save rollback point
            PREVIOUS_COMMIT=$(git rev-parse HEAD)

            git pull
            /home/ddp/.local/bin/uv run python manage.py migrate
            /home/ddp/.local/bin/uv run python manage.py loaddata seed/*.json
            /home/ddp/.yarn/bin/pm2 restart django-celery-worker django-backend-asgi

            # Health check (wait for server to start)
            sleep 15
            if ! curl -sf http://localhost:8002/api/docs > /dev/null 2>&1; then
              echo "Health check FAILED. Rolling back to $PREVIOUS_COMMIT"
              git checkout $PREVIOUS_COMMIT
              /home/ddp/.local/bin/uv run python manage.py migrate
              /home/ddp/.yarn/bin/pm2 restart django-celery-worker django-backend-asgi
              exit 1
            fi

            echo "Deploy successful! Commit: $(git rev-parse --short HEAD)"
```

### 9.8 Visual: Current vs Proposed Pipeline

#### Current (broken flow)

```
                    push to main
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
         dalgo-ci.yml          dalgo-cd.yml
         (tests run)           (deploys immediately)
              │                     │
              ▼                     ▼
         Codecov upload        SSH → git pull → pm2 restart
                               (NO dependency on CI!)
```

#### Proposed (gated flow)

```
         push to main / PR
              │
    ┌─────────┴─────────┐
    ▼                   ▼
  lint              security
  (black,isort,     (pip-audit)
   flake8)              │
    │                   │
    ▼                   │
  test ◄────────────────┘
  (migrate → seed →
   pytest → coverage)
    │
    ▼
  deploy (main only)
  (git pull → migrate →
   pm2 restart →
   health check →
   rollback on fail)
```

### 9.9 CI Changes — File-by-File

| File | Action | Description |
|------|--------|-------------|
| `.github/workflows/dalgo-ci-cd.yml` | **NEW** | Unified CI/CD workflow with lint → test → deploy stages |
| `.github/workflows/dalgo-ci.yml` | **DELETE** | Replaced by unified workflow |
| `.github/workflows/dalgo-cd.yml` | **DELETE** | Replaced by unified workflow |
| `.github/workflows/dalgo-docker-release.yml` | **KEEP** | Works fine as-is (triggers on release, not push) |
| `.github/workflows/docs.yml` | **KEEP** | Independent, works fine |

### 9.10 Required GitHub Repository Settings

After implementing the new workflow, configure these in GitHub repo settings:

1. **Branch protection rules** (Settings → Branches → main):
   - Require status checks to pass before merging: `lint`, `test`
   - Require branches to be up to date before merging
   - Require pull request reviews (optional but recommended)

2. **Environment protection** (Settings → Environments → production):
   - Required reviewers (optional — for manual deploy approval)
   - Deployment branches: `main` only

3. **Secrets** (Settings → Secrets → Actions):
   - Existing secrets work as-is: `CI_DBNAME`, `CI_DBPASSWORD`, `SERVERIP`, etc.
   - No new secrets needed

### 9.11 CI Improvements Roadmap

| Phase | What | When |
|-------|------|------|
| **Phase 1** | Unify CI+CD, add `needs: ci` gate, add isort+flake8 | With test pipeline (this plan) |
| **Phase 2** | Add migration check, health check + rollback in CD | Week after Phase 1 |
| **Phase 3** | Add `pip-audit` security scanning (advisory mode) | Same sprint |
| **Phase 4** | Add Codecov PR comments, coverage delta tracking | Next sprint |
| **Phase 5** | Add staging environment, deploy to staging first | Requires infra work |
| **Phase 6** | Add E2E tests (Playwright) in CI for webapp_v2 | Frontend team |
| **Phase 7** | Make security scanning strict (`continue-on-error: false`) | After fixing existing vulns |
| **Phase 8** | Add Python 3.11/3.12 to test matrix | When ready to upgrade |

---

## 10. Updated Implementation Order (with CI)

1. **Create `.env.test`** and modify `.env` (DBNAME=dalgo)
2. **Create `setup.cfg`** (flake8 config)
3. **Add isort config** to `pyproject.toml`
4. **Update `ddpui/settings.py`** (TESTING flag, test optimizations)
5. **Create `conftest.py`** at project root
6. **Create `ddpui/management/commands/seed.py`**
7. **Create `scripts/test.sh`** and `chmod +x`
8. **Create `Makefile`**
9. **Update `pyproject.toml`** pytest section
10. **Test the pipeline locally**: `make test`
11. **Create dev database**: `createdb dalgo && make migrate && make seed`
12. **Create `.github/workflows/dalgo-ci-cd.yml`** (unified CI/CD)
13. **Delete old workflows**: `dalgo-ci.yml` and `dalgo-cd.yml`
14. **Configure GitHub branch protection** (require `lint` + `test` status checks)
15. **Test CI**: Open a PR, verify lint + test stages run and gate the merge

---

## 11. Flake8 Rules — What We Ignore and Why

### Permanently Ignored (safe, will never be re-enabled)

These conflict with black or are incompatible with Django/pytest patterns:

| Rule | What it flags | Why we ignore it |
|------|--------------|------------------|
| **E203** | Whitespace before `:` | Conflicts with black's slice formatting (`x[1 : 2]`). Every black+flake8 project ignores this. |
| **W503** | Line break before binary operator | Conflicts with black. PEP8 itself now prefers breaks *before* operators. |
| **E402** | Module-level import not at top | Django test files use `pytestmark = pytest.mark.django_db` before imports. Django apps use `load_dotenv()` and `django.setup()` before imports. This is a standard Django pattern, not a code smell. |
| **E501** | Line too long (>100 chars) | Black handles line length for code. Remaining E501s are long strings, URLs, or comments that black can't/shouldn't wrap. Splitting them would hurt readability. |
| **E266** | Too many leading `#` for block comment | Purely cosmetic. `## Section` vs `# Section` is a style choice with no functional impact. |

### Temporarily Ignored (should be fixed and removed from ignore list)

These are real code quality issues. They're ignored now to unblock the pipeline, but each should be cleaned up incrementally:

| Rule | Count | What it flags | Why fix it | Priority |
|------|-------|--------------|------------|----------|
| **E712** | 9 | `== True` / `== False` instead of `is True` / `if not x` | In Python, `x == True` is different from `x is True` and `if x`. Using `==` with booleans is usually a mistake or unclear intent. Django `filter(field=True)` is fine (keyword arg), but `if var == False` in Python code should be `if not var`. | LOW |
| **E722** | 5 | Bare `except:` without exception type | Catches everything including `SystemExit`, `KeyboardInterrupt` — hides real bugs and makes debugging painful. Replace with `except Exception:` at minimum. | HIGH — easy fix, real safety issue |
| **E741** | 2 | Ambiguous variable name (`l`) | `l` looks like `1` in many fonts. Both occurrences are in `ddpui/utils/helpers.py`. Rename to `length`, `lst`, or something descriptive. | LOW — only 2 occurrences |
| **F841** | ~3 (non-test) | Local variable assigned but never used | Dead code. Either the variable should be used, the assignment removed, or replaced with `_` if the call has side effects. Current occurrences: `urls.py` (intentional Sentry trigger, has `noqa`), `createorganduser.py` (queries DB but ignores result), `dashboard_service.py` (builds payload dict that's never sent — WIP code). | MEDIUM |

### Per-File Ignores (permanent for test files)

These are false positives specific to pytest's fixture injection model:

| Rule | Applies to | What it flags | Why it's a false positive |
|------|-----------|--------------|--------------------------|
| **F401** | `__init__.py` | Imported but unused | `__init__.py` files re-export symbols — the import *is* the purpose. |
| **F811** | `ddpui/tests/*` | Redefinition of unused name | Pytest fixture overrides — a test file can redefine a fixture from `conftest.py` to customize it. This is standard pytest. |
| **F841** | `ddpui/tests/*` | Unused variable | Pytest fixtures are injected by parameter name. `def test_foo(orguser):` uses `orguser` as a fixture trigger even if the test body doesn't reference it directly. |
| **F405** | `ddpui/tests/*` | May be undefined from star import | Test helper files use `from helpers import *` for convenience. The names are defined — flake8 just can't statically verify them through `*`. |
| **F403** | `ddpui/tests/*` | Star import used | Same as F405 — test files use star imports from test helper modules. Acceptable in tests, never in production code. |

### Excluded Directories

| Path | Why excluded |
|------|-------------|
| `*/migrations/*` | Auto-generated by Django. Never lint migration files. |
| `.venv` | Virtual environment — third-party code. |
| `__pycache__` | Bytecode cache — not source code. |
| `ddpui/dbt_automation/assets/*` | Auto-generated DBT assets. |
| `ddpui/dbt_automation/seeds/*` | Auto-generated DBT seed files. |
| `ddpui/tests/integration_tests/*` | Integration tests have structural issues (F821 undefined names like `scaffold`, `sync_sources`) because they require external services to resolve. They're excluded from the test pipeline anyway (`--ignore`). |

### Tightening Roadmap

The goal is to shrink the ignore list to only the 5 permanent rules:

| Step | Action | Removes from ignore |
|------|--------|-------------------|
| 1 | Fix 5 bare `except:` in `auth.py`, `superset_service.py`, `aws_client.py` | E722 |
| 2 | Rename 2 `l` variables in `helpers.py` | E741 |
| 3 | Clean up 3 unused assignments in non-test code | F841 (global; keep in test per-file-ignores) |
| 4 | Fix 9 `== True/False` comparisons across test + service files | E712 |

Each step is a small PR. After all 4, `setup.cfg` shrinks to:

```ini
[flake8]
max-line-length = 100
extend-ignore =
    E203,  # Conflicts with black (whitespace before ':')
    W503,  # Conflicts with black (line break before binary operator)
    E402,  # Django/pytest import patterns
    E501,  # Black handles line length; remaining are strings/URLs
    E266   # Cosmetic comment formatting
```

### Best Practices for New Code

Going forward, new code should never introduce these issues:

- **Never** write bare `except:` — always `except Exception:` or a specific exception
- **Never** use single-letter names `l`, `O`, `I` (PEP8 ambiguous identifiers)
- **Never** use `== True` / `== False` — use `if cond:` / `if not cond:`
- **Never** leave unused variables — use `_` for intentional throwaway
- **Never** use `from x import *` outside test files
- **Always** run `make lint` before committing — if it passes, the code is clean
- **Always** run `make format` to auto-fix formatting (black + isort)
- When fixing an ignored rule, **remove it from `setup.cfg`** in the same PR
