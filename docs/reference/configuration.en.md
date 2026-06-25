# Complete Configuration Reference

Index of all configuration keys available in Caelix.

## etc/manifest.ini

### [orchestrator] Section

| Key | Type | Default | Description |
|---|---|---|---|
| `interval` | int | `15` | Reconciliation interval (seconds) |
| `max_repair` | int | `5` | Failure threshold before critical alert |
| `remove_orphans` | bool | `0` | Remove undeclared `caelix-*` containers |
| `log_level` | string | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `audit_log_backend` | string | `jsonl` | Audit backend: `jsonl` or `sqlite` |
| `audit_log_all` | bool | `0` | Audit all `caelix-*` containers |

### [proxy] Section

| Key | Type | Default | Description |
|---|---|---|---|
| `listen` | string | — | Global proxy listen address (e.g., `0.0.0.0:8080`) |
| `autoscale_port_range` | string | — | Replica port range (e.g., `18500-18999`) |
| `health_interval` | int | `3` | Backend health check interval (seconds) |
| `connect_timeout` | int | `5` | Backend connection timeout (seconds) |

### Service Section (per application)

#### Basic

| Key | Type | Default | Description |
|---|---|---|---|
| `image` | string | **required** | Docker image |
| `publish` | string | — | Ports CSV (e.g., `8080:80,443:443`) |
| `command` | string | — | Override CMD |
| `env` | string | — | Environment variables, `;`-separated (e.g., `KEY=val;KEY2=val2`) |
| `volumes_bind` | string | — | Bind mounts CSV (e.g., `/data:/app/data`) |
| `network` | string | `bridge` | Docker network |
| `config_version` | string | — | Increment to force re-creation |

#### Health Checks

| Key | Type | Default | Description |
|---|---|---|---|
| `health_type` | string | `http` | `http`, `https`, `tcp`, `none` |
| `health_url` | string | — | HTTP/HTTPS probe URL |
| `health_tcp_port` | int | — | TCP probe port |
| `health_expect_codes` | string | `200,204` | Accepted HTTP codes (CSV) |
| `health_timeout` | int | `5` | Probe timeout (seconds) |
| `health_max_bytes` | int | `0` | Max response size (0 = unlimited) |

#### Monitoring

| Key | Type | Default | Description |
|---|---|---|---|
| `monitoring_types` | string | `all` | Active types (CSV, `all`, `none`) |
| `monitoring_log_tail` | int | `120` | Log lines to analyze |
| `monitoring_log_error_regex` | string | (built-in) | Log anomaly regex |
| `monitoring_http_latency_max_ms` | int | `1200` | Max latency (ms) |
| `monitoring_http_error_rate_threshold_pct` | int | `30` | Error rate threshold (%) |
| `monitoring_http_error_rate_window` | int | `20` | Sliding window (requests) |
| `monitoring_disk_usage_max_pct` | int | `90` | Max disk usage (%) |

#### Memory

| Key | Type | Default | Description |
|---|---|---|---|
| `memory_limit_mb` | int | — | Docker limit (`-m`) |
| `memory_soft_mb` | int | — | Warning threshold |
| `memory_hard_mb` | int | — | Critical threshold |

#### Rollout & Repair

| Key | Type | Default | Description |
|---|---|---|---|
| `rollout_strategy` | string | `recreate` | `recreate` or `blue_green` |
| `candidate_publish` | string | — | Blue/green candidate port |
| `health_url_candidate` | string | — | Candidate health URL |
| `preflight_cmd` | string | — | Pre-switch command |
| `repair_strategy` | string | `auto` | `auto`, `restart-only`, `recreate-only`, `purge-only` |
| `purge_on_escalation` | bool | `0` | Remove volumes on purge |
| `post_repair_grace` | int | — | Post-repair delay (seconds) |
| `create_fail_max_attempts` | int | `0` | Max creation failures (0 = unlimited) |

#### Behavior

| Key | Type | Default | Description |
|---|---|---|---|
| `manual_stop_pause` | bool | `1` | Pause after manual stop |
| `container_audit_log` | bool | `0` | Enable audit for this service |

#### Automatic Updates

| Key | Type | Default | Description |
|---|---|---|---|
| `auto_update` | bool | `0` | Enable automatic image update checking |
| `auto_update_interval` | string | `86400` | Check interval (seconds, `hourly`, `daily`, `weekly`) |
| `auto_update_strategy` | string | `immediate` | `immediate` (apply automatically) or `notify` (alert only) |

#### Scheduled Backups

| Key | Type | Default | Description |
|---|---|---|---|
| `backup` | bool | `0` | Enable scheduled backups |
| `backup_schedule` | string | `86400` | Backup interval (seconds, `hourly`, `daily`, `weekly`) |
| `backup_target` | string | `local` | Target: `local`, `ftp`, `rsync`, `s3` |
| `backup_retention` | int | `7` | Number of backups to keep |
| `backup_ftp_host` | string | — | FTP host |
| `backup_ftp_user` | string | — | FTP username |
| `backup_ftp_pass` | string | — | FTP password |
| `backup_ftp_path` | string | `/backups` | FTP remote path |
| `backup_rsync_dest` | string | — | rsync destination |
| `backup_rsync_pass` | string | — | rsync password |
| `backup_rsync_opts` | string | `-az` | rsync options |
| `backup_s3_bucket` | string | — | S3 bucket |
| `backup_s3_prefix` | string | `/backups` | S3 prefix |
| `backup_s3_endpoint` | string | — | S3 endpoint (MinIO, etc.) |
| `backup_s3_region` | string | `us-east-1` | AWS region |

#### Autoscale

| Key | Type | Default | Description |
|---|---|---|---|
| `autoscale` | bool | `0` | Enable autoscaling |
| `autoscale_min` | int | `1` | Minimum replicas |
| `autoscale_max` | int | `5` | Maximum replicas |
| `autoscale_metric` | string | `http_latency` | Metrics CSV |
| `autoscale_up_threshold` | int | `800` | Scale-up threshold |
| `autoscale_down_threshold` | int | `200` | Scale-down threshold |
| `autoscale_cooldown` | int | `3` | Passes before action |
| `autoscale_container_port` | int | `8080` | Internal replica port |
| `autoscale_port_base` | int | (auto) | Starting host port |
| `autoscale_lb_publish` | string | — | Legacy LB address |
| `autoscale_health_path` | string | `/` | Backend health path |
| `autoscale_route` | string | — | Global proxy route |

## etc/notify.ini

### [discord] Section

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `0` | Enable Discord notifications |
| `webhook_url` | string | — | Discord webhook URL |

### [slack] Section

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `0` | Enable Slack notifications |
| `webhook_url` | string | — | Slack webhook URL |

### [teams] Section

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `0` | Enable Teams notifications |
| `webhook_url` | string | — | Teams webhook URL |

### [telegram] Section

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `0` | Enable Telegram notifications |
| `bot_token` | string | — | Telegram bot token |
| `chat_id` | string | — | Telegram chat ID |

### [smtp] Section

| Key | Type | Default | Description |
|---|---|---|---|
| `enabled` | bool | `0` | Enable email notifications |
| `host` | string | — | SMTP server |
| `port` | int | `587` | SMTP port |
| `user` | string | — | SMTP username |
| `pass` | string | — | SMTP password |
| `from` | string | — | Sender address |
| `to` | string | — | Recipient address(es) |

## Environment Variables

### Engine (Bash)

| Variable | Default | Description |
|---|---|---|
| `CAELIX_MANIFEST` | `etc/manifest.ini` | Manifest path |
| `CAELIX_NOTIFY_CONF` | `etc/notify.ini` | Notification config path |
| `CAELIX_DATA` | `./.caelix` | Data directory |
| `CAELIX_INTERVAL` | `15` | Reconciliation interval (sec) |
| `CAELIX_MAX_REPAIR` | `5` | Failure threshold before alert |
| `CAELIX_LOG_LEVEL` | `info` | Log level |
| `CAELIX_STRICT_LOCAL` | `0` | Force localhost health URLs |
| `CAELIX_LANG` | `fr` | Orchestrator language (fr, en) |
| `CAELIX_HTTP_USER_AGENT` | `Caelix/1.0` | User-Agent for HTTP health checks |

### Web Console (FastAPI backend)

| Variable | Default | Description |
|---|---|---|
| `PORT` | `8080` | Backend listen port |
| `CAELIX_UI_BIND` | `0.0.0.0` | Bind address |
| `CAELIX_ADMIN_PASSWORD` | `admin` | Initial admin password (only if no users exist) |
| `CAELIX_JWT_SECRET` | (auto) | JWT signing key; auto-generated and persisted to `.caelix/jwt_secret.key` if unset |
| `CAELIX_JWT_EXPIRE_MINUTES` | `480` | JWT token / session cookie lifetime (minutes) |
| `CAELIX_CORS_ORIGINS` | — | Allowed CORS origins (CSV). Empty = same-origin only (recommended) |
| `CAELIX_METRICS_PROTECT` | `0` | Require authentication on `/metrics` |
| `CAELIX_UI_TLS_CERT` | — | TLS certificate path (HTTPS) |
| `CAELIX_UI_TLS_KEY` | — | TLS key path (HTTPS) |
| `CAELIX_UI_TOKEN` | — | Legacy shared token. **Deprecated**: migrate to user accounts |
