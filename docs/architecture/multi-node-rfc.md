# RFC — Caelix multi-nœud (cluster HA)

| | |
|---|---|
| **Statut** | Brouillon — à débattre |
| **Date** | 2026-06-16 |
| **Auteur** | Arcneell |
| **Cible** | Cluster haute disponibilité conservant le moteur auto-réparateur Caelix |
| **Stack retenue** | Consul · WireGuard · 3 controllers co-localisés · NFSv4 · ingress maison (voir §1.1) |

---

## 1. Résumé (TL;DR)

Caelix est aujourd'hui un orchestrateur **mono-hôte** : le moteur Bash réconcilie un
`manifest.ini` contre le démon Docker **local**, l'état vit dans des fichiers
`.caelix/` + une base SQLite locale, et le load-balancing repose sur un proxy
`socat` qui bind des ports de l'hôte.

Cette RFC propose une architecture **controller + agents** qui :

- **conserve le moteur auto-réparateur** (`repair.sh`, `health.sh`, blue/green,
  autoscale) comme **exécuteur local sur chaque nœud** — c'est le différenciateur
  produit, il n'est pas remplacé par Swarm/Kubernetes ;
- ajoute un **plan de contrôle répliqué** (placement, reschedule sur panne, source
  de vérité partagée) pour obtenir une **vraie haute disponibilité** : un service
  survit à la perte de son nœud et est replanifié ailleurs automatiquement.

La transformation n'est **pas un port** mais un changement d'architecture majeur,
livrable par phases dont chacune a une valeur commerciale propre (multi-hôtes
managé → scale horizontal → HA complète).

## 1.1 Stack technique retenue

Les choix ci-dessous sont **arrêtés** (priorité : sécurité par défaut, efficacité,
cohérence avec le moteur existant). La justification détaillée et les alternatives
écartées figurent aux sections référencées.

| Brique | Décision | Pourquoi | Réf. |
|---|---|---|---|
| **Store / consensus / discovery** | **Consul** | Un seul binaire : KV Raft + élection de leader + service-discovery + health-checks + ACL + chiffrement gossip + TLS | §6.1 |
| **Réseau cross-host** | **Maillage WireGuard** (underlay chiffré) + sous-réseau conteneurs par nœud routé sur le mesh | Tout le trafic est-ouest chiffré par défaut, kernel-natif, faible surface ; indépendant de Swarm | §6.2 |
| **Ingress / LB** | **Proxy Caelix existant**, régénéré depuis Consul (façon `consul-template`) | Réutilise l'intégration certbot/domaines ; évite un second système TLS | §7 |
| **Topologie controller** | **3 controllers co-localisés** avec les 3 serveurs Consul ; leader élu, followers en lecture seule | Quorum Consul = 3 nœuds déjà requis ; survit à la perte d'un nœud sans intervention | §8.1 |
| **Stockage `shared`** | **NFSv4** d'abord (driver de volume Docker), trafic sur le mesh WireGuard ; CephFS/SFTP ensuite | Ubiquitaire et simple ; `pinned` reste le défaut sûr | §6.3 |
| **Sécurité contrôle** | **mTLS** partout + **ACL Consul** par nœud + **bail = autorité** (fencing) | Sûr par défaut, argument commercial | §12 |
| **Auth/UI (SQLite)** | Lecture partout, **écriture leader** ; migration KV/Postgres si la charge l'exige | Réutilise l'existant sans réécrire l'auth | §6.1 |
| **Install / ajout de nœud** | **Composants embarqués + join-token** (modèle k3s / Docker Swarm), une commande ou un bouton UI | Simplicité d'install = objectif produit : zéro commande Consul/WireGuard exposée | §1.2 |

## 1.2 Expérience d'installation & d'ajout de nœud (objectif produit)

!!! danger "Contrainte produit : simplicité d'installation de premier rang"
    Le multi-nœud Caelix doit s'installer **comme k3s ou Docker Swarm — pas comme
    Kubernetes/kubeadm**. Une commande pour démarrer, une commande (ou un bouton UI)
    pour ajouter un nœud. **L'utilisateur ne doit jamais taper une commande Consul ni
    `wg`** : ces composants sont des détails d'implémentation embarqués.

**Principe : tout est embarqué et auto-piloté.** Le binaire Caelix embarque et gère le
cycle de vie de Consul et de WireGuard (clés, interface, pairs, CA mTLS). L'opérateur
n'interagit qu'avec `caelix` et l'UI.

**Démarrer un cluster (1 nœud) :**

```bash
caelix cluster init
# → ce nœud devient controller ; Caelix démarre Consul en mode bootstrap,
#   génère la CA mTLS + les clés WireGuard, et affiche un join-token.
#   Un cluster à 1 nœud = exactement le comportement single-node actuel.
```

**Ajouter un nœud (une seule commande, comme `docker swarm join`) :**

```bash
caelix node join --token <join-token> <controller-ip>
# → Caelix configure tout seul : agent Consul, pair WireGuard (clé + route),
#   certificat mTLS (obtenu via le token), enregistrement du nœud.
#   Aucune édition de fichier, aucune commande tierce.
```

**Ajout guidé via l'UI :** bouton **« Ajouter un nœud »** → affiche la commande
copiable avec un **token à TTL court**, une vue temps réel des nœuds (santé, charge,
zone), et des actions **drain / remove** d'un nœud en un clic.

**Démarrage simple, montée en HA à la demande.** On peut démarrer avec **un seul
controller** (cas mono-serveur, le plus simple) ; le HA du plan de contrôle s'active
plus tard en promouvant deux nœuds supplémentaires :

```bash
caelix node promote <node-id>   # passe le plan de contrôle de 1 → 3 (quorum HA)
```

On n'impose donc **pas** 3 nœuds dès le jour 1 (contrairement à l'impression que peut
donner §8.1) : 3 nœuds est la cible **pour le HA du control-plane**, pas un prérequis
d'installation.

**Prérequis minimaux** (tout le reste est embarqué) : un Linux avec le module
**WireGuard** (présent dans les noyaux ≥ 5.6, standard aujourd'hui) et **Docker**.

**Sécurité du join** : le join-token est à **TTL court et révocable** ; il sert de
*bootstrap* pour obtenir un **certificat mTLS** par nœud (le token ne donne pas
d'accès durable). Rotation et révocation pilotées depuis l'UI. C'est l'équivalent
automatisé d'un bootstrap-token, sans la complexité opérationnelle.

---

## 2. Objectifs / Non-objectifs

### Objectifs

1. **HA des charges de travail** : la perte d'un nœud worker entraîne le
   redéploiement automatique de ses services stateless sur les nœuds survivants.
2. **HA du plan de contrôle** : pas de point de défaillance unique pour la prise
   de décision (élection de leader, état répliqué).
3. **Réutilisation maximale du moteur existant** : l'agent réutilise
   `reconcile_all`/`reconcile_app` quasiment tels quels, à partir d'un
   sous-manifest qui lui est propre.
4. **Continuité commerciale** : chaque phase est vendable seule.
5. **Compatibilité descendante** : un déploiement single-node reste un cas
   particulier (cluster à 1 nœud) sans régression.
6. **Installation & onboarding triviaux** : démarrer un cluster et ajouter un nœud
   doivent tenir en **une commande ou un bouton UI** (modèle k3s / Docker Swarm),
   sans jamais exposer Consul ni WireGuard (cf. §1.2). C'est un **critère
   d'acceptation produit**, pas un confort optionnel.

### Non-objectifs (pour cette itération)

- Remplacer le moteur par Docker Swarm ou Kubernetes.
- Le multi-région / multi-datacenter avec latence WAN élevée (on vise un cluster
  LAN/VPC à faible latence).
- La migration à chaud (live-migration) des conteneurs stateful : on traite le
  stateful par **pinning** ou **stockage partagé**, pas par déplacement d'état en
  cours d'exécution.
- Le multi-tenant fort (isolation réseau/quota par client) — hors périmètre.

---

## 3. Pourquoi Caelix est aujourd'hui mono-hôte

Chaque sous-système suppose « un seul hôte ». Inventaire des points qui doivent
changer :

| Sous-système | Hypothèse actuelle | Fichiers | Impact multi-nœud |
|---|---|---|---|
| Accès runtime | CLI `docker`/`podman` sur le **socket local** | `lib/runtime.sh` (`_rt`, `runtime_available`), `ui/backend/app/core/docker.py` (`runtime_bin`) | Doit viser N démons (agent local ou `DOCKER_HOST` TLS) |
| État | Fichiers `.caelix/` + SQLite WAL | `.caelix/state/`, `.caelix/autoscale/`, `auth.db` | Non partagé : un seul nœud peut décider |
| Réconciliation | Daemon unique, `flock` **global**, boucle sur les apps d'un hôte | `bin/caelix` (`reconcile_all`, `caelix_with_global_lock`) | Aucune notion de placement ni de « qui fait quoi » |
| Load-balancing | Proxy `socat` local, bind ports hôte, route vers conteneurs locaux | `lib/autoscale_proxy.sh`, `lib/proxy.sh` | `socat` ne traverse pas les hôtes |
| Nommage | `caelix-<app>`, répliques `…-rN` uniques **par hôte** | `lib/runtime.sh` (`caelix_cname`) | Collisions cluster-wide |
| Volumes | `volumes_bind` = chemins **hôte** | manifest, `container_create` | Un bind ne suit pas un conteneur replanifié |
| Placement | Inexistant | — | Nouveau sous-système |
| Heartbeat | `caelix-daemon-heartbeat` fichier local | `.caelix/state/` | Doit devenir une santé nœud observable par le cluster |

!!! note "Ce qui est réutilisable tel quel"
    Le **cœur intelligent** — escalade `restart → recreate → purge`
    (`repair.sh`), sondes HTTP/TCP (`health.sh`), blue/green, autoscale local,
    incidents, audit, notifications — fonctionne **par nœud** sans modification de
    fond. Ce sont les trois piliers d'un cluster (réseau cross-host, état
    partagé/consensus, stockage) qui sont absents et doivent être construits.

---

## 4. Architecture cible

```mermaid
flowchart TB
    UI[UI Vue 3 / API admin] --> LEADER

    subgraph CP["Plan de contrôle (HA — 3 nœuds, leader élu)"]
        LEADER["controller (leader)"]
        STBY1["controller (follower)"]
        STBY2["controller (follower)"]
        STORE[("Store répliqué Raft\nétat désiré + observé\nélection de leader")]
        LEADER --- STORE
        STBY1 --- STORE
        STBY2 --- STORE
    end

    LEADER -- "sous-manifest par nœud\n(watch / pull)" --> AGA
    LEADER -- "sous-manifest par nœud" --> AGB
    LEADER -- "sous-manifest par nœud" --> AGC

    subgraph W["Plan de données (workers)"]
        AGA["agent A\nmoteur Bash\n→ docker local"]
        AGB["agent B\nmoteur Bash\n→ docker local"]
        AGC["agent C\nmoteur Bash\n→ docker local"]
    end

    AGA <-- "heartbeat + état observé" --> STORE
    AGB <-- "heartbeat + état observé" --> STORE
    AGC <-- "heartbeat + état observé" --> STORE

    AGA -. "overlay réseau + ingress cross-host" .- AGB
    AGB -. .- AGC
```

**Principe directeur** : le `manifest.ini` global devient l'**état désiré du
cluster**. Le scheduler du controller le **découpe en sous-manifests par nœud**.
Chaque agent réconcilie son sous-manifest contre son Docker local — c'est-à-dire
qu'il exécute **le moteur Caelix actuel sans savoir qu'il est clusterisé**.

---

## 5. Composants

### 5.1 Controller (plan de contrôle)

Service (extension du backend FastAPI existant) qui, **uniquement sur le leader** :

- lit l'état désiré global (manifest cluster) depuis le store ;
- exécute le **scheduler** : décide sur quel(s) nœud(s) chaque app/réplique tourne,
  en fonction de l'affinité, des ressources, de l'anti-affinité et de la santé des
  nœuds ;
- écrit les **sous-manifests par nœud** dans le store ;
- surveille les **heartbeats** des agents ; à l'expiration d'un bail (lease), marque
  le nœud `down` et **replanifie** ses charges stateless ;
- sert l'UI et l'API (les followers peuvent servir en lecture seule et proxifier les
  écritures vers le leader).

### 5.2 Agent (plan de données, un par nœud)

Empaquetage du moteur Bash existant en service autonome :

- s'enregistre auprès du cluster (identité nœud, capacités, ressources) ;
- **watch** son sous-manifest dans le store ; à chaque changement, écrit un
  `manifest.ini` local et déclenche `reconcile_all` ;
- conserve son daemon de réconciliation périodique (l'auto-réparation locale reste
  active même si le controller est injoignable — **dégradation gracieuse**) ;
- publie son **heartbeat** et l'**état observé** de ses conteneurs dans le store ;
- enregistre ses backends pour le service-discovery (cf. §7).

!!! tip "Le verrou `flock` reste pertinent"
    `caelix_with_global_lock` sérialise les passes — il devient le **verrou local de
    l'agent** (un agent = un hôte). Aucun verrou distribué n'est requis *côté agent* ;
    la coordination cluster se fait via l'élection de leader du controller (§8).

### 5.3 Protocole controller ↔ agent

Modèle **pull/watch** (l'agent observe le store), préféré au push direct :

- découplage : le controller n'a pas à connaître l'adresse routable de chaque agent ;
- résilience : un agent qui redémarre re-synchronise en relisant le store ;
- HA naturelle : le basculement de leader n'interrompt pas les agents.

Sens des données :

| Sens | Donnée | Clé store (cf. §9) |
|---|---|---|
| controller → agent | sous-manifest désiré | `caelix/nodes/<id>/desired` |
| agent → controller | heartbeat + bail | `caelix/nodes/<id>/heartbeat` (TTL) |
| agent → controller | état observé (conteneurs, santé) | `caelix/nodes/<id>/observed` |
| agent → cluster | backends pour LB | `caelix/services/<app>/backends/<inst>` |

---

## 6. Décisions techniques — les trois problèmes durs

### 6.1 État partagé + consensus

**Problème.** Aujourd'hui : fichiers `.caelix/` locaux + SQLite. Pour du HA il faut
une source de vérité **répliquée** *et* une **élection de leader** ; sinon
split-brain (deux controllers replanifient la même app).

| Option | Avantages | Inconvénients |
|---|---|---|
| **Consul** *(retenu)* | KV Raft + élection de leader + health-checks + service-discovery + ACL + gossip chiffré + TLS, en un seul binaire ; Docker-natif | Dépendance externe à opérer (cluster 3 nœuds) |
| etcd | Référence k8s, robuste, watch performant | Pas de service-discovery/health intégré → à compléter |
| Postgres + advisory locks | Si une instance SQL est déjà exploitée ; SQL familier | HA SQL à gérer soi-même ; pas de watch natif (polling) ; pas de service-discovery |

**Décision : Consul.** Il couvre d'un coup les trois besoins — KV répliqué
(état désiré/observé), élection de leader (sessions + locks), et
service-discovery/health pour l'ingress (§7). SQLite (`auth.db`) reste pour
l'auth/UI : **lue partout, écrite par le leader** (puis répliquée, ou migrée vers le
KV/Postgres si la charge d'écriture le justifie).

### 6.2 Réseau cross-host + ingress

**Problème.** Aujourd'hui : `socat` bind sur ports de l'hôte et route vers conteneurs
**locaux** (`autoscale_proxy.sh`). Cela ne traverse pas les nœuds.

| Couche | Option | Note |
|---|---|---|
| Overlay | **Maillage WireGuard** (underlay chiffré) + sous-réseau conteneurs par nœud routé sur le mesh — **retenu** | Chiffré par défaut (ChaCha20-Poly1305), kernel-natif, faible surface ; indépendant de Swarm |
| Overlay | Docker overlay (Swarm-mode réseau seul) | Le plus natif, mais VXLAN **en clair** sauf `--opt encrypted` (IPsec, lourd) ; couple partiellement au mode Swarm |
| Overlay | Plugin CNI (flannel/cilium) | Puissant mais lourd pour le positionnement « léger » de Caelix |

**Décision : maillage WireGuard.** Le controller gère la configuration des pairs
(clés publiques, endpoints, sous-réseaux autorisés) et la distribue via le store ;
chaque nœud se voit attribuer un **sous-réseau conteneurs** (ex. `10.42.<n>.0/24`)
dont les routes transitent par le tunnel WireGuard. C'est le choix **le plus sûr** :
tout le trafic est-ouest (contrôle *et* données applicatives, y compris NFS) est
chiffré, sans dépendre du mode Swarm. À l'échelle visée (cluster LAN/VPC, nombre de
nœuds modéré), un maillage full-mesh piloté par le controller est efficace et simple
à raisonner. L'overlay Docker reste l'**alternative documentée** si un client refuse
WireGuard.

**Ingress : on fait évoluer le proxy existant.** `global_proxy_update_routes` génère
déjà `routes.conf` ; on remplace la source « ports locaux » par les **backends
découverts dans Consul** (régénération façon `consul-template` sur watch), de sorte
qu'un ingress sur n'importe quel nœud route — via le mesh — vers une réplique saine
sur n'importe quel autre nœud. On réutilise ainsi l'intégration TLS/certbot/domaines
existante (`proxy.sh`) au lieu d'introduire un second système (Traefik/Caddy reste
l'échappatoire si le besoin L7 dépasse le proxy actuel).

### 6.3 Stockage stateful + placement

**Problème.** `volumes_bind` = chemins **hôte**. Un bind ne suit pas un conteneur
replanifié → une app stateful ne peut pas migrer naïvement.

Deux modes exposés dans le manifest (cf. §10) :

- **`storage = pinned`** : l'app est **clouée à son nœud**. Simple, aucune dépendance
  ; mais **pas de HA pour cette app** (si le nœud meurt, l'app attend son retour). À
  réserver aux données qu'on ne veut pas répliquer.
- **`storage = shared`** : volume sur stockage partagé via driver de volume Docker.
  L'app **peut migrer** : le scheduler la replanifie sur un nœud ayant accès au même
  volume.

**Décision : NFSv4 comme premier backend `shared` supporté** (ubiquitaire, simple à
opérer, driver de volume Docker standard), son trafic transitant sur le mesh
WireGuard (donc chiffré). CephFS et SFTP sont prévus comme backends ultérieurs pour
les clients ayant des besoins de redondance/perf supérieurs. **`pinned` reste le
défaut** : on n'active `shared` qu'explicitement, pour ne pas créer de fausse
promesse de HA sur du stateful mal configuré.

Les apps **stateless** migrent librement (cible principale du HA en phase 1).

---

## 7. Découverte de services & load-balancing

Flux cible :

1. À chaque (re)déploiement, l'agent enregistre l'instance dans Consul :
   `caelix/services/<app>/backends/<node>-<replica> = host:port` + un health-check.
2. L'ingress de chaque nœud **watch** cette clé et régénère sa table de routage
   (`routes.conf`), filtrée sur les backends **sains**.
3. Une requête entrante sur n'importe quel nœud est routée — via l'overlay — vers
   une réplique saine, **quel que soit son nœud**.

Cela généralise l'autoscale actuel : les répliques `-r1/-r2` ne sont plus toutes sur
le même hôte mais **réparties**, et le proxy global devient un **ingress cluster**.

---

## 8. Haute disponibilité & gestion des pannes

### 8.1 Élection de leader (controller)

**Topologie retenue : 3 controllers co-localisés avec les 3 serveurs Consul** — mais
**uniquement quand on veut le HA du plan de contrôle**. Un cluster peut démarrer avec
**un seul controller** (install la plus simple, cf. §1.2) puis être promu à 3 via
`caelix node promote` sans interruption. Le quorum Consul exige déjà 3 nœuds ; on
réutilise ces mêmes nœuds plutôt que d'ajouter une couche séparée. Les controllers acquièrent un **lock de session Consul** ; le
détenteur est **leader** (scheduling + écritures de l'état désiré). Les deux
**followers** servent l'UI/API en **lecture seule** et **proxifient les écritures**
vers le leader. Perte de session (crash, partition) → le lock se libère → un follower
devient leader et reprend le scheduling. Les agents, en pull/watch, **ne sont pas
interrompus**. La perte d'un des 3 nœuds de contrôle préserve le quorum (2/3) et donc
la disponibilité du plan de contrôle.

### 8.2 Détection de panne nœud

Chaque agent renouvelle un **bail TTL** (`heartbeat`). Expiration → le leader marque
le nœud `down`, puis :

- **stateless** : replanifie les apps du nœud mort sur des nœuds sains (réécriture
  des sous-manifests cibles) ;
- **stateful `pinned`** : marque l'app `unavailable`, ouvre un incident, attend le
  retour du nœud (ou intervention) ;
- **stateful `shared`** : replanifie sur un nœud ayant accès au volume partagé.

### 8.3 Prévention du split-brain

- **Une seule autorité d'écriture** : seul le leader écrit l'état désiré → pas de
  décision concurrente.
- **Quorum** : le store (Consul/etcd) exige la majorité ; un nœud minoritaire dans
  une partition **ne peut pas** devenir leader.
- **Fencing applicatif** : avant de replanifier une charge, le leader **révoque le
  bail** du nœud suspect ; l'agent qui perd son bail **suspend ses créations** pour
  éviter le double-démarrage (réutilise la logique de suspension `suspend_reconcile`
  existante).

### 8.4 Dégradation gracieuse

Si le plan de contrôle est entièrement injoignable, **chaque agent continue
d'auto-réparer localement** selon son dernier sous-manifest connu. On ne perd que la
capacité de **reschedule**, pas l'auto-réparation locale — propriété cohérente avec
l'ADN « self-healing » de Caelix.

---

## 9. Schéma d'état (layout Consul KV)

```text
caelix/
├── cluster/
│   ├── leader                      # lock de session (élection controller)
│   ├── desired/manifest            # manifest cluster (état désiré global)
│   └── config/                     # paramètres cluster (intervalles, politiques)
├── nodes/
│   └── <node-id>/
│       ├── meta                    # capacités, ressources, labels, zone
│       ├── heartbeat               # clé TTL (bail) — absence = nœud down
│       ├── desired                 # sous-manifest assigné par le scheduler
│       └── observed                # état réel (conteneurs, santé, restart_count)
└── services/
    └── <app>/
        ├── spec                    # résumé désiré (replicas, route, storage)
        └── backends/
            └── <node>-<replica>    # host:port + statut santé (pour l'ingress)
```

L'arborescence `.caelix/` locale **subsiste sur chaque agent** (logs, audit,
incidents, state de réconciliation locale) ; seules les clés ci-dessus sont
**cluster-wide**.

---

## 10. Extensions du manifest (placement)

Nouveaux champs par section d'app (rétro-compatibles : absents = comportement
single-node) :

```ini
[mon-app]
image = nginx:1.27
# --- placement cluster -------------------------------------------------
total_replicas   = 3              # nb total dans le cluster (remplace l'autoscale par-hôte)
node_affinity    = zone=eu-west   # contrainte de placement (label nœud)
anti_affinity    = mon-app        # ne pas co-localiser deux répliques sur le même nœud
storage          = shared         # pinned | shared (cf. §6.3)
shared_volume    = mon-app-data   # nom du volume partagé si storage=shared
max_per_node     = 1              # plafond de répliques par nœud
```

Le scheduler traduit ces contraintes en affectations `nodes/<id>/desired`.

---

## 11. Nommage cluster-wide

`caelix_cname()` produit aujourd'hui `caelix-<app>`, et les répliques `…-rN` sont
uniques **par hôte**. En cluster, le nom doit être unique **globalement** :

```
caelix-<app>-<node-id-court>-r<N>
```

Le label `caelix.app=<app>` (déjà posé par `container_create`) reste l'identifiant
logique ; on ajoute `caelix.node=<id>` et on conserve `caelix.candidate=1` pour le
nettoyage blue/green. Le service-discovery indexe par label, pas par nom — le nom
n'a donc qu'à être **unique et déterministe**.

---

## 12. Sécurité

Caelix étant un **produit commercial** dont la crédibilité sécurité est un argument
de vente, le plan de contrôle distribué doit être sûr **par défaut** :

- **mTLS agent ↔ controller ↔ store** : certificats par nœud, rotation ; aucun trafic
  de contrôle en clair sur le réseau.
- **Distribution de secrets** : les secrets ne transitent pas dans les sous-manifests
  en clair ; référencés par clé, résolus côté agent depuis un magasin chiffré (Consul
  + chiffrement au repos, ou Vault en option). Prolonge la logique `write_text_secret`
  / fichiers `0600` déjà en place.
- **Bail = autorité** : un agent sans bail valide **ne crée pas** de conteneur
  (anti double-start, §8.3).
- **ACL du store** : chaque agent n'a accès qu'à `nodes/<son-id>/*` et à la lecture du
  service-discovery ; seul le controller écrit `desired`.
- **Surface réseau** : l'overlay (WireGuard) chiffre le trafic est-ouest entre nœuds.

---

## 13. Compatibilité descendante & migration

- **Cluster à 1 nœud = single-node actuel.** Si aucun champ de placement n'est défini
  et qu'un seul agent est enregistré, le comportement est identique à aujourd'hui.
- **Mode dégradé sans store** : on peut conserver un chemin « standalone » (pas de
  Consul) qui retombe exactement sur le moteur mono-hôte — utile pour les petits
  clients et le développement.
- **Migration** : un déploiement existant devient le **premier nœud** du cluster ;
  on ajoute des agents progressivement ; le manifest existant fonctionne sans
  modification jusqu'à ce qu'on introduise les champs de placement.

---

## 14. Roadmap de dé-risquage

Ordre conçu pour que **la partie risquée (consensus/HA) arrive en dernier**, une fois
le reste prouvé, et pour que **chaque phase soit vendable**.

| Phase | Contenu | Valeur livrée | Effort |
|---|---|---|---|
| **0 — Abstraction runtime** ✅ | `_rt`/`run_cmd` ciblent un démon distant via `CAELIX_DOCKER_HOST`/`_TLS_VERIFY`/`_CERT_PATH` (→ `DOCKER_*`), défaut = socket local inchangé | Base technique ; débloque tout | _fait_ |
| **1 — Agent autonome** 🚧 | Empaqueter le moteur en agent réconciliant son local depuis un sous-manifest reçu. **Livré** : identité de nœud (`caelix node-info`), commande `caelix agent`, publication du statut `agent/status.json` (meta + observed, le contrat lu par le controller en §9). **Reste** : abstraction de la source du sous-manifest (fichier → store) + enregistrement du nœud, en phase 2 | — | en cours |
| **2 — Controller mono-instance + store** 🚧 | Manifest cluster, scheduler statique, push par nœud. **Livré** : cœur de placement Python (`core/cluster/` — `schedule`, découpe, `FileStore` §9, `apply_cluster`) **+ boucle agent↔store** (`CAELIX_CLUSTER_STORE`) **+ API/UI controller** (`/api/cluster/*` ; vue « Cluster ») **+ backend Consul** (`ConsulStore` KV controller + **agent Bash lisant/écrivant dans Consul** via `curl`, sélection `CAELIX_CLUSTER_BACKEND=consul`) → chemin Consul **bout-en-bout**. | **« Multi-hôtes managé »** (console unique sur N serveurs) | ✅ |
| **3 — Réseau cross-host + ingress dynamique** 🚧 | Maillage WireGuard + ingress (proxy existant) alimenté par Consul. **Livré** : registre de services (`register/deregister_backend`, `backends_for`, `service_apps`) + `build_routes` + `GET /api/cluster/routes` **+ publication des backends par l'agent** (`node_publish_backends`) **+ alimentation de l'ingress** (`CAELIX_INGRESS=1`) **+ plan de maillage WireGuard** (`core/cluster/mesh.py` : `assign_subnets` 10.42.N.0/24 déterministe, `mesh_overview`, `render_wg_config` wg0.conf depuis les métas des pairs ; méta nœud `wg_pubkey`/`wg_endpoint` ; `GET /api/cluster/mesh`). **+ application système `wg`/`ip` côté agent** (`publish_mesh` pousse des directives sans secret par nœud ; `caelix mesh-keygen`/`mesh-up`/`mesh-down` appliquent le maillage avec la clé privée locale ; recette validée par `run-mesh.sh`). **Reste** : refresh ingress depuis Consul, harnais dind end-to-end | **Scale horizontal** (répliques réparties derrière un LB) | en cours |
| **4 — HA du controller** 🚧 | Élection de leader, reschedule sur node-down, anti split-brain. **Livré** : heartbeat + liveness + reschedule (`apply_cluster(live_ttl=…)`, `/apply?live=1`, `/nodes`+`/status`) **+ élection de leader** (session/lock Consul ; `FileStore` mono-instance ; `GET /api/cluster/leader`) **+ fencing anti-split-brain** (`node_agent_cycle`) **+ boucle controller leader-gated** (`core/cluster/loop.py` : `controller_tick` n'agit que si leader ; `controller_loop` daemon session+reschedule périodique ; `CAELIX_CONTROLLER=1` dans le lifespan backend). | **Vrai HA** | ✅ |
| **5 — Stateful** 🚧 | Modes `pinned`/`shared`, drain de nœud propre. **Livré** : placement storage-aware (`AppPlacement.storage`, pinned = instance unique non relocalisable → `pending` si son nœud tombe ; shared/stateless migrent) + **drain de nœud** (`set_node_drain`/`node_draining`, exclu du placement par `apply_cluster`, `POST /api/cluster/nodes/<id>/drain`, flag dans `/nodes`). **Reste** : volume `shared` réel (driver NFS), drain bloquant tant que des pinned restent | HA des apps à état | en cours |

!!! note "L'onboarding est transversal"
    L'expérience d'install/join (§1.2) n'est **pas une phase** : `caelix cluster init`
    arrive en phase 1, `caelix node join` + le bouton UI en phase 2, `caelix node
    promote` (montée HA) en phase 4. À chaque phase, la règle « une commande / un
    bouton, zéro Consul/WireGuard exposé » est un critère d'acceptation.

!!! warning "Point de non-retour"
    Les phases 0→2 sont incrémentales et peu risquées (l'état reste fondamentalement
    centralisé). La **phase 4** introduit le consensus distribué : c'est là que se
    concentrent la complexité et le risque (split-brain, partitions, fencing). Ne pas
    l'aborder avant d'avoir validé 0→3 en conditions réelles.

---

## 15. Plan de tests

- **Unitaire/moteur (existant)** : les suites BATS (`tests/bats/`) restent valides —
  l'agent exécute le même moteur. Ajouter des tests de génération `manifest.ini`
  local à partir d'un sous-manifest.
- **Scheduler** : tests sur les contraintes (affinité, anti-affinité, `max_per_node`,
  ressources insuffisantes → app `pending`).
- **Chaos / pannes** : tuer un agent → vérifier reschedule des stateless ; tuer le
  leader → vérifier basculement sans interruption des agents ; **partition réseau** →
  vérifier l'absence de double-start (fencing).
- **Réseau** : requête entrante sur nœud A routée vers réplique sur nœud B.
- **Charge** : N nœuds × M apps, mesurer la latence de reschedule et la fréquence des
  watches.
- **Compat** : un manifest single-node existant déployé sur un cluster à 1 nœud =
  zéro régression (rejouer la suite actuelle).
- **Intégration mono-hôte** : `tests/integration/run.sh` valide la traversée réelle
  agent (Bash) ↔ **Consul réel** ↔ controller (Python) — enregistrement, placement,
  registre de backends, reschedule sur node-down, élection de leader — sur une seule
  machine, sans dind ni WireGuard (hors CI).
- **Data-plane WireGuard** : `tests/integration/run-mesh.sh` (root) valide que
  `render_wg_config` produit une config établissant un **vrai tunnel chiffré** entre
  deux nœuds (network namespaces + underlay veth) — sous-réseaux déterministes,
  ping cross-nœud, handshake. Validé sur l'hôte de dev. Reste : `dockerd` par nœud
  (dind) pour le routage ingress cross-host bout-en-bout.

---

## 16. Risques & questions ouvertes

| Risque | Mitigation |
|---|---|
| Complexité du consensus distribué (split-brain) | Déléguer le Raft à Consul/etcd ; ne pas l'écrire soi-même ; fencing par bail |
| Stateful sans stockage partagé = fausse promesse de HA | Documenter clairement `pinned` (pas de HA) vs `shared` ; ne pas survendre |
| Dépendance opérationnelle (cluster Consul à exploiter) | Conserver le mode standalone ; fournir un install packagé du quorum |
| **Install perçue comme « usine à gaz » (effet Kubernetes)** | Composants embarqués + join-token + UI guidée (§1.2) ; **test d'acceptation : ajout d'un nœud en < 2 min, une commande** ; ne jamais exposer Consul/WireGuard |
| Réseau overlay = nouvelle surface de panne/sécurité | mTLS + WireGuard ; tests de partition |
| Charge d'écriture sur SQLite multi-nœud | Lecture partout / écriture leader ; migrer vers KV/Postgres si nécessaire |

**Décisions actées** (cf. §1.1) :

1. **Store de consensus → Consul.**
2. **Réseau cross-host → maillage WireGuard** + sous-réseau conteneurs par nœud.
3. **Topologie controller → 3 controllers co-localisés** avec les serveurs Consul,
   leader élu + followers en lecture seule.
4. **Stockage `shared` → NFSv4** en premier backend ; `pinned` par défaut.
5. **Ingress → proxy Caelix existant** régénéré depuis Consul.

**Questions restant ouvertes (n'engagent pas l'architecture) :**

- Seuils par défaut des baux/TTL heartbeat (compromis détection rapide vs faux
  positifs sur réseau chargé) — à calibrer en phase 4.
- Migration éventuelle de l'auth SQLite vers le KV/Postgres si la charge d'écriture
  multi-nœud le justifie (réévaluer après la phase 2).

---

## 17. Alternatives écartées

- **Déléguer à Docker Swarm / Kubernetes** : HA et réseau « gratuits », mais le moteur
  auto-réparateur — cœur de valeur du produit — passerait au second plan. Écarté car
  contraire à l'objectif « garder mon moteur ».
- **Multi-hôtes sans HA (un seul controller, état non répliqué)** : moins coûteux,
  mais ne fournit pas la vraie HA demandée. Conservé uniquement comme **phase
  intermédiaire** (phase 2) et comme mode standalone.
