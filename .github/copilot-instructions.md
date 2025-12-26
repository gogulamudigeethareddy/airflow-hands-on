# Copilot instructions — airflow-hands-on

Quick, focused notes for an AI coding agent working in this repository. Keep changes tight, reproducible, and focused on the files listed below.

## Big picture

- This repo is a local Airflow development stack based on the official Docker Compose setup (`docker-compose.yaml`).
- Services: `postgres`, `redis`, `airflow-apiserver`, `airflow-scheduler`, `airflow-worker`, `airflow-dag-processor`, `airflow-triggerer`, `airflow-init`, `airflow-cli`, `flower`.
- DAGS, config, plugins and logs are bind-mounted from the workspace into the containers:
  - `dags/` → `/opt/airflow/dags`
  - `config/airflow.cfg` → `/opt/airflow/config/airflow.cfg`
  - `plugins/` → `/opt/airflow/plugins`
  - `logs/` → `/opt/airflow/logs`

## What an agent should know / do first

- Run the stack locally with Docker Compose:
  - `docker compose up --build -d` (or `docker-compose up --build -d`)
  - Monitor init step: `docker compose up airflow-init` will run DB migrations and create the web user using `_AIRFLOW_WWW_USER_USERNAME` and `_AIRFLOW_WWW_USER_PASSWORD` env vars.
- Use the `airflow-cli` service for commands inside container context:
  - List DAGs: `docker compose run --rm airflow-cli airflow dags list`
  - Trigger a DAG: `docker compose run --rm airflow-cli airflow dags trigger <dag_id>`
  - Test a task: `docker compose run --rm airflow-cli airflow tasks test <dag_id> <task_id> <execution_date>`
- Check logs using `docker compose logs -f <service>` (e.g., `airflow-scheduler`, `airflow-worker`). Task run logs persist to local `logs/`.

## Project-specific patterns & conventions

- Executor: The Docker Compose env sets `AIRFLOW__CORE__EXECUTOR=CeleryExecutor` (distributed/Celery setup). The stack includes Redis (broker) and Postgres (metadata DB).
- DAG discovery: Airflow's default heuristic is used (see `config/airflow.cfg` `might_contain_dag_callable`), and `dags_are_paused_at_creation = True` — new DAGs are paused by default.
- Keep DAG modules import-time cheap: `dagbag_import_timeout = 30.0`. Avoid long/blocking top-level code in files under `dags/`.
- Adding dependencies:
  - Quick way: set `_PIP_ADDITIONAL_REQUIREMENTS` (in `.env`) to e.g. `apache-airflow-providers-google==...` — this installs on container start (slow and not recommended for persistent development).
  - Recommended: create/extend Docker image (commented hint in `docker-compose.yaml`) and bump `AIRFLOW_IMAGE_NAME` or use a custom `build:`.
- User creation: `airflow-init` uses env vars `_AIRFLOW_WWW_USER_USERNAME` and `_AIRFLOW_WWW_USER_PASSWORD` to create the default web user.
- Resource checks: `airflow-init` warns if the environment has <4GB RAM or <2 CPUs — good to keep in mind for CI or constrained dev machines.

## Files to inspect when making changes

- `docker-compose.yaml` — service definitions & env variables (single source of truth for running the stack).
- `config/airflow.cfg` — runtime configuration; contains `dags_folder`, `executor`, `dagbag_import_timeout`, auth manager and other important flags.
- `dags/` — place DAG Python modules here. Use small, import-safe DAG files.
- `plugins/` — Airflow custom operators/hooks/plugins; mounted into containers.
- `logs/` — task logs and local runtime logs persisted here.
- `pyproject.toml` — project metadata (currently minimal; add test/lint deps here if adding CI).

## Useful, concrete examples

- Start stack and follow logs:
  ```bash
  docker compose up --build -d
  docker compose logs -f airflow-scheduler
  docker compose run --rm airflow-cli airflow dags list
  ```

- Add a provider quickly (not recommended as long-term):
  - Add `_PIP_ADDITIONAL_REQUIREMENTS="apache-airflow-providers-postgres==<version>"` to `.env` then restart `airflow-init`.

- Unpause a DAG (when it doesn't auto-run):
  ```bash
  docker compose run --rm airflow-cli airflow dags unpause <dag_id>
  ```

## Safety, tests & commit guidance

- No CI workflows or tests exist in repo today; if you add code, also add tests and CI steps (suggested: `pytest` and a minimal `python -m pytest` job in `.github/workflows`).
- Do not commit secrets or long-lived credentials to the repo; use `.env` (and add to `.gitignore`).
- Keep Docker and Airflow versions pinned in `docker-compose.yaml` to avoid drifting behavior.

## When to ask the human

- If a DAG requires additional platform-specific providers (GCP/AWS/etc.), confirm whether to modify `_PIP_ADDITIONAL_REQUIREMENTS` or provide a custom image.
- If you need a baseline integration test pattern (e.g., smoke-testing DAG imports), ask which runner or test framework the project owner prefers.

---

If any section is unclear or you'd like more examples (e.g., a template DAG, example plugin, or a sample GitHub Actions workflow), tell me which area to expand and I will iterate. ✅
