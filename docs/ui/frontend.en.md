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

## Application Shell

v2.0 introduces flat navigation (NetBird / Portainer style), built for the cluster. The shell lives in `src/App.vue`:

- a single sidebar renders the section list (flat buttons, no collapsible groups);
- a header shows the page title, the cluster status strip (`components/cluster/ClusterStatusStrip.vue`), the language toggle, the light/dark theme toggle, the notifications bell, and the user menu;
- a tab bar (`components/layout/SectionTabs.vue`) for multi-facet sections.

### Section configuration

Sections and their tabs are declared centrally in `src/config/sections.ts` (`SECTIONS: NavSection[]`). Each section carries an `id`, an i18n `labelKey`, a Lucide icon, a `to` route, route `prefixes` (active state and tab resolution) and, optionally, `tabs` and a `clusterOnly` flag. The `sectionForPath()` helper does a longest-prefix match to resolve the active section. Existing view routes are kept; only the chrome is new.

| Section | Route | Tabs |
|---|---|---|
| **Overview** | `/` | — (cluster topology dashboard) |
| **Nodes** *(`clusterOnly`)* | `/nodes` | — |
| **Containers** | `/containers` | `/containers` · `/images` · `/volumes` · `/networks` · `/system` |
| **Services** | `/services` | `/services` · `/autoscale` |
| **Stacks** | `/stacks` | `/stacks` · `/apps` · `/app-store` |
| **Ingress** | `/apps/domains` | `/apps/domains` · `/apps/certificates` |
| **Activity** | `/logs` | `/logs` · `/events` · `/incidents` · `/journal` |
| **Settings** | `/settings` | `/settings` · `/cluster` |

Sections marked `clusterOnly` (Nodes) are hidden in single-host mode; the console then also drops the "Node" column from lists and the cluster status strip from the header.

### Notable views

- **Overview**: node cards (role / leader / VIP, health, CPU·RAM, per-node resource counts), cluster KPIs, quorum, and recent incidents.
- **Containers**: table with filters and a "Node" column (cluster), inline actions targeting the row's node, logs in a modal, access to the creation assistant (guided wizard: Docker Hub image search, ports/volumes/env auto-filled from metadata, dedicated or existing network, Caelix orchestrator with health checks, autoscale with dedicated proxy).
- **Services / Autoscale**: detailed orchestrated-service state, metrics, replicas, thresholds, manual scale.
- **Stacks**: Compose, deployed applications, and the template catalog (deployment assistant).
- **Activity**: centralized logs (daemon, real-time container, UI backend), Docker events, filterable incidents, audit journal.
- **Settings**: user management (admin), notifications, preferences, and cluster settings.

Long lists are virtualized and rendered progressively: a slow node does not block the display.

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
