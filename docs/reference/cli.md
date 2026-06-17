# Référence CLI

Le binaire `bin/caelix` est le point d'entrée principal de Caelix.

## Usage

```bash
bin/caelix <commande> [options]
```

Ou avec les variables d'environnement :

```bash
CAELIX_MANIFEST=etc/manifest.ini CAELIX_DATA=.caelix bin/caelix <commande>
```

## Commandes

### run

```bash
bin/caelix run
```

Lance la boucle de réconciliation infinie (mode daemon).

- Exécute `reconcile_all()` en boucle
- Intervalle configurable via `[orchestrator] interval` ou `CAELIX_INTERVAL`
- Écrit un heartbeat à chaque cycle
- S'arrête avec `Ctrl+C` ou `SIGTERM`

C'est la commande principale pour le fonctionnement en production.

### agent

```bash
bin/caelix agent
```

Identique à `run`, plus la publication du **statut de nœud** à chaque cycle
(multi-nœud, phase 1). L'agent :

- possède une **identité de nœud** stable (`node-info`) ;
- réconcilie son manifeste local comme en mode mono-hôte ;
- écrit `.caelix/agent/status.json` (méta + état observé) à chaque cycle.

Si un store cluster est configuré (`CAELIX_CLUSTER_BACKEND` = `file` via
`CAELIX_CLUSTER_STORE`, ou `consul`), l'agent publie sa méta dans le store et
**adopte le sous-manifest poussé par le controller** comme source d'état désiré ;
sinon il réconcilie son manifeste local (mono-hôte). Voir la
[RFC multi-nœud](../architecture/multi-node-rfc.md).

### node-info

```bash
bin/caelix node-info
```

Affiche l'identité et les méta-données du nœud en JSON (id, hostname, labels,
moteur, version, CPU, mémoire). L'identité provient de `CAELIX_NODE_ID` si défini,
sinon d'un id généré une fois et persisté dans `.caelix/node/id`. Les labels de
placement se déclarent via `CAELIX_NODE_LABELS` (ex. `zone=eu-west,disk=ssd`).

### node-status

```bash
bin/caelix node-status
```

Écrit puis affiche le statut de l'agent (`.caelix/agent/status.json`) : méta du
nœud + état observé des conteneurs gérés par Caelix (nom, app, état, image,
compteur d'échecs santé).

### once

```bash
bin/caelix once
```

Exécute un seul cycle de réconciliation puis s'arrête.

Cas d'usage :
- Premier lancement après installation
- Intégration dans un cron
- Tests et debug

### reconcile-app

```bash
bin/caelix reconcile-app <nom-du-service>
```

Réconcilie uniquement le service spécifié.

```bash
# Exemple
bin/caelix reconcile-app web
```

### validate

```bash
bin/caelix validate
```

Valide la syntaxe et les clés du manifest. Vérifie :

- Syntaxe INI correcte
- Clés requises présentes (`image`, `health_url` si http)
- Format des ports
- Valeurs valides pour les stratégies
- Si Python est disponible, utilise `manifest_doctor.py` pour une validation étendue

### doctor

```bash
bin/caelix doctor [--fix] [--strict-local]
```

Diagnostic complet du système.

**Options :**

| Option | Description |
|---|---|
| `--fix` | Tente de corriger automatiquement les problèmes détectés |
| `--strict-local` | Vérifie que les URLs de health ciblent localhost |

**Vérifications effectuées :**

- Clés requises présentes
- Format des ports valide
- Stratégies de rollout cohérentes (blue_green requiert `candidate_publish`)
- Types de health, repair, autoscale metric valides
- Écriture dans `CAELIX_DATA`
- Disponibilité Docker/Podman
- Présence de `notify.ini`

**Mode fix :**

```bash
bin/caelix doctor --fix
```

- Sauvegarde l'original en `.bak`
- Corrige les problèmes détectables automatiquement
- Réécrit le manifest corrigé

### show

```bash
bin/caelix show
```

Affiche le manifest chargé, section par section. Utile pour vérifier comment Caelix interprète la configuration.

### version

```bash
bin/caelix version
```

Affiche la version depuis le fichier `VERSION`.

### status

```bash
bin/caelix status
```

Affiche l'état du daemon :

- Dernier heartbeat
- Uptime depuis le dernier démarrage
- Runtime détecté (Docker/Podman)
- Chemins configurés (`CAELIX_MANIFEST`, `CAELIX_DATA`)
- Nombre de services dans le manifest

### resume

```bash
bin/caelix resume <nom-du-service>
```

Reprend la réconciliation d'un service suspendu ou en pause manuelle.

Supprime les fichiers :
- `.caelix/state/<app>.suspend_reconcile`
- `.caelix/state/<app>.manual_pause`

```bash
# Exemple : reprendre un service suspendu après des échecs de création
bin/caelix resume api
```
