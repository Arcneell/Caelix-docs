# Development Workflow

This document describes the development process for contributing to the Caelix project. It covers prerequisites, local development, testing, and code conventions.

## Prerequisites

| Tool        | Purpose                                           | Required |
|-------------|---------------------------------------------------|----------|
| Docker      | Container runtime, UI container build             | Yes      |
| Bash 4+     | Associative arrays used throughout `lib/`         | Yes      |
| Git         | Version control                                   | Yes      |
| Python 3    | Backend, manifest_doctor, audit_log               | Yes      |
| Node.js     | Frontend development (npm, Vite)                  | Yes      |
| ShellCheck  | Bash linting                                      | Yes      |
| socat       | Proxy functionality (only needed for autoscale)   | Optional |
| curl        | HTTP health checks (used by the bash engine)      | Yes      |

## Local Development

### Running the Orchestrator Daemon

Start the reconciliation loop:

```bash
bin/caelix run
```

Or run a single reconciliation pass:

```bash
bin/caelix once
```

Environment variables for local development:

```bash
export CAELIX_MANIFEST="$PWD/etc/manifest.ini"
export CAELIX_NOTIFY_CONF="$PWD/etc/notify.ini"
export CAELIX_DATA="$PWD/.caelix"
export CAELIX_INTERVAL=15
export CAELIX_LOG_LEVEL=debug
```

### Deploying the Web UI

Use the deploy script to build and run the UI container:

```bash
./scripts/deploy-ui.sh
```

This builds `caelix-ui:2` and runs it with the project root mounted at `/workspace`. The UI is available at `http://127.0.0.1:18100/`.

To expose on the LAN:

```bash
CAELIX_UI_PUBLISH_BIND=0.0.0.0 ./scripts/deploy-ui.sh
```

### Backend Development

The backend code (`ui/backend/app/`) is mounted as a volume in the container. To apply changes:

```bash
docker restart caelix-caelix-ui
```

The backend runs with Uvicorn. Environment variables like `CAELIX_LOG_LEVEL=DEBUG` can be set via the manifest's `env` key or directly with `docker exec`.

### Frontend Development

For hot-reload during frontend development:

```bash
cd ui/frontend
npm install
npm run dev
```

This starts a Vite development server with hot module replacement, typically on port 5173. API requests are proxied to the backend container.

For production builds:

```bash
cd ui/frontend
npm run build
```

The built assets land in `ui/frontend/dist/` and are copied into the Docker image during `docker build`.

To rebuild the full UI image after frontend changes:

```bash
docker build -t caelix-ui:2 ./ui
```

## Testing

### Full Check Suite

Run all automated checks:

```bash
./scripts/check-all.sh
```

This script executes the following steps in order:

1. **ShellCheck** (`shellcheck -x`): lint `bin/caelix`, `lib/*.sh`, `scripts/*.sh` with external source following enabled.
2. **Bash syntax** (`bash -n`): syntax-check all shell scripts.
3. **Python compile**: compile-check `lib/`, `ui/backend/app/` with `python3 -m compileall`.
4. **Python modules**: explicit compile of `manifest_doctor.py` and `audit_log.py`.
5. **Manifest validation**: run `manifest_doctor.py check` against the example manifest.
6. **Caelix validate**: run `bin/caelix validate` with the example manifest.
7. **Caelix doctor**: run `bin/caelix doctor` with the example manifest.
8. **Docker build** (optional): set `CHECK_UI_DOCKER_BUILD=1` to include a UI image build.

```bash
# Include Docker build in checks
CHECK_UI_DOCKER_BUILD=1 ./scripts/check-all.sh
```

### Manual Testing

Validate the manifest:

```bash
bin/caelix validate
```

Run the doctor with auto-repair:

```bash
bin/caelix doctor --fix
```

Run a single-service reconciliation:

```bash
bin/caelix reconcile-app myservice
```

Check the current state:

```bash
bin/caelix status
bin/caelix show
```

## Code Conventions

### Bash

- Use `set -euo pipefail` in all scripts.
- Prefix all Caelix functions with `caelix_` or use module-specific prefixes (`manifest_`, `container_`, `autoscale_`, `proxy_`, etc.).
- Container names follow the pattern `caelix-<app>` (managed by `caelix_cname()`).
- State files go in `$CAELIX_DATA/state/`, incidents in `$CAELIX_DATA/incidents/`.
- Use `caelix_log <level> <message>` for all logging (writes to stderr and JSON log file).
- Add `# shellcheck shell=bash` and relevant `# shellcheck source=` directives.
- Use `manifest_get_default` with sensible defaults rather than failing on missing keys.

### Python

- Type hints on function signatures.
- Use `_sanitize_input()` on all user-provided strings before passing to subprocess.
- Use `_valid_docker_name()` to validate container/image names.
- All routes require the `verify_auth` dependency (except `/api/ping` and optionally `/metrics`).
- Structured logging via `get_logger()` and `log_action()`.

### Frontend (Vue/TypeScript)

- Composition API with `<script setup>`.
- Tailwind CSS for styling.
- Lucide icons.
- Shared components in `components/shared/`.
- Views organized by domain in `views/docker/` and `views/orchestrator/`.

## Commit Conventions

This project follows [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <description>

[optional body]

[optional footer(s)]
```

Common types:

| Type       | When to use                                    |
|------------|------------------------------------------------|
| `feat`     | New feature                                    |
| `fix`      | Bug fix                                        |
| `refactor` | Code restructuring without behavior change     |
| `docs`     | Documentation only                             |
| `style`    | Formatting, whitespace, no code change         |
| `test`     | Adding or updating tests                       |
| `chore`    | Build process, dependencies, tooling           |
| `perf`     | Performance improvement                        |

Scopes: `engine`, `ui`, `backend`, `frontend`, `proxy`, `autoscale`, `manifest`, `docker`, `docs`.

Examples:

```
feat(autoscale): add http_error_rate metric support
fix(engine): prevent reconciliation loop on manual stop
docs: update configuration/manifest.md with new autoscale keys
refactor(backend): extract notification persistence to core module
```

## Project Structure Quick Reference

| Path                     | What to edit                                       |
|--------------------------|----------------------------------------------------|
| `bin/caelix`               | CLI commands, reconcile_all, do_run                |
| `lib/*.sh`               | Bash engine modules                                |
| `lib/*.py`               | Python helpers used by bash (audit, doctor)         |
| `ui/backend/app/main.py` | FastAPI app, middleware, router registration        |
| `ui/backend/app/routers/`| API endpoint implementations                      |
| `ui/backend/app/core/`   | Shared backend logic (auth, docker, logging, etc.) |
| `ui/backend/app/config.py`| Path constants, env vars, rate limiting           |
| `ui/frontend/src/views/` | Vue page components                                |
| `ui/frontend/src/components/shared/` | Reusable UI components              |
| `etc/manifest.ini`       | Local manifest (not committed)                     |
| `etc/manifest.ini.example`| Reference manifest (committed)                   |
| `etc/notify.ini.example` | Reference notification config (committed)          |
| `docs/`                  | Documentation                                      |
| `scripts/`               | Utility scripts (deploy, check, install)           |
