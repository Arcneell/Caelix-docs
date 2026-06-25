# Installation

## Quick Install (recommended)

Install Caelix on any Linux server in two steps:

**1. Authenticate to the Caelix registry** (credentials provided with your license):

```bash
echo "YOUR_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin
```

**2. Install (single-host):**

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh | bash -s -- --with-systemd
```

!!! tip "High-availability cluster (2.0)"
    The installer also supports **cluster mode** via `--mode controller|join` and the
    floating **VIP** via `--vip`. HA clustering ships in the **2.0** image
    (`ghcr.io/arcneell/caelix:2.0.0-beta.1` or the `:beta` channel; `:latest` stays on
    stable 1.x). The single-host install above stays the **default** and is unchanged.
    See [Multi-node cluster](cluster.en.md) for the full guide, and
    [Cluster mode](#cluster-mode-high-availability) below for the flags.

### What the Script Does

1. Checks for and installs Docker if missing
2. Extracts the orchestration engine to `/opt/caelix/`
3. Creates default configuration files
4. Runs an initial reconciliation (`caelix once`)
5. Installs the systemd service (if `--with-systemd`)
6. In cluster mode: also installs `wireguard-tools` (+ `modprobe wireguard`), `arping`,
   starts an HA Consul server and writes `/etc/caelix-cluster.env`

### Prerequisites

| Dependency | Minimum Version | Installed Automatically |
|---|---|---|
| **Docker** | 20.10+ | Yes |
| **Bash** | 4.0+ | No (present on most systems) |
| **curl** | 7.0+ | Yes |
| **socat** | 1.7+ | No (optional, for autoscale) |

!!! note "Python not required"
    Unlike the source installation, the image-based installation does **not** require Python or Node.js on the host. The Python backend is compiled and embedded in the Docker image.

### Install Options

```bash
install.sh [options]

  --ui-port PORT         Web console listen port (default: 18100)
  --ui-bind ADDR         UI bind address (default: 0.0.0.0;
                         127.0.0.1 = local machine only)
  --lang fr|en           CLI / notifications language (default: fr)
  --root PATH            Installation directory (default: /opt/caelix)
  --image IMAGE          Docker image (default: ghcr.io/arcneell/caelix:latest)
  --with-systemd         Install and start the systemd service
  --mode MODE            single | controller | join (cluster; default: single)
  --vip CIDR             (controller mode) cluster VIP carried by the leader (e.g. 10.0.0.10/32)
  --cluster-size N       (cluster) number of expected Consul servers (HA quorum; default: 3)
  --consul-addr URL      (join mode) cluster Consul address (e.g. http://10.0.0.1:8500)
  --consul-token TOK     (cluster) Consul ACL token (production hardening)
  --admin-password PW    initial admin password (same value on every node)
  --node-id ID           (cluster) identifier for this node (auto-generated otherwise)
  --no-install-docker    Don't install Docker (fail if missing)
  --skip-engine          Don't extract the engine (UI image only)
  --skip-pull            Don't pull the image (use local image)
  --dry-run              Print actions without executing
```

Examples:

```bash
# Custom UI port
... | bash -s -- --with-systemd --ui-port 9000

# UI reachable only locally
... | bash -s -- --with-systemd --ui-bind 127.0.0.1

# Install under the home dir with a custom port (the systemd unit adapts)
... | bash -s -- --with-systemd --root ~/caelix --ui-port 8088
```

Every option also has an environment-variable equivalent:
`CAELIX_UI_PORT`, `CAELIX_UI_BIND`, `CAELIX_LANG`, `CAELIX_ROOT`, `CAELIX_IMAGE`.

### Cluster mode (high availability)

As of **2.0**, the installer brings up an HA cluster. Use the 2.0 image
(`ghcr.io/arcneell/caelix:2.0.0-beta.1` or the `:beta` channel). Cluster mode forces
`--with-systemd` and automatically installs WireGuard, arping and a Consul server.

```bash
# Bootstrap node (controller) — carries the VIP, starts Consul + console
docker run --rm ghcr.io/arcneell/caelix:2.0.0-beta.1 cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32 \
      --cluster-size 3 --admin-password 'ChangeMe-Strong'

# Additional nodes — join the cluster
docker run --rm ghcr.io/arcneell/caelix:2.0.0-beta.1 cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join \
      --consul-addr http://<controller-IP>:8500 --admin-password 'ChangeMe-Strong'
```

The console is then reachable on the **VIP** (`http://10.0.0.10:18100`). Full guide
(verification, deploying a cluster service, HPA, failover, hardening):
[Multi-node cluster](cluster.en.md).

### Environment Variables

```bash
CAELIX_IMAGE="ghcr.io/arcneell/caelix:latest"   # Image to use
CAELIX_ROOT="/opt/caelix"                         # Installation directory
```

## Directory Structure After Install

```
/opt/caelix/
├── bin/caelix                  # Orchestrator CLI
├── lib/                      # Engine modules
│   ├── common.sh
│   ├── manifest.sh
│   ├── health.sh
│   ├── repair.sh
│   ├── ...
│   ├── audit_log.py
│   └── manifest_doctor.py
├── etc/
│   ├── manifest.ini          # Service configuration
│   └── notify.ini            # Notification configuration
├── .caelix/                    # Runtime data
│   ├── state/
│   ├── incidents/
│   ├── audit/
│   └── logs/
└── VERSION
```

## Post-Install Configuration

### Edit the Manifest

Add your services to `/opt/caelix/etc/manifest.ini`:

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1

[my-service]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
repair_strategy = auto
```

### Validate the Configuration

```bash
caelix validate
caelix doctor
```

### Configure Notifications

Edit `/opt/caelix/etc/notify.ini` to enable Discord, Slack, Teams, Telegram, or SMTP alerts.

## systemd Service

If you didn't use `--with-systemd` during installation:

```bash
# Re-run the installer with the option
install.sh --with-systemd --skip-pull --skip-engine

# Or install manually
sudo tee /etc/systemd/system/caelix.service <<EOF
[Unit]
Description=Caelix orchestrator (run mode)
After=network-online.target docker.service
Wants=network-online.target
Requires=docker.service

[Service]
Type=simple
User=$(whoami)
WorkingDirectory=/opt/caelix
Environment=CAELIX_MANIFEST=/opt/caelix/etc/manifest.ini
Environment=CAELIX_NOTIFY_CONF=/opt/caelix/etc/notify.ini
Environment=CAELIX_HOME=/opt/caelix
ExecStart=/opt/caelix/bin/caelix run
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now caelix
```

## Updating

To update Caelix to the latest version:

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh | bash -s -- --with-systemd
```

The script updates:

- The **engine** (bin/caelix, lib/) — always overwritten with the latest version
- The **UI Docker image** — re-pulled automatically
- The **systemd service** — restarted

!!! warning "Configuration preserved"
    The files `etc/manifest.ini` and `etc/notify.ini` are **never overwritten**. Your configuration is preserved. If a new version introduces new configuration keys, check the release notes to add them manually.

The UI container will be recreated on the next reconciliation cycle with the new image.

## Web Console

After installation, the web console is available at **http://SERVER_IP:&lt;port&gt;** (port `18100` by default).

- Login: `admin` / a **random password generated on first start** (no `admin`/`admin` default). Read it from `/opt/caelix/.caelix/initial-admin-password` (or `docker logs caelix-caelix-ui | grep -i password`), then change it and delete the file. Set `CAELIX_ADMIN_PASSWORD` / `--admin-password` to choose your own at install time.
- **The port and bind address are chosen at install time**: `--ui-port <PORT>` and `--ui-bind <ADDR>` (see [Install Options](#install-options)).
- By default the UI listens on all interfaces (`0.0.0.0`). To restrict it to local access, install with `--ui-bind 127.0.0.1`.

To change the port **afterwards**, edit `publish` (and `health_url`) in the manifest, increment `config_version`, then run `caelix once`:

```ini
[caelix-ui]
publish = 127.0.0.1:9000:8080
health_url = http://127.0.0.1:9000/api/ping
```

---

## Source Installation (developers)

For contributors and developers:

### Additional Prerequisites

| Dependency | Minimum Version | Usage |
|---|---|---|
| **Python 3** | 3.11+ | UI backend, manifest validation, audit |
| **Node.js** | 20+ | Vue 3 frontend build |
| **ShellCheck** | — | Bash script linting |

### Procedure

```bash
git clone https://github.com/Arcneell/Caelix.git
cd caelix
./scripts/install-all.sh
```

See [WORKFLOW-MODIFICATIONS.md](../WORKFLOW-MODIFICATIONS.en.md) for the full development workflow.
