# Troubleshooting

Guide for resolving common issues.

## Quick Diagnostic

```bash
# Check overall state
bin/caelix status

# Validate configuration
bin/caelix validate

# Full diagnostic
bin/caelix doctor

# View daemon logs
tail -50 .caelix/logs/caelix-daemon.log | python3 -m json.tool

# Latest incidents
tail -20 .caelix/incidents/incidents.log
```

## Common Issues

### The daemon does not start

**Symptom**: `bin/caelix run` fails immediately.

**Checks**:

1. Is Docker accessible?
   ```bash
   docker info
   ```

2. Is the manifest valid?
   ```bash
   bin/caelix validate
   ```

3. Is the CAELIX_DATA directory writable?
   ```bash
   ls -la .caelix/
   ```

### A service does not start

**Symptom**: the container is never created.

**Checks**:

1. Does the image exist?
   ```bash
   docker pull <image>
   ```

2. Is the service suspended?
   ```bash
   ls .caelix/state/<app>.suspend_reconcile
   # If the file exists:
   bin/caelix resume <app>
   ```

3. Is the service manually paused?
   ```bash
   ls .caelix/state/<app>.manual_pause
   # If the file exists:
   bin/caelix resume <app>
   ```

### Health checks keep failing

**Symptom**: the `.fail` counter keeps increasing.

**Checks**:

1. Is the health URL accessible?
   ```bash
   curl -v <health_url>
   ```

2. Is the port correctly exposed?
   ```bash
   docker port caelix-<app>
   ```

3. Does the service take time to start? Increase the grace period:
   ```ini
   post_repair_grace = 10
   ```

4. In strict mode, does the URL target localhost?
   ```bash
   CAELIX_STRICT_LOCAL=1 bin/caelix doctor
   ```

### The service keeps restarting

**Symptom**: constant restarts, notification flood.

**Possible causes**:

- **OOM**: the container is killed due to out of memory
  ```bash
  docker inspect caelix-<app> | grep OOMKilled
  ```
  Solution: increase `memory_limit_mb`

- **Crash at startup**: the application crashes immediately
  ```bash
  docker logs caelix-<app>
  ```

- **Port in use**: the port is already used by another process
  ```bash
  ss -tlnp | grep <port>
  ```

### Discord notifications are not working

**Checks**:

1. Is the webhook configured?
   ```bash
   cat etc/notify.ini
   ```

2. Is the webhook valid?
   ```bash
   curl -X POST <webhook_url> \
     -H "Content-Type: application/json" \
     -d '{"content": "Test Caelix"}'
   ```

3. Is `enabled` set to `1`?

### Autoscale is not working

**Checks**:

1. Is `autoscale = 1` defined?
2. Is `socat` installed?
   ```bash
   which socat
   ```

3. Is the port range free?
   ```bash
   ss -tlnp | grep -E '185[0-9]{2}'
   ```

4. Check the backends:
   ```bash
   cat .caelix/autoscale/<app>.backends
   ```

### The web console does not connect

**Checks**:

1. Is the UI container running?
   ```bash
   docker ps | grep caelix-ui
   ```

2. Is the Docker socket mounted?
   ```bash
   docker inspect caelix-ui | grep docker.sock
   ```

3. Is the auth token correct (if enabled)?

## Logs

### Daemon Logs (JSON lines)

```bash
# Latest logs
tail -20 .caelix/logs/caelix-daemon.log

# Format as readable JSON
tail -5 .caelix/logs/caelix-daemon.log | python3 -m json.tool

# Filter by level
grep '"level":"error"' .caelix/logs/caelix-daemon.log
```

### Container Logs

```bash
docker logs caelix-<app> --tail 50
docker logs caelix-<app> -f  # real-time follow
```

### Incidents

```bash
# Text
cat .caelix/incidents/incidents.log

# JSONL (with jq)
cat .caelix/incidents/$(date +%Y-%m-%d).jsonl | jq .
```

## Debug Mode

For more detail in logs:

```bash
CAELIX_LOG_LEVEL=debug bin/caelix once
```

Or in the manifest:

```ini
[orchestrator]
log_level = debug
```

## Full Reset

!!! danger "Warning"
    This operation deletes all Caelix state. Containers are not affected.

```bash
# Delete all state
rm -rf .caelix/

# Relaunch
bin/caelix once
```

Caelix will recreate the `.caelix/` directory and reconverge toward the desired state.
