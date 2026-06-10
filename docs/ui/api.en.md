# REST API

The FastAPI backend exposes 150+ REST endpoints across 21 routers, organized by category.

---

## Base URL and Authentication

```
Base: http://localhost:8080/api
```

Authentication is mandatory. CLI/API clients get a JWT token via `/api/auth/login`, then include it in each request via the `Authorization: Bearer` header. (The SPA, by contrast, authenticates through the httpOnly `caelix_session` cookie set at login — see [Authentication](../configuration/authentication.en.md).)

```bash
# Login
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"your_password"}' | jq -r .token)

# Usage
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/containers/

# For SSE (EventSource cannot set headers): get a single-use ticket
TICKET=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/auth/sse-ticket | jq -r .ticket)
curl "http://localhost:8080/api/stream?ticket=$TICKET"
```

See [Authentication](../configuration/authentication.en.md) for details (session cookie, SSE tickets, roles).

---

## System and Health

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/api/ping` | No | Health check (instant response) |
| `GET` | `/api/state` | Yes | Full aggregated state (containers, health, incidents, autoscale) |
| `GET` | `/api/stream` | Yes | SSE: state updates (~2s) |
| `GET` | `/metrics` | Optional | Prometheus metrics |
| `GET` | `/api/host/details` | Yes | OS, kernel, CPU, memory, disks, uptime |
| `GET` | `/api/docker/info` | Yes | `docker info` as JSON |
| `GET` | `/api/docker/version` | Yes | `docker version` as JSON |
| `GET` | `/api/docker/df` | Yes | Docker disk usage by type |
| `GET` | `/api/docker/summary` | Yes | Counters: containers, images, volumes, networks |

---

## Containers

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/containers/` | List (`all=1` to include stopped) |
| `GET` | `/api/containers/{name}/inspect` | Full details (`docker inspect`) |
| `GET` | `/api/containers/{name}/logs` | Logs (params: `tail`, `timestamps`, `search`) |
| `GET` | `/api/containers/{name}/logs/stream` | SSE: real-time logs |
| `GET` | `/api/containers/{name}/logs/download` | Download logs as text file |
| `GET` | `/api/containers/{name}/stats` | CPU/memory/network snapshot |
| `GET` | `/api/containers/{name}/stats/stream` | SSE: real-time stats (~2s) |
| `GET` | `/api/containers/{name}/top` | Running processes in the container |
| `GET` | `/api/containers/{name}/changes` | Filesystem modifications |
| `GET` | `/api/containers/{name}/export` | Download the container as TAR |
| `GET` | `/api/containers/{name}/clone-config` | Extract config from an existing container |
| `POST` | `/api/containers/` | Create a container (full spec) |
| `POST` | `/api/containers/create/preview` | Preview the `docker run` command |
| `POST` | `/api/containers/preflight` | Pre-creation checks (ports, names, volumes) |
| `POST` | `/api/containers/pull` | Pull an image: `{"image":"nginx:latest"}` |
| `GET` | `/api/containers/available-ports` | Available host ports (`count=3`) |
| `POST` | `/api/containers/{name}/action` | Action: `{"action":"start\|stop\|restart\|remove\|kill\|pause\|unpause"}` |
| `POST` | `/api/containers/{name}/exec` | Execute: `{"cmd":"ls -la /app"}` |
| `POST` | `/api/containers/{name}/commit` | Commit: `{"repo":"myimage","tag":"v1"}` |
| `POST` | `/api/containers/{name}/rename` | Rename: `{"name":"new-name"}` |

---

## Images

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/images/` | List all images (cached) |
| `GET` | `/api/images/{id}/inspect` | Image details |
| `GET` | `/api/images/{id}/history` | Layer history |
| `GET` | `/api/images/{id}/export` | Download the image as a `.tar` file |
| `GET` | `/api/images/search` | Registry search (`q=term`) |
| `GET` | `/api/images/metadata` | Image metadata (`image=nginx:latest`): ports, volumes, env. Supports Docker Hub without pull. |
| `POST` | `/api/images/pull` | Pull an image: `{"image":"nginx:latest"}` |
| `POST` | `/api/images/build` | Build an image: `{"dockerfile_content":"...","tag":"my-image:v1","no_cache":false}` |
| `POST` | `/api/images/import` | Import from a `.tar` file (raw body) |
| `POST` | `/api/images/{id}/tag` | Tag: `{"repo":"my-registry/image","tag":"latest"}` |
| `POST` | `/api/images/{id}/push` | Push to a registry |
| `POST` | `/api/images/{id}/remove` | Remove an image (`{"force":true}` optional) |
| `POST` | `/api/images/prune` | Remove dangling images |

---

## Volumes

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/volumes/` | List all volumes |
| `GET` | `/api/volumes/{name}/inspect` | Volume details |
| `POST` | `/api/volumes/create` | Create a volume |
| `POST` | `/api/volumes/{name}/remove` | Remove a volume |
| `POST` | `/api/volumes/prune` | Remove unused volumes |

---

## Networks

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/networks/` | List all networks |
| `GET` | `/api/networks/{name}/inspect` | Network details |
| `POST` | `/api/networks/create` | Create a network |
| `POST` | `/api/networks/{name}/remove` | Remove a network |
| `POST` | `/api/networks/{name}/connect` | Connect a container |
| `POST` | `/api/networks/{name}/disconnect` | Disconnect a container |

---

## Stacks (Docker Compose)

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/stacks/` | List stacks |
| `GET` | `/api/stacks/{name}/status` | Services + compose file |
| `GET` | `/api/stacks/{name}/content` | Raw compose file content |
| `GET` | `/api/stacks/{name}/env` | Stack environment variables |
| `GET` | `/api/stacks/{name}/webhook/trigger` | Trigger webhook via GET |
| `POST` | `/api/stacks/deploy` | Deploy a new stack |
| `POST` | `/api/stacks/deploy-git` | Deploy from a Git repository |
| `POST` | `/api/stacks/{name}/update` | Update compose (save + redeploy) |
| `POST` | `/api/stacks/{name}/down` | Stop a stack |
| `POST` | `/api/stacks/{name}/delete` | Delete a stack |
| `POST` | `/api/stacks/{name}/env` | Save environment variables |
| `POST` | `/api/stacks/{name}/webhook` | Create a redeployment webhook |
| `POST` | `/api/stacks/{name}/webhook/trigger` | Trigger webhook (pull + redeploy) |
| `POST` | `/api/stacks/{name}/save-template` | Save stack as a custom template |

---

## Orchestrator

### Manifest

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/manifest/show` | Formatted display (`caelix show`) |
| `GET` | `/api/manifest/raw` | Raw INI file content |
| `POST` | `/api/manifest/raw` | Save (validates before writing) |
| `POST` | `/api/validate` | Run `caelix validate` |
| `POST` | `/api/doctor` | Run `caelix doctor` (options: `--fix`, `--strict-local`) |

### Reconciliation

| Method | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/api/reconcile` | — | Run `caelix once` |
| `POST` | `/api/reconcile-app` | `{"app":"name"}` | Reconcile a single service |

### Services

| Method | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/api/service/action` | `{"app":"x","action":"start\|stop\|restart"}` | Service action |
| `POST` | `/api/service/clear_pause` | `{"app":"x"}` | Clear manual pause |
| `POST` | `/api/apps/upsert` | `{"name":"app","values":{...}}` | Create or update a service |
| `POST` | `/api/apps/delete` | `{"name":"app"}` | Delete a service + containers |
| `POST` | `/api/apps/bump_config` | `{"name":"app"}` | Increment config_version |

### Global Parameters

| Method | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/api/orchestrator/patch` | `{"orchestrator":{"interval":"8"}}` | Modify globals |
| `POST` | `/api/manifest/orchestrator/lang` | `{"value":"fr\|en"}` | Change the orchestrator language |
| `POST` | `/api/proxy/config` | `{"proxy":{"listen":"..."}}` | Modify proxy config |

---

## Autoscale

| Method | Endpoint | Body | Description |
|---|---|---|---|
| `GET` | `/api/autoscale/state` | — | State of all autoscaled services |
| `POST` | `/api/autoscale/scale` | `{"app":"x","action":"up\|down"}` | Manual scale |
| `POST` | `/api/autoscale/stress` | `{"app":"x","action":"add\|reset","amount":50}` | Latency injection for testing |

---

## Incidents and Alerts

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/incidents?limit=50` | List of recent incidents |
| `POST` | `/api/alerts/ack` | Acknowledge an incident by ID |
| `POST` | `/api/alerts/ack_all` | Acknowledge all incidents |

---

## Audit

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/audit/recent?limit=50` | Recent events |
| `POST` | `/api/audit/clear` | Clear audit logs |

---

## Notifications

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/notifications/` | List notifications |
| `GET` | `/api/notifications/stream` | SSE: real-time stream |
| `POST` | `/api/notify/save` | Save notification config (Discord, Slack, Teams, Telegram, SMTP) |
| `POST` | `/api/notify/test` | Send a test message to all enabled channels |

---

## Docker Cleanup

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/docker/prune/container` | Remove stopped containers |
| `POST` | `/api/docker/prune/image` | Remove dangling images |
| `POST` | `/api/docker/prune/volume` | Remove unused volumes |
| `POST` | `/api/docker/prune/network` | Remove unused networks |
| `POST` | `/api/docker/prune/build_cache` | Clear build cache |

---

## Docker Events

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/docker/events/stream` | SSE: live Docker events |
| `GET` | `/api/docker/events/recent?since=1h` | Recent events |

---

## Templates

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/templates/external` | List external templates (configured sources) |
| `GET` | `/api/templates/sources` | List template sources |
| `POST` | `/api/templates/sources` | Save template sources |
| `POST` | `/api/templates/deploy-full` | Deploy a template (full workflow with Caelix) |
| `POST` | `/api/templates/save` | Save Caelix service templates |
| `GET` | `/api/templates/custom` | List custom templates |
| `POST` | `/api/templates/custom/create` | Create a custom template |
| `POST` | `/api/templates/custom/{name}/update` | Update a custom template |
| `POST` | `/api/templates/custom/{name}/delete` | Delete a custom template |

---

## Webhooks

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/webhooks/` | List container webhooks |
| `POST` | `/api/webhooks/{name}/create` | Create a webhook for a container |
| `POST` | `/api/webhooks/{name}/delete` | Delete a webhook |
| `GET` | `/api/webhooks/{name}/trigger` | Trigger via GET (for CI integrations) |
| `POST` | `/api/webhooks/{name}/trigger` | Trigger via POST (pull + recreate) |

---

## Registries

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/registries/` | List configured registries |
| `POST` | `/api/registries/add` | Add a registry |
| `POST` | `/api/registries/{name}/remove` | Remove a registry |
| `POST` | `/api/registries/{name}/login` | Log in to a registry |

---

## Backup and Restore

### Caelix Configuration

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/backup` | Download a `.tar.gz` archive of the full Caelix configuration |
| `POST` | `/api/restore` | Restore configuration from an archive (raw body) |

### Scheduled Volume Backups

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/backup/status` | Backup status for all services |
| `POST` | `/api/backup/trigger/{name}` | Trigger an immediate backup |
| `GET` | `/api/backup/list/{name}` | List archives for a service |
| `GET` | `/api/backup/download/{name}/{filename}` | Download an archive |
| `POST` | `/api/backup/config/{name}` | Update backup configuration for a service |
| `GET` | `/api/backup/defaults` | Default configuration template |

---

## Automatic Updates

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/update/status` | Update status for all services (enabled, interval, strategy, digest, update_available) |
| `POST` | `/api/update/check/{name}` | Force an immediate check (docker pull + digest comparison) |
| `POST` | `/api/update/apply/{name}` | Apply a pending update (pull + rollout) |
| `POST` | `/api/update/config/{name}` | Update auto-update configuration for a service |

---

## Miscellaneous

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/logs/search` | Search in container logs |
| `GET` | `/api/openapi.json` | OpenAPI 3.0.3 spec |
| `POST` | `/api/data/clear` | Clear incidents and/or audit |

---

## Server-Sent Events (SSE)

SSE streams enable real-time updates without polling. Authentication uses a **single-use ticket** (`POST /api/auth/sse-ticket`, valid once, 30 s) passed as a `?ticket=` parameter — never the JWT directly:

```javascript
const { ticket } = await (await fetch('/api/auth/sse-ticket', { method: 'POST' })).json();
const evtSource = new EventSource(`/api/stream?ticket=${encodeURIComponent(ticket)}`);
evtSource.onmessage = (event) => {
  const state = JSON.parse(event.data);
  // Update the interface
};
```

Available streams:

| Endpoint | Content | Frequency |
|---|---|---|
| `/api/stream` | Global aggregated state | ~2s |
| `/api/containers/{name}/logs/stream` | Container logs | Real-time |
| `/api/containers/{name}/stats/stream` | Container stats | ~2s |
| `/api/docker/events/stream` | Docker events | Real-time |
