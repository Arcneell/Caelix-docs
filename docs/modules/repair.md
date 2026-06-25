# Repair & Rollout

Le module `repair.sh` implémente la réparation automatique par escalade et les stratégies de déploiement (recreate, blue/green).

---

## Stratégies de réparation

### Vue d'ensemble

```mermaid
graph TD
    subgraph auto["repair_strategy = auto"]
        A1["Phase 1: Restart"] -->|Échec| A2["Phase 2: Recreate"]
        A2 -->|Échec| A3["Phase 3: Purge"]
    end

    subgraph restart["repair_strategy = restart-only"]
        R1["Restart uniquement"]
    end

    subgraph recreate["repair_strategy = recreate-only"]
        RC1["Remove + Create"]
    end

    subgraph purge["repair_strategy = purge-only"]
        P1["Remove + Volumes + Create"]
    end

    style auto fill:#2c3e50,color:#fff
    style restart fill:#3498db,color:#fff
    style recreate fill:#e67e22,color:#fff
    style purge fill:#e74c3c,color:#fff
```

### Phase 1 : Restart

```bash
docker restart caelix-<app>
```

- Simple et rapide
- Conserve les volumes, la config, le filesystem du conteneur
- Suffisant pour les crashs temporaires et les fuites mémoire légères

### Phase 2 : Recreate

```bash
docker rm -f caelix-<app>
docker run ... (re-création complète depuis le manifest)
```

- Régénère le conteneur depuis zéro à partir de l'image
- Applique la configuration actuelle du manifest (ports, env, volumes, labels...)
- Les volumes bind sont réattachés
- Utile quand le filesystem du conteneur est corrompu

### Phase 3 : Purge

```bash
docker rm -f caelix-<app>
docker volume rm <volumes>     # si purge_on_escalation=1
docker run ... (re-création complète)
```

- Dernier recours
- Supprime les volumes nommés pour repartir d'une base propre
- **Perte de données possible** si les volumes ne sont pas sauvegardés

### Configuration

```ini
[mon-service]
repair_strategy = auto           # Escalade complète (défaut)
repair_strategy = restart-only   # Seulement restart
repair_strategy = recreate-only  # Seulement recreate
repair_strategy = purge-only     # Seulement purge

purge_on_escalation = 1          # Supprimer les volumes lors du purge (défaut: 0)
post_repair_grace = 5            # Secondes d'attente après réparation (défaut: 3)
```

### Logique de `repair_execute()`

La phase est déterminée par le compteur d'échecs `fail_count` :

En mode `auto`, l'escalade suit le `fail_count` :

```mermaid
graph LR
    A["fail = 1"] --> B["restart"]
    C["fail = 2"] --> D["recreate"]
    E["fail >= 3"] --> F["purge"]
```

Chaque phase appelle `caelix_audit_event()` et `incident_record()` après exécution.

---

## Déploiement Blue/Green

Le déploiement blue/green permet des mises à jour zero-downtime. Un conteneur candidat est créé, validé, puis basculé.

### Flux complet

```mermaid
sequenceDiagram
    participant M as Manifest
    participant S as Caelix
    participant Old as Ancien (port 3000)
    participant New as Candidat (port 3001)
    participant D as Docker

    Note over S: Changement d'image détecté

    S->>M: Lire candidate_publish, preflight_cmd...
    S->>D: docker run (nouvelle image sur port 3001)
    D-->>New: Conteneur créé

    S->>S: Attendre candidate_grace (défaut: 2s)

    opt preflight_cmd défini
        S->>New: docker exec: preflight_cmd
        alt Exit code != 0
            S->>D: docker rm -f (candidat)
            S->>S: incident_record(critical, bluegreen_fail)
            Note over S: Ancien conteneur reste actif
        end
    end

    S->>New: deep_diagnose_name(health_url_candidate)

    alt Candidat SAIN
        S->>D: docker rename (ancien → temp)
        S->>D: docker rename (candidat → caelix-app)
        S->>D: docker rm -f (ancien/temp)
        S->>S: incident_record(info, bluegreen_switch)
        S->>S: reset_fail_count()
        Note over S: Bascule réussie !
    else Candidat EN PANNE
        S->>D: docker rm -f (candidat)
        S->>S: incident_record(critical, bluegreen_fail)
        Note over S: Ancien conteneur reste actif
    end
```

### Configuration complète

```ini
[mon-service]
image = myapp:v2.0.0                                      # Nouvelle image
rollout_strategy = blue_green                              # Activer blue/green
publish = 127.0.0.1:3000:3000                              # Port de production
candidate_publish = 127.0.0.1:3001:3000                    # Port temporaire du candidat (REQUIS)
health_url = http://127.0.0.1:3000/health                  # Santé de la production
health_url_candidate = http://127.0.0.1:3001/health        # Santé du candidat (optionnel)
preflight_cmd = python manage.py migrate                    # Commande pré-bascule (optionnel)
```

!!! warning "candidate_publish est obligatoire"
    Si `rollout_strategy = blue_green` mais que `candidate_publish` n'est pas défini, `bin/caelix doctor` signalera une erreur.

---

## Pause manuelle

### Problème résolu

Sans pause manuelle, un opérateur qui fait `docker stop caelix-web` verrait Caelix redémarrer le service immédiatement au prochain cycle.

### Fonctionnement

```mermaid
graph LR
    A["docker stop"] --> B{"exit code"}
    B -->|"0 / 137 / 143"| C["set_manual_pause()"]
    B -->|autre| D["repair normal"]
    C --> E["skip reconciliation"]
    E --> F["caelix resume → reprend"]
```

### Configuration

```ini
[mon-service]
manual_stop_pause = 1   # Activé par défaut
manual_stop_pause = 0   # Désactiver : toujours redémarrer
```

### Fonctions impliquées

| Fonction | Description |
|---|---|
| `manual_pause_state_path(app)` | Chemin du fichier flag |
| `is_manual_pause_active(app)` | Le service est-il en pause ? |
| `set_manual_pause(app, reason)` | Activer la pause |
| `clear_manual_pause(app)` | Désactiver la pause |
| `manual_stop_pause_enabled(app)` | La pause est-elle configurée ? |
| `exit_code_looks_like_manual_stop(code)` | Exit code = arrêt manuel ? |

---

## Config version

Pour forcer la re-création d'un conteneur sans changer l'image :

```ini
[mon-service]
config_version = 2   # Incrémenter cette valeur
```

Caelix compare le label `caelix.config_version` du conteneur avec la valeur du manifest. Si elles diffèrent, le conteneur est recréé (ou déployé en blue/green selon la stratégie).

Les fonctions `desired_config_version()` et `current_config_version()` gèrent cette comparaison.

---

## Suspension automatique

```ini
[mon-service]
create_fail_max_attempts = 3   # 0 = illimité (défaut)
```

```mermaid
graph LR
    F1["Échec création 1"] --> F2["Échec création 2"]
    F2 --> F3["Échec création 3"]
    F3 --> S["SUSPENDU<br>.suspend_reconcile créé"]
    S --> N["Notification critique"]
    S --> R["bin/caelix resume app"]
    R --> OK["Réconciliation reprend"]

    style S fill:#e74c3c,color:#fff
    style OK fill:#27ae60,color:#fff
```

La fonction `caelix_clear_suspend_state()` supprime les fichiers de suspension et remet le compteur à zéro.

---

## Fonctions du module repair.sh

| Fonction | Description |
|---|---|
| `reconcile_app(app)` | Point d'entrée principal de la réconciliation par service |
| `ensure_desired_revision(app)` | Vérifie image et config_version, déclenche rollout si nécessaire |
| `rollout_blue_green(app)` | Déploiement blue/green complet |
| `repair_execute(app, reason)` | Escalade de réparation (restart → recreate → purge) |
| `candidate_preflight(app, cname)` | Exécute la commande preflight dans le candidat |
| `create_candidate_name(app)` | Génère le nom du conteneur candidat |
| `detect_unexpected_restart(app)` | Compare le restart count avec l'état sauvegardé |
| `remove_orphan_containers()` | Supprime les conteneurs caelix-* non déclarés |
| `caelix_section_reserved(section)` | Vérifie si une section est réservée |
| `is_manual_pause_active(app)` | Vérifie la pause manuelle |
| `set_manual_pause(app, reason)` | Active la pause |
| `clear_manual_pause(app)` | Désactive la pause |
