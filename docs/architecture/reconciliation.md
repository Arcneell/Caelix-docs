# Boucle de réconciliation

La boucle de réconciliation compare en continu l'état désiré (manifest) avec l'état réel (Docker) et applique les corrections nécessaires à chaque cycle.

---

## Vue d'ensemble du cycle

```mermaid
graph TD
    start["Début du cycle"] --> load["1. Charger manifest.ini<br>manifest_load()"]
    load -->|Échec| tryrepair["manifest_try_repair()<br>Réparation auto"]
    tryrepair -->|Échec| skip["Abandonner ce cycle"]
    tryrepair -->|OK| globals
    load -->|OK| globals["2. Appliquer les globales<br>[orchestrator] → interval, max_repair, log_level"]
    globals --> detect["3. Détecter le runtime<br>runtime_engine() → docker ou podman"]
    detect --> proxy{"4. Section [proxy]<br>définie ?"}
    proxy -->|Oui| proxyensure["global_proxy_ensure()<br>Démarrer/vérifier le proxy"]
    proxy -->|Non| apps
    proxyensure --> apps["5. Pour chaque app<br>dans MNF_APPS_ORDER"]
    apps --> reconcile["reconcile_app(app)"]
    reconcile --> next{"Encore des apps ?"}
    next -->|Oui| apps
    next -->|Non| orphans{"6. remove_orphans = 1 ?"}
    orphans -->|Oui| clean["remove_orphan_containers()<br>Supprimer les caelix-* inconnus"]
    orphans -->|Non| heartbeat
    clean --> heartbeat["7. caelix_daemon_heartbeat()<br>Écrire timestamp"]
    heartbeat --> sleep["8. sleep CAELIX_INTERVAL"]
    sleep --> start

    style start fill:#1abc9c,color:#fff
    style skip fill:#e74c3c,color:#fff
    style reconcile fill:#3498db,color:#fff
    style heartbeat fill:#2ecc71,color:#fff
```

### Paramètres du cycle

```ini
[orchestrator]
interval = 10       # Secondes entre chaque cycle (défaut: 15)
max_repair = 5      # Échecs avant alerte critique (défaut: 5)
remove_orphans = 1  # Supprimer les conteneurs caelix-* non déclarés
log_level = info    # debug, info, warn, error
```

!!! info "Heartbeat"
    Le fichier `.caelix/state/caelix-daemon-heartbeat` contient le timestamp du dernier cycle complet. La console web l'utilise pour afficher si le daemon est actif.

---

## Réconciliation par application

La fonction `reconcile_app()` est appelée pour chaque service. Voici sa logique complète :

### Pré-checks

```mermaid
graph LR
    A["reconcile_app"] --> B{"autoscale ?"}
    B -->|oui| C["autoscale_reconcile → return"]
    B -->|non| D{"pause manuelle ?"}
    D -->|oui| E["skip"]
    D -->|non| F{"suspendu ?"}
    F -->|oui| G["skip + alerte"]
    F -->|non| H["continuer"]
```

### Convergence

```mermaid
graph LR
    A{"conteneur existe ?"} -->|non| B["container_create()"]
    A -->|oui| C{"image/config changé ?"}
    C -->|oui| D["rollout blue_green ou recreate"]
    C -->|non| E{"en marche ?"}
    E -->|non| F["container_start()"]
    E -->|oui| G["deep_diagnose()"]
```

### Réparation

```mermaid
graph LR
    A["deep_diagnose()"] -->|sain| B["reset_fail_count"]
    A -->|échec| C["repair_execute()"]
    C --> D["grace period"]
    D --> E["re-check"]
    E -->|sain| B
    E -->|échec| F["inc_fail_count"]
    F -->|">= max_repair"| G["alerte critique"]
```

---

## Détection de révision

La fonction `ensure_desired_revision()` vérifie deux choses :

1. L'image du conteneur correspond-elle à celle du manifest ?
2. Le label `caelix.config_version` correspond-il au manifest ?

```mermaid
flowchart LR
    A["Image actuelle<br>vs manifest"] --> B{"Match ?"}
    C["config_version<br>label vs manifest"] --> B
    B -->|Oui| D["Pas de changement"]
    B -->|Non| E{"rollout_strategy"}
    E -->|recreate| F["Remove + Create"]
    E -->|blue_green| G["rollout_blue_green()"]
```

La comparaison d'images normalise les noms avant de comparer : `nginx` est équivalent à `docker.io/library/nginx:latest`.

---

## Stratégies de réparation

### Escalade automatique (`repair_strategy = auto`)

Le compteur `.fail` détermine la phase :

| fail_count | Phase | Action |
|---|---|---|
| 1 | **restart** | `docker restart caelix-<app>` |
| 2 | **recreate** | `docker rm -f` + `docker run` |
| 3+ | **purge** | Remove + suppression volumes + `docker run` |

```mermaid
graph LR
    F1["fail = 1"] --> R["Restart<br>docker restart"]
    F2["fail = 2"] --> RC["Recreate<br>rm -f + run"]
    F3["fail >= 3"] --> P["Purge<br>rm -f + volume rm + run"]

    R -->|Échec| F2
    RC -->|Échec| F3
    P -->|Échec| Alert["Alerte critique<br>+ notification Discord"]

    R -->|OK| Reset["fail = 0"]
    RC -->|OK| Reset
    P -->|OK| Reset

    style F1 fill:#f39c12,color:#fff
    style F2 fill:#e67e22,color:#fff
    style F3 fill:#e74c3c,color:#fff
    style Reset fill:#27ae60,color:#fff
    style Alert fill:#c0392b,color:#fff
```

### Stratégies spécifiques

```ini
repair_strategy = auto           # Escalade complète (défaut)
repair_strategy = restart-only   # Seulement restart, pas d'escalade
repair_strategy = recreate-only  # Seulement remove + create
repair_strategy = purge-only     # Seulement purge complète
```

### Grace period

```ini
post_repair_grace = 5  # Secondes d'attente après réparation avant re-vérification
```

---

## Déploiement blue/green

```mermaid
sequenceDiagram
    participant S as Caelix
    participant Old as Ancien conteneur<br>(port 3000)
    participant New as Candidat<br>(port 3001)
    participant D as Docker

    S->>D: Créer candidat sur candidate_publish (3001)
    S->>S: Attendre candidate_grace (2s)

    opt preflight_cmd défini
        S->>New: Exécuter preflight_cmd<br>(ex: python manage.py migrate)
        New-->>S: Exit code
        alt Échec preflight
            S->>D: Supprimer candidat
            S-->>S: incident_record(critical, bluegreen_fail)
        end
    end

    S->>New: deep_diagnose_name() via health_url_candidate
    alt Candidat sain
        S->>D: Renommer ancien → temp
        S->>D: Renommer candidat → caelix-app
        S->>D: Supprimer ancien
        S-->>S: incident_record(info, bluegreen_switch)
    else Candidat en panne
        S->>D: Supprimer candidat
        S-->>S: incident_record(critical, bluegreen_fail)
        Note over S: L'ancien conteneur reste actif
    end
```

Configuration requise :

```ini
[mon-service]
rollout_strategy = blue_green
publish = 127.0.0.1:3000:3000
candidate_publish = 127.0.0.1:3001:3000
health_url_candidate = http://127.0.0.1:3001/health  # optionnel
preflight_cmd = python manage.py migrate               # optionnel
```

---

## Protections contre les boucles

### Suspension automatique

```ini
create_fail_max_attempts = 3  # Suspendre après 3 échecs de création
```

Quand la création échoue N fois de suite :

1. Le fichier `.caelix/state/<app>.suspend_reconcile` est créé
2. Caelix arrête de toucher à ce service
3. Une alerte critique est envoyée

Pour reprendre : `bin/caelix resume <app>`

### Pause manuelle

Quand un opérateur arrête un conteneur manuellement (`docker stop`), Caelix détecte les exit codes d'arrêt propre :

| Exit code | Signification |
|---|---|
| `0` | Arrêt normal |
| `137` | SIGKILL |
| `143` | SIGTERM |

Caelix crée `.caelix/state/<app>.manual_pause` et ne redémarrera pas le conteneur.

```ini
manual_stop_pause = 1  # Activé par défaut
```

---

## Nettoyage des orphelins

Quand `remove_orphans = 1`, Caelix supprime les conteneurs qui :

- Ont un nom commençant par `caelix-`
- Ne correspondent à aucune section du manifest
- Ne sont pas des replicas (`-r<N>`) ou LB (`-lb`) d'un service existant

Les fichiers d'état associés sont aussi nettoyés.

---

## Modes d'exécution

| Commande | Usage | Quand l'utiliser |
|---|---|---|
| `bin/caelix run` | Boucle infinie | Production (daemon, systemd) |
| `bin/caelix once` | Un seul cycle | Tests, cron, premier lancement |
| `bin/caelix reconcile-app <app>` | Un seul service | Debug ciblé |
