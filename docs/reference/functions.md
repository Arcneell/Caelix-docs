# Fonctions internes

Référence complète des 173 fonctions du moteur Caelix, organisées par module.

---

## common.sh — Logging, état, ports (23 fonctions)

### Logging et heartbeat

| Fonction | Signature | Description |
|---|---|---|
| `caelix_log` | `(level, ...message)` | Log vers stderr + fichier JSON. Niveaux : debug, info, warn, error |
| `_caelix_ts` | `()` | Timestamp UTC ISO (YYYY-MM-DDTHH:MM:SSZ) |
| `caelix_daemon_heartbeat` | `()` | Écrit le timestamp dans `.caelix/state/caelix-daemon-heartbeat` |
| `_caelix_log_rotate` | `()` | Rotation du log daemon (10 Mo, 30 fichiers max) |
| `caelix_version_string` | `(root)` | Version depuis le fichier VERSION |

### Compteurs d'échecs

| Fonction | Signature | Description |
|---|---|---|
| `fail_count_path` | `(app)` | Chemin vers `.caelix/state/<app>.fail` |
| `get_fail_count` | `(app)` | Lire le compteur d'échecs |
| `set_fail_count` | `(app, count)` | Écrire le compteur |
| `inc_fail_count` | `(app)` | Incrémenter de 1 |
| `reset_fail_count` | `(app)` | Remettre à 0 + supprimer flag notification |

### Streak de création

| Fonction | Signature | Description |
|---|---|---|
| `create_fail_streak_path` | `(app)` | Chemin vers `.caelix/state/<app>.create_fail_streak` |
| `caelix_bump_create_fail_streak` | `(app)` | Incrémenter le compteur d'échecs consécutifs |
| `caelix_get_create_fail_streak` | `(app)` | Lire le compteur |
| `caelix_clear_create_fail_streak` | `(app)` | Remettre à zéro |

### Suspension

| Fonction | Signature | Description |
|---|---|---|
| `suspend_reconcile_path` | `(app)` | Chemin vers le flag de suspension |
| `suspend_reconcile_notified_path` | `(app)` | Chemin vers le flag de notification |
| `caelix_clear_suspend_state` | `(app)` | Supprimer suspension + streak |

### Allocation de ports

| Fonction | Signature | Description |
|---|---|---|
| `port_alloc_file` | `()` | Chemin vers `port_allocations` |
| `port_alloc_init` | `()` | Initialiser le fichier |
| `port_alloc_get` | `(app, idx)` | Retrouver un port existant |
| `port_alloc_next` | `(app, idx)` | Allouer le prochain port libre (flock) |
| `port_alloc_release` | `(app)` | Libérer tous les ports d'un service |
| `port_alloc_release_one` | `(app, idx)` | Libérer un seul port |

---

## manifest.sh — Parseur INI (4 fonctions)

| Fonction | Signature | Description |
|---|---|---|
| `manifest_clear` | `()` | Vider MNF[] et MNF_APPS_ORDER |
| `manifest_load` | `(path)` | Parser le fichier INI → MNF["section\|key"] |
| `manifest_get` | `(section, key)` | Lire une valeur (erreur si absente) |
| `manifest_get_default` | `(section, key, default)` | Lire avec fallback |

**Variables globales :**

- `MNF` : array associatif `MNF["section|key"] = "value"`
- `MNF_APPS_ORDER` : array des noms de sections dans l'ordre du fichier

---

## runtime.sh — Opérations Docker/Podman (18 fonctions)

### Détection et wrapper

| Fonction | Signature | Description |
|---|---|---|
| `runtime_engine` | `()` | Détecte docker ou podman dans PATH |
| `_rt` | `(...args)` | Wrapper vers $RT (docker/podman) |
| `caelix_cname` | `(app)` | Retourne `caelix-<app>` |

### Inspection

| Fonction | Signature | Description |
|---|---|---|
| `container_id_by_name` | `(name)` | ID du conteneur par son nom |
| `container_exists` | `(app)` | Le conteneur existe-t-il ? |
| `container_running` | `(app)` | Le conteneur tourne-t-il ? |
| `container_image` | `(app)` | Image actuelle du conteneur |
| `container_label` | `(app, key)` | Valeur d'un label Docker |
| `image_matches` | `(want, got)` | Comparaison intelligente d'images |
| `inspect_restart_count` | `(cid)` | Nombre de redémarrages Docker |
| `inspect_oom_killed` | `(cid)` | OOM Killed ? (true/false) |
| `inspect_state_status` | `(cid)` | Statut (running, exited...) |
| `inspect_exit_code` | `(cid)` | Code de sortie |

### Cycle de vie

| Fonction | Signature | Description |
|---|---|---|
| `container_create` | `(app, [publish], [name])` | Créer un conteneur avec toute la config du manifest |
| `container_remove_force` | `(app)` | `docker rm -f` |
| `container_restart` | `(app)` | `docker restart` |
| `container_start` | `(app)` | `docker start` |
| `container_exec` | `(cname, ...args)` | Exécuter une commande dans le conteneur |

---

## health.sh — Surveillance (16 fonctions)

| Fonction | Signature | Description |
|---|---|---|
| `monitoring_enabled` | `(app, type)` | Type de monitoring actif pour ce service ? |
| `health_tcp` | `(host, port, [timeout])` | Probe TCP (nc ou /dev/tcp) |
| `health_http` | `(url, [timeout], [expect], [max_bytes])` | Probe HTTP avec validation complète |
| `caelix_health_url_is_local` | `(url)` | URL cible localhost ? |
| `container_memory_usage_mb` | `(name)` | Usage mémoire en Mo |
| `container_recent_logs` | `(name, [tail])` | N dernières lignes de logs |
| `container_resource_snapshot` | `(name)` | Snapshot CPU + mémoire |
| `http_total_time_ms` | `(url, [timeout])` | Temps de réponse HTTP en ms |
| `http_probe_code` | `(url, [timeout])` | Code HTTP seulement |
| `http_error_rate_state_path` | `(app)` | Chemin fichier errrate |
| `http_error_rate_state_get` | `(app)` | Lire fail et total |
| `http_error_rate_state_set` | `(app, fail, total)` | Écrire fail et total |
| `disk_usage_pct_for_path` | `(path)` | Usage disque en % |
| `check_disk_usage_limit` | `(app)` | Vérifier tous les volumes |
| `deep_diagnose_name` | `(app, name)` | Diagnostic complet (8 types) |
| `deep_diagnose` | `(app)` | Wrapper avec nom standard |

---

## repair.sh — Réconciliation et réparation (24 fonctions)

### Réconciliation

| Fonction | Signature | Description |
|---|---|---|
| `reconcile_app` | `(app)` | Point d'entrée de la réconciliation par service |
| `ensure_desired_revision` | `(app)` | Vérifier image et config_version |
| `remove_orphan_containers` | `()` | Supprimer les caelix-* non déclarés |

### Réparation

| Fonction | Signature | Description |
|---|---|---|
| `repair_execute` | `(app, reason)` | Escalade : restart → recreate → purge |

### Blue/Green

| Fonction | Signature | Description |
|---|---|---|
| `rollout_blue_green` | `(app)` | Déploiement blue/green complet |
| `create_candidate_name` | `(app)` | Nom du candidat (avec timestamp) |
| `candidate_preflight` | `(app, cname)` | Exécuter preflight_cmd |

### Pause manuelle

| Fonction | Signature | Description |
|---|---|---|
| `manual_pause_state_path` | `(app)` | Chemin du flag |
| `is_manual_pause_active` | `(app)` | Pause active ? |
| `set_manual_pause` | `(app, [reason])` | Activer la pause |
| `clear_manual_pause` | `(app)` | Désactiver |
| `manual_stop_pause_enabled` | `(app)` | Config active ? |
| `exit_code_looks_like_manual_stop` | `(code)` | Code = arrêt manuel ? |
| `manual_pause_notified_path` | `(app)` | Chemin flag notification |
| `is_manual_pause_notified` | `(app)` | Déjà notifié ? |
| `set_manual_pause_notified` | `(app)` | Marquer comme notifié |
| `clear_manual_pause_notified` | `(app)` | Supprimer le flag |

### Config version

| Fonction | Signature | Description |
|---|---|---|
| `desired_config_version` | `(app)` | Version désirée (manifest) |
| `current_config_version` | `(app)` | Version actuelle (label Docker) |

### Divers

| Fonction | Signature | Description |
|---|---|---|
| `caelix_section_reserved` | `(section)` | Section réservée ? |
| `restart_count_state_path` | `(app)` | Chemin du restart count |
| `get_last_restart_count` | `(app)` | Dernier restart count sauvegardé |
| `set_last_restart_count` | `(app, count)` | Sauvegarder le restart count |
| `detect_unexpected_restart` | `(app)` | Détecter un restart inattendu |

---

## autoscale.sh — Scaling horizontal (31 fonctions)

### État et configuration

| Fonction | Signature | Description |
|---|---|---|
| `autoscale_enabled` | `(app)` | Autoscale activé ? |
| `autoscale_replica_name` | `(app, idx)` | Nom de la replica |
| `autoscale_lb_name` | `(app)` | Nom du load balancer |
| `autoscale_replica_port` | `(app, idx)` | Port de la replica |
| `autoscale_backends_file` | `(app)` | Chemin du fichier backends |
| `autoscale_cooldown_file` | `(app)` | Chemin du fichier cooldown |
| `autoscale_state_dir` | `()` | Répertoire autoscale |
| `autoscale_current_count` | `(app)` | Nombre total de replicas |
| `autoscale_running_count` | `(app)` | Nombre de replicas en marche |
| `autoscale_state_for_app` | `(app)` | État résumé pour l'UI |

### Proxy global

| Fonction | Signature | Description |
|---|---|---|
| `autoscale_global_proxy_active` | `()` | Section [proxy] configurée ? |
| `global_proxy_update_routes` | `()` | Régénérer routes.conf |
| `global_proxy_pid_file` | `()` | Chemin PID proxy global |
| `global_proxy_running` | `()` | Proxy global en vie ? |
| `global_proxy_ensure` | `()` | Démarrer si nécessaire |
| `global_proxy_stop` | `()` | Arrêter proprement |

### Opérations de scaling

| Fonction | Signature | Description |
|---|---|---|
| `autoscale_scale_up` | `(app)` | Créer une nouvelle replica |
| `autoscale_scale_down` | `(app)` | Supprimer la dernière replica |
| `autoscale_write_backends_file` | `(app)` | Mettre à jour le fichier backends |
| `autoscale_check_replicas` | `(app)` | Vérifier que toutes les replicas existent |
| `autoscale_recreate_replica` | `(app, idx)` | Recréer une replica spécifique |

### Load balancer

| Fonction | Signature | Description |
|---|---|---|
| `autoscale_lb_ensure` | `(app)` | Assurer que le proxy tourne |
| `_autoscale_lb_ensure_dedicated` | `(app, port)` | Proxy dédié (route port:N) |
| `autoscale_lb_running` | `(app)` | Proxy en vie ? |
| `autoscale_lb_reload` | `(app)` | Recharger la config |
| `autoscale_lb_stop` | `(app)` | Arrêter le proxy |

### Métriques et décisions

| Fonction | Signature | Description |
|---|---|---|
| `autoscale_collect_metric` | `(app)` | Collecter la valeur de la métrique |
| `autoscale_decision` | `(app, value)` | Décider : up, down, stable |

### Orchestration

| Fonction | Signature | Description |
|---|---|---|
| `autoscale_reconcile` | `(app)` | Cycle complet de réconciliation autoscale |
| `autoscale_process_proxy_events` | `(app)` | Traiter les events.queue du proxy |
| `autoscale_cleanup` | `(app)` | Nettoyage complet d'un service |

---

## proxy.sh — Reverse proxy TCP (15 fonctions)

| Fonction | Signature | Description |
|---|---|---|
| `proxy_log` | `(level, ...msg)` | Logging du proxy |
| `_proxy_ts` | `()` | Timestamp UTC |
| `_proxy_atomic_inc` | `(file, [delta])` | Compteur atomique (flock) |
| `_proxy_atomic_read` | `(file)` | Lecture compteur |
| `_proxy_rr_next` | `(file, count)` | Index round-robin atomique |
| `_proxy_read_backends` | `(bfile, state_dir)` | Lire backends + statut santé |
| `_proxy_pick_backend` | `(bfile, state_dir)` | Round-robin sur backends sains |
| `_proxy_route_lookup` | `(routes, host, path)` | Matching de route |
| `_proxy_app_state_dir` | `(state_dir, app)` | Répertoire d'état par app |
| `_proxy_handle_connection` | `()` | Traiter une connexion HTTP |
| `_proxy_build_state_json` | `(bfile, state_dir)` | JSON pour /caelix-proxy/state |
| `_proxy_build_state_json_global` | `(routes, state_dir)` | JSON multi-routes |
| `_proxy_build_metrics` | `(bfile, state_dir)` | Prometheus single app |
| `_proxy_build_metrics_global` | `(routes, state_dir)` | Prometheus multi-routes |
| `_proxy_health_loop` | `(bfile, state_dir)` | Boucle de health check background |

---

## notify.sh — Notifications Discord (11 fonctions)

| Fonction | Signature | Description |
|---|---|---|
| `notify_load` | `(path)` | Charger notify.ini |
| `notify_get` | `(section, key)` | Lire une config |
| `notify_get_default` | `(section, key, default)` | Lire avec fallback |
| `notify_all` | `(subject, body)` | Envoyer une notification |
| `_notify_discord` | `(subject, body)` | Poster l'embed Discord |
| `_notify_json_escape` | `(s)` | Échapper JSON |
| `_notify_event_title_fr` | `(event)` | Titre français |
| `_notify_reason_code_fr` | `(raw)` | Raison en français |
| `_notify_detection_fr` | `(event, detail)` | Description de détection |
| `_notify_actions_fr` | `(event, detail)` | Actions entreprises |
| `_notify_extract_kv` | `(key, string)` | Extraire key=value |

---

## incidents.sh — Journal d'incidents (4 fonctions)

| Fonction | Signature | Description |
|---|---|---|
| `incident_log_path` | `()` | Chemin vers incidents.log |
| `incident_archive_daily` | `()` | Archiver dans le fichier journalier |
| `_json_escape` | `(s)` | Échapper JSON |
| `incident_record` | `(app, severity, event, detail, [skip_discord])` | Enregistrer un incident |

---

## audit.sh — Trail d'audit (2 fonctions Bash)

| Fonction | Signature | Description |
|---|---|---|
| `caelix_audit_py` | `()` | Localiser audit_log.py |
| `caelix_audit_event` | `(app, cname, event, source, [detail])` | Enregistrer via Python |

---

## doctor.sh — Validation et diagnostic (4 fonctions)

| Fonction | Signature | Description |
|---|---|---|
| `caelix_doctor_py` | `()` | Localiser manifest_doctor.py |
| `caelix_manifest_try_repair` | `(mf, [example])` | Réparer le manifest (backup .bak) |
| `caelix_manifest_doctor_check` | `(mf, [strict])` | Valider le manifest (Python) |
| `caelix_doctor_env_checks` | `()` | Vérifier l'environnement (Docker, chemins...) |

---

## audit_log.py — Persistence audit (14 fonctions Python)

| Fonction | Signature | Description |
|---|---|---|
| `should_record` | `(manifest, app)` | Faut-il auditer ce service ? |
| `backend_for` | `(manifest)` | Backend à utiliser (jsonl/sqlite) |
| `append_jsonl` | `(path, record)` | Ajouter au JSONL |
| `append_sqlite` | `(path, record)` | Insérer dans SQLite |
| `append_event` | `(manifest_path, data_dir, app, container, event, source, detail)` | Point d'entrée unifié |
| `_read_jsonl_tail_records` | `(path, limit)` | Lire les N derniers records (seek arrière) |
| `read_recent` | `(manifest_path, data_dir, limit)` | Lire les événements récents |
| `clear_audit_storage` | `(data_dir)` | Supprimer tous les logs |

---

## manifest_doctor.py — Validation avancée (7 fonctions Python)

| Fonction | Signature | Description |
|---|---|---|
| `load_manifest` | `(path)` | Charger le manifest avec gestion d'erreurs |
| `validate_manifest` | `(cp, strict_local)` | Validation complète (ports, stratégies, autoscale...) |
| `fix_manifest` | `(path, example)` | Réparation automatique avec backup |
| `_ports_ok` | `(a, b)` | Valider plage de ports |
| `_parse_publish_entry` | `(raw)` | Valider format publish (IPv4/IPv6) |
| `_strict_local_url_ok` | `(url)` | URL localhost seulement |
