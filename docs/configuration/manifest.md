# Configuration du manifest

Le fichier `etc/manifest.ini` centralise la déclaration des services et les paramètres globaux de l'orchestrateur.

## Format

Le manifest utilise le format INI standard :

```ini
# Commentaire
[section]
cle = valeur
cle_avec_guillemets = "valeur avec espaces"
```

- Les commentaires commencent par `#`
- Les valeurs peuvent être entre guillemets
- Les clés dupliquées dans une même section génèrent une erreur

## Sections réservées

Ces sections ne sont pas traitées comme des services :

### [orchestrator]

Paramètres globaux du daemon.

```ini
[orchestrator]
interval = 10              # Intervalle de réconciliation (secondes)
max_repair = 5             # Seuil d'échecs avant alerte
remove_orphans = 1         # Supprimer les conteneurs caelix-* inconnus
log_level = info           # debug, info, warn, error
audit_log_backend = jsonl  # jsonl ou sqlite
audit_log_all = 0          # 1 = auditer tous les conteneurs caelix-*
```

### [proxy]

Configuration du reverse-proxy global.

```ini
[proxy]
listen = 0.0.0.0:8080          # Adresse d'écoute du proxy global
autoscale_port_range = 18500-18999  # Plage de ports pour les replicas
health_interval = 3             # Intervalle de health check proxy (sec)
connect_timeout = 5             # Timeout de connexion backend (sec)
```

### [notify] et [global]

Réservés pour usage futur. La configuration des notifications se fait dans `etc/notify.ini`.

## Définition d'un service

Chaque section non réservée définit un service :

```ini
[mon-service]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
```

### Paramètres de base

| Clé | Défaut | Description |
|---|---|---|
| `image` | **requis** | Image Docker (ex: `nginx:latest`, `myapp:v1.2.0`) |
| `publish` | — | Ports exposés, CSV (ex: `8080:80,443:443`) |
| `command` | — | Commande à exécuter (override CMD) |
| `env` | — | Variables d'environnement, séparées par `;` (ex: `KEY=val;KEY2=val2`) |
| `volumes_bind` | — | Bind mounts, CSV (ex: `/data:/app/data,/logs:/app/logs`) |
| `network` | `bridge` | Réseau Docker |
| `config_version` | — | Incrémenter pour forcer la re-création du conteneur |

### Health checks

| Clé | Défaut | Description |
|---|---|---|
| `health_type` | `http` | Type de probe : `http`, `https`, `tcp`, `none` |
| `health_url` | — | URL pour les probes HTTP/HTTPS |
| `health_tcp_port` | — | Port pour les probes TCP |
| `health_expect_codes` | `200,204` | Codes HTTP acceptés (CSV) |
| `health_timeout` | `5` | Timeout de la probe (secondes) |
| `health_max_bytes` | `0` | Taille max de la réponse (0 = illimité) |

### Monitoring

| Clé | Défaut | Description |
|---|---|---|
| `monitoring_types` | `all` | Types actifs (CSV ou `all`/`none`) |
| `monitoring_log_tail` | `120` | Nombre de lignes de log à analyser |
| `monitoring_log_error_regex` | (built-in) | Regex pour détecter les anomalies dans les logs |
| `monitoring_http_latency_max_ms` | `1200` | Latence max acceptable (ms) |
| `monitoring_http_error_rate_threshold_pct` | `30` | Seuil de taux d'erreur (%) |
| `monitoring_http_error_rate_window` | `20` | Taille de la fenêtre glissante |
| `monitoring_disk_usage_max_pct` | `90` | Utilisation disque max (%) |

Types de monitoring disponibles :

- `health` — probe HTTP/TCP
- `memory` — utilisation mémoire
- `oom` — détection OOM killed
- `restart` — redémarrages inattendus
- `logs` — analyse de logs par regex
- `http_latency` — temps de réponse
- `http_error_rate` — taux d'erreur HTTP
- `disk` — utilisation disque

### Mémoire

| Clé | Défaut | Description |
|---|---|---|
| `memory_limit_mb` | — | Limite mémoire Docker (`-m` flag) |
| `memory_soft_mb` | — | Seuil d'avertissement (warning) |
| `memory_hard_mb` | — | Seuil critique (déclenche réparation) |

### Rollout et réparation

| Clé | Défaut | Description |
|---|---|---|
| `rollout_strategy` | `recreate` | `recreate` ou `blue_green` |
| `candidate_publish` | — | Port du candidat blue/green (requis si blue_green) |
| `health_url_candidate` | — | URL de santé du candidat (si différente) |
| `preflight_cmd` | — | Commande à exécuter dans le candidat avant bascule |
| `repair_strategy` | `auto` | `auto`, `restart-only`, `recreate-only`, `purge-only` |
| `purge_on_escalation` | `0` | Supprimer les volumes nommés lors du purge |
| `post_repair_grace` | — | Délai après réparation avant health check (sec) |
| `create_fail_max_attempts` | `0` | Max échecs de création avant suspension (0 = illimité) |

### Comportement

| Clé | Défaut | Description |
|---|---|---|
| `manual_stop_pause` | `1` | Ne pas redémarrer après un arrêt manuel |
| `container_audit_log` | `0` | Activer l'audit pour ce service |

### Autoscale

| Clé | Défaut | Description |
|---|---|---|
| `autoscale` | `0` | Activer l'autoscaling (`1` ou `true`) |
| `autoscale_min` | `1` | Nombre minimum de replicas |
| `autoscale_max` | `5` | Nombre maximum de replicas |
| `autoscale_metric` | `http_latency` | Métriques CSV : `http_latency`, `memory`, `http_error_rate` |
| `autoscale_up_threshold` | `800` | Seuil de scale-up (ms/Mo/%) |
| `autoscale_down_threshold` | `200` | Seuil de scale-down |
| `autoscale_cooldown` | `3` | Passages consécutifs avant action |
| `autoscale_container_port` | `8080` | Port interne du conteneur replica |
| `autoscale_port_base` | (auto) | Port hôte de départ (auto-alloué si absent) |
| `autoscale_lb_publish` | — | Adresse du load balancer (mode legacy) |
| `autoscale_health_path` | `/` | Chemin HTTP pour les probes de backend |
| `autoscale_route` | — | Routage global : `host:hostname`, `path:/prefix`, `port:N`, `default` |

## Exemple complet

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

Toujours valider après modification :

```bash
# Validation rapide
bin/caelix validate

# Diagnostic complet (avec suggestions de correction)
bin/caelix doctor

# Correction automatique des problèmes détectés
bin/caelix doctor --fix
```
