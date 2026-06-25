# Premier lancement

Ce guide vous accompagne pour lancer Caelix avec un premier service. Deux chemins
rapides selon votre cible : mono-hôte (défaut) ou cluster HA.

## Chemin rapide : mono-hôte

Installez Caelix sur un serveur Linux (voir [Installation](installation.md) pour le
`docker login` au registry) :

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd
```

Ouvrez la console sur **http://IP_DU_SERVEUR:18100** (login `admin` ; mot de passe
affiché dans `docker logs caelix-caelix-ui | grep -i password`, ou fixé via
`--admin-password`). Depuis la console, ajoutez vos services. En CLI, le flux manifeste
ci-dessous fait la même chose.

## Chemin rapide : cluster HA

Trois nœuds, image `:latest`. Sur le nœud d'amorçage :

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32 \
      --cluster-size 3 --admin-password 'ChangeMoi-Fort'
```

Sur chaque autre nœud :

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join \
      --store-addr http://<IP-controller>:2379 --admin-password 'ChangeMoi-Fort'
```

Ouvrez la console sur la VIP (`http://10.0.0.10:18100`), vérifiez les 3 nœuds dans
la vue **Cluster**, puis déployez un service cluster. Exemple nginx servi sur la VIP :

```ini
[web]
image = nginx:latest
total_replicas = 3
publish = 8080:80
autoscale_route = default
anti_affinity = web
```

Le service est joignable sur `http://10.0.0.10/`. Pour le guide complet (vérification,
`caelix vip-status`, HPA, bascule, durcissement), voir
[Cluster multi-nœud](cluster.md).

---

## Flux manifeste (mono-hôte, en détail)

Les étapes ci-dessous montrent le flux mono-hôte piloté par le manifeste (équivalent
CLI de l'ajout d'un service). Depuis une installation par image, le manifeste est dans
`/opt/caelix/etc/manifest.ini` et la commande globale est `caelix` ; les exemples
`bin/caelix` ci-dessous valent pour une checkout des sources.

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

La boucle de réconciliation s'exécute toutes les 10 secondes (configurable via `interval`).

!!! tip "Arrêter le daemon"
    `Ctrl+C` pour arrêter. En mode systemd : `sudo systemctl stop caelix`.

## 6. Tester la réparation automatique

Simulez une panne en arrêtant manuellement le conteneur :

```bash
docker stop caelix-web
```

Au prochain cycle de réconciliation, Caelix détecte que le conteneur est arrêté et le redémarre automatiquement.

!!! note "Pause manuelle"
    Si `manual_stop_pause = 1` (défaut), Caelix ne redémarre pas un conteneur arrêté manuellement. Utilisez `bin/caelix resume web` pour reprendre la réconciliation.

## Étape suivante

- [Configuration du manifest](../configuration/manifest.md) pour ajouter plus de services
- [Health checks](../modules/health.md) pour configurer la surveillance
- [Notifications Discord](../configuration/notifications.md) pour recevoir les alertes
