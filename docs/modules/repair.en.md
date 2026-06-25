# Repair & Rollout

The `repair.sh` module implements automatic repair through escalation and deployment strategies (recreate, blue/green).

---

## Repair Strategies

### Overview

```mermaid
graph TD
    subgraph auto["repair_strategy = auto"]
        A1["Phase 1: Restart"] -->|Failure| A2["Phase 2: Recreate"]
        A2 -->|Failure| A3["Phase 3: Purge"]
    end

    subgraph restart["repair_strategy = restart-only"]
        R1["Restart only"]
    end

    subgraph recreate["repair_strategy = recreate-only"]
        RC1["Remove + Create"]
    end

    subgraph purge["repair_strategy = purge-only"]
        P1["Remove + Volumes + Create"]
    end

    style auto fill:#2c3e50,color:#fff
    style restart fill:#3498db,color:#fff
    style recreate fill:#e67e22,color:#fff
    style purge fill:#e74c3c,color:#fff
```

### Phase 1: Restart

```bash
docker restart caelix-<app>
```

- Simple and fast
- Preserves volumes, config, and the container filesystem
- Sufficient for temporary crashes and mild memory leaks

### Phase 2: Recreate

```bash
docker rm -f caelix-<app>
docker run ... (full re-creation from the manifest)
```

- Regenerates the container from scratch using the image
- Applies the current manifest configuration (ports, env, volumes, labels...)
- Bind volumes are reattached
- Useful when the container filesystem is corrupted

### Phase 3: Purge

```bash
docker rm -f caelix-<app>
docker volume rm <volumes>     # if purge_on_escalation=1
docker run ... (full re-creation)
```

- Last resort
- Removes named volumes to start from a clean slate
- **Potential data loss** if volumes are not backed up

### Configuration

```ini
[mon-service]
repair_strategy = auto           # Full escalation (default)
repair_strategy = restart-only   # Restart only
repair_strategy = recreate-only  # Recreate only
repair_strategy = purge-only     # Purge only

purge_on_escalation = 1          # Remove volumes during purge (default: 0)
post_repair_grace = 5            # Seconds to wait after repair (default: 3)
```

### `repair_execute()` Logic

The phase is determined by the `fail_count` failure counter:

In `auto` mode, escalation follows the `fail_count`:

```mermaid
graph LR
    A["fail = 1"] --> B["restart"]
    C["fail = 2"] --> D["recreate"]
    E["fail >= 3"] --> F["purge"]
```

Each phase calls `caelix_audit_event()` and `incident_record()` after execution.

---

## Blue/Green Deployment

Blue/green deployment enables zero-downtime updates. A candidate container is created, validated, then switched over.

### Full Flow

```mermaid
sequenceDiagram
    participant M as Manifest
    participant S as Caelix
    participant Old as Old (port 3000)
    participant New as Candidate (port 3001)
    participant D as Docker

    Note over S: Image change detected

    S->>M: Read candidate_publish, preflight_cmd...
    S->>D: docker run (new image on port 3001)
    D-->>New: Container created

    S->>S: Wait candidate_grace (default: 2s)

    opt preflight_cmd defined
        S->>New: docker exec: preflight_cmd
        alt Exit code != 0
            S->>D: docker rm -f (candidate)
            S->>S: incident_record(critical, bluegreen_fail)
            Note over S: Old container remains active
        end
    end

    S->>New: deep_diagnose_name(health_url_candidate)

    alt Candidate HEALTHY
        S->>D: docker rename (old → temp)
        S->>D: docker rename (candidate → caelix-app)
        S->>D: docker rm -f (old/temp)
        S->>S: incident_record(info, bluegreen_switch)
        S->>S: reset_fail_count()
        Note over S: Switchover successful!
    else Candidate UNHEALTHY
        S->>D: docker rm -f (candidate)
        S->>S: incident_record(critical, bluegreen_fail)
        Note over S: Old container remains active
    end
```

### Complete Configuration

```ini
[mon-service]
image = myapp:v2.0.0                                      # New image
rollout_strategy = blue_green                              # Enable blue/green
publish = 127.0.0.1:3000:3000                              # Production port
candidate_publish = 127.0.0.1:3001:3000                    # Candidate temporary port (REQUIRED)
health_url = http://127.0.0.1:3000/health                  # Production health
health_url_candidate = http://127.0.0.1:3001/health        # Candidate health (optional)
preflight_cmd = python manage.py migrate                    # Pre-switch command (optional)
```

!!! warning "candidate_publish is required"
    If `rollout_strategy = blue_green` but `candidate_publish` is not defined, `bin/caelix doctor` will report an error.

---

## Manual Pause

### Problem Solved

Without manual pause, an operator running `docker stop caelix-web` would see Caelix restart the service immediately on the next cycle.

### Operation

```mermaid
graph LR
    A["docker stop"] --> B{"exit code"}
    B -->|"0 / 137 / 143"| C["set_manual_pause()"]
    B -->|other| D["normal repair"]
    C --> E["skip reconciliation"]
    E --> F["caelix resume → resumes"]
```

### Configuration

```ini
[mon-service]
manual_stop_pause = 1   # Enabled by default
manual_stop_pause = 0   # Disable: always restart
```

### Involved Functions

| Function | Description |
|---|---|
| `manual_pause_state_path(app)` | Flag file path |
| `is_manual_pause_active(app)` | Is the service paused? |
| `set_manual_pause(app, reason)` | Enable pause |
| `clear_manual_pause(app)` | Disable pause |
| `manual_stop_pause_enabled(app)` | Is pause configured? |
| `exit_code_looks_like_manual_stop(code)` | Exit code = manual stop? |

---

## Config Version

To force container re-creation without changing the image:

```ini
[mon-service]
config_version = 2   # Increment this value
```

Caelix compares the `caelix.config_version` label of the container with the manifest value. If they differ, the container is recreated (or deployed via blue/green depending on the strategy).

The `desired_config_version()` and `current_config_version()` functions handle this comparison.

---

## Automatic Suspension

```ini
[mon-service]
create_fail_max_attempts = 3   # 0 = unlimited (default)
```

```mermaid
graph LR
    F1["Creation failure 1"] --> F2["Creation failure 2"]
    F2 --> F3["Creation failure 3"]
    F3 --> S["SUSPENDED<br>.suspend_reconcile created"]
    S --> N["Critical notification"]
    S --> R["bin/caelix resume app"]
    R --> OK["Reconciliation resumes"]

    style S fill:#e74c3c,color:#fff
    style OK fill:#27ae60,color:#fff
```

The `caelix_clear_suspend_state()` function removes suspension files and resets the counter to zero.

---

## repair.sh Module Functions

| Function | Description |
|---|---|
| `reconcile_app(app)` | Main entry point for per-service reconciliation |
| `ensure_desired_revision(app)` | Check image and config_version, trigger rollout if needed |
| `rollout_blue_green(app)` | Complete blue/green deployment |
| `repair_execute(app, reason)` | Repair escalation (restart → recreate → purge) |
| `candidate_preflight(app, cname)` | Execute preflight command in the candidate |
| `create_candidate_name(app)` | Generate candidate container name |
| `detect_unexpected_restart(app)` | Compare restart count with saved state |
| `remove_orphan_containers()` | Remove undeclared caelix-* containers |
| `caelix_section_reserved(section)` | Check if a section is reserved |
| `is_manual_pause_active(app)` | Check manual pause |
| `set_manual_pause(app, reason)` | Enable pause |
| `clear_manual_pause(app)` | Disable pause |
