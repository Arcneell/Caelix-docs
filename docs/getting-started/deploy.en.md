# Deploy Documentation

Public documentation is hosted in the **[Caelix-docs](https://github.com/Arcneell/Caelix-docs)** repository and served via GitHub Pages:

**https://arcneell.github.io/Caelix-docs/**

The **Caelix** repository (private) is the source. A CI workflow syncs `docs/` and `mkdocs.yml` to Caelix-docs on every push to `master`.

---

## Local prerequisites

```bash
pip3 install mkdocs-material mkdocs-static-i18n
```

!!! note
    The `mkdocs-static-i18n` plugin is **required** (bilingual FR/EN documentation).

## Local preview

From the Caelix repository root:

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

## CI: automatic sync (Caelix → Caelix-docs)

Workflow: `.github/workflows/sync-docs.yml`

Triggered when `docs/**` or `mkdocs.yml` changes on `master`.

### Required secret: `DOCS_PUSH_TOKEN`

The Caelix `GITHUB_TOKEN` cannot push to Caelix-docs, which is a separate repository. A dedicated PAT is required.

**Create the token**

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. Scopes: **`repo`** (or **`public_repo`** if Caelix-docs remains public)
3. Copy the token

**Or fine-grained token**:

1. **Settings** → **Developer settings** → **Fine-grained tokens**
2. Repository access: **Only select repositories** → `Caelix-docs`
3. Permissions → **Contents**: **Read and write**

**Add the secret**

1. **Caelix** repo → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** → name: **`DOCS_PUSH_TOKEN`**, value: the PAT
3. Re-run the workflow: **Actions** → **Sync docs to Caelix-docs** → **Run workflow**

### Error "Invalid username or token"

The `DOCS_PUSH_TOKEN` secret is expired, revoked, or incorrect. Regenerate a PAT, update the secret, then re-run the workflow.

---

## Pages deployment (Caelix-docs)

The **Caelix-docs** repository has its own `deploy-docs.yml` workflow:

1. Push to `main` on Caelix-docs (via the sync above)
2. MkDocs build `--strict`
3. GitHub Pages deployment

No manual action is required after a successful sync.

---

## Manual sync (fallback)

If CI is unavailable:

```bash
git clone https://github.com/Arcneell/Caelix-docs.git /tmp/caelix-docs
rm -rf /tmp/caelix-docs/docs /tmp/caelix-docs/mkdocs.yml
cp -r docs /tmp/caelix-docs/docs
cp mkdocs.yml /tmp/caelix-docs/mkdocs.yml
cd /tmp/caelix-docs
git add -A
git commit -m "docs: manual sync from Caelix"
git push origin main
```
