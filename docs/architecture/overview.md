# Vue d'ensemble de l'architecture

## Principes de conception

| Principe | Description |
|---|---|
| **Single-node** | Pas de coordination multi-machines. Conçu pour orchestrer des conteneurs sur un seul hôte. |
| **Déclaratif** | L'état désiré est défini dans un fichier INI. Le moteur converge vers cet état à chaque cycle. |
| **Self-healing** | Réparation automatique par escalade (restart → recreate → purge) sans intervention. |
| **Minimal** | Dépendances : Bash 5, curl, Docker ou Podman. Pas de runtime additionnel requis. |
| **Observable** | Audit trail, journal d'incidents, alertes Discord, métriques Prometheus. |

---

## Composants

### Pipeline de réconciliation

```mermaid
graph LR
    A["manifest.ini"] --> B["manifest_load()"]
    B --> C["reconcile_app()"]
    C --> D["deep_diagnose_name()"]
    D -->|sain| E["OK"]
    D -->|échec| F["repair_execute()"]
    F --> G["Docker / Podman"]
```

### Pipeline autoscale

```mermaid
graph LR
    A["autoscale_reconcile()"] --> B["collect_metric()"]
    B --> C["autoscale_decision()"]
    C -->|up| D["scale_up()"]
    C -->|down| E["scale_down()"]
    D --> F["write_backends + proxy reload"]
    E --> F
```

### Observabilité

```mermaid
graph LR
    A["Événement"] --> B["incident_record()"]
    B --> C["incidents.log"]
    B --> D["YYYY-MM-DD.jsonl"]
    B --> E["Discord webhook"]
    A --> F["caelix_audit_event()"]
    F --> G["JSONL / SQLite"]
```

---

## Flux de données principal

```mermaid
sequenceDiagram
    participant M as manifest.ini
    participant S as bin/caelix run
    participant R as reconcile_app()
    participant H as deep_diagnose_name()
    participant D as Docker
    participant N as Discord
    participant I as incidents.log

    loop Toutes les N secondes
        S->>M: manifest_load()
        S->>S: Détecter runtime (docker/podman)
        S->>R: Pour chaque app dans MNF_APPS_ORDER
        R->>D: container_exists() ?
        alt N'existe pas
            R->>D: container_create()
            R->>I: incident_record(info, created)
        else Existe
            R->>D: container_running() ?
            alt En marche
                R->>H: deep_diagnose_name()
                alt Sain
                    H-->>R: return 0 (OK)
                else En panne
                    H-->>R: return 1 (raison)
                    R->>D: repair_execute() (restart/recreate/purge)
                    R->>I: incident_record(severity, event)
                    R->>N: notify_all() → Discord embed
                end
            else Arrêté
                R->>D: container_start()
            end
        end
        S->>S: remove_orphan_containers()
        S->>S: caelix_daemon_heartbeat()
        S->>S: sleep CAELIX_INTERVAL
    end
```

---

## Communication entre modules

```mermaid
graph LR
    manifest["manifest.sh<br>MNF[] global array"] --> health
    manifest --> repair
    manifest --> autoscale
    manifest --> runtime
    manifest --> proxy
    manifest --> doctor

    common["common.sh<br>Logging, état, ports"] --> health
    common --> repair
    common --> autoscale
    common --> runtime

    health["health.sh"] -->|diagnostic| repair["repair.sh"]
    repair -->|docker ops| runtime["runtime.sh"]
    repair -->|alertes| notify["notify.sh"]
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

## Conventions de nommage

### Conteneurs

| Type | Format | Exemple |
|---|---|---|
| Service standard | `caelix-<app>` | `caelix-web` |
| Candidat blue/green | `caelix-<app>-candidate-<timestamp>` | `caelix-web-candidate-1705312200` |
| Replica autoscale | `caelix-<app>-r<N>` | `caelix-web-r3` |
| Load balancer (legacy) | `caelix-<app>-lb` | `caelix-web-lb` |

### Labels Docker

| Label | Valeur | Usage |
|---|---|---|
| `caelix.app` | Nom du service | Identification |
| `caelix.config_version` | Version de config | Détection de changement |
| `caelix.role` | `replica` | Distinction replicas autoscale |
| `caelix.replica` | Numéro (1, 2, 3...) | Index de la replica |

### Fichiers d'état

| Pattern | Exemple | Contenu |
|---|---|---|
| `.caelix/state/<app>.fail` | `.caelix/state/web.fail` | Compteur d'échecs (entier) |
| `.caelix/state/<app>.manual_pause` | `.caelix/state/web.manual_pause` | Raison de la pause |
| `.caelix/state/<app>.suspend_reconcile` | `.caelix/state/web.suspend_reconcile` | Flag de suspension |
| `.caelix/state/<app>.create_fail_streak` | `.caelix/state/web.create_fail_streak` | Échecs consécutifs |
| `.caelix/state/<app>.restart_count` | `.caelix/state/web.restart_count` | Dernier restart count Docker |
| `.caelix/state/<app>.http_errrate` | `.caelix/state/web.http_errrate` | `fail total` (fenêtre glissante) |
| `.caelix/state/<app>.autoscale_cooldown` | `.caelix/state/web.autoscale_cooldown` | Streak de décisions |
| `.caelix/autoscale/<app>.backends` | `.caelix/autoscale/web.backends` | `name host port` par ligne |
