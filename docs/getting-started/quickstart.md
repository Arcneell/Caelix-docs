# Premier lancement

Ce guide vous accompagne pour lancer Caelix avec un premier service.

## 1. Créer le manifest

```bash
cd caelix
cp etc/manifest.ini.example etc/manifest.ini
```

Éditez `etc/manifest.ini` pour définir votre premier service :

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1
log_level = info

[web]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
health_expect_codes = 200
health_timeout = 5
repair_strategy = auto
```

## 2. Valider la configuration

```bash
bin/caelix validate
```

Sortie attendue : aucune erreur. Pour un diagnostic plus complet :

```bash
bin/caelix doctor
```

## 3. Lancer un premier passage

```bash
bin/caelix once
```

Caelix va :

1. Charger le manifest
2. Détecter le runtime (Docker ou Podman)
3. Constater que le conteneur `caelix-web` n'existe pas
4. Le créer avec l'image `nginx:latest` et le port `8080:80`
5. Lancer un health check sur `http://127.0.0.1:8080/`
6. Afficher le résultat

## 4. Vérifier le résultat

```bash
# Voir le statut de Caelix
bin/caelix status

# Voir le conteneur créé
docker ps | grep caelix-web
```

Votre service Nginx est maintenant accessible sur `http://localhost:8080`.

## 5. Lancer le daemon

Pour que Caelix surveille vos services en continu :

```bash
bin/caelix run
```

La boucle de réconciliation va s'exécuter toutes les 10 secondes (configurable via `interval`).

!!! tip "Arrêter le daemon"
    `Ctrl+C` pour arrêter. En mode systemd : `sudo systemctl stop caelix`.

## 6. Tester la réparation automatique

Simulez une panne en arrêtant manuellement le conteneur :

```bash
docker stop caelix-web
```

Au prochain cycle de réconciliation, Caelix détectera que le conteneur est arrêté et le redémarrera automatiquement.

!!! note "Pause manuelle"
    Si `manual_stop_pause = 1` (défaut), Caelix ne redémarrera pas un conteneur arrêté manuellement. Utilisez `bin/caelix resume web` pour reprendre la réconciliation.

## Étape suivante

- [Configuration du manifest](../configuration/manifest.md) pour ajouter plus de services
- [Health checks](../modules/health.md) pour configurer la surveillance
- [Notifications Discord](../configuration/notifications.md) pour recevoir les alertes
