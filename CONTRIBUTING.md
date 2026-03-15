# Contributing to fleet optimization

Thank you for taking the time to contribute. This project exists to help fleets reduce operational costs through open, vendor-neutral infrastructure — and every pull request, bug report, and documentation improvement moves that mission forward.

This guide covers everything you need to go from zero to a merged contribution. Please read it fully before opening a PR.

---

## Table of contents

- [Code of conduct](#code-of-conduct)
- [Ways to contribute](#ways-to-contribute)
- [Architecture overview](#architecture-overview)
- [Development setup](#development-setup)
- [Branching and workflow](#branching-and-workflow)
- [Coding standards](#coding-standards)
- [Testing requirements](#testing-requirements)
- [Submitting a pull request](#submitting-a-pull-request)
- [Dependency management](#dependency-management)
- [Security vulnerabilities](#security-vulnerabilities)
- [Commit message convention](#commit-message-convention)
- [Review process and SLAs](#review-process-and-slas)
- [Good first issues](#good-first-issues)
- [Recognition](#recognition)

---

## Code of conduct

This project follows the [Contributor Covenant v2.1](https://www.contributor-covenant.org/version/2/1/code_of_conduct/). By participating, you agree to uphold a respectful, inclusive environment. Report unacceptable behaviour to **conduct@fleetopt.io** — all reports are handled confidentially.

---

## Ways to contribute

You do not need to write code to make a meaningful contribution.

| Type | Examples |
|---|---|
| Bug reports | Incorrect route calculations, stale telemetry data, API errors |
| Feature requests | New ML models, routing engine integrations, dashboard widgets |
| Code | Bug fixes, performance improvements, new modules |
| Tests | Unit tests, integration tests, fixtures — coverage is critically low right now |
| Documentation | README improvements, API docs, inline docstrings, architecture diagrams |
| Dependency hygiene | CVE patches, version upgrades, conflict resolution |
| DevOps | CI/CD improvements, Docker/Kubernetes configs, observability |

If you are unsure whether an idea fits the project, open a [Discussion](https://github.com/fleetopt/fleet-optimization/discussions) before investing time in implementation.

---

## Architecture overview

Understanding the system before contributing avoids misaligned PRs. The platform is composed of five layers:

```
┌─────────────────────────────────────────────────────────┐
│  API layer          FastAPI + Uvicorn                    │
│  Orchestration      Prefect 3.x flows                    │
│  Stream processing  kafka-python-ng → Faust              │
│  ML pipeline        scikit-learn · XGBoost · LightGBM    │
│                     CatBoost · MLflow (experiment tracking)│
│  Data stores        PostgreSQL + TimescaleDB (telemetry)  │
│                     MinIO (model artefacts / object store)│
│  Geospatial         osmnx · geopy · geopandas             │
│  Infrastructure     Docker · Kubernetes · Dask            │
└─────────────────────────────────────────────────────────┘
```

Key directories:

```
fleet-optimization/
├── api/               FastAPI routers and request/response models
├── core/              Domain logic: vehicles, routes, maintenance
├── ml/                Training pipelines, feature engineering, model registry
├── streams/           Kafka consumers and Faust stream processors
├── flows/             Prefect flow definitions
├── geo/               Geospatial utilities and routing helpers
├── db/                SQLAlchemy models, migrations (Alembic)
├── infra/             Docker, Kubernetes manifests
├── tests/             All tests — mirror source structure
└── docs/              Architecture decision records (ADRs), API specs
```

If a PR touches module boundaries or introduces a new service, include or update the relevant ADR in `docs/adr/`.

---

## Development setup

### Prerequisites

- Python 3.11+
- Docker and Docker Compose
- Git

### 1. Fork and clone

```bash
git clone https://github.com/<your-username>/fleet-optimization.git
cd fleet-optimization
```

### 2. Create a virtual environment

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
```

### 3. Install dependencies

We use split requirements files. Install the set relevant to your work:

```bash
pip install -r requirements/base.txt          # always required
pip install -r requirements/dev.txt           # linting, testing, type checking
pip install -r requirements/ml.txt            # ML stack (optional for non-ML work)
pip install -r requirements/infra.txt         # Kubernetes, Docker SDK (optional)
```

> **Note:** `requirements.txt` in the root is the legacy flat file. It will be replaced by the split structure above — contributions to this migration are welcome ([#good-first-issues](#good-first-issues)).

### 4. Set up environment variables

```bash
cp .env.example .env
# Edit .env — never commit secrets or real credentials
```

All secrets must be supplied via environment variables. Hard-coding credentials — even in tests — will cause an automatic PR rejection.

### 5. Start local services

```bash
docker compose up -d postgres kafka minio
```

### 6. Run database migrations

```bash
alembic upgrade head
```

### 7. Verify setup

```bash
pytest tests/smoke/ -v
```

All smoke tests must pass before you begin development.

---

## Branching and workflow

```
main              ← production-ready, protected
develop           ← integration branch for all feature work
feature/<slug>    ← your working branch, branched from develop
fix/<slug>        ← bug fix, branched from develop
hotfix/<slug>     ← critical production fix, branched from main
```

**Always branch from `develop`**, not from `main`, unless you are patching a critical production issue.

```bash
git checkout develop
git pull origin develop
git checkout -b feature/async-telemetry-ingestion
```

Keep branches short-lived. Large features should be broken into incremental PRs that each leave the codebase in a working state.

---

## Coding standards

### Style

We use `ruff` for linting and formatting. Configuration is in `pyproject.toml`.

```bash
ruff check .          # lint
ruff format .         # format
```

Your editor should run `ruff format` on save. CI will reject PRs that fail either check.

### Type annotations

All new functions and methods must include type annotations. We use `mypy` in strict mode for core modules.

```bash
mypy core/ api/ ml/
```

**Good:**
```python
def calculate_eta(
    origin: tuple[float, float],
    destination: tuple[float, float],
    avg_speed_kmh: float,
) -> datetime:
    ...
```

**Not accepted:**
```python
def calculate_eta(origin, destination, avg_speed):
    ...
```

### Docstrings

Public functions, classes, and modules must have Google-style docstrings.

```python
def score_vehicle_health(telemetry: VehicleTelemetry) -> HealthScore:
    """Compute a health score for a vehicle from its latest telemetry.

    Args:
        telemetry: Telemetry snapshot including odometer, engine temp,
            and fault codes recorded at the point of observation.

    Returns:
        A HealthScore with a 0–100 numeric score and a list of
        contributing risk factors ranked by severity.

    Raises:
        InvalidTelemetryError: If required fields are missing or
            out of acceptable range.
    """
```

### No magic numbers

Route thresholds, maintenance intervals, model hyperparameters, and API timeouts must not be hard-coded as literals. Place them in `core/config.py` or the relevant module's constants file.

```python
# Bad
if odometer_km > 15000:
    schedule_service(vehicle)

# Good
from core.config import MAINTENANCE_INTERVAL_KM
if odometer_km > MAINTENANCE_INTERVAL_KM:
    schedule_service(vehicle)
```

### Async-first for I/O

All database queries, external API calls, and Kafka interactions in the `api/` and `streams/` layers must use `async`/`await`. Synchronous blocking calls in request handlers will not be merged.

```python
# Bad — blocks the event loop
@router.get("/vehicles/{vehicle_id}/status")
def get_status(vehicle_id: UUID) -> VehicleStatus:
    return db.query(Vehicle).filter_by(id=vehicle_id).one()

# Good
@router.get("/vehicles/{vehicle_id}/status")
async def get_status(vehicle_id: UUID, db: AsyncSession = Depends(get_db)) -> VehicleStatus:
    result = await db.execute(select(Vehicle).where(Vehicle.id == vehicle_id))
    return result.scalar_one()
```

### Structured logging

Use the project logger — never `print()`. Include context fields on every log call that touches a vehicle, route, or job.

```python
from core.logging import get_logger

logger = get_logger(__name__)

logger.info("route_calculated", vehicle_id=str(vehicle.id), stops=len(stops), duration_ms=elapsed)
logger.error("maintenance_scoring_failed", vehicle_id=str(vehicle.id), exc_info=True)
```

---

## Testing requirements

Test coverage is currently around 12% — raising it is one of the project's highest priorities. Every PR is expected to include tests.

| Contribution type | Minimum requirement |
|---|---|
| Bug fix | Regression test that fails before the fix and passes after |
| New feature | Unit tests covering the happy path and at least two error cases |
| Refactor | No net reduction in coverage; all existing tests must pass |
| ML model change | Performance benchmark comparison vs. baseline in PR description |

### Running tests

```bash
pytest                            # full suite
pytest tests/unit/                # unit tests only
pytest tests/integration/         # requires Docker services running
pytest --cov=. --cov-report=term-missing   # with coverage report
```

### Test structure

```
tests/
├── unit/                  # pure logic, no I/O — fast
│   ├── test_routing.py
│   ├── test_maintenance.py
│   └── ml/
│       └── test_feature_engineering.py
├── integration/           # hits real services via Docker Compose
│   ├── test_api_vehicles.py
│   └── test_kafka_consumers.py
└── smoke/                 # minimal sanity checks, used in CI gate
```

Test files must mirror the module they test: `core/routing.py` → `tests/unit/test_routing.py`.

### Fixtures

Shared fixtures live in `tests/conftest.py`. Add fixtures there — do not duplicate factory logic across test files.

---

## Submitting a pull request

### Before you open the PR

- [ ] Tests pass locally: `pytest`
- [ ] Linting passes: `ruff check . && ruff format --check .`
- [ ] Type checks pass: `mypy core/ api/`
- [ ] No secrets or credentials in any file
- [ ] `.env.example` updated if you added a new environment variable
- [ ] Docstrings added for all public interfaces you introduced
- [ ] `CHANGELOG.md` updated under `[Unreleased]`

### PR title

Follow the commit convention (see below). The PR title becomes the squash commit message.

```
feat(ml): add gradient-boosted ETA model for highway routes
fix(api): validate vehicle ID before telemetry lookup
```

### PR description

Use the template provided when you open the PR. Required fields:

- **What does this PR do?** — plain English, one paragraph
- **Why is this the right approach?** — especially for architectural decisions
- **How was it tested?** — reference specific test files or commands
- **Breaking changes** — list any API or schema changes that affect callers
- **Related issues** — use `Closes #<number>` to auto-close

### Draft PRs

Open a draft PR early if you want feedback on an approach before the implementation is complete. Mark it ready for review only when all checklist items above are satisfied.

---

## Dependency management

Dependency changes require extra care given the existing conflicts in `requirements.txt`.

### Rules

1. **Never add a new dependency without opening a discussion first** if the package adds more than ~5 MB to the install footprint or introduces a competing implementation of something already in the stack.
2. **Always pin to a minimum version** (`package>=1.2.0`), not an exact version, unless a regression has been confirmed at a higher version. Document the reason in a comment.
3. **Run `pip-audit` before and after** any dependency change and include the output diff in your PR description.
4. **Do not add `kafka==1.3.5` back.** This package is abandoned. Use `kafka-python-ng` for all Kafka interactions.
5. **Security CVEs are treated as critical bugs.** A PR that upgrades a CVE-affected package (`requests`, `boto3`, etc.) will be reviewed and merged within 48 hours.

```bash
pip install pip-audit
pip-audit -r requirements/base.txt
```

### Adding a new dependency

```bash
# Add to the correct split file
echo "newpackage>=1.0.0" >> requirements/base.txt   # or ml.txt, infra.txt

# Verify no conflicts
pip install -r requirements/base.txt -r requirements/dev.txt --dry-run

# Run audit
pip-audit -r requirements/base.txt
```

---

## Security vulnerabilities

**Do not open a public GitHub issue for a security vulnerability.**

Report security issues by emailing **security@fleetopt.io**. Include:

- A description of the vulnerability and affected component
- Steps to reproduce or a proof-of-concept
- Your assessment of impact and severity

We follow a 90-day responsible disclosure window. You will receive an acknowledgement within 48 hours and a status update within 7 days.

---

## Commit message convention

We follow [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/).

```
<type>(<scope>): <short summary>

[optional body — explain the why, not the what]

[optional footer — breaking changes, closes #issue]
```

**Types:**

| Type | Use for |
|---|---|
| `feat` | New user-facing feature |
| `fix` | Bug fix |
| `perf` | Performance improvement |
| `refactor` | Code change with no behaviour change |
| `test` | Adding or updating tests |
| `docs` | Documentation only |
| `deps` | Dependency update |
| `ci` | CI/CD configuration changes |
| `chore` | Tooling, build scripts, config |

**Scopes** (use the affected module): `api`, `ml`, `streams`, `geo`, `db`, `flows`, `infra`, `core`

**Examples:**

```
feat(api): add bulk vehicle registration endpoint

fix(ml): correct feature scaling for cold-start vehicles
Odometer readings on new vehicles were not normalised before
being passed to the XGBoost ETA model, causing >30% prediction
errors on routes under 500 km.
Closes #148

deps: upgrade requests to 2.32.3 to address CVE-2024-35195

refactor(core): extract VehicleFleetManager into service modules
Splits the 850-line god class into FleetRegistry, MaintenanceScheduler,
and TelemetryAggregator. No behaviour changes.
```

Commits that do not follow this format will be flagged during review.

---

## Review process and SLAs

| PR type | First response | Target merge |
|---|---|---|
| Security / CVE fix | 24 hours | 48 hours |
| Bug fix (with tests) | 3 business days | 7 business days |
| Feature | 5 business days | Varies by scope |
| Documentation | 3 business days | 5 business days |
| Dependency update | 3 business days | 5 business days |

### What reviewers look for

- Correctness and edge case handling
- Test quality, not just quantity — tests must actually assert meaningful behaviour
- Async-safety in the API and streams layers
- No new blocking I/O introduced on request paths
- Performance impact on fleet queries at 500+ vehicle scale
- No regression in `pip-audit` results

Reviewers may request changes. Address feedback in new commits — do not force-push to a PR that is under review. Once all conversations are resolved, re-request review.

---

## Good first issues

These are concrete, well-scoped contributions that do not require deep domain knowledge. They are labelled [`good first issue`](https://github.com/fleetopt/fleet-optimization/issues?q=label%3A%22good+first+issue%22) on GitHub.

**Dependency & security (high impact, low risk):**
- Upgrade `requests` from 2.31.0 to 2.32.3+ to resolve CVE-2024-35195 and CVE-2024-47081
- Upgrade `boto3` from 1.28.0 to 1.34+ (18 months of security patches)
- Remove `kafka==1.3.5` and migrate all usages to `kafka-python-ng`
- Split the flat `requirements.txt` into `base`, `ml`, `infra`, and `dev` files

**Testing (critical need):**
- Add unit tests for `geo/` distance and ETA utilities (currently 0% coverage)
- Add regression tests for the maintenance scoring thresholds
- Add smoke tests for the FastAPI health and readiness endpoints

**Code quality:**
- Replace all `print()` calls with structured `logger` calls
- Extract magic numbers in `core/` into `core/config.py` constants
- Add type annotations to any 5 public functions currently missing them

**Documentation:**
- Add Google-style docstrings to the `ml/` feature engineering module
- Document the `.env.example` variables with descriptions and valid ranges
- Write an ADR for the decision to use Prefect over Apache Airflow

Comment on the issue before starting work so we can assign it to you and avoid duplication.

---

## Recognition

Contributors are listed in [`CONTRIBUTORS.md`](./CONTRIBUTORS.md), updated on each release. Significant contributions are called out in release notes. If you contribute a fix for a confirmed CVE, you will be credited in the security advisory.

We are grateful for every contribution, regardless of size.

---

*For questions not covered here, open a [Discussion](https://github.com/fleetopt/fleet-optimization/discussions) or reach out via the project's community channels.*
