# Multi-node cluster

Caelix runs single-host by default. This guide covers enabling cluster mode, adding
a node, and the test bench. For the internals, see
[Architecture › Cluster](../architecture/cluster.md).

Consul and WireGuard are embedded and driven by Caelix: you only interact with
`caelix` and the console.

In a cluster, **every node is a Consul server (Raft quorum) and a controller**: high
availability is automatic. With **≥ 3 nodes**, losing the leader node is absorbed (the
2/3 quorum survives), another node takes over leadership and the **VIP** within
seconds. The **WireGuard mesh is mandatory**: a cluster node cannot exist without it
(the installer requires `wireguard-tools` and the agent applies the mesh automatically).

---

## 1. Enable cluster mode

Cluster mode is enabled through environment variables. Without them, Caelix stays
single-host and nothing changes.

| Variable | Role | Example |
|---|---|---|
| `CAELIX_CLUSTER_BACKEND` | `file` (single controller) or `consul` (HA) | `consul` |
| `CAELIX_CLUSTER_STORE` | Store path (`file` backend) | `/opt/caelix/.caelix/cluster` |
| `CAELIX_CONSUL_ADDR` | Consul address (`consul` backend) | `http://127.0.0.1:8500` |
| `CAELIX_CONSUL_TOKEN` | Consul ACL token (optional) | — |
| `CAELIX_NODE_ID` | Node identity (generated and persisted otherwise) | `node-a` |
| `CAELIX_NODE_ADDR` | Node IP on the cluster network | `10.0.0.11` |
| `CAELIX_NODE_LABELS` | Labels for affinity (`k=v,k=v`) | `zone=eu,disk=ssd` |
| `CAELIX_NODE_TTL` | Heartbeat TTL in seconds | `30` |
| `CAELIX_DOCKER_ADDR` | Published Docker endpoint (for UI targeting) | `tcp://10.0.0.11:2375` |
| `CAELIX_CONTROLLER` | `1` on **all** cluster nodes (HA: leader loop, elected via Consul) | `1` |
| `CAELIX_INGRESS` | `1` to publish backends / refresh the ingress | `1` |
| `CAELIX_WG_ENDPOINT` | WireGuard endpoint advertised to peers | `10.0.0.11:51820` |
| `CAELIX_CLUSTER_VIP` | Floating VIP held by the leader (stable console + ingress address) | `10.0.0.10/32` |
| `CAELIX_VIP_IFACE` | Interface carrying the VIP (default: the default-route interface) | `enp5s0` |

A clean way to set these is an environment file `/etc/caelix-cluster.env` loaded by
the systemd units (see §3).

---

## 2. Start a node (agent)

On each node the engine runs in **agent mode**: like `caelix run`, plus publishing
the identity, heartbeat and backends.

```bash
# Generate (once) the WireGuard key pair; prints the public key
caelix mesh-keygen

# Apply the WireGuard mesh from the store directives (root)
sudo caelix mesh-up

# Start the agent (reconciles the sub-manifest pushed by the controller)
caelix agent
```

Handy inspection commands:

```bash
caelix node-info      # node identity + metadata (JSON)
caelix node-status    # writes + prints the agent status (meta + observed state)
```

The **controller** is the FastAPI backend started with `CAELIX_CONTROLLER=1`: it
reads the desired state and the live nodes, schedules placement and writes one
sub-manifest per node. On the `consul` backend it self-elects as leader; on the
`file` backend it is the sole controller.

---

## 3. Example systemd unit

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

On the control node, a second unit starts the backend with `CAELIX_CONTROLLER=1`
(the leader loop is enabled there).

---

## 4. Add or remove a node

### Add

On the controller console, **Cluster › Add a node** shows the command to run on the
new machine. Otherwise, directly:

```bash
# Guided install: choose "join an existing cluster"
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --mode join --consul-addr http://<controller-IP>:8500

# Or, if Caelix is already installed on the machine:
caelix node join --consul-addr http://<controller-IP>:8500 --start
```

The node generates its WireGuard keys, registers in the store, and appears in the
**Cluster** view and the header **node selector**.

### Remove

In the **Cluster** view, a row's **Remove** button drains the node (its workloads
are rescheduled elsewhere) then removes it from the cluster. If pinned workloads
block the drain, the console offers to force it. On the node, `caelix node leave`
stops the agent and tears down the mesh.

---

## 5. Target a node from the console

In cluster mode a **node selector** appears in the console header. Picking a node
routes **all** Docker-backed actions (containers, images, volumes, networks, stacks,
logs, metrics, deployments) to that node's daemon. The choice is remembered and
reloads the views. "Controller (local)" returns to the local daemon.

Technically, the console sends an `X-Caelix-Node` header that the backend resolves to
the node's Docker endpoint — see
[Architecture › Cluster §9](../architecture/cluster.md#9-targeting-a-node).

---

## 6. Cluster VIP (stable address)

Node IPs can change (DHCP) and the leader can fail over. To provide a **stable access
address** for the console and the ingress, Caelix manages a **floating VIP** carried
by the **leader** node:

- Set `CAELIX_CLUSTER_VIP` (e.g. `10.0.0.10/32`) on controller-eligible nodes — at
  install time: `--mode controller --vip 10.0.0.10/32`.
- The leader's **agent** adds the VIP to its interface (`CAELIX_VIP_IFACE`, otherwise
  the default-route interface) and sends a gratuitous ARP; other nodes release it. The
  console (`:18100`) and the ingress proxy (`:80`), which listen on `0.0.0.0`, then
  answer on the VIP.
- The Consul lock guarantees a **single leader** (no split-brain). On failover the new
  leader takes over the VIP within ~1–2 ticks; the outgoing leader (or one that lost
  its lease) releases it. The configured VIP is also **published to the store** so a
  promoted node can take it over without reconfiguration.

```bash
caelix vip-status     # current leader, VIP, interface, local ownership
```

!!! note "Automatic failover"
    Every cluster node is controller-eligible (`CAELIX_CONTROLLER=1`) and a Consul
    server: failover is **automatic** from **≥ 3 nodes** (Raft quorum kept when a node
    is lost). The new leader takes over the VIP within ~seconds; the WireGuard mesh
    stays active on the surviving nodes.

---

## 7. Bring up a real cluster

On each VM or server, install via the official package (see
[Installation](installation.md)):

```bash
# Controller (first node) — starts Consul, console + controller, carries the VIP
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32

# Workers
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join --consul-addr http://<controller-IP>:8500
```

Then, on each node: `caelix mesh-keygen` and `sudo caelix mesh-up`. Nodes appear in
the console's **Cluster** view, reachable on the VIP (`http://10.0.0.10:18100`).
