# Web Console

Web-based management and monitoring interface, built with FastAPI (REST backend + SSE) and Vue 3 (TypeScript SPA).

---

## Architecture

```mermaid
graph LR
    A["Vue 3 SPA"] -->|"/api/*"| B["FastAPI Backend"]
    B --> C["Docker socket"]
    B --> D["Caelix data (/workspace)"]
    B --> E["SSE streams"]
    B --> F["/metrics (Prometheus)"]
```

### Tech Stack

| Component | Technology | Version |
|---|---|---|
| Backend | Python / FastAPI | 3.11+ |
| Frontend | Vue 3 + TypeScript | 3.5 / TS 5.9 |
| Build | Vite | 8.x |
| Styling | Tailwind CSS | 4.x |
| Icons | Lucide | — |
| State management | Pinia | 3.x |
| Routing / i18n | Vue Router 4 + Vue I18n 11 | — |
| Container | Docker multi-stage | Node + Python Alpine |

---

## Getting Started

### Via Docker (production)

```bash
./scripts/deploy-ui.sh
```

The UI is accessible at `http://localhost:18100`.

For LAN access:

```bash
CAELIX_UI_PUBLISH_BIND=0.0.0.0 ./scripts/deploy-ui.sh
```

### Local Development

=== "Backend"

    ```bash
    cd ui/backend
    pip install -r requirements.txt
    uvicorn app.main:app --reload --port 8080
    ```

=== "Frontend"

    ```bash
    cd ui/frontend
    npm install
    npm run dev
    # → http://localhost:5173 (proxies /api to the backend)
    ```

---

## Authentication

Authentication is **mandatory**. Caelix uses a multi-user system with JWT and two roles (`admin` / `technicien`).

On first launch, an `admin` / `admin` account is automatically created. The interface forces a password change for the default account.

The **SPA** authenticates via an **httpOnly** session cookie (`caelix_session`, `SameSite=strict`) set at login — the token is never stored in `localStorage`. **CLI/API clients** use the `Authorization: Bearer` header.

### API Login (CLI / scripts)

```bash
# 1. Get a JWT token
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "your_password"}'

# 2. Use the token
curl -H "Authorization: Bearer eyJ..." http://localhost:8080/api/containers/
```

SSE streams use a **single-use ticket** (`EventSource` cannot set headers): call `POST /api/auth/sse-ticket`, then open the stream with `?ticket=<ticket>`.

See [Configuration > Authentication](../configuration/authentication.en.md) for full details (session cookie, SSE tickets, roles, security, user API).

---

## Interface Sections

### Dashboard

Overview of your infrastructure:

- **Daemon status**: indicator based on heartbeat (active, inactive, unknown)
- **Service grid**: card for each service with state, uptime, quick actions
- **System metrics**: host server CPU, memory, disk
- **Recent alerts**: latest unacknowledged notifications

### Docker

Direct management of all Docker resources:

| Subsection | Features |
|---|---|
| **Containers** | List, creation (multi-step wizard), start/stop/restart, logs, exec, stats, export, commit |
| **Images** | List, pull, build, remove, prune, registry search |
| **Volumes** | List, create, remove, prune |
| **Networks** | List, create, remove, connect/disconnect |
| **Stacks** | Docker Compose management (deploy, down, status) |
| **System Info** | Docker version, storage driver, resources |
| **Events** | Real-time Docker events stream (SSE) |

### Orchestrator

Caelix-specific interface:

| Subsection | Features |
|---|---|
| **Services** | Detailed state, actions (start/stop/restart), resume |
| **Manifest Editor** | Syntax-aware INI file editing with validation |
| **Autoscale Dashboard** | Metrics, replicas, thresholds, manual scale, stress test |
| **Incidents** | Filterable history by date/service/severity, acknowledgment |
| **Audit Journal** | Container operation timeline with filtering |

### AppStore

Simplified deployment via preconfigured templates:

- Built-in template catalog
- Remote template sources
- Multi-step deployment assistant (WizardModal)

### Logs

Centralized log viewer:

- Caelix daemon logs (formatted JSON)
- Container logs (with real-time streaming)
- UI backend logs
- Log search

### Settings

- User management (admin only)
- Notification management (Discord config)
- Display preferences

---

## Mounted Volumes

The UI container requires two volumes:

| Host Volume | Container Target | Usage |
|---|---|---|
| Caelix project root | `/workspace` | Access to manifest, state, logs, bin/caelix |
| `/var/run/docker.sock` | `/var/run/docker.sock` | Communication with the Docker daemon |

---

## UI Environment Variables

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | Backend listen port |
| `CAELIX_UI_BIND` | `0.0.0.0` | Bind address |
| `CAELIX_ADMIN_PASSWORD` | `admin` | Initial admin account password |
| `CAELIX_JWT_SECRET` | (auto) | JWT signing key (auto-generated if absent) |
| `CAELIX_JWT_EXPIRE_MINUTES` | `480` | JWT token validity duration (minutes) |
| `CAELIX_RUNTIME` | (auto) | `docker` or `podman` |
| `CAELIX_UI_TLS_CERT` | — | TLS certificate path |
| `CAELIX_UI_TLS_KEY` | — | TLS key path |
| `CAELIX_METRICS_PROTECT` | `0` | Protect /metrics with authentication |
| `CAELIX_CORS_ORIGINS` | — | Allowed CORS origins (CSV). Empty = same-origin only (recommended) |
| `CAELIX_UI_VERBOSE` | `0` | Detailed HTTP logs |

---

## Frontend Components

| Component | Description |
|---|---|
| `DataTable` | Table with sorting, filtering, pagination |
| `StatusBadge` | Colored badge (running, stopped, unhealthy) |
| `JsonViewer` | Formatted JSON display |
| `ConfirmModal` | Confirmation dialog for destructive actions |
| `FeedbackToast` | Temporary notification (success, error, info) |
| `DeployProgress` | Deployment progress bar |
| `WizardModal` | Multi-step assistant |
| `ArrayField` | Form field for lists (ports, volumes, env) |
| `ContainerWizard` | Complete container creation form |
