# Distribution & Release

Ce guide explique comment construire, publier et distribuer SORK sous forme d'image Docker pré-compilée, sans exposer le code source.

## Architecture de distribution

SORK est distribué sous forme d'une **image Docker unique** qui contient :

| Composant | Contenu | Protection |
|-----------|---------|------------|
| Backend Python | Bytecode `.pyc` compilé | Source `.py` supprimé de l'image |
| Frontend Vue | JS/CSS minifié par Vite | Code source non inclus |
| Moteur Bash | Scripts `bin/` et `lib/` | Inclus pour extraction sur l'hôte |
| Templates | `manifest.ini.dist`, `notify.ini.example` | Fichiers de configuration |

L'installeur (`scripts/install.sh`) extrait le moteur depuis l'image et configure le serveur hôte.

## Prérequis (une seule fois)

### 1. Créer un token GitHub

1. Aller sur [github.com/settings/tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. Scopes requis : `write:packages`, `read:packages`
4. Copier le token

### 2. Se connecter au registry

```bash
echo "ghp_VOTRE_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin
```

### 3. Configurer la visibilité du package

Après le premier push :

1. Aller sur [github.com/Arcneell?tab=packages](https://github.com/Arcneell?tab=packages)
2. Cliquer sur le package `sork`
3. **Package settings** > **Danger Zone** > **Change visibility**
4. Choisir **Public** (ou configurer l'accès pour des utilisateurs spécifiques)

### 4. CI GitHub Actions — secret `GHCR_TOKEN` (requis si push 403)

Le workflow Release (`.github/workflows/release.yml`) pousse l'image vers GHCR. Si le job `build-and-push` échoue avec **`403 Forbidden`**, le `GITHUB_TOKEN` intégré n'a pas le droit d'écrire sur le package.

**Option A — PAT dédié (recommandé)**

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. Scopes : `write:packages`, `read:packages`
3. Copier le token
4. Repo **SORK** → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
5. Nom : `GHCR_TOKEN`, valeur : le PAT
6. Relancer le workflow Release (Actions → Release → Re-run, ou re-push du tag `v1.3.0`)

**Option B — Permissions workflow**

1. Repo **SORK** → **Settings** → **Actions** → **General**
2. **Workflow permissions** → cocher **Read and write permissions**
3. Sauvegarder, puis relancer le workflow

**Lier le package au dépôt** (si le package existe déjà) :

1. [Packages `sork`](https://github.com/users/Arcneell/packages/container/sork/settings) → **Package settings**
2. **Manage Actions access** / **Connect repository** → ajouter `Arcneell/SORK`

## Build et publication

### Build local (test)

```bash
./scripts/build-release.sh
```

Le script :

1. Build l'image multi-stage depuis le `Dockerfile` racine
2. Compile le backend Python en `.pyc` (supprime les `.py`)
3. Bundle le frontend Vue minifié
4. Embarque le moteur Bash dans `/opt/sork/`
5. Vérifie qu'aucun fichier `.py` source n'est présent
6. Tag l'image en `:latest` et `:<version>` (depuis le fichier `VERSION`)

### Push vers le registry

```bash
./scripts/build-release.sh --push
```

### Options

```bash
./scripts/build-release.sh [options]

  --push              Push vers le registry après le build
  --tag TAG           Tag supplémentaire (peut être répété)
  --registry REG      Registry (défaut : ghcr.io/arcneell)
  --no-cache          Build sans cache Docker
```

### Exemples

```bash
# Release stable
./scripts/build-release.sh --push

# Version beta
./scripts/build-release.sh --push --tag beta

# Version spécifique
./scripts/build-release.sh --push --tag 1.3.0
```

## Installation côté client

Le client s'authentifie au registry puis extrait et lance l'installeur depuis l'image :

```bash
# 1. Authentification (token fourni avec la licence)
echo "TOKEN_CLIENT" | docker login ghcr.io -u Arcneell --password-stdin

# 2. Installation
docker run --rm ghcr.io/arcneell/sork:latest cat /opt/sork/install.sh | bash -s -- --with-systemd
```

Le script d'installation est embarqué dans l'image — aucun accès au code source ni au dépôt GitHub n'est nécessaire.

Voir [Installation](installation.md) pour toutes les options.

## Checklist de release

1. Mettre à jour le fichier `VERSION`
2. Mettre à jour le badge version dans `README.md`
3. Lancer `./scripts/build-release.sh --push`
4. Taguer le commit git :

```bash
git tag v$(cat VERSION)
git push --tags
```

## Structure de l'image

```
ghcr.io/arcneell/sork:<version>
├── /app/
│   ├── app/                  # Backend Python (bytecode .pyc uniquement)
│   │   ├── main.pyc            # Backend compilé (bytecode sourceless, compileall -b)
│   │   ├── core/*.pyc          # Modules core compilés
│   │   └── routers/*.pyc       # Routers compilés
│   └── static/               # Frontend Vue (JS/CSS minifié)
└── /opt/sork/                # Moteur (extrait par l'installeur)
    ├── bin/sork
    ├── lib/*.sh
    ├── lib/*.py
    ├── etc/
    └── VERSION
```

!!! note "Sécurité du code"
    Le backend Python est compilé en bytecode et les fichiers `.py` source sont supprimés de l'image. Le frontend est minifié par Vite. Le code n'est pas directement lisible, mais aucune protection logicielle n'est inviolable. La vraie protection réside dans la **licence** et la **valeur du service** (mises à jour, support, documentation).
