# RFC — Caelix multi-node (HA cluster)

| | |
|---|---|
| **Status** | Draft — for discussion |
| **Date** | 2026-06-16 |
| **Author** | Arcneell |
| **Goal** | A high-availability cluster that keeps the Caelix self-healing engine |
| **Chosen stack** | Consul · WireGuard · 3 co-located controllers · NFSv4 · in-house ingress (see §1.1) |

---

## 1. Summary (TL;DR)

Caelix is today a **single-host** orchestrator: the Bash engine reconciles a
`manifest.ini` against the **local** Docker daemon, state lives in `.caelix/` files
plus a local SQLite database, and load-balancing relies on a `socat` proxy that binds
host ports.

This RFC proposes a **controller + agents** architecture that:

- **keeps the self-healing engine** (`repair.sh`, `health.sh`, blue/green, autoscale)
  as the **local executor on each node** — it is the product differentiator and is
  not replaced by Swarm/Kubernetes;
- adds a **replicated control plane** (placement, reschedule on failure, shared source
  of truth) to deliver **true high availability**: a service survives the loss of its
  node and is automatically rescheduled elsewhere.

This is **not a port** but a major architectural shift, deliverable in phases each of
which has its own commercial value (managed multi-host → horizontal scale → full HA).

## 1.1 Chosen technical stack

The choices below are **locked** (priorities: secure by default, efficiency,
consistency with the existing engine). Detailed rationale and rejected alternatives
are in the referenced sections.

| Building block | Decision | Why | Ref. |
|---|---|---|---|
| **Store / consensus / discovery** | **Consul** | One binary: Raft KV + leader election + service discovery + health checks + ACL + gossip encryption + TLS | §6.1 |
| **Cross-host network** | **WireGuard mesh** (encrypted underlay) + per-node container subnet routed over the mesh | All east-west traffic encrypted by default, kernel-native, small surface; independent of Swarm | §6.2 |
| **Ingress / LB** | **Existing Caelix proxy**, regenerated from Consul (`consul-template` style) | Reuses the certbot/domains integration; avoids a second TLS system | §7 |
| **Controller topology** | **3 co-located controllers** with the 3 Consul servers; elected leader, read-only followers | Consul quorum already needs 3 nodes; survives the loss of one node with no intervention | §8.1 |
| **`shared` storage** | **NFSv4** first (Docker volume driver), traffic over the WireGuard mesh; CephFS/SFTP later | Ubiquitous and simple; `pinned` stays the safe default | §6.3 |
| **Control-plane security** | **mTLS** everywhere + per-node **Consul ACL** + **lease = authority** (fencing) | Secure by default, a sales argument | §12 |
| **Auth/UI (SQLite)** | Read everywhere, **leader writes**; migrate to KV/Postgres if write load demands it | Reuses the existing auth without rewriting it | §6.1 |
| **Install / add a node** | **Embedded components + join token** (k3s / Docker Swarm model), one command or a UI button | Install simplicity is a product goal: zero Consul/WireGuard command exposed | §1.2 |

## 1.2 Install & add-a-node experience (product goal)

!!! danger "Product constraint: install simplicity is first-class"
    Caelix multi-node must install **like k3s or Docker Swarm — not like
    Kubernetes/kubeadm**. One command to start, one command (or a UI button) to add a
    node. **The user must never type a Consul or `wg` command**: these components are
    embedded implementation details.

**Principle: everything is embedded and self-driven.** The Caelix binary embeds and
manages the lifecycle of Consul and WireGuard (keys, interface, peers, mTLS CA). The
operator only interacts with `caelix` and the UI.

**Start a cluster (1 node):**

```bash
caelix cluster init
# → this node becomes the controller; Caelix starts Consul in bootstrap mode,
#   generates the mTLS CA + the WireGuard keys, and prints a join token.
#   A 1-node cluster = exactly the current single-node behaviour.
```

**Add a node (a single command, like `docker swarm join`):**

```bash
caelix node join --token <join-token> <controller-ip>
# → Caelix configures everything itself: Consul agent, WireGuard peer (key + route),
#   mTLS certificate (obtained via the token), node registration.
#   No file editing, no third-party command.
```

**Guided add via the UI:** an **"Add a node"** button → shows the copyable command
with a **short-TTL token**, a live node view (health, load, zone), and one-click
**drain / remove** actions for a node.

**Simple start, HA on demand.** A cluster can start with a **single controller** (the
simplest, single-server case); control-plane HA is enabled later by promoting two more
nodes:

```bash
caelix node promote <node-id>   # moves the control plane from 1 → 3 (HA quorum)
```

So 3 nodes are **not** required on day 1 (despite the impression §8.1 may give): 3
nodes is the target **for control-plane HA**, not an install prerequisite.

**Minimal prerequisites** (everything else is embedded): a Linux host with the
**WireGuard** module (present in kernels ≥ 5.6, standard today) and **Docker**.

**Join security**: the join token is **short-lived and revocable**; it acts as a
*bootstrap* to obtain a per-node **mTLS certificate** (the token grants no durable
access). Rotation and revocation are driven from the UI. It is the automated
equivalent of a bootstrap token, without the operational complexity.

---

## 2. Goals / Non-goals

### Goals

1. **Workload HA**: losing a worker node automatically redeploys its stateless
   services onto the surviving nodes.
2. **Control-plane HA**: no single point of failure for decision-making (leader
   election, replicated state).
3. **Maximum reuse of the existing engine**: the agent reuses
   `reconcile_all`/`reconcile_app` almost as-is, from a sub-manifest of its own.
4. **Commercial continuity**: each phase is sellable on its own.
5. **Backward compatibility**: a single-node deployment stays a special case (a
   1-node cluster) with no regression.
6. **Trivial install & onboarding**: starting a cluster and adding a node must fit in
   **one command or a UI button** (k3s / Docker Swarm model), without ever exposing
   Consul or WireGuard (see §1.2). It is a **product acceptance criterion**, not an
   optional nicety.

### Non-goals (for this iteration)

- Replacing the engine with Docker Swarm or Kubernetes.
- Multi-region / multi-datacenter with high WAN latency (we target a low-latency
  LAN/VPC cluster).
- Live-migration of stateful containers: stateful is handled by **pinning** or
  **shared storage**, not by moving running state.
- Strong multi-tenancy (per-customer network/quota isolation) — out of scope.

---

## 3. Why Caelix is single-host today

Every subsystem assumes "a single host". Inventory of what must change:

| Subsystem | Current assumption | Files | Multi-node impact |
|---|---|---|---|
| Runtime access | `docker`/`podman` CLI on the **local socket** | `lib/runtime.sh` (`_rt`, `runtime_available`), `ui/backend/app/core/docker.py` (`runtime_bin`) | Must target N daemons (local agent or `DOCKER_HOST` TLS) |
| State | `.caelix/` files + SQLite WAL | `.caelix/state/`, `.caelix/autoscale/`, `auth.db` | Not shared: only one node can decide |
| Reconciliation | Single daemon, **global** `flock`, loops over one host's apps | `bin/caelix` (`reconcile_all`, `caelix_with_global_lock`) | No notion of placement or "who does what" |
| Load-balancing | Local `socat` proxy, binds host ports, routes to local containers | `lib/autoscale_proxy.sh`, `lib/proxy.sh` | `socat` does not cross hosts |
| Naming | `caelix-<app>`, replicas `…-rN` unique **per host** | `lib/runtime.sh` (`caelix_cname`) | Cluster-wide collisions |
| Volumes | `volumes_bind` = **host** paths | manifest, `container_create` | A bind does not follow a rescheduled container |
| Placement | Nonexistent | — | New subsystem |
| Heartbeat | `caelix-daemon-heartbeat` local file | `.caelix/state/` | Must become cluster-observable node health |

!!! note "What is reusable as-is"
    The **intelligent core** — `restart → recreate → purge` escalation
    (`repair.sh`), HTTP/TCP probes (`health.sh`), blue/green, local autoscale,
    incidents, audit, notifications — works **per node** without deep changes. It is
    the three pillars of a cluster (cross-host network, shared state/consensus,
    storage) that are missing and must be built.

---

## 4. Target architecture

```mermaid
flowchart TB
    UI[Vue 3 UI / admin API] --> LEADER

    subgraph CP["Control plane (HA — 3 nodes, elected leader)"]
        LEADER["controller (leader)"]
        STBY1["controller (follower)"]
        STBY2["controller (follower)"]
        STORE[("Replicated Raft store\ndesired + observed state\nleader election")]
        LEADER --- STORE
        STBY1 --- STORE
        STBY2 --- STORE
    end

    LEADER -- "per-node sub-manifest\n(watch / pull)" --> AGA
    LEADER -- "per-node sub-manifest" --> AGB
    LEADER -- "per-node sub-manifest" --> AGC

    subgraph W["Data plane (workers)"]
        AGA["agent A\nBash engine\n→ local docker"]
        AGB["agent B\nBash engine\n→ local docker"]
        AGC["agent C\nBash engine\n→ local docker"]
    end

    AGA <-- "heartbeat + observed state" --> STORE
    AGB <-- "heartbeat + observed state" --> STORE
    AGC <-- "heartbeat + observed state" --> STORE

    AGA -. "overlay network + cross-host ingress" .- AGB
    AGB -. .- AGC
```

**Guiding principle**: the global `manifest.ini` becomes the **cluster desired state**.
The controller's scheduler **splits it into per-node sub-manifests**. Each agent
reconciles its sub-manifest against its local Docker — that is, it runs **the current
Caelix engine without knowing it is clustered**.

---

## 5. Components

### 5.1 Controller (control plane)

A service (extension of the existing FastAPI backend) that, **only on the leader**:

- reads the global desired state (cluster manifest) from the store;
- runs the **scheduler**: decides on which node(s) each app/replica runs, based on
  affinity, resources, anti-affinity and node health;
- writes the **per-node sub-manifests** to the store;
- monitors agent **heartbeats**; on lease expiry, marks the node `down` and
  **reschedules** its stateless workloads;
- serves the UI and API (followers can serve read-only and proxy writes to the
  leader).

### 5.2 Agent (data plane, one per node)

Packaging of the existing Bash engine as a standalone service:

- registers with the cluster (node identity, capabilities, resources);
- **watches** its sub-manifest in the store; on each change, writes a local
  `manifest.ini` and triggers `reconcile_all`;
- keeps its periodic reconciliation daemon (local self-healing stays active even if
  the controller is unreachable — **graceful degradation**);
- publishes its **heartbeat** and the **observed state** of its containers to the
  store;
- registers its backends for service discovery (see §7).

!!! tip "The `flock` lock stays relevant"
    `caelix_with_global_lock` serializes passes — it becomes the agent's **local
    lock** (one agent = one host). No distributed lock is needed *on the agent side*;
    cluster coordination happens via the controller's leader election (§8).

### 5.3 Controller ↔ agent protocol

A **pull/watch** model (the agent watches the store), preferred over direct push:

- decoupling: the controller need not know each agent's routable address;
- resilience: a restarting agent re-syncs by re-reading the store;
- natural HA: leader failover does not interrupt the agents.

Data flow:

| Direction | Data | Store key (see §9) |
|---|---|---|
| controller → agent | desired sub-manifest | `caelix/nodes/<id>/desired` |
| agent → controller | heartbeat + lease | `caelix/nodes/<id>/heartbeat` (TTL) |
| agent → controller | observed state (containers, health) | `caelix/nodes/<id>/observed` |
| agent → cluster | backends for LB | `caelix/services/<app>/backends/<inst>` |

---

## 6. Technical decisions — the three hard problems

### 6.1 Shared state + consensus

**Problem.** Today: local `.caelix/` files + SQLite. For HA we need a **replicated**
source of truth *and* a **leader election**; otherwise split-brain (two controllers
reschedule the same app).

| Option | Pros | Cons |
|---|---|---|
| **Consul** *(chosen)* | Raft KV + leader election + health checks + service discovery + ACL + encrypted gossip + TLS, in one binary; Docker-native | External dependency to operate (3-node cluster) |
| etcd | k8s reference, robust, fast watches | No built-in service discovery/health → must be completed |
| Postgres + advisory locks | If a SQL instance is already operated; familiar SQL | SQL HA to manage yourself; no native watch (polling); no service discovery |

**Decision: Consul.** It covers all three needs at once — replicated KV (desired/
observed state), leader election (sessions + locks), and service discovery/health for
the ingress (§7). SQLite (`auth.db`) stays for auth/UI: **read everywhere, written by
the leader** (then replicated, or migrated to KV/Postgres if the write load justifies
it).

### 6.2 Cross-host network + ingress

**Problem.** Today: `socat` binds host ports and routes to **local** containers
(`autoscale_proxy.sh`). This does not cross nodes.

| Layer | Option | Note |
|---|---|---|
| Overlay | **WireGuard mesh** (encrypted underlay) + per-node container subnet routed over the mesh — **chosen** | Encrypted by default (ChaCha20-Poly1305), kernel-native, small surface; independent of Swarm |
| Overlay | Docker overlay (Swarm-mode network only) | Most native, but VXLAN is **cleartext** unless `--opt encrypted` (IPsec, heavy); partially couples to Swarm mode |
| Overlay | CNI plugin (flannel/cilium) | Powerful but heavy for Caelix's "lightweight" positioning |

**Decision: WireGuard mesh.** The controller manages peer configuration (public keys,
endpoints, allowed subnets) and distributes it via the store; each node is assigned a
**container subnet** (e.g. `10.42.<n>.0/24`) whose routes go through the WireGuard
tunnel. It is the **most secure** choice: all east-west traffic (control *and* app
data, including NFS) is encrypted, with no dependency on Swarm mode. At the target
scale (LAN/VPC cluster, modest node count), a controller-driven full mesh is efficient
and simple to reason about. The Docker overlay remains the **documented alternative**
if a customer refuses WireGuard.

**Ingress: we evolve the existing proxy.** `global_proxy_update_routes` already
generates `routes.conf`; we replace the "local ports" source with the **backends
discovered in Consul** (`consul-template`-style regeneration on watch), so that an
ingress on any node routes — via the mesh — to a healthy replica on any other node.
This reuses the existing TLS/certbot/domains integration (`proxy.sh`) instead of
introducing a second system (Traefik/Caddy stays the escape hatch if L7 needs outgrow
the current proxy).

### 6.3 Stateful storage + placement

**Problem.** `volumes_bind` = **host** paths. A bind does not follow a rescheduled
container → a stateful app cannot migrate naively.

Two modes exposed in the manifest (see §10):

- **`storage = pinned`**: the app is **pinned to its node**. Simple, no dependency;
  but **no HA for that app** (if the node dies, the app waits for its return). Reserve
  it for data you do not want to replicate.
- **`storage = shared`**: a volume on shared storage via a Docker volume driver. The
  app **can migrate**: the scheduler reschedules it onto a node with access to the same
  volume.

**Decision: NFSv4 as the first supported `shared` backend** (ubiquitous, simple to
operate, standard Docker volume driver), its traffic riding the WireGuard mesh (thus
encrypted). CephFS and SFTP are planned as later backends for customers with higher
redundancy/perf needs. **`pinned` stays the default**: `shared` is enabled only
explicitly, to avoid a false HA promise on misconfigured stateful workloads.

**Stateless** apps migrate freely (the main HA target in phase 1).

---

## 7. Service discovery & load-balancing

Target flow:

1. On each (re)deploy, the agent registers the instance in Consul:
   `caelix/services/<app>/backends/<node>-<replica> = host:port` + a health check.
2. Each node's ingress **watches** that key and regenerates its routing table
   (`routes.conf`), filtered on **healthy** backends.
3. An inbound request on any node is routed — via the overlay — to a healthy replica,
   **regardless of its node**.

This generalizes the current autoscale: replicas `-r1/-r2` are no longer all on the
same host but **distributed**, and the global proxy becomes a **cluster ingress**.

---

## 8. High availability & failure handling

### 8.1 Leader election (controller)

**Chosen topology: 3 controllers co-located with the 3 Consul servers** — but **only
when control-plane HA is wanted**. A cluster can start with a **single controller**
(simplest install, see §1.2) then be promoted to 3 via `caelix node promote` without
interruption. Consul quorum already needs 3 nodes; we reuse those same nodes rather
than adding a separate layer. Controllers acquire a **Consul session lock**; the holder
is **leader** (scheduling + desired-state writes). The two **followers** serve the
UI/API **read-only** and **proxy writes** to the leader. Session loss (crash,
partition) → the lock is released → a follower becomes leader and resumes scheduling.
Agents, in pull/watch, **are not interrupted**. Losing one of the 3 control nodes
preserves quorum (2/3) and thus control-plane availability.

### 8.2 Node failure detection

Each agent renews a **TTL lease** (`heartbeat`). On expiry → the leader marks the node
`down`, then:

- **stateless**: reschedules the dead node's apps onto healthy nodes (rewriting the
  target sub-manifests);
- **stateful `pinned`**: marks the app `unavailable`, opens an incident, waits for the
  node's return (or intervention);
- **stateful `shared`**: reschedules onto a node with access to the shared volume.

### 8.3 Split-brain prevention

- **A single write authority**: only the leader writes the desired state → no
  concurrent decision.
- **Quorum**: the store (Consul/etcd) requires the majority; a minority node in a
  partition **cannot** become leader.
- **Application-level fencing**: before rescheduling a workload, the leader **revokes
  the lease** of the suspect node; an agent that loses its lease **suspends its
  creations** to avoid double-start (reuses the existing `suspend_reconcile` logic).

### 8.4 Graceful degradation

If the control plane is entirely unreachable, **each agent keeps self-healing locally**
according to its last known sub-manifest. We only lose the **reschedule** capability,
not local self-healing — a property consistent with Caelix's "self-healing" DNA.

---

## 9. State schema (Consul KV layout)

```text
caelix/
├── cluster/
│   ├── leader                      # session lock (controller election)
│   ├── desired/manifest            # cluster manifest (global desired state)
│   └── config/                     # cluster parameters (intervals, policies)
├── nodes/
│   └── <node-id>/
│       ├── meta                    # capabilities, resources, labels, zone
│       ├── heartbeat               # TTL key (lease) — absence = node down
│       ├── desired                 # sub-manifest assigned by the scheduler
│       └── observed                # real state (containers, health, restart_count)
└── services/
    └── <app>/
        ├── spec                    # desired summary (replicas, route, storage)
        └── backends/
            └── <node>-<replica>    # host:port + health status (for the ingress)
```

The local `.caelix/` tree **remains on each agent** (logs, audit, incidents, local
reconciliation state); only the keys above are **cluster-wide**.

---

## 10. Manifest extensions (placement)

New per-app-section fields (backward compatible: absent = single-node behaviour):

```ini
[my-app]
image = nginx:1.27
# --- cluster placement -------------------------------------------------
total_replicas   = 3              # total count across the cluster (replaces per-host autoscale)
node_affinity    = zone=eu-west   # placement constraint (node label)
anti_affinity    = my-app         # do not co-locate two replicas on the same node
storage          = shared         # pinned | shared (see §6.3)
shared_volume    = my-app-data    # shared volume name if storage=shared
max_per_node     = 1              # cap on replicas per node
```

The scheduler translates these constraints into `nodes/<id>/desired` assignments.

---

## 11. Cluster-wide naming

`caelix_cname()` currently produces `caelix-<app>`, and replicas `…-rN` are unique
**per host**. In a cluster, the name must be globally unique:

```
caelix-<app>-<short-node-id>-r<N>
```

The `caelix.app=<app>` label (already set by `container_create`) stays the logical
identifier; we add `caelix.node=<id>` and keep `caelix.candidate=1` for blue/green
cleanup. Service discovery indexes by label, not by name — so the name only has to be
**unique and deterministic**.

---

## 12. Security

Since Caelix is a **commercial product** whose security credibility is a sales
argument, the distributed control plane must be secure **by default**:

- **mTLS agent ↔ controller ↔ store**: per-node certificates, rotation; no control
  traffic in cleartext on the network.
- **Secret distribution**: secrets do not travel in sub-manifests in cleartext;
  referenced by key, resolved on the agent side from an encrypted store (Consul +
  encryption at rest, or Vault as an option). Extends the existing `write_text_secret`
  / `0600` files logic.
- **Lease = authority**: an agent without a valid lease **does not create** a container
  (anti double-start, §8.3).
- **Store ACL**: each agent only has access to `nodes/<its-id>/*` and read access to
  service discovery; only the controller writes `desired`.
- **Network surface**: the overlay (WireGuard) encrypts east-west traffic between
  nodes.

---

## 13. Backward compatibility & migration

- **A 1-node cluster = current single-node.** If no placement field is set and a single
  agent is registered, behaviour is identical to today.
- **Degraded mode without store**: a "standalone" path (no Consul) can be kept that
  falls back exactly onto the single-host engine — useful for small customers and
  development.
- **Migration**: an existing deployment becomes the **first node** of the cluster;
  agents are added progressively; the existing manifest works unchanged until placement
  fields are introduced.

---

## 14. De-risking roadmap

Ordered so that **the risky part (consensus/HA) comes last**, once the rest is proven,
and so **each phase is sellable**.

| Phase | Content | Value delivered | Effort |
|---|---|---|---|
| **0 — Runtime abstraction** ✅ | `_rt`/`run_cmd` target a remote daemon via `CAELIX_DOCKER_HOST`/`_TLS_VERIFY`/`_CERT_PATH` (→ `DOCKER_*`), default = unchanged local socket | Technical base; unblocks everything | _done_ |
| **1 — Standalone agent** 🚧 | Package the engine as an agent reconciling its local from a received sub-manifest. **Delivered**: node identity (`caelix node-info`), `caelix agent` command, `agent/status.json` publishing (meta + observed, the contract the controller reads in §9). **Remaining**: sub-manifest source abstraction (file → store) + node registration, in phase 2 | — | in progress |
| **2 — Single-instance controller + store** 🚧 | Cluster manifest, static scheduler, per-node push. **Delivered**: Python placement core (`core/cluster/` — `schedule`, manifest split, `FileStore` §9, `apply_cluster`) **+ agent↔store loop** (`CAELIX_CLUSTER_STORE`) **+ controller API/UI** (`/api/cluster/*`; "Cluster" view) **+ Consul backend** (`ConsulStore` KV, same interface as `FileStore`, selected via `CAELIX_CLUSTER_BACKEND=consul`). **Remaining**: the Bash agent reading its sub-manifest straight from Consul (today via the `CAELIX_CLUSTER_STORE` FS) | **"Managed multi-host"** (single console over N servers) | nearly complete |
| **3 — Cross-host network + dynamic ingress** | WireGuard mesh + ingress (existing proxy) fed by Consul | **Horizontal scale** (replicas spread behind an LB) | weeks |
| **4 — Controller HA** | Leader election, reschedule on node-down, anti split-brain | **True HA** | hardest |
| **5 — Stateful** | `pinned`/`shared` modes, clean node drain | HA for stateful apps | ongoing |

!!! note "Onboarding is cross-cutting"
    The install/join experience (§1.2) is **not a phase**: `caelix cluster init` lands
    in phase 1, `caelix node join` + the UI button in phase 2, `caelix node promote`
    (HA upgrade) in phase 4. At every phase, the rule "one command / one button, zero
    Consul/WireGuard exposed" is an acceptance criterion.

!!! warning "Point of no return"
    Phases 0→2 are incremental and low-risk (state stays fundamentally centralized).
    **Phase 4** introduces distributed consensus: that is where complexity and risk
    concentrate (split-brain, partitions, fencing). Do not tackle it before 0→3 are
    validated in real conditions.

---

## 15. Test plan

- **Unit/engine (existing)**: the BATS suites (`tests/bats/`) stay valid — the agent
  runs the same engine. Add tests for local `manifest.ini` generation from a
  sub-manifest.
- **Scheduler**: tests on constraints (affinity, anti-affinity, `max_per_node`,
  insufficient resources → app `pending`).
- **Chaos / failures**: kill an agent → verify stateless reschedule; kill the leader →
  verify failover without interrupting agents; **network partition** → verify no
  double-start (fencing).
- **Network**: an inbound request on node A routed to a replica on node B.
- **Load**: N nodes × M apps, measure reschedule latency and watch frequency.
- **Compat**: an existing single-node manifest deployed on a 1-node cluster = zero
  regression (replay the current suite).

---

## 16. Risks & open questions

| Risk | Mitigation |
|---|---|
| Distributed-consensus complexity (split-brain) | Delegate Raft to Consul/etcd; do not write it yourself; lease-based fencing |
| Stateful without shared storage = false HA promise | Clearly document `pinned` (no HA) vs `shared`; do not oversell |
| Operational dependency (a Consul cluster to operate) | Keep standalone mode; ship a packaged quorum install |
| **Install perceived as a "gas factory" (Kubernetes effect)** | Embedded components + join token + guided UI (§1.2); **acceptance test: add a node in < 2 min, one command**; never expose Consul/WireGuard |
| Overlay network = new failure/security surface | mTLS + WireGuard; partition tests |
| SQLite write load in multi-node | Read everywhere / leader writes; migrate to KV/Postgres if needed |

**Locked decisions** (see §1.1):

1. **Consensus store → Consul.**
2. **Cross-host network → WireGuard mesh** + per-node container subnet.
3. **Controller topology → 3 co-located controllers** with the Consul servers,
   elected leader + read-only followers.
4. **`shared` storage → NFSv4** as the first backend; `pinned` by default.
5. **Ingress → existing Caelix proxy** regenerated from Consul.

**Open questions (do not affect the architecture):**

- Default lease/heartbeat TTL thresholds (trade-off between fast detection and false
  positives on a busy network) — to calibrate in phase 4.
- Possible migration of SQLite auth to the KV/Postgres if multi-node write load
  justifies it (reassess after phase 2).

---

## 17. Rejected alternatives

- **Delegating to Docker Swarm / Kubernetes**: HA and network "for free", but the
  self-healing engine — the product's core value — would take a back seat. Rejected as
  contrary to the "keep my engine" goal.
- **Multi-host without HA (a single controller, non-replicated state)**: cheaper, but
  does not provide the requested true HA. Kept only as an **intermediate phase** (phase
  2) and as standalone mode.
