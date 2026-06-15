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
