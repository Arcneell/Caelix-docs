# Deploy Documentation

Public documentation is hosted in the **[SORK-docs](https://github.com/Arcneell/SORK-docs)** repository and served via GitHub Pages:

**https://arcneell.github.io/SORK-docs/**

The **SORK** repository (private) is the source; a CI workflow syncs `docs/` and `mkdocs.yml` to SORK-docs on every push to `master`.

---

## Local prerequisites

```bash
pip3 install mkdocs-material mkdocs-static-i18n
```

!!! note
    The `mkdocs-static-i18n` plugin is **required** (bilingual FR/EN documentation).

## Local preview

From the SORK repository root:

```bash
mkdocs serve
```

→ http://127.0.0.1:8000

## Static build (without deploying)

```bash
mkdocs build --strict
```

Output in `site/`.

---

## CI — automatic sync (SORK → SORK-docs)

Workflow: `.github/workflows/sync-docs.yml`

Triggered when `docs/**` or `mkdocs.yml` changes on `master`.

### Required secret: `DOCS_PUSH_TOKEN`

The SORK `GITHUB_TOKEN` **cannot** push to SORK-docs (separate repository). A dedicated PAT is required.

**Create the token**

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. Scopes: **`repo`** (or **`public_repo`** if SORK-docs remains public)
3. Copy the token

**Or fine-grained token**:

1. **Settings** → **Developer settings** → **Fine-grained tokens**
2. Repository access: **Only select repositories** → `SORK-docs`
3. Permissions → **Contents**: **Read and write**

**Add the secret**

1. **SORK** repo → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** → name: **`DOCS_PUSH_TOKEN`**, value: the PAT
3. Re-run the workflow: **Actions** → **Sync docs to SORK-docs** → **Run workflow**

### Error « Invalid username or token »

The `DOCS_PUSH_TOKEN` secret is **expired, revoked, or incorrect**. Regenerate a PAT, update the secret, then re-run the workflow.

---

## Pages deployment (SORK-docs)

The **SORK-docs** repository has its own `deploy-docs.yml` workflow:

1. Push to `main` on SORK-docs (via sync above)
2. MkDocs build `--strict`
3. GitHub Pages deployment

No manual action required after a successful sync.

---

## Manual sync (fallback)

If CI is unavailable:

```bash
git clone https://github.com/Arcneell/SORK-docs.git /tmp/sork-docs
rm -rf /tmp/sork-docs/docs /tmp/sork-docs/mkdocs.yml
cp -r docs /tmp/sork-docs/docs
cp mkdocs.yml /tmp/sork-docs/mkdocs.yml
cd /tmp/sork-docs
git add -A
git commit -m "docs: manual sync from SORK"
git push origin main
```
