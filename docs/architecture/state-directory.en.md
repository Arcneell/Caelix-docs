# State Directory (.caelix/)

The `.caelix/` directory contains all Caelix runtime data. It is automatically created on first launch.

## Structure

```
.caelix/
├── state/                          # Current service state
│   ├── caelix-daemon-heartbeat       # Timestamp of last cycle
│   ├── <app>.fail                  # Health check failure counter
│   ├── <app>.restart_count         # Last Docker restart count
│   ├── <app>.create_fail_streak    # Consecutive creation failures
│   ├── <app>.suspend_reconcile     # Suspension flag
│   ├── <app>.manual_pause          # Manual pause flag
│   ├── <app>.autoscale_cooldown    # Autoscale streak
│   ├── <app>.http_errrate          # HTTP error rate (fail total)
│   └── manifest_load_warn.notified # Manifest alert flag
│
├── incidents/                      # Incident journal
│   ├── incidents.log               # Text log (append-only)
│   └── YYYY-MM-DD.jsonl            # Daily JSONL archive
│
├── archive/                        # Rotated archives
│   └── incidents-YYYY-MM-DD.log    # Old incident logs
│
├── audit/                          # Audit trail
│   ├── container-audit.jsonl       # JSONL backend (default)
│   └── container-audit.sqlite      # SQLite backend (alternative)
│
├── autoscale/                      # Autoscale data
│   ├── port_allocations            # Replica port allocations
│   ├── <app>.backends              # Backend list for the proxy
│   ├── global-proxy.pid            # Global proxy PID
│   ├── routes.conf                 # Global routing table
│   └── global-proxy/
│       └── health_<replica>        # Per-replica health status (0/1)
│
├── logs/                           # Log files
│   ├── caelix-daemon.log             # Current daemon log (JSON lines)
│   ├── caelix-daemon.YYYY-MM-DD.log  # Rotated logs
│   └── caelix-ui.log                 # UI backend log
│
├── stacks/                         # Docker Compose data
│   └── <stack>/                    # Per stack
│
├── node/                           # Node identity (agent mode, multi-node)
│   └── id                          # Generated, persisted node id
│
├── agent/                          # Status published by the agent (multi-node)
│   └── status.json                 # Node meta + observed state (read by the controller)
│
├── notifications.json              # Notification buffer (200 max)
├── webhooks.json                   # Deployment webhooks
└── registries.json                 # Docker registry credentials
```

## State Files in Detail

### caelix-daemon-heartbeat

Contains the Unix timestamp of the last completed reconciliation cycle. Used by the web console to verify that the daemon is active.

### \<app\>.fail

Consecutive health check failure counter. Incremented on each failure, reset to zero when the service becomes healthy again. When this counter reaches `CAELIX_MAX_REPAIR`, an alert is sent.

### \<app\>.manual_pause

Automatically created when Caelix detects a manual stop (exit codes 0, 137, 143). As long as this file exists, reconciliation ignores this service. Removed by `bin/caelix resume <app>`.

### \<app\>.suspend_reconcile

Created when the number of creation failures exceeds `create_fail_max_attempts`. Prevents any reconciliation attempt until an explicit `resume`.

### \<app\>.http_errrate

Contains two values: the number of failures and the total number of requests in the sliding window. Used to calculate the HTTP error rate.

## Log Rotation

- **Daemon logs**: daily rotation, 10 MB threshold, 30-day retention
- **Incidents**: daily archival to `.caelix/archive/`
- **Notifications**: circular buffer of 200 entries

## Customizing the Path

```bash
export CAELIX_DATA=/var/lib/caelix
bin/caelix run
```

By default, `CAELIX_DATA` is `./.caelix` (relative to the working directory).

!!! warning "Permissions"
    The `CAELIX_DATA` directory must be readable and writable by the user running Caelix.
