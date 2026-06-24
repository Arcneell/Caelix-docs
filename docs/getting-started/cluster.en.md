# Multi-node cluster

Caelix runs single-host by default. As of **2.0**, cluster mode is **highly available
by design** and is operated **like a single host** from the console. This guide covers
installing a real cluster (bootstrap node + joining nodes), verifying it, deploying a
**cluster service**, and failover. For the internals, see
[Architecture › Cluster](../architecture/cluster.md).

Consul and WireGuard are embedded and driven by Caelix: you only interact with
`caelix` and the console. **The installer sets up every prerequisite** (Docker, socat,
`wireguard-tools` + `modprobe wireguard`, `arping` for the VIP): a cluster node is
self-contained.

In a cluster, **every node is a Consul server (Raft quorum) and a controller**
(`CAELIX_CONTROLLER=1`): high availability is automatic. With **≥ 3 nodes**, losing the
leader node is absorbed (the 2/3 quorum survives), another node takes over leadership
and the **VIP** within seconds. The **WireGuard mesh is mandatory**: a cluster node
cannot exist without it (the installer requires `wireguard-tools` and the agent applies
the mesh automatically).

!!! note "Cluster / VIP strictly opt-in"
    Single-host stays the **default** mode and is unchanged. Cluster and the VIP are
    only enabled via `--mode controller|join` (and `--vip …`). Without those options,
    Caelix behaves exactly as in 1.x.

!!! warning "2.0 image (beta)"
    HA clustering ships in **2.0**. Pull the image
    `ghcr.io/arcneell/caelix:2.0.0-beta.1` (or the `:beta` channel). `:latest` stays on
    stable 1.x. To authenticate to the registry, see [Installation](installation.en.md).

---

## 1. Prerequisites

- At least **3 Linux machines/VMs** for real automatic failover (the Raft quorum is
  kept when one node is lost). 1 or 2 nodes work but without a fault-tolerant quorum.
- Private network connectivity between nodes (private IPs carry Consul `:8500`, dockerd
  `:2375` and the WireGuard mesh `:51820`).
- A **free VIP address** on the nodes' subnet (e.g. `10.0.0.10`), which becomes the
  stable address for the console and the ingress.
- Access to the Caelix registry (`docker login ghcr.io`, see [Installation](installation.en.md)).

Everything else (Docker, socat, WireGuard, arping, Consul, Compose) is installed
automatically by `install.sh`.

---

## 2. Install the bootstrap node (controller)

On the **first** machine, run the installer in `controller` mode. This node bootstraps
the Consul raft, starts the console + controller loop, and **carries the VIP**.

```bash
docker run --rm ghcr.io/arcneell/caelix:2.0.0-beta.1 cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32 \
      --cluster-size 3 --admin-password 'ChangeMe-Strong'
```

Useful cluster install options:

| Option | Role |
|---|---|
| `--mode controller` | Bootstrap node: bootstraps Consul (`-bootstrap-expect=1`), starts the console + controller |
| `--vip 10.0.0.10/32` | Floating VIP carried by the leader (stable console + ingress address) |
| `--cluster-size 3` | Number of expected Consul servers (HA quorum, default **3**) |
| `--admin-password <pw>` | Initial admin password. **Same value on every node** → consistent admin after failover. Otherwise a strong password is generated and printed once in the console logs |
| `--consul-token <tok>` | Consul ACL token (production hardening) |
| `--ui-bind <addr>` | Console listen address (default `0.0.0.0`) |

!!! tip "Admin password"
    Without `--admin-password`, retrieve the generated one once:
    `docker logs caelix-caelix-ui | grep -i password`.

When done, the node exposes Consul (`caelix-consul`), writes `/etc/caelix-cluster.env`
and starts the agent via systemd (`caelix-agent.service`). The console is reachable on
the VIP: **http://10.0.0.10:18100**.

---

## 3. Join the other nodes

On **each** additional node, run the installer in `join` mode, pointing at the
bootstrap node's Consul address (its private IP, port 8500):

```bash
docker run --rm ghcr.io/arcneell/caelix:2.0.0-beta.1 cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join \
      --consul-addr http://<controller-IP>:8500 \
      --admin-password 'ChangeMe-Strong'
```

Pass the **same** `--admin-password` as the controller to keep a consistent admin after
failover. The node joins the Consul gossip (`-retry-join`), grows the Raft quorum,
becomes controller-eligible too, generates its WireGuard keys and applies the mesh
automatically.

!!! note "Cluster forces systemd"
    Outside single-host, `--with-systemd` is implicit: the agent runs as a service
    (`caelix-agent.service`) and loads `/etc/caelix-cluster.env`.

If Caelix is **already installed** on a machine, you can also have it join without
re-running the installer:

```bash
caelix node join --consul-addr http://<controller-IP>:8500 --start
```

---

## 4. Verify the cluster

1. **Console on the VIP** — open **http://10.0.0.10:18100** (the VIP, whichever node is
   leader). The **Cluster** view must list **all 3 live nodes**.
2. **VIP status** — on any node:

   ```bash
   caelix vip-status     # current leader, VIP, interface, local ownership
   ```

   Exactly one node (the leader) holds the VIP.
3. **VIP reachable** — from another machine on the network:

   ```bash
   curl -I http://10.0.0.10:18100     # console
   curl -I http://10.0.0.10:80        # ingress (once a service is published)
   ```

Node selector: in cluster mode a selector appears in the console header. Picking a node
routes **all** Docker-backed actions (containers, images, volumes, networks, stacks,
logs, metrics, deployments) to that node's daemon, via the `X-Caelix-Node` header.
"Controller (local)" returns to the local daemon.

---

## 5. Deploy a cluster service (the essentials)

A **cluster service** is described in the cluster manifest (via the console: an app
section of the manifest). Unlike a single-host service, you declare a **total number of
replicas** that the scheduler spreads across nodes, and an **ingress route** that the
leader's global proxy serves on the VIP.

Keys of a cluster app section:

| Key | Role |
|---|---|
| `image` | Container image (e.g. `nginx:latest`) |
| `total_replicas` | Total number of replicas to spread across nodes |
| `publish` | `<hostport>:<containerport>` — the ingress backend is `<node-addr>:<hostport>` |
| `autoscale_route` | Ingress route key. Use **`default`** for the catch-all `VIP:80` route |
| `anti_affinity` | (optional) App name → at most 1 replica per node |
| `node_affinity` | (optional) Pins replicas onto labeled nodes |
| `max_per_node` | (optional) Cap on replicas per node |

!!! note "Default health = none"
    A service with no explicit `health_type` **defaults to `none`** in a cluster: it is
    not "repaired to death". Crash recovery goes through `create_missing`. Declare an
    explicit probe (`health_type = http`, `health_url`, …) to enable monitoring.

### Example: nginx served on the VIP

```ini
[web]
image = nginx:latest
total_replicas = 3
publish = 8080:80
autoscale_route = default
anti_affinity = web
```

The scheduler places 3 replicas (1 per node thanks to `anti_affinity`), each published
on `<node>:8080`. The **leader's global proxy** load-balances `VIP:80` across these
backends and **drops dead ones**. The service is then reachable at:

```bash
curl http://10.0.0.10/        # → nginx response, whichever node is leader
```

---

## 6. Horizontal autoscale (HPA)

To adjust `total_replicas` automatically based on CPU load, add to the app section:

| Key | Role |
|---|---|
| `hpa = 1` | Enables horizontal autoscale |
| `hpa_min` / `hpa_max` | Replica count bounds |
| `hpa_target` | Target CPU utilization (%) |
| `hpa_cooldown` | Delay (s) between two adjustments |

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

The leader measures replica CPU, adjusts `total_replicas` (between `hpa_min` and
`hpa_max`), the scheduler re-places, and the ingress load-balances the new backends.

---

## 7. Failover

Every node is controller-eligible and a Consul server. The **Consul lock** guarantees a
**single leader** (no split-brain). When the leader is lost:

- with **≥ 3 nodes**, the 2/3 quorum survives and another node is elected leader;
- the new leader **takes over the VIP** within ~seconds (adds it to its interface +
  gratuitous ARP); the outgoing leader releases it;
- the **console state is shared via Consul** (login, users, config, stacks, certs): the
  console is identical after failover;
- the **WireGuard mesh** stays active on the surviving nodes;
- the ingress (`VIP:80`) keeps serving the still-alive backends.

```bash
caelix vip-status     # confirms the new leader and VIP ownership
```

The configured VIP is **published to the store**: a promoted node takes it over without
reconfiguration.

---

## 8. Remove a node

In the **Cluster** view, a row's **Remove** button drains the node (its workloads are
rescheduled elsewhere) then removes it from the cluster. If pinned workloads block the
drain, the console offers to force it. On the node:

```bash
caelix node leave     # stops the agent and tears down the mesh
```

---

## 9. Security hardening (production)

At install, **dockerd `:2375` and the Consul API `:8500` are bound to the node's private
IP** (never `0.0.0.0`), and a best-effort firewall restricts 2375 to private networks.
That is the first barrier, but **not sufficient on its own in production**:

- The **Consul KV is the control plane**: it holds the JWT secret, password hashes and
  TLS keys. Anyone who reaches the Consul API can read them.
- In production, the operator **must** enable:
  - **Consul ACLs** (`default_policy = deny` + token via `CAELIX_CONSUL_TOKEN`, passed at
    install with `--consul-token`);
  - **gossip encryption**;
  - **TLS** on the Consul API and **mTLS** on dockerd.

---

## 10. Reference: cluster environment variables

The installer writes `/etc/caelix-cluster.env`, loaded by `caelix-agent.service`. For
manual or advanced tuning:

| Variable | Role | Example |
|---|---|---|
| `CAELIX_CLUSTER_BACKEND` | `file` (single controller) or `consul` (HA) | `consul` |
| `CAELIX_CONSUL_ADDR` | Consul address (`consul` backend) | `http://127.0.0.1:8500` |
| `CAELIX_CONSUL_TOKEN` | Consul ACL token (optional) | — |
| `CAELIX_NODE_ID` | Node identity (generated and persisted otherwise) | `node-a` |
| `CAELIX_NODE_ADDR` | Node IP on the cluster network | `10.0.0.11` |
| `CAELIX_NODE_LABELS` | Labels for affinity (`k=v,k=v`) | `zone=eu,disk=ssd` |
| `CAELIX_DOCKER_ADDR` | Published Docker endpoint (`X-Caelix-Node` targeting) | `tcp://10.0.0.11:2375` |
| `CAELIX_CONTROLLER` | `1` on **all** cluster nodes (leader loop elected via Consul) | `1` |
| `CAELIX_INGRESS` | `1` to publish backends / refresh the ingress | `1` |
| `CAELIX_WG_ENDPOINT` | WireGuard endpoint advertised to peers | `10.0.0.11:51820` |
| `CAELIX_CLUSTER_VIP` | Floating VIP held by the leader | `10.0.0.10/32` |
| `CAELIX_VIP_IFACE` | Interface carrying the VIP (default: the default-route interface) | `enp5s0` |

Inspection commands: `caelix node-info`, `caelix node-status`, `caelix vip-status`.
