# Reconciliation Loop

The reconciliation loop continuously compares the desired state (manifest) with the actual state (Docker) and applies the necessary corrections on each cycle.

---

## Cycle Overview

```mermaid
graph TD
    start["Cycle start"] --> load["1. Load manifest.ini<br>manifest_load()"]
    load -->|Failure| tryrepair["manifest_try_repair()<br>Auto-repair"]
    tryrepair -->|Failure| skip["Abort this cycle"]
    tryrepair -->|OK| globals
    load -->|OK| globals["2. Apply globals<br>[orchestrator] → interval, max_repair, log_level"]
    globals --> detect["3. Detect runtime<br>runtime_engine() → docker or podman"]
    detect --> proxy{"4. [proxy] section<br>defined?"}
    proxy -->|Yes| proxyensure["global_proxy_ensure()<br>Start/verify the proxy"]
    proxy -->|No| apps
    proxyensure --> apps["5. For each app<br>in MNF_APPS_ORDER"]
    apps --> reconcile["reconcile_app(app)"]
    reconcile --> next{"More apps?"}
    next -->|Yes| apps
    next -->|No| orphans{"6. remove_orphans = 1?"}
    orphans -->|Yes| clean["remove_orphan_containers()<br>Remove unknown caelix-*"]
    orphans -->|No| heartbeat
    clean --> heartbeat["7. caelix_daemon_heartbeat()<br>Write timestamp"]
    heartbeat --> sleep["8. sleep CAELIX_INTERVAL"]
    sleep --> start

    style start fill:#1abc9c,color:#fff
    style skip fill:#e74c3c,color:#fff
    style reconcile fill:#3498db,color:#fff
    style heartbeat fill:#2ecc71,color:#fff
```

### Cycle Parameters

```ini
[orchestrator]
interval = 10       # Seconds between each cycle (default: 15)
max_repair = 5      # Failures before critical alert (default: 5)
remove_orphans = 1  # Remove undeclared caelix-* containers
log_level = info    # debug, info, warn, error
```

!!! info "Heartbeat"
    The file `.caelix/state/caelix-daemon-heartbeat` contains the timestamp of the last completed cycle. The web console uses it to display whether the daemon is active.

---

## Per-Application Reconciliation

The `reconcile_app()` function is called for each service. Here is its complete logic:

### Pre-checks

```mermaid
graph LR
    A["reconcile_app"] --> B{"autoscale?"}
    B -->|yes| C["autoscale_reconcile → return"]
    B -->|no| D{"manual pause?"}
    D -->|yes| E["skip"]
    D -->|no| F{"suspended?"}
    F -->|yes| G["skip + alert"]
    F -->|no| H["continue"]
```

### Convergence

```mermaid
graph LR
    A{"container exists?"} -->|no| B["container_create()"]
    A -->|yes| C{"image/config changed?"}
    C -->|yes| D["rollout blue_green or recreate"]
    C -->|no| E{"running?"}
    E -->|no| F["container_start()"]
    E -->|yes| G["deep_diagnose()"]
```

### Repair

```mermaid
graph LR
    A["deep_diagnose()"] -->|healthy| B["reset_fail_count"]
    A -->|failure| C["repair_execute()"]
    C --> D["grace period"]
    D --> E["re-check"]
    E -->|healthy| B
    E -->|failure| F["inc_fail_count"]
    F -->|">= max_repair"| G["critical alert"]
```

---

## Revision Detection

The `ensure_desired_revision()` function checks two things:

1. Does the container image match the one in the manifest?
2. Does the `caelix.config_version` label match the manifest?

```mermaid
flowchart LR
    A["Current image<br>vs manifest"] --> B{"Match?"}
    C["config_version<br>label vs manifest"] --> B
    B -->|Yes| D["No change"]
    B -->|No| E{"rollout_strategy"}
    E -->|recreate| F["Remove + Create"]
    E -->|blue_green| G["rollout_blue_green()"]
```

Image comparison normalizes names before comparing: `nginx` is equivalent to `docker.io/library/nginx:latest`.

---

## Repair Strategies

### Automatic Escalation (`repair_strategy = auto`)

The `.fail` counter determines the phase:

| fail_count | Phase | Action |
|---|---|---|
| 1 | **restart** | `docker restart caelix-<app>` |
| 2 | **recreate** | `docker rm -f` + `docker run` |
| 3+ | **purge** | Remove + volume deletion + `docker run` |

```mermaid
graph LR
    F1["fail = 1"] --> R["Restart<br>docker restart"]
    F2["fail = 2"] --> RC["Recreate<br>rm -f + run"]
    F3["fail >= 3"] --> P["Purge<br>rm -f + volume rm + run"]

    R -->|Failure| F2
    RC -->|Failure| F3
    P -->|Failure| Alert["Critical alert<br>+ Discord notification"]

    R -->|OK| Reset["fail = 0"]
    RC -->|OK| Reset
    P -->|OK| Reset

    style F1 fill:#f39c12,color:#fff
    style F2 fill:#e67e22,color:#fff
    style F3 fill:#e74c3c,color:#fff
    style Reset fill:#27ae60,color:#fff
    style Alert fill:#c0392b,color:#fff
```

### Specific Strategies

```ini
repair_strategy = auto           # Full escalation (default)
repair_strategy = restart-only   # Restart only, no escalation
repair_strategy = recreate-only  # Remove + create only
repair_strategy = purge-only     # Full purge only
```

### Grace Period

```ini
post_repair_grace = 5  # Seconds to wait after repair before re-verification
```

---

## Blue/Green Deployment

```mermaid
sequenceDiagram
    participant S as Caelix
    participant Old as Old container<br>(port 3000)
    participant New as Candidate<br>(port 3001)
    participant D as Docker

    S->>D: Create candidate on candidate_publish (3001)
    S->>S: Wait candidate_grace (2s)

    opt preflight_cmd defined
        S->>New: Run preflight_cmd<br>(e.g., python manage.py migrate)
        New-->>S: Exit code
        alt Preflight failure
            S->>D: Remove candidate
            S-->>S: incident_record(critical, bluegreen_fail)
        end
    end

    S->>New: deep_diagnose_name() via health_url_candidate
    alt Candidate healthy
        S->>D: Rename old → temp
        S->>D: Rename candidate → caelix-app
        S->>D: Remove old
        S-->>S: incident_record(info, bluegreen_switch)
    else Candidate unhealthy
        S->>D: Remove candidate
        S-->>S: incident_record(critical, bluegreen_fail)
        Note over S: Old container remains active
    end
```

Required configuration:

```ini
[mon-service]
rollout_strategy = blue_green
publish = 127.0.0.1:3000:3000
candidate_publish = 127.0.0.1:3001:3000
health_url_candidate = http://127.0.0.1:3001/health  # optional
preflight_cmd = python manage.py migrate               # optional
```

---

## Loop Protection

### Automatic Suspension

```ini
create_fail_max_attempts = 3  # Suspend after 3 creation failures
```

When creation fails N consecutive times:

1. The file `.caelix/state/<app>.suspend_reconcile` is created
2. Caelix stops touching this service
3. A critical alert is sent

To resume: `bin/caelix resume <app>`

### Manual Pause

When an operator manually stops a container (`docker stop`), Caelix detects clean shutdown exit codes:

| Exit code | Meaning |
|---|---|
| `0` | Normal shutdown |
| `137` | SIGKILL |
| `143` | SIGTERM |

Caelix creates `.caelix/state/<app>.manual_pause` and will not restart the container.

```ini
manual_stop_pause = 1  # Enabled by default
```

---

## Orphan Cleanup

When `remove_orphans = 1`, Caelix removes containers that:

- Have a name starting with `caelix-`
- Do not match any manifest section
- Are not replicas (`-r<N>`) or LB (`-lb`) of an existing service

Associated state files are also cleaned up.

---

## Execution Modes

| Command | Usage | When to use |
|---|---|---|
| `bin/caelix run` | Infinite loop | Production (daemon, systemd) |
| `bin/caelix once` | Single cycle | Tests, cron, first launch |
| `bin/caelix reconcile-app <app>` | Single service | Targeted debugging |
