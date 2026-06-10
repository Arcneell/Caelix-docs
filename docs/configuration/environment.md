# Variables d'environnement

SORK peut être configuré via des variables d'environnement en complément du manifest.

## Variables principales

| Variable | Défaut | Description |
|---|---|---|
| `SORK_MANIFEST` | `etc/manifest.ini` | Chemin vers le fichier manifest |
| `SORK_NOTIFY_CONF` | `etc/notify.ini` | Chemin vers la configuration des notifications |
| `SORK_DATA` | `./.sork` | Répertoire de données d'exécution |
| `SORK_INTERVAL` | `15` | Intervalle de réconciliation en secondes |
| `SORK_MAX_REPAIR` | `5` | Seuil d'échecs avant alerte |
| `SORK_LOG_LEVEL` | `info` | Niveau de log : `debug`, `info`, `warn`, `error` |
| `SORK_STRICT_LOCAL` | `0` | Si `1`, les URLs de health check doivent cibler localhost |

## Priorité de configuration

Les variables d'environnement sont écrasées par les valeurs du manifest si elles sont définies dans `[orchestrator]` :

```
Variable d'env < manifest.ini [orchestrator]
```

Par exemple, si `SORK_INTERVAL=15` et que le manifest définit `interval = 10`, c'est `10` qui sera utilisé.

## Exemples d'utilisation

### Lancement avec un manifest personnalisé

```bash
SORK_MANIFEST=/etc/sork/manifest.ini bin/sork run
```

### Répertoire de données séparé

```bash
SORK_DATA=/var/lib/sork bin/sork run
```

### Mode debug

```bash
SORK_LOG_LEVEL=debug bin/sork once
```

### Mode strict (health checks localhost uniquement)

```bash
SORK_STRICT_LOCAL=1 bin/sork validate
```

Ce mode est utile pour s'assurer que les health checks ne ciblent pas des services distants, ce qui pourrait causer des faux positifs ou des fuites d'information.

## Service systemd

Le fichier `sork.global.service` configure les variables pour un fonctionnement en tant que service :

```ini
[Service]
Environment=SORK_MANIFEST=/opt/sork/etc/manifest.ini
Environment=SORK_NOTIFY_CONF=/opt/sork/etc/notify.ini
```

Pour ajouter des variables supplémentaires :

```bash
sudo systemctl edit sork
```

```ini
[Service]
Environment=SORK_LOG_LEVEL=debug
Environment=SORK_DATA=/var/lib/sork
```
