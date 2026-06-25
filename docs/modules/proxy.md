# Proxy & Load Balancer

Le module `proxy.sh` implémente un reverse-proxy TCP basé sur socat. Il assure un round-robin health-aware, le hot-reload des backends et des endpoints d'introspection (health, state, metrics Prometheus, routes).

---

## Architecture

```mermaid
graph LR
    client["Client HTTP"] --> proxy["Proxy Caelix<br>(socat)"]
    proxy --> backend1["caelix-app-r1<br>:18501 ✓ sain"]
    proxy --> backend2["caelix-app-r2<br>:18502 ✓ sain"]
    proxy -.->|ignoré| backend3["caelix-app-r3<br>:18503 ✗ en panne"]

    subgraph Health["Health Loop (background)"]
        hloop["_proxy_health_loop()"]
        hloop -->|"curl /health"| backend1
        hloop -->|"curl /health"| backend2
        hloop -->|"curl /health"| backend3
    end

    style proxy fill:#9b59b6,color:#fff
    style backend1 fill:#27ae60,color:#fff
    style backend2 fill:#27ae60,color:#fff
    style backend3 fill:#e74c3c,color:#fff
    style Health fill:#2c3e50,color:#fff
```

Le proxy :

- Distribue le trafic en round-robin entre les backends sains uniquement.
- Ignore automatiquement les backends en panne.
- Se recharge à chaud quand les fichiers backends ou routes changent.
- Écrit un événement dans `events.queue` quand un backend change d'état.

---

## Modes de fonctionnement

### Mode legacy (par service)

Un proxy indépendant pour chaque service autoscalé :

```ini
[mon-service]
autoscale = 1
autoscale_lb_publish = 127.0.0.1:8080:80
```

Le proxy lit le fichier `.caelix/autoscale/<app>.backends`.

### Mode global

Un seul proxy pour tous les services :

```ini
[proxy]
listen = 0.0.0.0:8080           # Adresse d'écoute
autoscale_port_range = 18500-18999
health_interval = 3              # Intervalle de health check (sec)
connect_timeout = 5              # Timeout connexion backend (sec)
log_level = info                 # debug, info, warn, error
```

Le proxy lit le fichier `.caelix/autoscale/routes.conf` pour déterminer vers quel service router chaque requête.

### Routage

L'ordre de matching dans `_proxy_route_lookup()` :

```mermaid
graph LR
    A["Requête"] --> B{"host: match ?"}
    B -->|oui| C["Backend du service"]
    B -->|non| D{"path: prefix match ?"}
    D -->|oui| C
    D -->|non| E{"route default ?"}
    E -->|oui| C
    E -->|non| F["503"]
    C --> G["Round-robin health-aware"]
```

Configuration des routes par service :

```ini
# Routage par hostname (header Host)
[web]
autoscale_route = host:www.example.com

# Routage par préfixe de chemin URL
[api]
autoscale_route = path:/api

# Port dédié (proxy séparé lancé sur ce port)
[admin]
autoscale_route = port:9090

# Route par défaut (catch-all)
[fallback]
autoscale_route = default
```

La fonction `_proxy_route_lookup()` effectue le matching dans l'ordre : host → path → default.

### Ingress cluster / VIP (mode 2.0)

En mode cluster, le proxy global tourne sur le nœud leader et sert l'ingress sur
`VIP:80` (la VIP flottante, cf. [variables d'environnement](../configuration/environment.md#vip-de-cluster-ingress-flottante) `CAELIX_CLUSTER_VIP` / `CAELIX_VIP_IFACE`). Il
load-balance le trafic à travers les backends des services cluster publiés par
chaque nœud.

Mécanique de bout en bout :

1. Chaque agent qui a une `CAELIX_NODE_ADDR` publie, pour ses services, un backend
   `<CAELIX_NODE_ADDR>:<hostport>` dans le registre de services (clé `publish` du
   manifest cluster).
2. Sur un nœud ingress (`CAELIX_INGRESS=1`), `build_routes` lit le registre et regroupe,
   par clé de route (`autoscale_route`, ou le nom de l'app), la liste dédupliquée et
   triée des adresses de backend de tous les nœuds. Plusieurs apps partageant la même
   clé sont fusionnées derrière une seule route. `default` est la route catch-all
   servie sur `VIP:80`.
3. À chaque passe, `routes.conf` est régénéré depuis le registre puis relu par
   connexion. Un changement de route (replica ajouté ou retiré, nœud disparu) ne
   nécessite aucun redémarrage du proxy.
4. Les backends morts sont écartés : la boucle de health du proxy sonde chaque
   backend et le round-robin n'envoie le trafic qu'aux backends sains.

Comme en mono-hôte, ce proxy global gère aussi le LB par application de l'autoscale et
le routage par domaine/TLS. L'ingress cluster réutilise le même moteur `proxy.sh`.

---

## Endpoints intégrés

Le proxy intercepte certaines requêtes pour exposer son état :

### GET /caelix-proxy/health

Retourne `200 OK` si au moins un backend est sain, `503 Service Unavailable` sinon.

### GET /caelix-proxy/state

JSON détaillant chaque backend avec son statut :

```json
{
  "backends": [
    {"name": "caelix-web-r1", "host": "127.0.0.1", "port": 18501, "healthy": true},
    {"name": "caelix-web-r2", "host": "127.0.0.1", "port": 18502, "healthy": true},
    {"name": "caelix-web-r3", "host": "127.0.0.1", "port": 18503, "healthy": false}
  ]
}
```

En mode global, inclut tous les routes et leurs backends respectifs.

### GET /caelix-proxy/metrics

Métriques au format Prometheus :

```
caelix_proxy_requests_total{backend="caelix-web-r1"} 1523
caelix_proxy_requests_total{backend="caelix-web-r2"} 1487
caelix_proxy_errors_total{backend="caelix-web-r1"} 3
caelix_proxy_backend_healthy{backend="caelix-web-r1"} 1
caelix_proxy_backend_healthy{backend="caelix-web-r3"} 0
```

### GET /caelix-proxy/routes

Table de routage actuelle (mode global uniquement).

---

## Boucle de health check

La fonction `_proxy_health_loop()` tourne en arrière-plan et vérifie périodiquement chaque backend :

```mermaid
sequenceDiagram
    participant H as Health Loop
    participant B1 as Backend R1
    participant B2 as Backend R2
    participant F as Fichier health_*
    participant Q as events.queue

    loop Toutes les health_interval secondes
        H->>B1: curl autoscale_health_path
        B1-->>H: 200 OK
        H->>F: health_caelix-web-r1 = 1

        H->>B2: curl autoscale_health_path
        B2-->>H: Timeout
        H->>F: health_caelix-web-r2 = 0

        alt Changement d'état détecté
            H->>Q: proxy_backend_down caelix-web-r2
        end
    end
```

Les fichiers d'état se trouvent dans `.caelix/autoscale/global-proxy/health_<replica>` (valeur : `0` ou `1`).

Les événements de transition sont écrits dans `events.queue` et traités par `autoscale_process_proxy_events()` au prochain cycle de réconciliation.

---

## Hot-reload

Le proxy surveille ses fichiers de configuration :

| Fichier | Quand il change | Effet |
|---|---|---|
| `.caelix/autoscale/<app>.backends` | Scale up/down, replica recréée | Backends mis à jour sans interruption |
| `.caelix/autoscale/routes.conf` | Manifest modifié, service ajouté/retiré | Routes mises à jour |

Le rechargement se fait par relecture du fichier, sans redémarrer le processus proxy.

---

## Sélection de backend (round-robin)

La fonction `_proxy_pick_backend()` :

1. Lit le fichier backends
2. Filtre les backends marqués comme sains (`health_*` = 1)
3. Sélectionne le prochain backend en round-robin atomique (`_proxy_rr_next()` avec flock)
4. Retourne `name host port` ou erreur si aucun backend sain

---

## Gestion des processus

| Fichier PID | Usage |
|---|---|
| `.caelix/autoscale/global-proxy.pid` | PID du proxy global |
| `.caelix/autoscale/<app>-lb.pid` | PID du proxy legacy par service |
| `.caelix/autoscale/<app>-dedicated.pid` | PID du proxy dédié (route port:N) |

Les fonctions `global_proxy_running()` et `autoscale_lb_running()` vérifient que le processus est vivant.

En cas de crash, `autoscale_reconcile()` détecte l'absence et relance le proxy, avec un incident `autoscale_lb_restarted` si un PID file existait avant.

---

## Variables d'environnement du proxy

Le proxy est configuré via des variables d'environnement passées à son processus :

| Variable | Défaut | Description |
|---|---|---|
| `CAELIX_PROXY_LISTEN` | `0.0.0.0:8080` | Adresse d'écoute |
| `CAELIX_PROXY_BACKENDS` | — | Fichier backends (mode legacy) |
| `CAELIX_PROXY_ROUTES` | — | Fichier routes (mode global) |
| `CAELIX_PROXY_HEALTH_INTERVAL` | `3` | Intervalle de health check (sec) |
| `CAELIX_PROXY_HEALTH_PATH` | `/` | Chemin HTTP pour les probes |
| `CAELIX_PROXY_HEALTH_TIMEOUT` | `2` | Timeout de probe (sec) |
| `CAELIX_PROXY_CONNECT_TIMEOUT` | `5` | Timeout de connexion (sec) |
| `CAELIX_PROXY_LOG_LEVEL` | `info` | Niveau de log |
| `CAELIX_PROXY_STATE_DIR` | — | Répertoire d'état (health, metrics) |
| `CAELIX_PROXY_APP` | — | Nom de l'application (pour le logging) |

---

## Fonctions du module proxy.sh

| Fonction | Description |
|---|---|
| `_proxy_handle_connection()` | Traite une connexion HTTP unique (routing, relay, metrics) |
| `_proxy_pick_backend(bfile, state_dir)` | Round-robin sur backends sains |
| `_proxy_route_lookup(routes, host, path)` | Matching de route (host/path/default) |
| `_proxy_health_loop(bfile, state_dir)` | Boucle de health check en arrière-plan |
| `_proxy_read_backends(bfile, state_dir)` | Lecture backends avec statut santé |
| `_proxy_build_state_json(...)` | JSON pour /caelix-proxy/state |
| `_proxy_build_metrics(...)` | Prometheus pour /caelix-proxy/metrics |
| `_proxy_build_metrics_global(...)` | Prometheus multi-routes |
| `_proxy_atomic_inc(file, delta)` | Compteur atomique (flock) |
| `_proxy_rr_next(file, count)` | Index round-robin atomique |
| `proxy_log(level, message)` | Logging du proxy |
