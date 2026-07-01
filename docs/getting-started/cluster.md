# Cluster multi-nœud

Caelix fonctionne en mono-hôte par défaut. Depuis la 2.2, le mode cluster est
haute disponibilité et se pilote depuis la console comme un mono-hôte. Ce guide
couvre l'installation d'un cluster réel (nœud d'amorçage puis nœuds qui
rejoignent), sa vérification, le déploiement d'un service cluster et la bascule
(failover). Pour le fonctionnement interne, voir
[Architecture › Cluster](../architecture/cluster.md).

etcd et WireGuard sont embarqués et pilotés par Caelix : on n'interagit qu'avec
`caelix` et la console. L'installeur met en place tous les prérequis (Docker,
socat, `wireguard-tools` avec `modprobe wireguard`, `arping` pour la VIP), de
sorte qu'un nœud cluster est autonome.

En cluster, chaque nœud est un membre etcd (quorum Raft) et un controller
(`CAELIX_CONTROLLER=1`), et la haute disponibilité est automatique. Avec au moins
3 nœuds, la perte du nœud leader est absorbée (le quorum 2/3 survit) : un autre
nœud reprend le leadership et la VIP en quelques secondes. Le maillage WireGuard
est obligatoire. Un nœud cluster ne peut exister sans lui : l'installeur exige
`wireguard-tools` et l'agent applique le maillage automatiquement.

!!! note "Cluster et VIP opt-in"
    Le mono-hôte reste le mode par défaut et ne change pas. Le cluster et la VIP
    ne s'activent que via `--mode controller|join` (et `--vip …`). Sans ces
    options, le comportement de Caelix est identique à la 1.x.

Le cluster HA est livré dans la 2.2, désormais publiée sous `:latest`. Les
exemples ci-dessous tirent `ghcr.io/arcneell/caelix:latest`. Pour authentifier le
registry, voir [Installation](installation.md).

---

## 1. Prérequis

- Au moins 3 machines ou VM Linux pour un vrai failover automatique (le quorum
  Raft est maintenu à la perte d'un nœud). 1 ou 2 nœuds fonctionnent, mais sans
  quorum tolérant aux pannes.
- Connectivité réseau privée entre les nœuds. Les IP privées portent etcd
  (`:2379` API client, `:2380` pairs Raft), dockerd (`:2375`) et le maillage
  WireGuard (`:51820`).
- Une adresse VIP libre sur le sous-réseau des nœuds (par exemple `10.0.0.10`),
  qui servira d'adresse stable pour la console et l'ingress.
- Accès au registry Caelix (`docker login ghcr.io`, voir
  [Installation](installation.md)).

Tout le reste (Docker, socat, WireGuard, arping, etcd, Compose) est installé
automatiquement par `install.sh`.

---

## 2. Installer le nœud d'amorçage (controller)

Sur la première machine, lancez l'installeur en mode `controller`. Ce nœud
bootstrappe le raft etcd, lance la console et la boucle controller, et porte la
VIP.

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32 \
      --cluster-size 3 --admin-password 'ChangeMoi-Fort'
```

Options cluster utiles à l'install :

| Option | Rôle |
|---|---|
| `--mode controller` | Nœud d'amorçage : bootstrappe etcd (cluster initial à un membre), lance la console + controller |
| `--vip 10.0.0.10/32` | VIP flottante portée par le leader (adresse stable console + ingress) |
| `--cluster-size 3` | Nombre de membres etcd attendus (quorum HA, défaut 3) |
| `--admin-password <pw>` | Mot de passe admin initial. La même valeur sur tous les nœuds donne un admin cohérent après bascule. Sinon un mot de passe fort est généré et affiché une fois dans les logs de la console |
| `--ui-bind <addr>` | Adresse d'écoute de la console (défaut `0.0.0.0`) |

!!! tip "Mot de passe admin"
    Sans `--admin-password`, récupérez le mot de passe généré une fois :
    `docker logs caelix-caelix-ui | grep -i password`.

À la fin, le nœud expose etcd (`caelix-etcd`), écrit `/etc/caelix-cluster.env`
et démarre l'agent via systemd (`caelix-agent.service`). La console est joignable
sur la VIP : **http://10.0.0.10:18100**.

---

## 3. Faire rejoindre les autres nœuds

Sur chaque nœud supplémentaire, lancez l'installeur en mode `join` en pointant
l'adresse du store etcd du nœud d'amorçage (son IP privée, port 2379) :

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join \
      --store-addr http://<IP-controller>:2379 \
      --admin-password 'ChangeMoi-Fort'
```

Passez le même `--admin-password` qu'au controller pour garder un admin cohérent
après bascule. L'installeur enregistre le nœud dans le cluster etcd
(`etcdctl member add`, automatique), grossit le quorum Raft, devient lui aussi
controller-éligible, génère ses clés WireGuard et applique le maillage
automatiquement.

!!! note "Le cluster force systemd"
    Hors mono-hôte, `--with-systemd` est implicite : l'agent tourne en service
    (`caelix-agent.service`) et charge `/etc/caelix-cluster.env`.

Si Caelix est déjà installé sur une machine, on peut aussi la faire rejoindre sans
relancer l'installeur :

```bash
caelix node join --store-addr http://<IP-controller>:2379 --start
```

---

## 4. Vérifier le cluster

1. Console sur la VIP : ouvrez **http://10.0.0.10:18100** (la VIP, quel que soit
   le leader). La vue **Cluster** doit lister les 3 nœuds vivants.
2. Statut VIP, sur n'importe quel nœud :

   ```bash
   caelix vip-status     # leader courant, VIP, interface, détention locale
   ```

   Un seul nœud (le leader) détient la VIP.
3. VIP joignable, depuis une autre machine du réseau :

   ```bash
   curl -I http://10.0.0.10:18100     # console
   curl -I http://10.0.0.10:80        # ingress (une fois un service publié)
   ```

Ciblage des nœuds : il n'y a pas de sélecteur de nœud global dans la console.
Chaque action cible le nœud de sa ligne, et les vues de ressources agrègent
l'ensemble des nœuds — la console résout et route automatiquement vers le démon
qui héberge la ressource, via l'en-tête `X-Caelix-Node` en coulisse. En mono-hôte,
la console se simplifie (pas de section Nodes, pas de colonne Node).

---

## 5. Déployer un service cluster (l'essentiel)

Un service cluster se décrit dans le manifeste cluster (via la console, dans la
section d'app du manifeste). Contrairement à un service mono-hôte, on déclare un
nombre total de réplicas que le scheduler répartit sur les nœuds, et une route
ingress que le proxy global du leader sert sur la VIP.

Clés d'une section d'app cluster :

| Clé | Rôle |
|---|---|
| `image` | Image du conteneur (ex. `nginx:latest`) |
| `total_replicas` | Nombre total de réplicas à répartir sur les nœuds |
| `publish` | `<hostport>:<containerport>` — le backend ingress est `<adresse-nœud>:<hostport>` |
| `autoscale_route` | Clé de route ingress. Utilisez `default` pour la route fourre-tout `VIP:80` |
| `anti_affinity` | (optionnel) Nom d'app → 1 réplica par nœud max |
| `node_affinity` | (optionnel) Épingle les réplicas sur des nœuds étiquetés |
| `max_per_node` | (optionnel) Plafond de réplicas par nœud |

!!! note "Health par défaut = none"
    Un service sans `health_type` explicite **défaut à `none`** en cluster : il
    n'est pas « réparé à mort ». La reprise au crash passe par `create_missing`.
    Déclarez une sonde explicite (`health_type = http`, `health_url`, …) pour
    activer la surveillance.

### Exemple : nginx servi sur la VIP

```ini
[web]
image = nginx:latest
total_replicas = 3
publish = 8080:80
autoscale_route = default
anti_affinity = web
```

Le scheduler place 3 réplicas (1 par nœud grâce à `anti_affinity`), chacun publié
sur `<nœud>:8080`. Le proxy global du leader load-balance `VIP:80` sur ces
backends et retire les backends morts. Le service est alors joignable sur :

```bash
curl http://10.0.0.10/        # → réponse nginx, quel que soit le leader
```

---

## 6. Autoscale horizontal (HPA)

Pour ajuster `total_replicas` automatiquement selon la charge CPU, ajoutez à la
section d'app :

| Clé | Rôle |
|---|---|
| `hpa = 1` | Active l'autoscale horizontal |
| `hpa_min` / `hpa_max` | Bornes du nombre de réplicas |
| `hpa_target` | Cible d'utilisation CPU (%) |
| `hpa_cooldown` | Ticks consécutifs entre deux ajustements |

```ini
[web]
image = nginx:latest
total_replicas = 3
publish = 8080:80
autoscale_route = default
anti_affinity = web
hpa = 1
hpa_min = 2
hpa_max = 8
hpa_target = 70
hpa_cooldown = 60
```

Le leader mesure le CPU des réplicas, ajuste `total_replicas` (entre `hpa_min` et
`hpa_max`), le scheduler replace et l'ingress load-balance les nouveaux backends.

---

## 7. Bascule (failover)

Tous les nœuds sont controller-éligibles et membres etcd. Le bail (lease) etcd
garantit un seul leader, donc pas de split-brain. À la perte du leader :

- avec au moins 3 nœuds, le quorum 2/3 survit et un autre nœud est élu leader ;
- le nouveau leader reprend la VIP en quelques secondes (pose sur son interface et
  ARP gratuit) ; le leader sortant la relâche ;
- à l'arrêt gracieux d'un agent (systemd `ExecStop`, drain), la VIP est relâchée
  proprement et bascule immédiatement, sans attendre l'expiration du bail — en
  complément de la bascule sur perte de bail lors d'une panne brutale ;
- l'état console est partagé via etcd (login, utilisateurs, config, stacks,
  certs), donc la console reste identique après bascule ;
- le maillage WireGuard reste actif sur les nœuds survivants ;
- l'ingress (`VIP:80`) continue de servir les backends encore vivants.

```bash
caelix vip-status     # confirme le nouveau leader et la détention de la VIP
```

La VIP configurée est publiée dans le store : un nœud promu leader la reprend sans
reconfiguration.

---

## 8. Retirer un nœud

Dans la vue **Cluster**, le bouton **Supprimer** d'une ligne draine le nœud (ses
charges sont replanifiées ailleurs) puis le retire du cluster. Si des charges
épinglées (`pinned`) bloquent le drain, la console propose de forcer. Côté nœud :

```bash
caelix node leave     # arrête l'agent et démonte le maillage
```

---

## 9. Durcissement sécurité (production)

À l'install, dockerd (`:2375`) et l'API etcd (`:2379`) sont liés à l'IP privée
du nœud (jamais `0.0.0.0`), et un pare-feu best-effort restreint 2375 aux réseaux
privés. C'est la première barrière, mais elle est insuffisante seule en
production :

- Le KV etcd est le plan de contrôle : il contient le secret JWT, les hash de
  mots de passe et les clés TLS. Quiconque atteint l'API etcd peut les lire.
- En production, l'opérateur **doit** activer :
  - l'authentification etcd (RBAC : utilisateurs, rôles et `auth enable`) ;
  - TLS client et TLS pair (`:2380`) sur etcd ;
  - mTLS sur dockerd.

---

## 10. Référence : variables d'environnement cluster

L'installeur écrit `/etc/caelix-cluster.env`, chargé par `caelix-agent.service`.
Pour un réglage manuel ou avancé :

| Variable | Rôle | Exemple |
|---|---|---|
| `CAELIX_CLUSTER_BACKEND` | `file` (mono-controller) ou `etcd` (HA) | `etcd` |
| `CAELIX_ETCD_ADDR` | Adresse etcd (backend `etcd`) | `http://127.0.0.1:2379` |
| `CAELIX_NODE_ID` | Identité du nœud (sinon généré et persisté) | `node-a` |
| `CAELIX_NODE_ADDR` | IP du nœud sur le réseau cluster | `10.0.0.11` |
| `CAELIX_NODE_LABELS` | Étiquettes pour l'affinité (`k=v,k=v`) | `zone=eu,disk=ssd` |
| `CAELIX_DOCKER_ADDR` | Endpoint Docker publié (ciblage `X-Caelix-Node`) | `tcp://10.0.0.11:2375` |
| `CAELIX_CONTROLLER` | `1` sur tous les nœuds cluster (boucle leader élue par etcd) | `1` |
| `CAELIX_INGRESS` | `1` pour publier les backends / rafraîchir l'ingress | `1` |
| `CAELIX_WG_ENDPOINT` | Endpoint WireGuard annoncé aux pairs | `10.0.0.11:51820` |
| `CAELIX_CLUSTER_VIP` | VIP flottante portée par le leader | `10.0.0.10/32` |
| `CAELIX_VIP_IFACE` | Interface portant la VIP (défaut : interface de la route par défaut) | `enp5s0` |

Commandes d'inspection : `caelix node-info`, `caelix node-status`,
`caelix vip-status`.
