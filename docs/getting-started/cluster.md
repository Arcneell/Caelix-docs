# Cluster multi-nœud

Caelix fonctionne en mono-hôte par défaut. Ce guide couvre l'activation du mode
cluster, l'ajout d'un nœud et le banc de test. Pour le fonctionnement interne, voir
[Architecture › Cluster](../architecture/cluster.md).

Consul et WireGuard sont embarqués et pilotés par Caelix : on n'interagit qu'avec
`caelix` et la console.

En cluster, **chaque nœud est un serveur Consul (quorum Raft) et un controller** :
la haute disponibilité est automatique. Avec **≥ 3 nœuds**, la perte du nœud leader est
absorbée (le quorum 2/3 survit), un autre nœud reprend le leadership et la **VIP** en
quelques secondes. Le **maillage WireGuard est obligatoire** : un nœud cluster ne peut
exister sans lui (l'installeur exige `wireguard-tools` et l'agent applique le maillage
automatiquement).

---

## 1. Activer le mode cluster

Le mode cluster s'active par variables d'environnement. Hors de ces variables,
Caelix reste mono-hôte et rien ne change.

| Variable | Rôle | Exemple |
|---|---|---|
| `CAELIX_CLUSTER_BACKEND` | `file` (mono-controller) ou `consul` (HA) | `consul` |
| `CAELIX_CLUSTER_STORE` | Chemin du store (backend `file`) | `/opt/caelix/.caelix/cluster` |
| `CAELIX_CONSUL_ADDR` | Adresse Consul (backend `consul`) | `http://127.0.0.1:8500` |
| `CAELIX_CONSUL_TOKEN` | Token ACL Consul (optionnel) | — |
| `CAELIX_NODE_ID` | Identité du nœud (sinon généré et persisté) | `node-a` |
| `CAELIX_NODE_ADDR` | IP du nœud sur le réseau cluster | `10.0.0.11` |
| `CAELIX_NODE_LABELS` | Étiquettes pour l'affinité (`k=v,k=v`) | `zone=eu,disk=ssd` |
| `CAELIX_NODE_TTL` | TTL du heartbeat en secondes | `30` |
| `CAELIX_DOCKER_ADDR` | Endpoint Docker publié (pour le ciblage depuis l'UI) | `tcp://10.0.0.11:2375` |
| `CAELIX_CONTROLLER` | `1` sur **tous** les nœuds cluster (HA : boucle leader élue par Consul) | `1` |
| `CAELIX_INGRESS` | `1` pour publier les backends / rafraîchir l'ingress | `1` |
| `CAELIX_WG_ENDPOINT` | Endpoint WireGuard annoncé aux pairs | `10.0.0.11:51820` |
| `CAELIX_CLUSTER_VIP` | VIP flottante portée par le leader (adresse stable console + ingress) | `10.0.0.10/32` |
| `CAELIX_VIP_IFACE` | Interface portant la VIP (défaut : interface de la route par défaut) | `enp5s0` |

Une façon propre de les poser est un fichier d'environnement
`/etc/caelix-cluster.env` chargé par les units systemd (voir §3).

---

## 2. Démarrer un nœud (agent)

Sur chaque nœud, le moteur tourne en **mode agent** : comme `caelix run`, plus la
publication de l'identité, du heartbeat et des backends.

```bash
# Génère (une fois) la paire de clés WireGuard ; affiche la clé publique
caelix mesh-keygen

# Applique le maillage WireGuard depuis les directives du store (root)
sudo caelix mesh-up

# Démarre l'agent (réconcilie le sous-manifeste poussé par le controller)
caelix agent
```

Commandes d'inspection utiles :

```bash
caelix node-info      # identité + méta-données du nœud (JSON)
caelix node-status    # écrit + affiche le statut de l'agent (meta + état observé)
```

Le **controller** est le backend FastAPI lancé avec `CAELIX_CONTROLLER=1` : il lit
l'état désiré et les nœuds vivants, planifie le placement et écrit un sous-manifeste
par nœud. En backend `consul`, il s'auto-élit leader ; en backend `file`, il est le
seul controller.

---

## 3. Exemple de unit systemd

```ini
# /etc/systemd/system/caelix-agent.service
[Unit]
Description=Caelix agent
After=docker.service
Wants=docker.service

[Service]
EnvironmentFile=/etc/caelix-cluster.env
Environment=CAELIX_DATA=/opt/caelix/.caelix
ExecStart=/bin/bash /opt/caelix/bin/caelix agent
Restart=always

[Install]
WantedBy=multi-user.target
```

Sur le nœud de contrôle, une seconde unit lance le backend avec
`CAELIX_CONTROLLER=1` (la boucle leader y est activée).

---

## 4. Ajouter ou supprimer un nœud

### Ajouter

Dans la console du controller, **Cluster › Ajouter un nœud** affiche la commande à
lancer sur la nouvelle machine. Sinon, directement :

```bash
# Installation guidée : choisir « rejoindre un cluster »
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --mode join --consul-addr http://<IP-controller>:8500

# Ou, si Caelix est déjà installé sur la machine :
caelix node join --consul-addr http://<IP-controller>:8500 --start
```

Le nœud génère ses clés WireGuard, s'enregistre dans le store, et apparaît dans la
vue **Cluster** ainsi que dans le **sélecteur de nœud** de l'en-tête.

### Supprimer

Dans la vue **Cluster**, le bouton **Supprimer** d'une ligne vide le nœud (ses
charges sont replanifiées ailleurs) puis le retire du cluster. Si des charges
épinglées (`pinned`) bloquent le drain, la console propose de forcer. Côté nœud,
`caelix node leave` arrête l'agent et démonte le maillage.

---

## 5. Cibler un nœud depuis la console

En mode cluster, un **sélecteur de nœud** apparaît dans l'en-tête de la console.
Choisir un nœud route **toutes** les actions adossées à Docker (conteneurs, images,
volumes, réseaux, stacks, logs, métriques, déploiements) vers le démon de ce nœud.
Le choix est mémorisé et recharge les vues. « Controller (local) » revient au démon
local.

Techniquement, la console envoie un en-tête `X-Caelix-Node` que le backend résout
vers l'endpoint Docker du nœud — voir [Architecture › Cluster §9](../architecture/cluster.md#9-ciblage-dun-nud).

---

## 6. VIP de cluster (adresse stable)

Les IP des nœuds peuvent changer (DHCP) et le leader peut basculer. Pour offrir une
**adresse d'accès stable** à la console et à l'ingress, Caelix gère une **VIP
flottante** portée par le nœud **leader** :

- Définir `CAELIX_CLUSTER_VIP` (ex. `10.0.0.10/32`) sur les nœuds controller-éligibles
  — à l'install : `--mode controller --vip 10.0.0.10/32`.
- L'**agent** du leader pose la VIP sur son interface (`CAELIX_VIP_IFACE`, sinon
  l'interface de la route par défaut) et émet un ARP gratuit ; les autres nœuds la
  relâchent. La console (`:18100`) et le proxy d'ingress (`:80`), qui écoutent en
  `0.0.0.0`, répondent alors sur la VIP.
- Le verrou Consul garantit **un seul leader** (pas de split-brain). À la bascule, le
  nouveau leader reprend la VIP en ~1–2 ticks ; le leader sortant (ou en perte de bail)
  la relâche. La VIP configurée est aussi **publiée dans le store**, pour qu'un nœud
  promu leader la reprenne sans reconfiguration.

```bash
caelix vip-status     # leader courant, VIP, interface, détention locale
```

!!! note "Failover automatique"
    Tous les nœuds d'un cluster sont controller-éligibles (`CAELIX_CONTROLLER=1`) et
    serveurs Consul : la bascule est **automatique** dès **≥ 3 nœuds** (quorum Raft
    maintenu à la perte d'un nœud). Le nouveau leader reprend la VIP en ~quelques
    secondes ; le maillage WireGuard reste actif sur les nœuds survivants.

---

## 7. Monter un cluster réel

Sur chaque VM ou serveur, installez via le paquet officiel (voir
[Installation](installation.md)) :

```bash
# Controller (1er nœud) — démarre Consul, console + controller, porte la VIP
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32

# Workers
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join --consul-addr http://<IP-controller>:8500
```

Puis, sur chaque nœud : `caelix mesh-keygen` et `sudo caelix mesh-up`. Les nœuds
apparaissent dans la vue **Cluster** de la console, joignable sur la VIP
(`http://10.0.0.10:18100`).
