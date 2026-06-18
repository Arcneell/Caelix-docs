# Multi-node cluster

Caelix runs single-host by default. This guide covers enabling cluster mode, adding
a node, and the test bench. For the internals, see
[Architecture › Cluster](../architecture/cluster.md).

Consul and WireGuard are embedded and driven by Caelix: you only interact with
`caelix` and the console.

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
| `CAELIX_CONTROLLER` | `1` on control nodes (enables the leader loop) | `1` |
| `CAELIX_INGRESS` | `1` to publish backends / refresh the ingress | `1` |
| `CAELIX_WG_ENDPOINT` | WireGuard endpoint advertised to peers | `10.0.0.11:51820` |

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

## 6. Incus test bench

A **real** multi-node bench is provided in `tests/integration/vmlab/`: it creates
**Incus VMs** (KVM), installs Docker + WireGuard + Caelix, and brings up a 3-node
cluster (1 controller + Consul, 2 workers) with real Consul and WireGuard.

```bash
cd tests/integration/vmlab
./up.sh         # build the frontend, create + provision the VMs, console forwards
# … test via the console (port-forward) …
./down.sh       # destroy the VMs
```

Provisioning (`provision.sh`) writes `/etc/caelix-cluster.env`, exposes each node's
`dockerd` over TCP (so the controller can target nodes from the UI) and installs the
`caelix-agent` + `caelix-ui` units. See the folder's `README.md` for details and
console access.

!!! warning "TCP Docker = isolated network only"
    The bench exposes `dockerd` over **plain** TCP on the isolated VM network. In
    production, restrict that endpoint to the **WireGuard subnet + mTLS**.
