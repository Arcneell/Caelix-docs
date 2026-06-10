# Frontend

The Caelix frontend is a Single Page Application (SPA) built with Vue 3 and TypeScript.

## Stack

| Tool | Version | Role |
|---|---|---|
| **Vue 3** | 3.5 | Reactive framework (Composition API, `<script setup>`) |
| **TypeScript** | 5.9 | Static typing (build via `vue-tsc -b`, typed API responses in `src/types/`) |
| **Vite** | 8.x | Build tool & dev server |
| **Tailwind CSS** | 4.x | Utility-first styles |
| **Pinia** | 3.x | State management (`stores/auth.ts`, `stores/app.ts`) |
| **Vue Router** | 4.x | SPA navigation (hash mode `#/`) + guards |
| **Vue I18n** | 11.x | FR/EN internationalization |
| **Lucide** | — | Icons |
| **Vitest** | 4.x | Unit tests (jsdom + Vue Test Utils) |

## View Structure

### Dashboard

The home page displays:

- **Daemon status**: indicator based on heartbeat (active, inactive, unknown)
- **Services**: summary card for each service with state, quick actions
- **System**: host server CPU, memory, disk
- **Recent alerts**: latest notifications

### Docker

Docker resource management organized in sub-views:

- **Containers**: table with filters, inline actions, logs in modal, creation assistant access button
- **Creation Assistant**: guided 6-step wizard to create a group of containers (Docker Hub image search, auto-filled ports/volumes/env from image metadata, dedicated or existing network, Caelix orchestrator with health checks, full autoscale with dedicated proxy)
- **Images**: gallery with pull, build, removal
- **Volumes**: list with size and attachments
- **Networks**: network topology
- **Stacks**: Docker Compose management
- **System Info**: Docker version, storage driver, container count
- **Events**: live Docker events stream

### Orchestrator

Caelix-specific interface:

- **Services**: detailed state of each orchestrated service (health, replicas, last action)
- **Manifest Editor**: syntax-aware INI file editing with live validation
- **Autoscale Dashboard**: metric graphs, replica count, active thresholds
- **Incidents**: table filterable by date, service, severity
- **Audit Journal**: container operation timeline with filtering

### AppStore

Simplified deployment:

- Template catalog (preconfigured services)
- Remote template sources
- **WizardModal**: multi-step deployment assistant

### Logs

Centralized log viewer:

- Caelix daemon logs (formatted JSON)
- Container logs (with real-time streaming)
- UI backend logs

### Settings

- Authentication token configuration
- Read/unread notification management
- Display preferences

## Reusable Components

| Component | Description |
|---|---|
| `DataTable` | Table with sorting, filtering, pagination |
| `StatusBadge` | Colored badge based on state (running, stopped, unhealthy) |
| `JsonViewer` | Formatted JSON display |
| `ConfirmModal` | Confirmation dialog for destructive actions |
| `FeedbackToast` | Temporary notification (success, error) |
| `DeployProgress` | Deployment progress bar |
| `WizardModal` | Multi-step assistant |
| `ArrayField` | Form field for lists (ports, volumes, env) |
| `ContainerWizard` | Complete container creation form |
| `ServiceAssistant` | Guided multi-container creation assistant, split into step components (`ServiceAssistantImageStep`, `…ContainerStep`, `…OrchestratorStep`, `…SummaryStep`) |
| `StackAssistant` | Application-stack deployment assistant (templates) |
| `UsersSettingsPanel` / `WebhooksSettingsPanel` / `BackupSettingsPanel` / `RegistriesSettingsPanel` | Panels extracted from `SettingsView` |

!!! note "Splitting the large components"
    Both the `ServiceAssistant` and `StackAssistant` assistants (~1600 lines each originally) were split into per-step subcomponents, and `SettingsView` into dedicated panels, to improve readability and testability.

## Authentication and State

- **SPA session**: no token is stored in `localStorage`. The JWT is held in memory for the tab's lifetime (`stores/auth.ts`), and the httpOnly `caelix_session` cookie re-authenticates after a refresh via `/api/auth/me`.
- **Requests**: the client (`src/api/client.ts`) adds the `Authorization: Bearer` header when a token is in memory; the cookie is sent automatically by the browser.
- **SSE streams**: `src/api/sse.ts` first exchanges the JWT for a single-use ticket (`/api/auth/sse-ticket`), then opens the `EventSource` with `?ticket=`, with automatic reconnection.

## Tests

Unit tests (`src/__tests__/`, ~80 cases with Vitest) cover the Pinia stores, router guards, the API client (401/429/timeout handling, token never persisted), utilities, and shared components (`DataTable`, `StatusBadge`, `ConfirmModal`, `ArrayField`, `DeployProgress`, `WebhookNotificationCard`). Run with `npm run test`.

## Production Build

```bash
cd ui/frontend
npm install
npm run build
```

Assets are generated in `dist/` and served as static files by the FastAPI backend.

## Development

```bash
cd ui/frontend
npm install
npm run dev
```

The Vite development server runs on `http://localhost:5173` with hot-reload. It proxies `/api` requests to the FastAPI backend.
