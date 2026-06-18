# Cluster multi-nœud

Caelix fonctionne en mono-hôte par défaut. Ce guide couvre l'activation du mode
cluster, l'ajout d'un nœud et le banc de test. Pour le fonctionnement interne, voir
[Architecture › Cluster](../architecture/cluster.md).

Consul et WireGuard sont embarqués et pilotés par Caelix : on n'interagit qu'avec
`caelix` et la console.

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
| `CAELIX_CONTROLLER` | `1` sur les nœuds de contrôle (active la boucle leader) | `1` |
| `CAELIX_INGRESS` | `1` pour publier les backends / rafraîchir l'ingress | `1` |
| `CAELIX_WG_ENDPOINT` | Endpoint WireGuard annoncé aux pairs | `10.0.0.11:51820` |

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

## 4. Ajouter un nœud

1. Installer Caelix sur le nouveau nœud (même version que le cluster).
2. Renseigner `/etc/caelix-cluster.env` : `CAELIX_CLUSTER_BACKEND=consul`,
   `CAELIX_CONSUL_ADDR`, un `CAELIX_NODE_ID` unique, `CAELIX_NODE_ADDR`,
   `CAELIX_DOCKER_ADDR`, `CAELIX_WG_ENDPOINT`.
3. `caelix mesh-keygen` puis `sudo caelix mesh-up`.
4. Démarrer `caelix-agent`. Le nœud publie sa meta, le controller le voit vivant et
   commence à y placer des charges.

Le nœud apparaît alors dans la vue **Cluster** de la console, et dans le **sélecteur
de nœud** de l'en-tête.

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

## 6. Banc de test Incus

Un banc multi-nœud **réel** est fourni dans `tests/integration/vmlab/` : il crée des
**VM Incus** (KVM), installe Docker + WireGuard + Caelix, et monte un cluster
3 nœuds (1 controller + Consul, 2 workers) avec Consul et WireGuard réels.

```bash
cd tests/integration/vmlab
./up.sh         # build du frontend, création + provisioning des VM, forwards console
# … tester via la console (port-forward) …
./down.sh       # détruit les VM
```

Le provisioning (`provision.sh`) écrit `/etc/caelix-cluster.env`, expose le `dockerd`
de chaque nœud en TCP (pour que le controller cible les nœuds depuis l'UI) et
installe les units `caelix-agent` + `caelix-ui`. Voir le `README.md` du dossier pour
les détails et l'accès console.

!!! warning "TCP Docker = réseau isolé uniquement"
    Le banc expose `dockerd` en TCP **clair** sur le réseau isolé des VM. En
    production, restreindre cet endpoint au **sous-réseau WireGuard + mTLS**.
