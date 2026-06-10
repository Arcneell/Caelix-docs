# Variables d'environnement

Caelix peut être configuré via des variables d'environnement en complément du manifest.

## Variables principales

| Variable | Défaut | Description |
|---|---|---|
| `CAELIX_MANIFEST` | `etc/manifest.ini` | Chemin vers le fichier manifest |
| `CAELIX_NOTIFY_CONF` | `etc/notify.ini` | Chemin vers la configuration des notifications |
| `CAELIX_DATA` | `./.caelix` | Répertoire de données d'exécution |
| `CAELIX_INTERVAL` | `15` | Intervalle de réconciliation en secondes |
| `CAELIX_MAX_REPAIR` | `5` | Seuil d'échecs avant alerte |
| `CAELIX_LOG_LEVEL` | `info` | Niveau de log : `debug`, `info`, `warn`, `error` |
| `CAELIX_STRICT_LOCAL` | `0` | Si `1`, les URLs de health check doivent cibler localhost |

## Priorité de configuration

Les variables d'environnement sont écrasées par les valeurs du manifest si elles sont définies dans `[orchestrator]` :

```
Variable d'env < manifest.ini [orchestrator]
```

Par exemple, si `CAELIX_INTERVAL=15` et que le manifest définit `interval = 10`, c'est `10` qui sera utilisé.

## Exemples d'utilisation

### Lancement avec un manifest personnalisé

```bash
CAELIX_MANIFEST=/etc/caelix/manifest.ini bin/caelix run
```

### Répertoire de données séparé

```bash
CAELIX_DATA=/var/lib/caelix bin/caelix run
```

### Mode debug

```bash
CAELIX_LOG_LEVEL=debug bin/caelix once
```

### Mode strict (health checks localhost uniquement)

```bash
CAELIX_STRICT_LOCAL=1 bin/caelix validate
```

Ce mode est utile pour s'assurer que les health checks ne ciblent pas des services distants, ce qui pourrait causer des faux positifs ou des fuites d'information.

## Service systemd

Le fichier `caelix.global.service` configure les variables pour un fonctionnement en tant que service :

```ini
[Service]
Environment=CAELIX_MANIFEST=/opt/caelix/etc/manifest.ini
Environment=CAELIX_NOTIFY_CONF=/opt/caelix/etc/notify.ini
```

Pour ajouter des variables supplémentaires :

```bash
sudo systemctl edit caelix
```

```ini
[Service]
Environment=CAELIX_LOG_LEVEL=debug
Environment=CAELIX_DATA=/var/lib/caelix
```
