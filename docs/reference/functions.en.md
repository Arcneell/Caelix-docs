# Internal Functions

Complete reference of the 173 functions in the Caelix engine, organized by module.

---

## common.sh — Logging, State, Ports (23 functions)

### Logging and Heartbeat

| Function | Signature | Description |
|---|---|---|
| `caelix_log` | `(level, ...message)` | Log to stderr + JSON file. Levels: debug, info, warn, error |
| `_caelix_ts` | `()` | UTC ISO timestamp (YYYY-MM-DDTHH:MM:SSZ) |
| `caelix_daemon_heartbeat` | `()` | Write timestamp to `.caelix/state/caelix-daemon-heartbeat` |
| `_caelix_log_rotate` | `()` | Daemon log rotation (10 MB, 30 files max) |
| `caelix_version_string` | `(root)` | Version from the VERSION file |

### Failure Counters

| Function | Signature | Description |
|---|---|---|
| `fail_count_path` | `(app)` | Path to `.caelix/state/<app>.fail` |
| `get_fail_count` | `(app)` | Read the failure counter |
| `set_fail_count` | `(app, count)` | Write the counter |
| `inc_fail_count` | `(app)` | Increment by 1 |
| `reset_fail_count` | `(app)` | Reset to 0 + remove notification flag |

### Creation Streak

| Function | Signature | Description |
|---|---|---|
| `create_fail_streak_path` | `(app)` | Path to `.caelix/state/<app>.create_fail_streak` |
| `caelix_bump_create_fail_streak` | `(app)` | Increment the consecutive failure counter |
| `caelix_get_create_fail_streak` | `(app)` | Read the counter |
| `caelix_clear_create_fail_streak` | `(app)` | Reset to zero |

### Suspension

| Function | Signature | Description |
|---|---|---|
| `suspend_reconcile_path` | `(app)` | Path to the suspension flag |
| `suspend_reconcile_notified_path` | `(app)` | Path to the notification flag |
| `caelix_clear_suspend_state` | `(app)` | Remove suspension + streak |

### Port Allocation

| Function | Signature | Description |
|---|---|---|
| `port_alloc_file` | `()` | Path to `port_allocations` |
| `port_alloc_init` | `()` | Initialize the file |
| `port_alloc_get` | `(app, idx)` | Retrieve an existing port |
| `port_alloc_next` | `(app, idx)` | Allocate the next free port (flock) |
| `port_alloc_release` | `(app)` | Release all ports for a service |
| `port_alloc_release_one` | `(app, idx)` | Release a single port |

---

## manifest.sh — INI Parser (4 functions)

| Function | Signature | Description |
|---|---|---|
| `manifest_clear` | `()` | Clear MNF[] and MNF_APPS_ORDER |
| `manifest_load` | `(path)` | Parse the INI file → MNF["section\|key"] |
| `manifest_get` | `(section, key)` | Read a value (error if missing) |
| `manifest_get_default` | `(section, key, default)` | Read with fallback |

**Global variables:**

- `MNF`: associative array `MNF["section|key"] = "value"`
- `MNF_APPS_ORDER`: array of section names in file order

---

## runtime.sh — Docker/Podman Operations (18 functions)

### Detection and Wrapper

| Function | Signature | Description |
|---|---|---|
| `runtime_engine` | `()` | Detect docker or podman in PATH |
| `_rt` | `(...args)` | Wrapper for $RT (docker/podman) |
| `caelix_cname` | `(app)` | Returns `caelix-<app>` |

### Inspection

| Function | Signature | Description |
|---|---|---|
| `container_id_by_name` | `(name)` | Container ID by name |
| `container_exists` | `(app)` | Does the container exist? |
| `container_running` | `(app)` | Is the container running? |
| `container_image` | `(app)` | Current container image |
| `container_label` | `(app, key)` | Docker label value |
| `image_matches` | `(want, got)` | Smart image comparison |
| `inspect_restart_count` | `(cid)` | Docker restart count |
| `inspect_oom_killed` | `(cid)` | OOM Killed? (true/false) |
| `inspect_state_status` | `(cid)` | Status (running, exited...) |
| `inspect_exit_code` | `(cid)` | Exit code |

### Lifecycle

| Function | Signature | Description |
|---|---|---|
| `container_create` | `(app, [publish], [name])` | Create a container with full manifest config |
| `container_remove_force` | `(app)` | `docker rm -f` |
| `container_restart` | `(app)` | `docker restart` |
| `container_start` | `(app)` | `docker start` |
| `container_exec` | `(cname, ...args)` | Execute a command in the container |

---

## health.sh — Monitoring (16 functions)

| Function | Signature | Description |
|---|---|---|
| `monitoring_enabled` | `(app, type)` | Is this monitoring type active for this service? |
| `health_tcp` | `(host, port, [timeout])` | TCP probe (nc or /dev/tcp) |
| `health_http` | `(url, [timeout], [expect], [max_bytes])` | HTTP probe with full validation |
| `caelix_health_url_is_local` | `(url)` | Does URL target localhost? |
| `container_memory_usage_mb` | `(name)` | Memory usage in MB |
| `container_recent_logs` | `(name, [tail])` | Last N log lines |
| `container_resource_snapshot` | `(name)` | CPU + memory snapshot |
| `http_total_time_ms` | `(url, [timeout])` | HTTP response time in ms |
| `http_probe_code` | `(url, [timeout])` | HTTP code only |
| `http_error_rate_state_path` | `(app)` | Error rate file path |
| `http_error_rate_state_get` | `(app)` | Read fail and total |
| `http_error_rate_state_set` | `(app, fail, total)` | Write fail and total |
| `disk_usage_pct_for_path` | `(path)` | Disk usage in % |
| `check_disk_usage_limit` | `(app)` | Check all volumes |
| `deep_diagnose_name` | `(app, name)` | Full diagnostic (8 types) |
| `deep_diagnose` | `(app)` | Wrapper with standard name |

---

## repair.sh — Reconciliation and Repair (24 functions)

### Reconciliation

| Function | Signature | Description |
|---|---|---|
| `reconcile_app` | `(app)` | Per-service reconciliation entry point |
| `ensure_desired_revision` | `(app)` | Check image and config_version |
| `remove_orphan_containers` | `()` | Remove undeclared caelix-* containers |

### Repair

| Function | Signature | Description |
|---|---|---|
| `repair_execute` | `(app, reason)` | Escalation: restart → recreate → purge |

### Blue/Green

| Function | Signature | Description |
|---|---|---|
| `rollout_blue_green` | `(app)` | Complete blue/green deployment |
| `create_candidate_name` | `(app)` | Candidate name (with timestamp) |
| `candidate_preflight` | `(app, cname)` | Execute preflight_cmd |

### Manual Pause

| Function | Signature | Description |
|---|---|---|
| `manual_pause_state_path` | `(app)` | Flag path |
| `is_manual_pause_active` | `(app)` | Is pause active? |
| `set_manual_pause` | `(app, [reason])` | Enable pause |
| `clear_manual_pause` | `(app)` | Disable pause |
| `manual_stop_pause_enabled` | `(app)` | Is config active? |
| `exit_code_looks_like_manual_stop` | `(code)` | Code = manual stop? |
| `manual_pause_notified_path` | `(app)` | Notification flag path |
| `is_manual_pause_notified` | `(app)` | Already notified? |
| `set_manual_pause_notified` | `(app)` | Mark as notified |
| `clear_manual_pause_notified` | `(app)` | Remove the flag |

### Config Version

| Function | Signature | Description |
|---|---|---|
| `desired_config_version` | `(app)` | Desired version (manifest) |
| `current_config_version` | `(app)` | Current version (Docker label) |

### Miscellaneous

| Function | Signature | Description |
|---|---|---|
| `caelix_section_reserved` | `(section)` | Is section reserved? |
| `restart_count_state_path` | `(app)` | Restart count path |
| `get_last_restart_count` | `(app)` | Last saved restart count |
| `set_last_restart_count` | `(app, count)` | Save the restart count |
| `detect_unexpected_restart` | `(app)` | Detect an unexpected restart |

---

## autoscale.sh — Horizontal Scaling (31 functions)

### State and Configuration

| Function | Signature | Description |
|---|---|---|
| `autoscale_enabled` | `(app)` | Is autoscale active? |
| `autoscale_replica_name` | `(app, idx)` | Replica name |
| `autoscale_lb_name` | `(app)` | Load balancer name |
| `autoscale_replica_port` | `(app, idx)` | Replica port |
| `autoscale_backends_file` | `(app)` | Backends file path |
| `autoscale_cooldown_file` | `(app)` | Cooldown file path |
| `autoscale_state_dir` | `()` | Autoscale directory |
| `autoscale_current_count` | `(app)` | Total replica count |
| `autoscale_running_count` | `(app)` | Running replica count |
| `autoscale_state_for_app` | `(app)` | Summary state for the UI |

### Global Proxy

| Function | Signature | Description |
|---|---|---|
| `autoscale_global_proxy_active` | `()` | Is [proxy] section configured? |
| `global_proxy_update_routes` | `()` | Regenerate routes.conf |
| `global_proxy_pid_file` | `()` | Global proxy PID path |
| `global_proxy_running` | `()` | Is the global proxy alive? |
| `global_proxy_ensure` | `()` | Start if needed |
| `global_proxy_stop` | `()` | Graceful stop |

### Scaling Operations

| Function | Signature | Description |
|---|---|---|
| `autoscale_scale_up` | `(app)` | Create a new replica |
| `autoscale_scale_down` | `(app)` | Remove the last replica |
| `autoscale_write_backends_file` | `(app)` | Update the backends file |
| `autoscale_check_replicas` | `(app)` | Verify all replicas exist |
| `autoscale_recreate_replica` | `(app, idx)` | Recreate a specific replica |

### Load Balancer

| Function | Signature | Description |
|---|---|---|
| `autoscale_lb_ensure` | `(app)` | Ensure the proxy is running |
| `_autoscale_lb_ensure_dedicated` | `(app, port)` | Dedicated proxy (route port:N) |
| `autoscale_lb_running` | `(app)` | Is the proxy alive? |
| `autoscale_lb_reload` | `(app)` | Reload config |
| `autoscale_lb_stop` | `(app)` | Stop the proxy |

### Metrics and Decisions

| Function | Signature | Description |
|---|---|---|
| `autoscale_collect_metric` | `(app)` | Collect the metric value |
| `autoscale_decision` | `(app, value)` | Decide: up, down, stable |

### Orchestration

| Function | Signature | Description |
|---|---|---|
| `autoscale_reconcile` | `(app)` | Complete autoscale reconciliation cycle |
| `autoscale_process_proxy_events` | `(app)` | Process the proxy events.queue |
| `autoscale_cleanup` | `(app)` | Full service cleanup |

---

## proxy.sh — TCP Reverse Proxy (15 functions)

| Function | Signature | Description |
|---|---|---|
| `proxy_log` | `(level, ...msg)` | Proxy logging |
| `_proxy_ts` | `()` | UTC timestamp |
| `_proxy_atomic_inc` | `(file, [delta])` | Atomic counter (flock) |
| `_proxy_atomic_read` | `(file)` | Read counter |
| `_proxy_rr_next` | `(file, count)` | Atomic round-robin index |
| `_proxy_read_backends` | `(bfile, state_dir)` | Read backends + health status |
| `_proxy_pick_backend` | `(bfile, state_dir)` | Round-robin across healthy backends |
| `_proxy_route_lookup` | `(routes, host, path)` | Route matching |
| `_proxy_app_state_dir` | `(state_dir, app)` | Per-app state directory |
| `_proxy_handle_connection` | `()` | Handle an HTTP connection |
| `_proxy_build_state_json` | `(bfile, state_dir)` | JSON for /caelix-proxy/state |
| `_proxy_build_state_json_global` | `(routes, state_dir)` | Multi-route JSON |
| `_proxy_build_metrics` | `(bfile, state_dir)` | Prometheus single app |
| `_proxy_build_metrics_global` | `(routes, state_dir)` | Prometheus multi-route |
| `_proxy_health_loop` | `(bfile, state_dir)` | Background health check loop |

---

## notify.sh — Discord Notifications (11 functions)

| Function | Signature | Description |
|---|---|---|
| `notify_load` | `(path)` | Load notify.ini |
| `notify_get` | `(section, key)` | Read a config value |
| `notify_get_default` | `(section, key, default)` | Read with fallback |
| `notify_all` | `(subject, body)` | Send a notification |
| `_notify_discord` | `(subject, body)` | Post the Discord embed |
| `_notify_json_escape` | `(s)` | Escape JSON |
| `_notify_event_title_fr` | `(event)` | French title |
| `_notify_reason_code_fr` | `(raw)` | French reason |
| `_notify_detection_fr` | `(event, detail)` | Detection description |
| `_notify_actions_fr` | `(event, detail)` | Actions taken |
| `_notify_extract_kv` | `(key, string)` | Extract key=value |

---

## incidents.sh — Incident Journal (4 functions)

| Function | Signature | Description |
|---|---|---|
| `incident_log_path` | `()` | Path to incidents.log |
| `incident_archive_daily` | `()` | Archive to the daily file |
| `_json_escape` | `(s)` | Escape JSON |
| `incident_record` | `(app, severity, event, detail, [skip_discord])` | Record an incident |

---

## audit.sh — Audit Trail (2 Bash functions)

| Function | Signature | Description |
|---|---|---|
| `caelix_audit_py` | `()` | Locate audit_log.py |
| `caelix_audit_event` | `(app, cname, event, source, [detail])` | Record via Python |

---

## doctor.sh — Validation and Diagnostics (4 functions)

| Function | Signature | Description |
|---|---|---|
| `caelix_doctor_py` | `()` | Locate manifest_doctor.py |
| `caelix_manifest_try_repair` | `(mf, [example])` | Repair the manifest (backup .bak) |
| `caelix_manifest_doctor_check` | `(mf, [strict])` | Validate the manifest (Python) |
| `caelix_doctor_env_checks` | `()` | Check the environment (Docker, paths...) |

---

## audit_log.py — Audit Persistence (14 Python functions)

| Function | Signature | Description |
|---|---|---|
| `should_record` | `(manifest, app)` | Should this service be audited? |
| `backend_for` | `(manifest)` | Which backend to use (jsonl/sqlite) |
| `append_jsonl` | `(path, record)` | Append to JSONL |
| `append_sqlite` | `(path, record)` | Insert into SQLite |
| `append_event` | `(manifest_path, data_dir, app, container, event, source, detail)` | Unified entry point |
| `_read_jsonl_tail_records` | `(path, limit)` | Read last N records (backward seek) |
| `read_recent` | `(manifest_path, data_dir, limit)` | Read recent events |
| `clear_audit_storage` | `(data_dir)` | Delete all logs |

---

## manifest_doctor.py — Advanced Validation (7 Python functions)

| Function | Signature | Description |
|---|---|---|
| `load_manifest` | `(path)` | Load manifest with error handling |
| `validate_manifest` | `(cp, strict_local)` | Full validation (ports, strategies, autoscale...) |
| `fix_manifest` | `(path, example)` | Automatic repair with backup |
| `_ports_ok` | `(a, b)` | Validate port range |
| `_parse_publish_entry` | `(raw)` | Validate publish format (IPv4/IPv6) |
| `_strict_local_url_ok` | `(url)` | URL localhost only |
