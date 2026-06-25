# Getting Started

This guide walks you through launching Caelix with your first service. Two fast paths
depending on your target: single-host (default) or an HA cluster.

## Fast path: single-host

Install Caelix on a Linux server (see [Installation](installation.en.md) for the
registry `docker login`):

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd
```

Open the console at **http://SERVER_IP:18100** (login `admin`; password shown by
`docker logs caelix-caelix-ui | grep -i password`, or set via `--admin-password`). From
the console, add your services. On the CLI, the manifest flow below does the same.

## Fast path: HA cluster

Three nodes, `:latest` image. On the bootstrap node:

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32 \
      --cluster-size 3 --admin-password 'ChangeMe-Strong'
```

On each other node:

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join \
      --consul-addr http://<controller-IP>:8500 --admin-password 'ChangeMe-Strong'
```

Open the console on the VIP (`http://10.0.0.10:18100`), check all 3 nodes in the
**Cluster** view, then deploy a cluster service. Example nginx served on the VIP:

```ini
[web]
image = nginx:latest
total_replicas = 3
publish = 8080:80
autoscale_route = default
anti_affinity = web
```

The service is reachable at `http://10.0.0.10/`. For the full guide (verification,
`caelix vip-status`, HPA, failover, hardening), see [Multi-node cluster](cluster.en.md).

---

## Manifest flow (single-host, in detail)

The steps below show the single-host, manifest-driven flow (the CLI equivalent of
adding a service). From an image-based install, the manifest lives at
`/opt/caelix/etc/manifest.ini` and the global command is `caelix`; the `bin/caelix`
examples below apply to a source checkout.

## 1. Create the Manifest

```bash
cd caelix
cp etc/manifest.ini.example etc/manifest.ini
```

Edit `etc/manifest.ini` to define your first service:

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1
log_level = info

[web]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
health_expect_codes = 200
health_timeout = 5
repair_strategy = auto
```

## 2. Validate the Configuration

```bash
bin/caelix validate
```

Expected output: no errors. For a more thorough diagnostic:

```bash
bin/caelix doctor
```

## 3. Run a First Pass

```bash
bin/caelix once
```

Caelix will:

1. Load the manifest
2. Detect the runtime (Docker or Podman)
3. Determine that the `caelix-web` container does not exist
4. Create it with the `nginx:latest` image and port `8080:80`
5. Run a health check on `http://127.0.0.1:8080/`
6. Display the result

## 4. Verify the Result

```bash
# Check Caelix status
bin/caelix status

# View the created container
docker ps | grep caelix-web
```

Your Nginx service is now accessible at `http://localhost:8080`.

## 5. Start the Daemon

To have Caelix continuously monitor your services:

```bash
bin/caelix run
```

The reconciliation loop runs every 10 seconds (configurable via `interval`).

!!! tip "Stop the daemon"
    `Ctrl+C` to stop. In systemd mode: `sudo systemctl stop caelix`.

## 6. Test Automatic Repair

Simulate a failure by manually stopping the container:

```bash
docker stop caelix-web
```

On the next reconciliation cycle, Caelix detects that the container is stopped and automatically restarts it.

!!! note "Manual pause"
    If `manual_stop_pause = 1` (default), Caelix will not restart a manually stopped container. Use `bin/caelix resume web` to resume reconciliation.

## Next Steps

- [Manifest configuration](../configuration/manifest.md) to add more services
- [Health checks](../modules/health.md) to configure monitoring
- [Discord notifications](../configuration/notifications.md) to receive alerts
