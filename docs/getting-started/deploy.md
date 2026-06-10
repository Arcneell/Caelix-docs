# Déploiement de la documentation

La documentation publique est hébergée sur le dépôt **[Caelix-docs](https://github.com/Arcneell/Caelix-docs)** et servie via GitHub Pages :

**https://arcneell.github.io/Caelix-docs/**

Le dépôt **Caelix** (privé) est la source ; un workflow CI synchronise `docs/` et `mkdocs.yml` vers Caelix-docs à chaque push sur `master`.

---

## Prérequis locaux

```bash
pip3 install mkdocs-material mkdocs-static-i18n
```

!!! note
    Le plugin `mkdocs-static-i18n` est **obligatoire** (documentation bilingue FR/EN).

## Prévisualisation locale

Depuis la racine du dépôt Caelix :

```bash
mkdocs serve
```

→ http://127.0.0.1:8000

## Build statique (sans déployer)

```bash
mkdocs build --strict
```

Résultat dans `site/`.

---

## CI — synchronisation automatique (Caelix → Caelix-docs)

Workflow : `.github/workflows/sync-docs.yml`

Déclenché quand `docs/**` ou `mkdocs.yml` change sur `master`.

### Secret requis : `DOCS_PUSH_TOKEN`

Le `GITHUB_TOKEN` du dépôt Caelix **ne peut pas** pousser vers Caelix-docs (dépôt séparé). Un PAT dédié est nécessaire.

**Créer le token**

1. [github.com/settings/tokens](https://github.com/settings/tokens) → **Generate new token (classic)**
2. Scopes : **`repo`** (ou **`public_repo`** si Caelix-docs reste public)
3. Copier le token

**Ou token fine-grained** :

1. **Settings** → **Developer settings** → **Fine-grained tokens**
2. Repository access : **Only select repositories** → `Caelix-docs`
3. Permissions → **Contents** : **Read and write**

**Ajouter le secret**

1. Dépôt **Caelix** → **Settings** → **Secrets and variables** → **Actions**
2. **New repository secret** → nom : **`DOCS_PUSH_TOKEN`**, valeur : le PAT
3. Relancer le workflow : **Actions** → **Sync docs to Caelix-docs** → **Run workflow**

### Erreur « Invalid username or token »

Le secret `DOCS_PUSH_TOKEN` est **expiré, révoqué ou incorrect**. Regénérez un PAT et mettez à jour le secret, puis relancez le workflow.

---

## Déploiement Pages (Caelix-docs)

Le dépôt **Caelix-docs** possède son propre workflow `deploy-docs.yml` :

1. Push sur `main` de Caelix-docs (via sync ci-dessus)
2. Build MkDocs `--strict`
3. Déploiement GitHub Pages

Aucune action manuelle requise après une sync réussie.

---

## Sync manuelle (secours)

Si la CI est indisponible :

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
