# CLI Reference

The `bin/caelix` binary is the main entry point for Caelix.

## Usage

```bash
bin/caelix <command> [options]
```

Or with environment variables:

```bash
CAELIX_MANIFEST=etc/manifest.ini CAELIX_DATA=.caelix bin/caelix <command>
```

## Commands

### run

```bash
bin/caelix run
```

Starts the infinite reconciliation loop (daemon mode).

- Executes `reconcile_all()` in a loop
- Interval configurable via `[orchestrator] interval` or `CAELIX_INTERVAL`
- Writes a heartbeat on each cycle
- Stops with `Ctrl+C` or `SIGTERM`

This is the primary command for production operation.

### agent

```bash
bin/caelix agent
```

Same as `run`, plus publishing the node status on each cycle (multi-node,
phase 1). The agent:

- has a stable node identity (`node-info`);
- reconciles its local manifest just like single-host mode;
- writes `.caelix/agent/status.json` (meta + observed state) on each cycle.

If a cluster store is configured (`CAELIX_CLUSTER_BACKEND` = `file` via
`CAELIX_CLUSTER_STORE`, or `consul`), the agent publishes its meta to the store and
adopts the sub-manifest pushed by the controller as its desired-state source;
otherwise it reconciles its local manifest (single-host). See the
[multi-node RFC](../architecture/multi-node-rfc.md).

### node-info

```bash
bin/caelix node-info
```

Prints the node identity and metadata as JSON (id, hostname, labels, engine,
version, CPU, memory). The identity comes from `CAELIX_NODE_ID` if set, otherwise
from an id generated once and persisted in `.caelix/node/id`. Placement labels are
declared via `CAELIX_NODE_LABELS` (e.g. `zone=eu-west,disk=ssd`).

### node-status

```bash
bin/caelix node-status
```

Writes then prints the agent status (`.caelix/agent/status.json`): node meta +
observed state of Caelix-managed containers (name, app, state, image, health
failure count).

### mesh-keygen / mesh-up / mesh-down

```bash
bin/caelix mesh-keygen   # generate (once) the node's WireGuard keypair
sudo bin/caelix mesh-up  # apply the mesh from the directives in the store
sudo bin/caelix mesh-down
```

Cross-host network layer (multi-node, RFC §6.2). `mesh-keygen` creates the local
keypair (the private key is never published) and prints the public key, then
published in the node meta. `mesh-up` applies the secret-free directives
pushed by the controller (`address` + peer table) with the local private key, via
`wg`/`ip` (root required). `mesh-down` tears the interface down. See the
[multi-node RFC](../architecture/multi-node-rfc.md).

### vip-status

```bash
bin/caelix vip-status
```

Prints the state of the cluster VIP (floating ingress) from the current node's
point of view:

- `node` — the local node identity
- `leader` — the node currently holding leadership (which should carry the VIP)
- `vip` — the configured VIP address (`CAELIX_CLUSTER_VIP`), or "not configured"
- `interface` — the interface carrying the VIP (`CAELIX_VIP_IFACE` or the default route)
- `held` — `yes` if the VIP is actually held locally, `no` otherwise

Handy to quickly check which node holds the VIP after a failover. See the
[Proxy](../modules/proxy.md) module (cluster ingress) and the
`CAELIX_CLUSTER_VIP` / `CAELIX_VIP_IFACE` / `CAELIX_VIP_ARP` variables.

### once

```bash
bin/caelix once
```

Executes a single reconciliation cycle then exits.

Use cases:
- First launch after installation
- Integration into a cron job
- Testing and debugging

### reconcile-app

```bash
bin/caelix reconcile-app <service-name>
```

Reconciles only the specified service.

```bash
# Example
bin/caelix reconcile-app web
```

### validate

```bash
bin/caelix validate
```

Validates the manifest syntax and keys. Checks:

- Correct INI syntax
- Required keys present (`image`, `health_url` if http)
- Port format
- Valid values for strategies
- If Python is available, uses `manifest_doctor.py` for extended validation

### doctor

```bash
bin/caelix doctor [--fix] [--strict-local]
```

Full system diagnostic.

**Options:**

| Option | Description |
|---|---|
| `--fix` | Attempt to automatically fix detected issues |
| `--strict-local` | Verify that health URLs target localhost |

**Checks performed:**

- Required keys present
- Valid port format
- Consistent rollout strategies (blue_green requires `candidate_publish`)
- Valid health, repair, autoscale metric types
- Write access to `CAELIX_DATA`
- Docker/Podman availability
- Presence of `notify.ini`

**Fix mode:**

```bash
bin/caelix doctor --fix
```

- Backs up the original as `.bak`
- Fixes automatically detectable issues
- Rewrites the corrected manifest

### show

```bash
bin/caelix show
```

Displays the loaded manifest, section by section. Useful to verify how Caelix interprets the configuration.

### version

```bash
bin/caelix version
```

Displays the version from the `VERSION` file.

### status

```bash
bin/caelix status
```

Displays the daemon state:

- Last heartbeat
- Uptime since last start
- Detected runtime (Docker/Podman)
- Configured paths (`CAELIX_MANIFEST`, `CAELIX_DATA`)
- Number of services in the manifest

### resume

```bash
bin/caelix resume <service-name>
```

Resumes reconciliation for a suspended or manually paused service.

Removes the files:
- `.caelix/state/<app>.suspend_reconcile`
- `.caelix/state/<app>.manual_pause`

```bash
# Example: resume a service suspended after creation failures
bin/caelix resume api
```
