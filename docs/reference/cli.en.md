# CLI Reference

The `bin/caelix` binary is the main entry point for Caelix.

## Usage

```bash
bin/caelix <command> [options]
```

Or with environment variables:

```bash
CAELIX_MANIFEST=etc/manifest.ini CAELIX_DATA=.caelix bin/caelix <command>
```

## Commands

### run

```bash
bin/caelix run
```

Starts the infinite reconciliation loop (daemon mode).

- Executes `reconcile_all()` in a loop
- Interval configurable via `[orchestrator] interval` or `CAELIX_INTERVAL`
- Writes a heartbeat on each cycle
- Stops with `Ctrl+C` or `SIGTERM`

This is the primary command for production operation.

### once

```bash
bin/caelix once
```

Executes a single reconciliation cycle then exits.

Use cases:
- First launch after installation
- Integration into a cron job
- Testing and debugging

### reconcile-app

```bash
bin/caelix reconcile-app <service-name>
```

Reconciles only the specified service.

```bash
# Example
bin/caelix reconcile-app web
```

### validate

```bash
bin/caelix validate
```

Validates the manifest syntax and keys. Checks:

- Correct INI syntax
- Required keys present (`image`, `health_url` if http)
- Port format
- Valid values for strategies
- If Python is available, uses `manifest_doctor.py` for extended validation

### doctor

```bash
bin/caelix doctor [--fix] [--strict-local]
```

Full system diagnostic.

**Options:**

| Option | Description |
|---|---|
| `--fix` | Attempt to automatically fix detected issues |
| `--strict-local` | Verify that health URLs target localhost |

**Checks performed:**

- Required keys present
- Valid port format
- Consistent rollout strategies (blue_green requires `candidate_publish`)
- Valid health, repair, autoscale metric types
- Write access to `CAELIX_DATA`
- Docker/Podman availability
- Presence of `notify.ini`

**Fix mode:**

```bash
bin/caelix doctor --fix
```

- Backs up the original as `.bak`
- Fixes automatically detectable issues
- Rewrites the corrected manifest

### show

```bash
bin/caelix show
```

Displays the loaded manifest, section by section. Useful to verify how Caelix interprets the configuration.

### version

```bash
bin/caelix version
```

Displays the version from the `VERSION` file.

### status

```bash
bin/caelix status
```

Displays the daemon state:

- Last heartbeat
- Uptime since last start
- Detected runtime (Docker/Podman)
- Configured paths (`CAELIX_MANIFEST`, `CAELIX_DATA`)
- Number of services in the manifest

### resume

```bash
bin/caelix resume <service-name>
```

Resumes reconciliation for a suspended or manually paused service.

Removes the files:
- `.caelix/state/<app>.suspend_reconcile`
- `.caelix/state/<app>.manual_pause`

```bash
# Example: resume a service suspended after creation failures
bin/caelix resume api
```
