# Distribution & Release

This guide explains how to build, publish, and distribute Caelix as a pre-compiled Docker image without exposing the source code.

## Distribution Architecture

Caelix is distributed as a **single Docker image** containing:

| Component | Contents | Protection |
|-----------|----------|------------|
| Python backend | Compiled `.pyc` bytecode | `.py` source removed from image |
| Vue frontend | Minified JS/CSS by Vite | Source code not included |
| Bash engine | `bin/` and `lib/` scripts | Included for host extraction |
| Templates | `manifest.ini.dist`, `notify.ini.example` | Configuration files |

The installer (`scripts/install.sh`) extracts the engine from the image and configures the host server.

## Prerequisites (once)

### 1. Create a GitHub Token

1. Go to [github.com/settings/tokens](https://github.com/settings/tokens)
2. **Generate new token (classic)**
3. Required scopes: `write:packages`, `read:packages`
4. Copy the token

### 2. Log in to the registry

```bash
echo "ghp_YOUR_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin
```

### 3. Configure package visibility

After the first push:

1. Go to [github.com/Arcneell?tab=packages](https://github.com/Arcneell?tab=packages)
2. Click the `caelix` package
3. **Package settings** > **Danger Zone** > **Change visibility**
4. Choose **Public** (or configure access for specific users)

### 4. GitHub Actions CI — `GHCR_TOKEN` secret (required on 403)

The Release workflow (`.github/workflows/release.yml`) pushes to GHCR. If `build-and-push` fails with **`403 Forbidden`**, the built-in `GITHUB_TOKEN` cannot write to the package.

**Option A — Dedicated PAT (recommended)**

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. Scopes: `write:packages`, `read:packages`
3. Copy the token
4. Repo **Caelix** → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**
5. Name: `GHCR_TOKEN`, value: the PAT
6. Re-run the Release workflow (Actions → Release → Re-run, or re-push tag `v1.4.0`)

**Option B — Workflow permissions**

1. Repo **Caelix** → **Settings** → **Actions** → **General**
2. **Workflow permissions** → enable **Read and write permissions**
3. Save, then re-run the workflow

**Link package to repository** (if the package already exists):

1. [Package `caelix` settings](https://github.com/users/Arcneell/packages/container/caelix/settings) → **Connect repository** → add `Arcneell/Caelix`

## Build and Publish

### Local build (test)

```bash
./scripts/build-release.sh
```

The script:

1. Builds the multi-stage Docker image from the root `Dockerfile`
2. Compiles the Python backend to `.pyc` (removes `.py` source)
3. Bundles the minified Vue frontend
4. Embeds the Bash engine at `/opt/caelix/`
5. Verifies no `.py` source leaked into the image
6. Tags the image as `:latest` and `:<version>` (from `VERSION` file)

### Push to registry

```bash
./scripts/build-release.sh --push
```

### Options

```bash
./scripts/build-release.sh [options]

  --push              Push to registry after build
  --tag TAG           Additional tag (can be repeated)
  --registry REG      Registry (default: ghcr.io/arcneell)
  --no-cache          Build without Docker cache
```

### Examples

```bash
# Stable release
./scripts/build-release.sh --push

# Beta version
./scripts/build-release.sh --push --tag beta

# Specific version
./scripts/build-release.sh --push --tag 1.4.0
```

## Client-Side Installation

The client authenticates to the registry then extracts and runs the installer from the image:

```bash
# 1. Authenticate (token provided with the license)
echo "CLIENT_TOKEN" | docker login ghcr.io -u Arcneell --password-stdin

# 2. Install
docker run --rm ghcr.io/arcneell/caelix:latest cat /opt/caelix/install.sh | bash -s -- --with-systemd
```

The install script is embedded in the image — no access to the source code or GitHub repository is needed.

See [Installation](installation.en.md) for all options.

## Release Checklist

Full process: **[Package Update Process](package-release.en.md)**.

Summary:

```bash
# 1. Create docs/getting-started/release-notes/vX.Y.Z.md (+ .en.md)
# 2. Publish (CI — recommended)
./scripts/release-package.sh X.Y.Z
```

Or manually:

1. Update the `VERSION` file
2. Update the version badge in `README.md`
3. Run `./scripts/build-release.sh --push` (local) **or** push tag `vX.Y.Z` (CI)
4. Tag the commit:

```bash
git tag v$(cat VERSION)
git push --tags
```

## Image Structure

```
ghcr.io/arcneell/caelix:<version>
├── /app/
│   ├── app/                  # Python backend (bytecode .pyc only)
│   │   ├── main.pyc            # Compiled backend (sourceless bytecode, compileall -b)
│   │   ├── core/*.pyc          # Compiled core modules
│   │   └── routers/*.pyc       # Compiled routers
│   └── static/               # Vue frontend (minified JS/CSS)
└── /opt/caelix/                # Engine (extracted by installer)
    ├── bin/caelix
    ├── lib/*.sh
    ├── lib/*.py
    ├── etc/
    └── VERSION
```

!!! note "Code Security"
    The Python backend is compiled to bytecode and `.py` source files are removed from the image. The frontend is minified by Vite. The code is not directly readable, but no software protection is unbreakable. The real protection lies in the **license** and the **service value** (updates, support, documentation).
