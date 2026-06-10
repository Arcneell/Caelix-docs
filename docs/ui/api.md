# API REST

Le backend FastAPI expose plus de 150 endpoints REST, répartis sur 21 routers et organisés par catégorie.

---

## Base URL et authentification

```
Base : http://localhost:8080/api
```

L'authentification est obligatoire. Les clients CLI/API obtiennent un token JWT via `/api/auth/login`, puis l'incluent dans chaque requête via l'en-tête `Authorization: Bearer`. (La SPA, elle, s'authentifie via le cookie httpOnly `caelix_session` posé à la connexion — voir [Authentification](../configuration/authentication.md).)

```bash
# Connexion
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"votre_mdp"}' | jq -r .token)

# Utilisation
curl -H "Authorization: Bearer $TOKEN" http://localhost:8080/api/containers/

# Pour SSE (EventSource ne supporte pas les headers) : obtenir un ticket à usage unique
TICKET=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" \
  http://localhost:8080/api/auth/sse-ticket | jq -r .ticket)
curl "http://localhost:8080/api/stream?ticket=$TICKET"
```

Voir [Authentification](../configuration/authentication.md) pour les détails (cookie de session, tickets SSE, rôles).

---

## Système et santé

| Méthode | Endpoint | Auth | Description |
|---|---|---|---|
| `GET` | `/api/ping` | Non | Health check (réponse instantanée) |
| `GET` | `/api/state` | Oui | État agrégé complet (conteneurs, santé, incidents, autoscale) |
| `GET` | `/api/stream` | Oui | SSE : mises à jour d'état (~2s) |
| `GET` | `/metrics` | Optionnel | Métriques Prometheus |
| `GET` | `/api/host/details` | Oui | OS, kernel, CPU, mémoire, disques, uptime |
| `GET` | `/api/docker/info` | Oui | `docker info` en JSON |
| `GET` | `/api/docker/version` | Oui | `docker version` en JSON |
| `GET` | `/api/docker/df` | Oui | Utilisation disque Docker par type |
| `GET` | `/api/docker/summary` | Oui | Compteurs : conteneurs, images, volumes, réseaux |

---

## Conteneurs

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/containers/` | Lister (`all=1` pour inclure les arrêtés) |
| `GET` | `/api/containers/{name}/inspect` | Détails complets (`docker inspect`) |
| `GET` | `/api/containers/{name}/logs` | Logs (params : `tail`, `timestamps`, `search`) |
| `GET` | `/api/containers/{name}/logs/stream` | SSE : logs en temps réel |
| `GET` | `/api/containers/{name}/logs/download` | Télécharger les logs en fichier texte |
| `GET` | `/api/containers/{name}/stats` | Snapshot CPU/mémoire/réseau |
| `GET` | `/api/containers/{name}/stats/stream` | SSE : stats temps réel (~2s) |
| `GET` | `/api/containers/{name}/top` | Processus en cours dans le conteneur |
| `GET` | `/api/containers/{name}/changes` | Modifications du filesystem |
| `GET` | `/api/containers/{name}/export` | Télécharger le conteneur en TAR |
| `GET` | `/api/containers/{name}/clone-config` | Extraire la config d'un conteneur existant |
| `POST` | `/api/containers/` | Créer un conteneur (spec complète) |
| `POST` | `/api/containers/create/preview` | Prévisualiser la commande `docker run` |
| `POST` | `/api/containers/preflight` | Vérifications pré-création (ports, noms, volumes) |
| `POST` | `/api/containers/pull` | Tirer une image : `{"image":"nginx:latest"}` |
| `GET` | `/api/containers/available-ports` | Ports hôte disponibles (`count=3`) |
| `POST` | `/api/containers/{name}/action` | Action : `{"action":"start\|stop\|restart\|remove\|kill\|pause\|unpause"}` |
| `POST` | `/api/containers/{name}/exec` | Exécuter : `{"cmd":"ls -la /app"}` |
| `POST` | `/api/containers/{name}/commit` | Committer : `{"repo":"myimage","tag":"v1"}` |
| `POST` | `/api/containers/{name}/rename` | Renommer : `{"name":"nouveau-nom"}` |

---

## Images

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/images/` | Lister toutes les images (cache) |
| `GET` | `/api/images/{id}/inspect` | Détails d'une image |
| `GET` | `/api/images/{id}/history` | Historique des layers |
| `GET` | `/api/images/{id}/export` | Télécharger l'image en fichier `.tar` |
| `GET` | `/api/images/search` | Recherche registry (`q=terme`) |
| `GET` | `/api/images/metadata` | Métadonnées d'une image (`image=nginx:latest`) : ports, volumes, env. Supporte Docker Hub sans pull. |
| `POST` | `/api/images/pull` | Tirer une image : `{"image":"nginx:latest"}` |
| `POST` | `/api/images/build` | Construire une image : `{"dockerfile_content":"...","tag":"mon-image:v1","no_cache":false}` |
| `POST` | `/api/images/import` | Importer depuis un `.tar` (body = fichier brut) |
| `POST` | `/api/images/{id}/tag` | Taguer : `{"repo":"mon-registry/image","tag":"latest"}` |
| `POST` | `/api/images/{id}/push` | Pousser vers un registry |
| `POST` | `/api/images/{id}/remove` | Supprimer une image (`{"force":true}` optionnel) |
| `POST` | `/api/images/prune` | Supprimer les images dangling |

---

## Volumes

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/volumes/` | Lister tous les volumes |
| `GET` | `/api/volumes/{name}/inspect` | Détails d'un volume |
| `POST` | `/api/volumes/create` | Créer un volume |
| `POST` | `/api/volumes/{name}/remove` | Supprimer un volume |
| `POST` | `/api/volumes/prune` | Supprimer les volumes inutilisés |

---

## Réseaux

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/networks/` | Lister tous les réseaux |
| `GET` | `/api/networks/{name}/inspect` | Détails d'un réseau |
| `POST` | `/api/networks/create` | Créer un réseau |
| `POST` | `/api/networks/{name}/remove` | Supprimer un réseau |
| `POST` | `/api/networks/{name}/connect` | Connecter un conteneur |
| `POST` | `/api/networks/{name}/disconnect` | Déconnecter un conteneur |

---

## Stacks (Docker Compose)

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/stacks/` | Lister les stacks |
| `GET` | `/api/stacks/{name}/status` | Services + fichier compose |
| `GET` | `/api/stacks/{name}/content` | Contenu brut du fichier compose |
| `GET` | `/api/stacks/{name}/env` | Variables d'environnement de la stack |
| `GET` | `/api/stacks/{name}/webhook/trigger` | Déclencher le webhook via GET |
| `POST` | `/api/stacks/deploy` | Déployer une nouvelle stack |
| `POST` | `/api/stacks/deploy-git` | Déployer depuis un dépôt Git |
| `POST` | `/api/stacks/{name}/update` | Mettre à jour le compose (sauvegarder + redéployer) |
| `POST` | `/api/stacks/{name}/down` | Arrêter une stack |
| `POST` | `/api/stacks/{name}/delete` | Supprimer une stack |
| `POST` | `/api/stacks/{name}/env` | Sauvegarder les variables d'environnement |
| `POST` | `/api/stacks/{name}/webhook` | Créer un webhook de redéploiement |
| `POST` | `/api/stacks/{name}/webhook/trigger` | Déclencher le webhook (pull + redéploiement) |
| `POST` | `/api/stacks/{name}/save-template` | Sauvegarder la stack comme template personnalisé |

---

## Orchestrator

### Manifest

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/manifest/show` | Affichage formatté (`caelix show`) |
| `GET` | `/api/manifest/raw` | Contenu brut du fichier INI |
| `POST` | `/api/manifest/raw` | Sauvegarder (validé avant écriture) |
| `POST` | `/api/validate` | Lancer `caelix validate` |
| `POST` | `/api/doctor` | Lancer `caelix doctor` (options : `--fix`, `--strict-local`) |

### Réconciliation

| Méthode | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/api/reconcile` | — | Lancer `caelix once` |
| `POST` | `/api/reconcile-app` | `{"app":"nom"}` | Réconcilier un seul service |

### Services

| Méthode | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/api/service/action` | `{"app":"x","action":"start\|stop\|restart"}` | Action sur un service |
| `POST` | `/api/service/clear_pause` | `{"app":"x"}` | Lever la pause manuelle |
| `POST` | `/api/apps/upsert` | `{"name":"app","values":{...}}` | Créer ou modifier un service |
| `POST` | `/api/apps/delete` | `{"name":"app"}` | Supprimer un service + conteneurs |
| `POST` | `/api/apps/bump_config` | `{"name":"app"}` | Incrémenter config_version |

### Paramètres globaux

| Méthode | Endpoint | Body | Description |
|---|---|---|---|
| `POST` | `/api/orchestrator/patch` | `{"orchestrator":{"interval":"8"}}` | Modifier les globales |
| `POST` | `/api/manifest/orchestrator/lang` | `{"value":"fr\|en"}` | Changer la langue de l'orchestrateur |
| `POST` | `/api/proxy/config` | `{"proxy":{"listen":"..."}}` | Modifier la config proxy |

---

## Autoscale

| Méthode | Endpoint | Body | Description |
|---|---|---|---|
| `GET` | `/api/autoscale/state` | — | État de tous les services autoscalés |
| `POST` | `/api/autoscale/scale` | `{"app":"x","action":"up\|down"}` | Scale manuel |
| `POST` | `/api/autoscale/stress` | `{"app":"x","action":"add\|reset","amount":50}` | Injection de latence pour test |

---

## Incidents et alertes

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/incidents?limit=50` | Liste des incidents récents |
| `POST` | `/api/alerts/ack` | Acquitter un incident par ID |
| `POST` | `/api/alerts/ack_all` | Acquitter tous les incidents |

---

## Audit

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/audit/recent?limit=50` | Événements récents |
| `POST` | `/api/audit/clear` | Effacer les logs d'audit |

---

## Notifications

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/notifications/` | Lister les notifications |
| `GET` | `/api/notifications/stream` | SSE : flux temps réel |
| `POST` | `/api/notify/save` | Sauvegarder la config notifications (Discord, Slack, Teams, Telegram, SMTP) |
| `POST` | `/api/notify/test` | Envoyer un message test sur tous les canaux activés |

---

## Nettoyage Docker

| Méthode | Endpoint | Description |
|---|---|---|
| `POST` | `/api/docker/prune/container` | Supprimer conteneurs arrêtés |
| `POST` | `/api/docker/prune/image` | Supprimer images dangling |
| `POST` | `/api/docker/prune/volume` | Supprimer volumes inutilisés |
| `POST` | `/api/docker/prune/network` | Supprimer réseaux inutilisés |
| `POST` | `/api/docker/prune/build_cache` | Vider le cache de build |

---

## Événements Docker

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/docker/events/stream` | SSE : événements Docker en direct |
| `GET` | `/api/docker/events/recent?since=1h` | Événements récents |

---

## Templates

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/templates/external` | Lister les templates externes (sources configurées) |
| `GET` | `/api/templates/sources` | Lister les sources de templates |
| `POST` | `/api/templates/sources` | Sauvegarder les sources de templates |
| `POST` | `/api/templates/deploy-full` | Déployer un template (workflow complet avec Caelix) |
| `POST` | `/api/templates/save` | Sauvegarder les templates de services Caelix |
| `GET` | `/api/templates/custom` | Lister les templates personnalisés |
| `POST` | `/api/templates/custom/create` | Créer un template personnalisé |
| `POST` | `/api/templates/custom/{name}/update` | Modifier un template personnalisé |
| `POST` | `/api/templates/custom/{name}/delete` | Supprimer un template personnalisé |

---

## Webhooks

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/webhooks/` | Lister les webhooks de conteneurs |
| `POST` | `/api/webhooks/{name}/create` | Créer un webhook pour un conteneur |
| `POST` | `/api/webhooks/{name}/delete` | Supprimer un webhook |
| `GET` | `/api/webhooks/{name}/trigger` | Déclencher via GET (pour intégrations CI) |
| `POST` | `/api/webhooks/{name}/trigger` | Déclencher via POST (pull + recréation) |

---

## Registries

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/registries/` | Lister les registries configurés |
| `POST` | `/api/registries/add` | Ajouter un registry |
| `POST` | `/api/registries/{name}/remove` | Supprimer un registry |
| `POST` | `/api/registries/{name}/login` | Se connecter à un registry |

---

## Sauvegarde et restauration

### Configuration Caelix

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/backup` | Télécharger une archive `.tar.gz` de toute la configuration Caelix |
| `POST` | `/api/restore` | Restaurer la configuration depuis une archive (body = fichier brut) |

### Sauvegarde planifiée des volumes

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/backup/status` | État des sauvegardes pour tous les services |
| `POST` | `/api/backup/trigger/{name}` | Déclencher une sauvegarde immédiate |
| `GET` | `/api/backup/list/{name}` | Lister les archives d'un service |
| `GET` | `/api/backup/download/{name}/{filename}` | Télécharger une archive |
| `POST` | `/api/backup/config/{name}` | Modifier la configuration backup d'un service |
| `GET` | `/api/backup/defaults` | Template de configuration par défaut |

---

## Mise à jour automatique

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/update/status` | État des mises à jour pour tous les services (enabled, interval, strategy, digest, update_available) |
| `POST` | `/api/update/check/{name}` | Forcer une vérification immédiate (docker pull + comparaison digest) |
| `POST` | `/api/update/apply/{name}` | Appliquer une mise à jour en attente (pull + rollout) |
| `POST` | `/api/update/config/{name}` | Modifier la configuration auto-update d'un service |

---

## Divers

| Méthode | Endpoint | Description |
|---|---|---|
| `GET` | `/api/logs/search` | Recherche dans les logs conteneurs |
| `GET` | `/api/openapi.json` | Spec OpenAPI 3.0.3 |
| `POST` | `/api/data/clear` | Effacer incidents et/ou audit |

---

## Server-Sent Events (SSE)

Les flux SSE permettent des mises à jour temps réel sans polling. L'authentification se fait via un **ticket à usage unique** (`POST /api/auth/sse-ticket`, valable une fois, 30 s) passé en paramètre `?ticket=` — jamais le JWT directement :

```javascript
const { ticket } = await (await fetch('/api/auth/sse-ticket', { method: 'POST' })).json();
const evtSource = new EventSource(`/api/stream?ticket=${encodeURIComponent(ticket)}`);
evtSource.onmessage = (event) => {
  const state = JSON.parse(event.data);
  // Mettre à jour l'interface
};
```

Flux disponibles :

| Endpoint | Contenu | Fréquence |
|---|---|---|
| `/api/stream` | État global agrégé | ~2s |
| `/api/containers/{name}/logs/stream` | Logs du conteneur | Temps réel |
| `/api/containers/{name}/stats/stream` | Stats du conteneur | ~2s |
| `/api/docker/events/stream` | Événements Docker | Temps réel |
