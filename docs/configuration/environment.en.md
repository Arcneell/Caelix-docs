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
