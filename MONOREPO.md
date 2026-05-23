The core philosophy of this monorepo is **strict isolation of concerns**. It creates a hard boundary between infrastructure (Docker, global databases), source code (`src/`), and individual deployable units (apps and code locations).

### The Directory Structure

```
my_enterprise_repo/
├── .github/                    # CI/CD pipelines
├── pyproject.toml              # ROOT: Defines the workspace & global linting rules
├── uv.lock                     # (or poetry.lock) Single lockfile for the repo
├── compose.yaml                # Root level compose with includes of the docker stacks
│
├── docker/                     # INFRASTRUCTURE & BUILD
│   ├── stacks/                 # Orchestration (e.g., app-stack.yml, data-stack.yml)
│   └── images/                 # Build Definitions (Dockerfile)
│
├── db/                         # SHARED DATABASES
│   ├── main_postgres/          # Managed via Flyway
│   └── analytics_mongo/        # Managed via Atlas
│
└── src/                        # SOURCE CODE
    ├── libs/                   # Shared local packages
    │   └── data_models/
    │       └── pyproject.toml
    │
    ├── apps/                   # Deployable applications
    │   └── django_api/
    │       ├── pyproject.toml
    │       └── core_app/
    │           └── migrations/ # Local, app-owned database migrations
    │
    ├── dagster/                # ORCHESTRATION
    │   ├── workspace.yaml      # Routes UI to specific locations
    │   ├── de_location/        # Data Engineering Python code
    │   │   └── pyproject.toml
    │   └── ds_location/        # Data Science Python code
    │       └── pyproject.toml
    │
    └── dbt/                    # TRANSFORMATIONS
        ├── finance/            # Standalone dbt project 1
        └── marketing/          # Standalone dbt project 2
```

## 1. The Root Level: Workspace Management

Instead of one massive virtual environment, the monorepo utilizes a **Python Workspace** (supported by modern tools like `uv`, `Poetry`, or `Rye`).

- **The Root `pyproject.toml`:** Does not contain application dependencies (like `pandas` or `django`). Instead, it defines the workspace members (`members = ["src/libs/*", "src/apps/*", "src/dagster/*"]`) and houses global configurations for tooling (Ruff, Pytest, Mypy).
- **The Child `pyproject.toml` files:** Every subfolder inside `libs/`, `apps/`, and `dagster/` contains its own `pyproject.toml` explicitly declaring only the dependencies it needs.
- **Path Dependencies:** If an application needs a shared library, it references the local path. For example, the Django API imports `data_models` via: `data_models = { path = "../../libs/data_models" }`.

## 2. Infrastructure & Builds (`docker/`)

The repository uses the **Centralized Build Directory** pattern to avoid "build context" errors.

- **`docker/images/`:** All Dockerfiles live here. Because a deployable unit (like Dagster) often needs files from multiple places (e.g., its own code, a shared library, and a dbt project), the Dockerfile cannot live inside the specific `src/` folder.
- **The Build Context Rule:** Docker builds and `docker compose` commands are always executed from the **repository root**. This allows the `Dockerfile` to copy files from anywhere in the monorepo.

## 3. The Source Code Boundary (`src/`)

By placing all application logic, libraries, and pipelines inside `src/`, your tooling configuration becomes trivial. You instruct Ruff, Mypy, and Pytest to strictly target the `src/` directory, naturally ignoring Docker scripts, CI/CD configs, and global database schemas.

### Dagster and dbt Integration

- **`src/dagster/workspace.yaml`:** This is Dagster's master config. It is completely ignorant of your APIs or shared libraries. It simply points the Dagster daemon to the individual code locations (`de_location`, `ds_location`).
- **`src/dbt/`:** By keeping dbt projects separate from Dagster code, you allow analytics engineers to develop, test, and run dbt natively using the dbt CLI, without needing to know Python or Dagster. Dagster simply wraps these directories at runtime using `dagster-dbt`.

## 4. Dependencies

```toml
[dependencies]
django = "^4.2"
data_models = { path = "../../libs/data_models", develop = true }
```

## 5. Docker Profiles

`docker compose --profile data up`

```yaml
# docker/stacks/data-stack.yml
services:
  dagster_de:
    profiles: ["data", "all"] # Only starts if explicitly requested
    # ...

# docker/stacks/app-stack.yml
services:
  django_api:
    profiles: ["app", "all"]
    # ...
```