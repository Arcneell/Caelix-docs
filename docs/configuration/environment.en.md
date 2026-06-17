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

By default, Caelix drives the **local** Docker (or Podman) daemon via its socket.
These variables let it target a **remote** daemon, including over TLS — the first
building block of multi-node support. **When unset, single-node behaviour is
strictly unchanged.**

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

> Example — drive a remote daemon over TLS:
> `-e CAELIX_DOCKER_HOST=tcp://node-b:2376 -e CAELIX_DOCKER_TLS_VERIFY=1 -e CAELIX_DOCKER_CERT_PATH=/etc/caelix/certs`.
> The same target is honoured by the Bash engine **and** by the web console's Docker calls.

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
