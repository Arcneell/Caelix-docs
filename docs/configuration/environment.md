# Variables d'environnement

Caelix peut être configuré via des variables d'environnement en complément du manifest.

## Variables principales

| Variable | Défaut | Description |
|---|---|---|
| `CAELIX_MANIFEST` | `etc/manifest.ini` | Chemin vers le fichier manifest |
| `CAELIX_NOTIFY_CONF` | `etc/notify.ini` | Chemin vers la configuration des notifications |
| `CAELIX_DATA` | `./.caelix` | Répertoire de données d'exécution |
| `CAELIX_INTERVAL` | `15` | Intervalle de réconciliation en secondes |
| `CAELIX_MAX_REPAIR` | `5` | Seuil d'échecs avant alerte |
| `CAELIX_LOG_LEVEL` | `info` | Niveau de log : `debug`, `info`, `warn`, `error` |
| `CAELIX_STRICT_LOCAL` | `0` | Si `1`, les URLs de health check doivent cibler localhost |

## Runtime conteneur (démon local ou distant)

Par défaut, Caelix pilote le démon Docker (ou Podman) **local** via son socket. Ces
variables permettent de cibler un démon **distant**, y compris en TLS — première
brique de la prise en charge multi-nœud. **Non définies, le comportement single-node
est strictement inchangé.**

| Variable | Défaut | Description |
|---|---|---|
| `CAELIX_RUNTIME` | _(auto)_ | Force le moteur : `docker` ou `podman`. Auto-détection sinon. |
| `CAELIX_DOCKER_HOST` | _(vide)_ | Démon cible (ex. `tcp://10.0.0.5:2376`). Si défini, est propagé aux variables Docker standard (`DOCKER_HOST`, et `CONTAINER_HOST` pour Podman). Vide = socket local. |
| `CAELIX_DOCKER_TLS_VERIFY` | _(vide)_ | Si `1`, active la vérification TLS du client Docker (`DOCKER_TLS_VERIFY`). |
| `CAELIX_DOCKER_CERT_PATH` | _(vide)_ | Répertoire des certificats client TLS (`ca.pem`, `cert.pem`, `key.pem`), mappé sur `DOCKER_CERT_PATH`. |
| `CAELIX_RT_TIMEOUT` | `60` | Timeout (s) des commandes runtime courtes ; `0` désactive. Les verbes longs/streaming (`pull`, `run`, `exec`, `logs`…) ne sont jamais coupés. |
| `CAELIX_RUNTIME_PROBE_TIMEOUT` | `10` | Timeout (s) de la sonde de disponibilité du démon (`info`). |
| `CAELIX_NODE_ID` | _(auto)_ | Identité du nœud (mode `agent`, multi-nœud). Si absent, un id est généré une fois et persisté dans `.caelix/node/id`. Sanitizé en `[a-z0-9-]`. |
| `CAELIX_NODE_LABELS` | _(vide)_ | Labels de placement du nœud, `clé=valeur` séparés par des virgules (ex. `zone=eu-west,disk=ssd`). Exposés par `caelix node-info`. |
| `CAELIX_CLUSTER_STORE` | _(vide)_ | Racine du store cluster fichier (layout RFC §9). Si défini, l'agent (`caelix agent`) publie sa méta dans `nodes/<id>/meta.json` et adopte le sous-manifest `nodes/<id>/desired.ini` poussé par le controller. Vide = mode mono-hôte. |
| `CAELIX_CLUSTER_BACKEND` | `file` | Backend du store cluster côté controller : `file` (utilise `CAELIX_CLUSTER_STORE`) ou `consul` (KV Consul). |
| `CAELIX_CONSUL_ADDR` | `http://127.0.0.1:8500` | Adresse HTTP de l'agent Consul (si `CAELIX_CLUSTER_BACKEND=consul`). |
| `CAELIX_CONSUL_TOKEN` | _(vide)_ | Token ACL Consul (en-tête `X-Consul-Token`), pour un cluster Consul sécurisé par ACL. |

> Exemple — piloter un démon distant en TLS :
> `-e CAELIX_DOCKER_HOST=tcp://node-b:2376 -e CAELIX_DOCKER_TLS_VERIFY=1 -e CAELIX_DOCKER_CERT_PATH=/etc/caelix/certs`.
> La même cible est honorée par le moteur Bash **et** par les appels Docker de la console web.

## Console web et reverse proxy

Ces variables s'appliquent au conteneur de la console web (API FastAPI) lorsqu'il est placé derrière un reverse proxy (TLS, load balancer) :

| Variable | Défaut | Description |
|---|---|---|
| `CAELIX_FORWARDED_ALLOW_IPS` | `*` | IP/CIDR des proxys amont dont uvicorn accepte les en-têtes `X-Forwarded-*`. Restreindre à l'IP du reverse proxy empêche un client d'usurper `X-Forwarded-Proto`/`X-Forwarded-For`. |
| `CAELIX_TRUSTED_PROXY` | `0` | Si `1`, l'application fait confiance à `X-Forwarded-For` pour l'IP client (verrouillage anti-bruteforce, audit). À n'activer que derrière un proxy de confiance qui réécrit l'en-tête. |
| `CAELIX_FORCE_SECURE_COOKIE` | `0` | Si `1`, force l'attribut `Secure` sur le cookie de session même sans `X-Forwarded-Proto` (TLS terminé en amont qui ne propage pas l'en-tête). |
| `CAELIX_METRICS_PROTECT` | `0` | Si `1`, l'endpoint `/metrics` exige une authentification. |
| `CAELIX_CORS_ORIGINS` | _(vide)_ | Origines CORS autorisées, séparées par des virgules. Vide = same-origin uniquement (le plus sûr). |

> Exemple : derrière un nginx unique en `172.18.0.2`, lancer le conteneur avec `-e CAELIX_FORWARDED_ALLOW_IPS=172.18.0.2 -e CAELIX_TRUSTED_PROXY=1` durcit la confiance accordée aux en-têtes transférés.

## Priorité de configuration

Les variables d'environnement sont écrasées par les valeurs du manifest si elles sont définies dans `[orchestrator]` :

```
Variable d'env < manifest.ini [orchestrator]
```

Par exemple, si `CAELIX_INTERVAL=15` et que le manifest définit `interval = 10`, c'est `10` qui sera utilisé.

## Exemples d'utilisation

### Lancement avec un manifest personnalisé

```bash
CAELIX_MANIFEST=/etc/caelix/manifest.ini bin/caelix run
```

### Répertoire de données séparé

```bash
CAELIX_DATA=/var/lib/caelix bin/caelix run
```

### Mode debug

```bash
CAELIX_LOG_LEVEL=debug bin/caelix once
```

### Mode strict (health checks localhost uniquement)

```bash
CAELIX_STRICT_LOCAL=1 bin/caelix validate
```

Ce mode est utile pour s'assurer que les health checks ne ciblent pas des services distants, ce qui pourrait causer des faux positifs ou des fuites d'information.

## Service systemd

Le fichier `caelix.global.service` configure les variables pour un fonctionnement en tant que service :

```ini
[Service]
Environment=CAELIX_MANIFEST=/opt/caelix/etc/manifest.ini
Environment=CAELIX_NOTIFY_CONF=/opt/caelix/etc/notify.ini
```

Pour ajouter des variables supplémentaires :

```bash
sudo systemctl edit caelix
```

```ini
[Service]
Environment=CAELIX_LOG_LEVEL=debug
Environment=CAELIX_DATA=/var/lib/caelix
```
