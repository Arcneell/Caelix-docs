# Environment Variables

Caelix can be configured via environment variables in addition to the manifest.

## Main Variables

| Variable | Default | Description |
|---|---|---|
| `CAELIX_MANIFEST` | `etc/manifest.ini` | Path to the manifest file |
| `CAELIX_NOTIFY_CONF` | `etc/notify.ini` | Path to the notification configuration |
| `CAELIX_DATA` | `./.caelix` | Runtime data directory |
| `CAELIX_INTERVAL` | `15` | Reconciliation interval in seconds |
| `CAELIX_MAX_REPAIR` | `5` | Failure threshold before alert |
| `CAELIX_LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `CAELIX_STRICT_LOCAL` | `0` | If `1`, health check URLs must target localhost |

## Container Runtime (local or remote daemon)

By default, Caelix drives the local Docker (or Podman) daemon via its socket.
These variables let it target a remote daemon, including over TLS: this is the first
building block of multi-node support. When unset, single-node behaviour is
strictly unchanged.

| Variable | Default | Description |
|---|---|---|
| `CAELIX_RUNTIME` | _(auto)_ | Force the engine: `docker` or `podman`. Auto-detected otherwise. |
| `CAELIX_DOCKER_HOST` | _(empty)_ | Target daemon (e.g. `tcp://10.0.0.5:2376`). When set, propagated to the standard Docker env vars (`DOCKER_HOST`, and `CONTAINER_HOST` for Podman). Empty = local socket. |
| `CAELIX_DOCKER_TLS_VERIFY` | _(empty)_ | If `1`, enables TLS verification in the Docker client (`DOCKER_TLS_VERIFY`). |
| `CAELIX_DOCKER_CERT_PATH` | _(empty)_ | Directory of the TLS client certificates (`ca.pem`, `cert.pem`, `key.pem`), mapped to `DOCKER_CERT_PATH`. |
| `CAELIX_RT_TIMEOUT` | `60` | Timeout (s) for short runtime commands; `0` disables it. Long/streaming verbs (`pull`, `run`, `exec`, `logs`…) are never cut off. |
| `CAELIX_RUNTIME_PROBE_TIMEOUT` | `10` | Timeout (s) for the daemon availability probe (`info`). |
| `CAELIX_NODE_ID` | _(auto)_ | Node identity (`agent` mode, multi-node). If unset, an id is generated once and persisted in `.caelix/node/id`. Sanitized to `[a-z0-9-]`. |
| `CAELIX_NODE_LABELS` | _(empty)_ | Node placement labels, comma-separated `key=value` (e.g. `zone=eu-west,disk=ssd`). Exposed by `caelix node-info`. |
| `CAELIX_NODE_ADDR` | _(empty)_ | Node advertise address (IP reachable by other nodes, over the mesh or the LAN). Used as the host of the backends published to the service registry (ingress). Without it, the agent publishes no backend. |
| `CAELIX_INGRESS` | `0` | If `1`, the node acts as an ingress: on each cycle the global proxy is fed from the cluster backend registry (cross-node load-balancing). Both `file` **and** `consul` backends. |
| `CAELIX_WG_PUBKEY` | _(empty)_ | The node's WireGuard public key, published in its meta so the controller/peers can build the mesh. Empty until the mesh is configured. |
| `CAELIX_WG_ENDPOINT` | _(empty)_ | The node's WireGuard endpoint (`host:port`) advertised to peers. |
| `CAELIX_WG_IFACE` | `caelix-wg0` | Name of the WireGuard interface created by `caelix mesh-up`. |
| `CAELIX_WG_LISTEN_PORT` | `51820` | Local WireGuard listen port applied by `caelix mesh-up`. |
| `CAELIX_NODE_TTL` | `30` | Heartbeat lease duration (s): past it without renewal, the controller treats the node as *down* (reschedule) and the agent that can no longer renew *fences* itself. |
| `CAELIX_CONTROLLER` | `0` | If `1`, the backend runs the **controller loop** (HA): acquire leadership then periodically reschedule onto live nodes (only the leader acts). Enable it on control nodes. |
| `CAELIX_CONTROLLER_INTERVAL` | `10` | Interval (s) between controller-loop passes. |
| `CAELIX_CLUSTER_STORE` | _(empty)_ | File cluster store root (RFC §9 layout). When set, the agent (`caelix agent`) publishes its meta to `nodes/<id>/meta.json` and adopts the `nodes/<id>/desired.ini` sub-manifest pushed by the controller. Empty = single-host mode. |
| `CAELIX_CLUSTER_BACKEND` | `file` | Cluster store backend, **on both the controller AND the agent**: `file` (uses `CAELIX_CLUSTER_STORE`) or `consul` (Consul KV). The agent (`caelix agent`) publishes its meta and reads its sub-manifest through this backend. |
| `CAELIX_CONSUL_ADDR` | `http://127.0.0.1:8500` | Consul agent HTTP address (when `CAELIX_CLUSTER_BACKEND=consul`). On the agent side, `curl` is required. |
| `CAELIX_CONSUL_TOKEN` | _(empty)_ | Consul ACL token (`X-Consul-Token` header), for an ACL-secured Consul cluster. |
| `CAELIX_DOCKER_ADDR` | _(empty)_ | TCP address of the node's Docker daemon (e.g. `tcp://10.0.0.5:2375`), published in the node meta. The console (and the cluster HPA) consume it to drive the right per-node daemon (`X-Caelix-Node` header). |
| `CAELIX_PIN_LOCAL_SECTIONS` | _(empty)_ | Local manifest sections to **pin** (CSV): re-injected after adopting the controller's pushed sub-manifest (e.g. `caelix-ui,proxy`), so a node keeps its local services. |

### Cluster VIP (floating ingress)

The floating VIP gives a stable access address (console + ingress) held by the
**leader** node. See the [Proxy](../modules/proxy.md) module and `node_vip_*` (lib/node.sh).

| Variable | Default | Description |
|---|---|---|
| `CAELIX_CLUSTER_VIP` | _(empty)_ | Cluster VIP address as a CIDR (e.g. `10.0.0.10/32`). Placed on the interface by the node holding leadership; source of truth for the VIP. Empty = no VIP. |
| `CAELIX_VIP_IFACE` | _(auto)_ | Interface carrying the VIP. If unset, the default-route interface is used. |
| `CAELIX_VIP_ARP` | `1` | If `1`, sends a gratuitous ARP (`arping`) when taking the VIP, so the LAN immediately reroutes to the new leader. |
| `CAELIX_ADMIN_PASSWORD` | _(empty)_ | Initial console admin password, to set identically on all nodes (the hash is stored in the cluster store). Also settable via `install.sh --admin-password`. |

> **Security (cluster)**: in cluster mode, `dockerd:2375` (cf. `CAELIX_DOCKER_ADDR`) and
> Consul `:8500` are bound to the node's private IP. In production, enable Consul ACLs
> + token (`CAELIX_CONSUL_TOKEN`) + TLS: the Consul KV holds the JWT secret, password
> hashes and TLS keys.

> Example, drive a remote daemon over TLS:
> `-e CAELIX_DOCKER_HOST=tcp://node-b:2376 -e CAELIX_DOCKER_TLS_VERIFY=1 -e CAELIX_DOCKER_CERT_PATH=/etc/caelix/certs`.
> The same target is honoured by the Bash engine and by the web console's Docker calls.

## Web Console and Reverse Proxy

These variables apply to the web console container (FastAPI API) when it sits behind a reverse proxy (TLS, load balancer):

| Variable | Default | Description |
|---|---|---|
| `CAELIX_FORWARDED_ALLOW_IPS` | `*` | IP/CIDR of upstream proxies whose `X-Forwarded-*` headers uvicorn trusts. Restricting it to the reverse proxy IP prevents a client from spoofing `X-Forwarded-Proto`/`X-Forwarded-For`. |
| `CAELIX_TRUSTED_PROXY` | `0` | If `1`, the app trusts `X-Forwarded-For` for the client IP (brute-force lockout, audit). Enable only behind a trusted proxy that rewrites the header. |
| `CAELIX_FORCE_SECURE_COOKIE` | `0` | If `1`, forces the `Secure` attribute on the session cookie even without `X-Forwarded-Proto` (TLS terminated upstream that does not forward the header). |
| `CAELIX_METRICS_PROTECT` | `0` | If `1`, the `/metrics` endpoint requires authentication. |
| `CAELIX_CORS_ORIGINS` | _(empty)_ | Comma-separated allowed CORS origins. Empty = same-origin only (most secure). |

> Example: behind a single nginx at `172.18.0.2`, run the container with `-e CAELIX_FORWARDED_ALLOW_IPS=172.18.0.2 -e CAELIX_TRUSTED_PROXY=1` to harden the trust placed in forwarded headers.

## Configuration Priority

Environment variables are overridden by manifest values if defined in `[orchestrator]`:

```
Environment variable < manifest.ini [orchestrator]
```

For example, if `CAELIX_INTERVAL=15` and the manifest defines `interval = 10`, `10` will be used.

## Usage Examples

### Launch with a Custom Manifest

```bash
CAELIX_MANIFEST=/etc/caelix/manifest.ini bin/caelix run
```

### Separate Data Directory

```bash
CAELIX_DATA=/var/lib/caelix bin/caelix run
```

### Debug Mode

```bash
CAELIX_LOG_LEVEL=debug bin/caelix once
```

### Strict Mode (localhost health checks only)

```bash
CAELIX_STRICT_LOCAL=1 bin/caelix validate
```

This mode is useful to ensure that health checks do not target remote services, which could cause false positives or information leaks.

## systemd Service

The `caelix.global.service` file configures the variables for running as a service:

```ini
[Service]
Environment=CAELIX_MANIFEST=/opt/caelix/etc/manifest.ini
Environment=CAELIX_NOTIFY_CONF=/opt/caelix/etc/notify.ini
```

To add additional variables:

```bash
sudo systemctl edit caelix
```

```ini
[Service]
Environment=CAELIX_LOG_LEVEL=debug
Environment=CAELIX_DATA=/var/lib/caelix
```
