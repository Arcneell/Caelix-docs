# Environment Variables

SORK can be configured via environment variables in addition to the manifest.

## Main Variables

| Variable | Default | Description |
|---|---|---|
| `SORK_MANIFEST` | `etc/manifest.ini` | Path to the manifest file |
| `SORK_NOTIFY_CONF` | `etc/notify.ini` | Path to the notification configuration |
| `SORK_DATA` | `./.sork` | Runtime data directory |
| `SORK_INTERVAL` | `15` | Reconciliation interval in seconds |
| `SORK_MAX_REPAIR` | `5` | Failure threshold before alert |
| `SORK_LOG_LEVEL` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `SORK_STRICT_LOCAL` | `0` | If `1`, health check URLs must target localhost |

## Configuration Priority

Environment variables are overridden by manifest values if defined in `[orchestrator]`:

```
Environment variable < manifest.ini [orchestrator]
```

For example, if `SORK_INTERVAL=15` and the manifest defines `interval = 10`, `10` will be used.

## Usage Examples

### Launch with a Custom Manifest

```bash
SORK_MANIFEST=/etc/sork/manifest.ini bin/sork run
```

### Separate Data Directory

```bash
SORK_DATA=/var/lib/sork bin/sork run
```

### Debug Mode

```bash
SORK_LOG_LEVEL=debug bin/sork once
```

### Strict Mode (localhost health checks only)

```bash
SORK_STRICT_LOCAL=1 bin/sork validate
```

This mode is useful to ensure that health checks do not target remote services, which could cause false positives or information leaks.

## systemd Service

The `sork.global.service` file configures the variables for running as a service:

```ini
[Service]
Environment=SORK_MANIFEST=/opt/sork/etc/manifest.ini
Environment=SORK_NOTIFY_CONF=/opt/sork/etc/notify.ini
```

To add additional variables:

```bash
sudo systemctl edit sork
```

```ini
[Service]
Environment=SORK_LOG_LEVEL=debug
Environment=SORK_DATA=/var/lib/sork
```
