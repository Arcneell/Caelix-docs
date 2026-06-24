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
| `CAELIX_NODE_ADDR` | _(vide)_ | Adresse d'annonce du nœud (IP joignable par les autres nœuds, sur le mesh ou le LAN). Sert d'hôte des backends publiés dans le registre de services (ingress). Sans elle, l'agent ne publie pas de backend. |
| `CAELIX_INGRESS` | `0` | Si `1`, le nœud agit comme ingress : à chaque cycle, le proxy global est alimenté par les backends cluster du registre (load-balancing cross-nœuds). Backends `file` **et** `consul`. |
| `CAELIX_WG_PUBKEY` | _(vide)_ | Clé publique WireGuard du nœud, publiée dans sa méta pour que le controller/pairs construisent le maillage. Vide tant que le mesh n'est pas configuré. |
| `CAELIX_WG_ENDPOINT` | _(vide)_ | Endpoint WireGuard du nœud (`host:port`) annoncé aux pairs. |
| `CAELIX_WG_IFACE` | `caelix-wg0` | Nom de l'interface WireGuard créée par `caelix mesh-up`. |
| `CAELIX_WG_LISTEN_PORT` | `51820` | Port d'écoute WireGuard local appliqué par `caelix mesh-up`. |
| `CAELIX_NODE_TTL` | `30` | Durée (s) du bail de heartbeat : au-delà sans renouvellement, le controller considère le nœud *down* (reschedule) et l'agent qui ne peut plus renouveler se *fence*. |
| `CAELIX_CONTROLLER` | `0` | Si `1`, le backend lance la **boucle controller** (HA) : acquisition du leadership puis reschedule périodique sur les nœuds vivants (seul le leader agit). À activer sur les nœuds de contrôle. |
| `CAELIX_CONTROLLER_INTERVAL` | `10` | Intervalle (s) entre deux passes de la boucle controller. |
| `CAELIX_CLUSTER_STORE` | _(vide)_ | Racine du store cluster fichier (layout RFC §9). Si défini, l'agent (`caelix agent`) publie sa méta dans `nodes/<id>/meta.json` et adopte le sous-manifest `nodes/<id>/desired.ini` poussé par le controller. Vide = mode mono-hôte. |
| `CAELIX_CLUSTER_BACKEND` | `file` | Backend du store cluster, **côté controller ET agent** : `file` (utilise `CAELIX_CLUSTER_STORE`) ou `consul` (KV Consul). L'agent (`caelix agent`) publie sa méta et lit son sous-manifest via ce backend. |
| `CAELIX_CONSUL_ADDR` | `http://127.0.0.1:8500` | Adresse HTTP de l'agent Consul (si `CAELIX_CLUSTER_BACKEND=consul`). Côté agent, `curl` est requis. |
| `CAELIX_CONSUL_TOKEN` | _(vide)_ | Token ACL Consul (en-tête `X-Consul-Token`), pour un cluster Consul sécurisé par ACL. |
| `CAELIX_DOCKER_ADDR` | _(vide)_ | Adresse TCP du démon Docker du nœud (ex. `tcp://10.0.0.5:2375`), publiée dans la méta du nœud. La console (et le HPA cluster) la consomment pour piloter le bon démon par nœud (en-tête `X-Caelix-Node`). |
| `CAELIX_PIN_LOCAL_SECTIONS` | _(vide)_ | Sections du manifeste local à **épingler** (CSV) : réinjectées après adoption du sous-manifest poussé par le controller (ex. `caelix-ui,proxy`), pour qu'un nœud garde ses services locaux. |

### VIP de cluster (ingress flottante)

La VIP flottante donne une adresse d'accès stable (console + ingress) portée par le
nœud **leader**. Voir le module [Proxy](../modules/proxy.md) et `node_vip_*` (lib/node.sh).

| Variable | Défaut | Description |
|---|---|---|
| `CAELIX_CLUSTER_VIP` | _(vide)_ | Adresse VIP du cluster en CIDR (ex. `10.0.0.10/32`). Posée sur l'interface par le nœud qui détient le leadership ; source de vérité de la VIP. Vide = pas de VIP. |
| `CAELIX_VIP_IFACE` | _(auto)_ | Interface portant la VIP. Si absente, l'interface de la route par défaut est utilisée. |
| `CAELIX_VIP_ARP` | `1` | Si `1`, émet un ARP gratuit (`arping`) lors de la prise de la VIP, pour que le LAN reroute immédiatement vers le nouveau leader. |
| `CAELIX_ADMIN_PASSWORD` | _(vide)_ | Mot de passe admin initial de la console, à fixer identique sur tous les nœuds (le hash est stocké dans le store cluster). Aussi posable via `install.sh --admin-password`. |

> **Sécurité (cluster)** — en mode cluster, `dockerd:2375` (cf. `CAELIX_DOCKER_ADDR`) et
> Consul `:8500` sont liés à l'**IP privée** du nœud. En production, activez les ACL
> Consul + token (`CAELIX_CONSUL_TOKEN`) + TLS : le KV Consul détient le secret JWT, les
> hash de mots de passe et les clés TLS.

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
