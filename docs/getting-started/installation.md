# Installation

## Installation rapide (recommandée)

Installez Caelix sur n'importe quel serveur Linux en deux étapes :

**1. Authentification au registry Caelix** (identifiants fournis avec votre licence) :

```bash
echo "VOTRE_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin
```

**2. Installation (mono-hôte) :**

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh | bash -s -- --with-systemd
```

!!! tip "Cluster haute disponibilité (2.0)"
    L'installeur prend aussi en charge le **mode cluster** via `--mode controller|join`
    et la **VIP** flottante via `--vip`. Le cluster HA est livré dans l'image **2.0**
    (`ghcr.io/arcneell/caelix:2.0.0-beta.1` ou canal `:beta` ; `:latest` reste sur la 1.x
    stable). Le mono-hôte ci-dessus reste le mode **par défaut** et inchangé. Voir
    [Cluster multi-nœud](cluster.md) pour le guide complet, et
    [Mode cluster](#mode-cluster-haute-disponibilité) plus bas pour les flags.

### Ce que fait le script

1. Vérifie et installe Docker si absent
2. Extrait le moteur d'orchestration dans `/opt/caelix/`
3. Crée les fichiers de configuration par défaut
4. Lance une première réconciliation (`caelix once`)
5. Installe le service systemd (si `--with-systemd`)
6. En cluster : installe aussi `wireguard-tools` (+ `modprobe wireguard`), `arping`,
   démarre un serveur Consul HA et écrit `/etc/caelix-cluster.env`

### Prérequis

| Dépendance | Version minimale | Installé automatiquement |
|---|---|---|
| **Docker** | 20.10+ | Oui |
| **Bash** | 4.0+ | Non (présent sur la plupart des systèmes) |
| **curl** | 7.0+ | Oui |
| **socat** | 1.7+ | Non (optionnel, pour autoscale) |

!!! note "Python non requis"
    Contrairement à l'installation depuis les sources, l'installation par image ne nécessite **pas** Python ni Node.js sur l'hôte. Le backend Python est compilé et embarqué dans l'image Docker.

### Options d'installation

```bash
install.sh [options]

  --ui-port PORT         Port d'écoute de la console web (défaut : 18100)
  --ui-bind ADDR         Adresse d'écoute de l'UI (défaut : 0.0.0.0 ;
                         127.0.0.1 = machine locale uniquement)
  --lang fr|en           Langue CLI / notifications (défaut : fr)
  --root PATH            Répertoire d'installation (défaut : /opt/caelix)
  --image IMAGE          Image Docker (défaut : ghcr.io/arcneell/caelix:latest)
  --with-systemd         Installe et active le service systemd
  --mode MODE            single | controller | join (cluster ; défaut : single)
  --vip CIDR             (mode controller) VIP de cluster portée par le leader (ex: 10.0.0.10/32)
  --cluster-size N       (cluster) Nombre de serveurs Consul attendus (quorum HA ; défaut : 3)
  --consul-addr URL      (mode join) Adresse Consul du cluster (ex: http://10.0.0.1:8500)
  --consul-token TOK     (cluster) Token ACL Consul (durcissement prod)
  --admin-password PW    Mot de passe admin initial (même valeur sur tous les nœuds)
  --node-id ID           (cluster) Identifiant de ce nœud (sinon auto-généré)
  --no-install-docker    N'installe pas Docker (échoue s'il est absent)
  --skip-engine          Ne pas extraire le moteur (seulement l'image UI)
  --skip-pull            Ne pas pull l'image (utilise l'image locale)
  --dry-run              Affiche les actions sans les exécuter
```

Exemples :

```bash
# Port UI personnalisé
... | bash -s -- --with-systemd --ui-port 9000

# UI accessible uniquement en local
... | bash -s -- --with-systemd --ui-bind 127.0.0.1

# Installation dans le home avec port custom (l'unité systemd s'adapte)
... | bash -s -- --with-systemd --root ~/caelix --ui-port 8088
```

Toutes les options ont un équivalent en variable d'environnement :
`CAELIX_UI_PORT`, `CAELIX_UI_BIND`, `CAELIX_LANG`, `CAELIX_ROOT`, `CAELIX_IMAGE`.

### Mode cluster (haute disponibilité)

À partir de la **2.0**, l'installeur monte un cluster HA. Utilisez l'image 2.0
(`ghcr.io/arcneell/caelix:2.0.0-beta.1` ou canal `:beta`). Le cluster force
`--with-systemd` et installe automatiquement WireGuard, arping et un serveur Consul.

```bash
# Nœud d'amorçage (controller) — porte la VIP, démarre Consul + console
docker run --rm ghcr.io/arcneell/caelix:2.0.0-beta.1 cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode controller --vip 10.0.0.10/32 \
      --cluster-size 3 --admin-password 'ChangeMoi-Fort'

# Nœuds supplémentaires — rejoignent le cluster
docker run --rm ghcr.io/arcneell/caelix:2.0.0-beta.1 cat /opt/caelix/install.sh \
  | bash -s -- --with-systemd --mode join \
      --consul-addr http://<IP-controller>:8500 --admin-password 'ChangeMoi-Fort'
```

La console est ensuite joignable sur la **VIP** (`http://10.0.0.10:18100`). Guide
complet (vérification, déploiement d'un service cluster, HPA, bascule, durcissement) :
[Cluster multi-nœud](cluster.md).

### Variables d'environnement

```bash
CAELIX_IMAGE="ghcr.io/arcneell/caelix:latest"   # Image à utiliser
CAELIX_ROOT="/opt/caelix"                         # Répertoire d'installation
```

## Arborescence après installation

```
/opt/caelix/
├── bin/caelix                  # CLI de l'orchestrateur
├── lib/                      # Modules du moteur
│   ├── common.sh
│   ├── manifest.sh
│   ├── health.sh
│   ├── repair.sh
│   ├── ...
│   ├── audit_log.py
│   └── manifest_doctor.py
├── etc/
│   ├── manifest.ini          # Configuration des services
│   └── notify.ini            # Configuration des notifications
├── .caelix/                    # Données runtime
│   ├── state/
│   ├── incidents/
│   ├── audit/
│   └── logs/
└── VERSION
```

## Configuration post-installation

### Éditer le manifest

Ajoutez vos services dans `/opt/caelix/etc/manifest.ini` :

```ini
[orchestrator]
interval = 10
max_repair = 5
remove_orphans = 1

[mon-service]
image = nginx:latest
publish = 8080:80
health_type = http
health_url = http://127.0.0.1:8080/
repair_strategy = auto
```

### Valider la configuration

```bash
caelix validate
caelix doctor
```

### Configurer les notifications

Éditez `/opt/caelix/etc/notify.ini` pour activer les alertes Discord, Slack, Teams, Telegram ou SMTP.

## Service systemd

Si vous n'avez pas utilisé `--with-systemd` lors de l'installation :

```bash
# Relancer l'installeur avec l'option
install.sh --with-systemd --skip-pull --skip-engine

# Ou installer manuellement
sudo tee /etc/systemd/system/caelix.service <<EOF
[Unit]
Description=Caelix orchestrateur (mode run)
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

## Mise à jour

Pour mettre à jour Caelix vers la dernière version :

```bash
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh | bash -s -- --with-systemd
```

Le script met à jour :

- Le **moteur** (bin/caelix, lib/) — toujours écrasé avec la dernière version
- L'**image Docker** de l'UI — re-pullée automatiquement
- Le **service systemd** — redémarré

!!! warning "Configuration préservée"
    Les fichiers `etc/manifest.ini` et `etc/notify.ini` ne sont **jamais écrasés**. Votre configuration est conservée. Si une nouvelle version introduit de nouvelles clés de configuration, consultez les notes de version pour les ajouter manuellement.

Le conteneur UI sera recréé au prochain cycle de réconciliation avec la nouvelle image.

## Console web

Après installation, la console web est accessible sur **http://IP_DU_SERVEUR:&lt;port&gt;** (port `18100` par défaut).

- Login par défaut : `admin` / `admin` (changement forcé à la première connexion)
- **Le port et l'adresse d'écoute se choisissent à l'installation** : `--ui-port <PORT>` et `--ui-bind <ADDR>` (voir [Options d'installation](#options-dinstallation)).
- Par défaut, l'UI écoute sur toutes les interfaces (`0.0.0.0`). Pour la restreindre à l'accès local, installez avec `--ui-bind 127.0.0.1`.

Pour changer le port **après coup**, éditez `publish` (et `health_url`) dans le manifest, incrémentez `config_version`, puis lancez `caelix once` :

```ini
[caelix-ui]
publish = 127.0.0.1:9000:8080
health_url = http://127.0.0.1:9000/api/ping
```

---

## Installation depuis les sources (développeurs)

Pour contribuer au projet ou modifier le code :

### Prérequis supplémentaires

| Dépendance | Version minimale | Usage |
|---|---|---|
| **Python 3** | 3.11+ | Backend UI, validation manifest, audit |
| **Node.js** | 20+ | Build du frontend Vue 3 |
| **ShellCheck** | — | Linting des scripts bash |

### Procédure

```bash
git clone https://github.com/Arcneell/Caelix.git
cd caelix
./scripts/install-all.sh
```

Voir [WORKFLOW-MODIFICATIONS.md](../WORKFLOW-MODIFICATIONS.md) pour le workflow de développement complet.
