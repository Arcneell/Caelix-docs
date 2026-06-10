# Manifest Configuration

The `etc/manifest.ini` file centralizes the declaration of services and the global orchestrator parameters.

## Format

The manifest uses the standard INI format:

```ini
# Comment
[section]
key = value
key_with_quotes = "value with spaces"
```

- Comments start with `#`
- Values can be quoted
- Duplicate keys within the same section generate an error

## Reserved Sections

These sections are not treated as services:

### [orchestrator]

Global daemon parameters.

```ini
[orchestrator]
interval = 10              # Reconciliation interval (seconds)
max_repair = 5             # Failure threshold before alert
remove_orphans = 1         # Remove unknown caelix-* containers
log_level = info           # debug, info, warn, error
audit_log_backend = jsonl  # jsonl or sqlite
audit_log_all = 0          # 1 = audit all caelix-* containers
```

### [proxy]

Global reverse proxy configuration.

```ini
[proxy]
listen = 0.0.0.0:8080          # Global proxy listen address
autoscale_port_range = 18500-18999  # Port range for replicas
health_interval = 3             # Proxy health check interval (sec)
connect_timeout = 5             # Backend connection timeout (sec)
```

### [notify] and [global]

Reserved for future use. Notification configuration is in `etc/notify.ini`.

## Service Definition

Each non-reserved section defines a service:

```ini
[mon-service]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
```

### Basic Parameters

| Key | Default | Description |
|---|---|---|
| `image` | **required** | Docker image (e.g., `nginx:latest`, `myapp:v1.2.0`) |
| `publish` | — | Exposed ports, CSV (e.g., `8080:80,443:443`) |
| `command` | — | Command to execute (overrides CMD) |
| `env` | — | Environment variables, semicolon-separated (e.g., `KEY=val;KEY2=val2`) |
| `volumes_bind` | — | Bind mounts, CSV (e.g., `/data:/app/data,/logs:/app/logs`) |
| `network` | `bridge` | Docker network |
| `config_version` | — | Increment to force container re-creation |

### Health Checks

| Key | Default | Description |
|---|---|---|
| `health_type` | `http` | Probe type: `http`, `https`, `tcp`, `none` |
| `health_url` | — | URL for HTTP/HTTPS probes |
| `health_tcp_port` | — | Port for TCP probes |
| `health_expect_codes` | `200,204` | Accepted HTTP codes (CSV) |
| `health_timeout` | `5` | Probe timeout (seconds) |
| `health_max_bytes` | `0` | Max response size (0 = unlimited) |

### Monitoring

| Key | Default | Description |
|---|---|---|
| `monitoring_types` | `all` | Active types (CSV or `all`/`none`) |
| `monitoring_log_tail` | `120` | Number of log lines to analyze |
| `monitoring_log_error_regex` | (built-in) | Regex to detect anomalies in logs |
| `monitoring_http_latency_max_ms` | `1200` | Max acceptable latency (ms) |
| `monitoring_http_error_rate_threshold_pct` | `30` | Error rate threshold (%) |
| `monitoring_http_error_rate_window` | `20` | Sliding window size |
| `monitoring_disk_usage_max_pct` | `90` | Max disk usage (%) |

Available monitoring types:

- `health` — HTTP/TCP probe
- `memory` — memory usage
- `oom` — OOM killed detection
- `restart` — unexpected restarts
- `logs` — log analysis by regex
- `http_latency` — response time
- `http_error_rate` — HTTP error rate
- `disk` — disk usage

### Memory

| Key | Default | Description |
|---|---|---|
| `memory_limit_mb` | — | Docker memory limit (`-m` flag) |
| `memory_soft_mb` | — | Warning threshold |
| `memory_hard_mb` | — | Critical threshold (triggers repair) |

### Rollout and Repair

| Key | Default | Description |
|---|---|---|
| `rollout_strategy` | `recreate` | `recreate` or `blue_green` |
| `candidate_publish` | — | Blue/green candidate port (required if blue_green) |
| `health_url_candidate` | — | Candidate health URL (if different) |
| `preflight_cmd` | — | Command to run in the candidate before switch |
| `repair_strategy` | `auto` | `auto`, `restart-only`, `recreate-only`, `purge-only` |
| `purge_on_escalation` | `0` | Remove named volumes during purge |
| `post_repair_grace` | — | Delay after repair before health check (sec) |
| `create_fail_max_attempts` | `0` | Max creation failures before suspension (0 = unlimited) |

### Behavior

| Key | Default | Description |
|---|---|---|
| `manual_stop_pause` | `1` | Do not restart after manual stop |
| `container_audit_log` | `0` | Enable audit for this service |

### Autoscale

| Key | Default | Description |
|---|---|---|
| `autoscale` | `0` | Enable autoscaling (`1` or `true`) |
| `autoscale_min` | `1` | Minimum number of replicas |
| `autoscale_max` | `5` | Maximum number of replicas |
| `autoscale_metric` | `http_latency` | Metrics CSV: `http_latency`, `memory`, `http_error_rate` |
| `autoscale_up_threshold` | `800` | Scale-up threshold (ms/MB/%) |
| `autoscale_down_threshold` | `200` | Scale-down threshold |
| `autoscale_cooldown` | `3` | Consecutive passes before action |
| `autoscale_container_port` | `8080` | Internal container port for replicas |
| `autoscale_port_base` | (auto) | Starting host port (auto-allocated if absent) |
| `autoscale_lb_publish` | — | Load balancer address (legacy mode) |
| `autoscale_health_path` | `/` | HTTP path for backend probes |
| `autoscale_route` | — | Global routing: `host:hostname`, `path:/prefix`, `port:N`, `default` |

## Complete Example

```ini
[orchestrator]
interval = 8
max_repair = 4
remove_orphans = 1
log_level = info
audit_log_backend = sqlite

[proxy]
listen = 0.0.0.0:8080
autoscale_port_range = 18500-18999
health_interval = 3

[web-app]
image = myapp:latest
publish = 127.0.0.1:3000:3000
health_type = http
health_url = http://127.0.0.1:3000/health
health_expect_codes = 200
config_version = 1
rollout_strategy = blue_green
candidate_publish = 127.0.0.1:3001:3000
memory_soft_mb = 256
memory_hard_mb = 512
monitoring_types = health,memory,oom,logs,http_latency
repair_strategy = auto
container_audit_log = 1

[api]
image = api:v2.0.0
autoscale = 1
autoscale_min = 2
autoscale_max = 6
autoscale_container_port = 8080
autoscale_metric = http_latency,memory
autoscale_up_threshold = 600
autoscale_down_threshold = 150
autoscale_cooldown = 3
autoscale_route = host:api.example.com
health_type = http
health_url = http://127.0.0.1:8080/health
monitoring_types = all

[redis]
image = redis:7-alpine
publish = 127.0.0.1:6379:6379
health_type = tcp
health_tcp_port = 6379
monitoring_types = health,memory,oom
repair_strategy = restart-only
```

## Validation

Always validate after modification:

```bash
# Quick validation
bin/caelix validate

# Full diagnostic (with fix suggestions)
bin/caelix doctor

# Automatic correction of detected issues
bin/caelix doctor --fix
```
