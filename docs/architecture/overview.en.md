# Architecture Overview

## Design Principles

| Principle | Description |
|---|---|
| **Single-node** | No multi-machine coordination. Designed to orchestrate containers on a single host. |
| **Declarative** | The desired state is defined in an INI file. The engine converges toward that state on each cycle. |
| **Self-healing** | Automatic repair through escalation (restart → recreate → purge) without intervention. |
| **Minimal** | Dependencies: Bash 5, curl, Docker or Podman. No additional runtime required. |
| **Observable** | Audit trail, incident journal, Discord alerts, Prometheus metrics. |

---

## Components

### Reconciliation Pipeline

```mermaid
graph LR
    A["manifest.ini"] --> B["manifest_load()"]
    B --> C["reconcile_app()"]
    C --> D["deep_diagnose_name()"]
    D -->|healthy| E["OK"]
    D -->|failure| F["repair_execute()"]
    F --> G["Docker / Podman"]
```

### Autoscale Pipeline

```mermaid
graph LR
    A["autoscale_reconcile()"] --> B["collect_metric()"]
    B --> C["autoscale_decision()"]
    C -->|up| D["scale_up()"]
    C -->|down| E["scale_down()"]
    D --> F["write_backends + proxy reload"]
    E --> F
```

### Observability

```mermaid
graph LR
    A["Event"] --> B["incident_record()"]
    B --> C["incidents.log"]
    B --> D["YYYY-MM-DD.jsonl"]
    B --> E["Discord webhook"]
    A --> F["caelix_audit_event()"]
    F --> G["JSONL / SQLite"]
```

---

## Main Data Flow

```mermaid
sequenceDiagram
    participant M as manifest.ini
    participant S as bin/caelix run
    participant R as reconcile_app()
    participant H as deep_diagnose_name()
    participant D as Docker
    participant N as Discord
    participant I as incidents.log

    loop Every N seconds
        S->>M: manifest_load()
        S->>S: Detect runtime (docker/podman)
        S->>R: For each app in MNF_APPS_ORDER
        R->>D: container_exists() ?
        alt Does not exist
            R->>D: container_create()
            R->>I: incident_record(info, created)
        else Exists
            R->>D: container_running() ?
            alt Running
                R->>H: deep_diagnose_name()
                alt Healthy
                    H-->>R: return 0 (OK)
                else Unhealthy
                    H-->>R: return 1 (reason)
                    R->>D: repair_execute() (restart/recreate/purge)
                    R->>I: incident_record(severity, event)
                    R->>N: notify_all() → Discord embed
                end
            else Stopped
                R->>D: container_start()
            end
        end
        S->>S: remove_orphan_containers()
        S->>S: caelix_daemon_heartbeat()
        S->>S: sleep CAELIX_INTERVAL
    end
```

---

## Inter-Module Communication

```mermaid
graph LR
    manifest["manifest.sh<br>MNF[] global array"] --> health
    manifest --> repair
    manifest --> autoscale
    manifest --> runtime
    manifest --> proxy
    manifest --> doctor

    common["common.sh<br>Logging, state, ports"] --> health
    common --> repair
    common --> autoscale
    common --> runtime

    health["health.sh"] -->|diagnostic| repair["repair.sh"]
    repair -->|docker ops| runtime["runtime.sh"]
    repair -->|alerts| notify["notify.sh"]
    repair -->|journal| incidents["incidents.sh"]
    repair -->|trail| audit["audit.sh"]

    autoscale["autoscale.sh"] -->|replicas| runtime
    autoscale -->|backends| proxy["proxy.sh"]
    autoscale -->|journal| incidents

    audit -->|python| auditpy["audit_log.py"]
    doctor["doctor.sh"] -->|python| doctorpy["manifest_doctor.py"]

    style manifest fill:#e67e22,color:#fff
    style common fill:#95a5a6,color:#fff
    style health fill:#e74c3c,color:#fff
    style repair fill:#3498db,color:#fff
    style autoscale fill:#9b59b6,color:#fff
    style proxy fill:#2ecc71,color:#fff
    style notify fill:#f39c12,color:#fff
    style incidents fill:#8e44ad,color:#fff
    style audit fill:#1abc9c,color:#fff
```

---

## Naming Conventions

### Containers

| Type | Format | Example |
|---|---|---|
| Standard service | `caelix-<app>` | `caelix-web` |
| Blue/green candidate | `caelix-<app>-candidate-<timestamp>` | `caelix-web-candidate-1705312200` |
| Autoscale replica | `caelix-<app>-r<N>` | `caelix-web-r3` |
| Load balancer (legacy) | `caelix-<app>-lb` | `caelix-web-lb` |

### Docker Labels

| Label | Value | Usage |
|---|---|---|
| `caelix.app` | Service name | Identification |
| `caelix.config_version` | Config version | Change detection |
| `caelix.role` | `replica` | Autoscale replica distinction |
| `caelix.replica` | Number (1, 2, 3...) | Replica index |

### State Files

| Pattern | Example | Content |
|---|---|---|
| `.caelix/state/<app>.fail` | `.caelix/state/web.fail` | Failure counter (integer) |
| `.caelix/state/<app>.manual_pause` | `.caelix/state/web.manual_pause` | Pause reason |
| `.caelix/state/<app>.suspend_reconcile` | `.caelix/state/web.suspend_reconcile` | Suspension flag |
| `.caelix/state/<app>.create_fail_streak` | `.caelix/state/web.create_fail_streak` | Consecutive failures |
| `.caelix/state/<app>.restart_count` | `.caelix/state/web.restart_count` | Last Docker restart count |
| `.caelix/state/<app>.http_errrate` | `.caelix/state/web.http_errrate` | `fail total` (sliding window) |
| `.caelix/state/<app>.autoscale_cooldown` | `.caelix/state/web.autoscale_cooldown` | Decision streak |
| `.caelix/autoscale/<app>.backends` | `.caelix/autoscale/web.backends` | `name host port` per line |
