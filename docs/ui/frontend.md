# Frontend

Le frontend Caelix est une Single Page Application (SPA) construite avec Vue 3 et TypeScript.

## Stack

| Outil | Version | Rôle |
|---|---|---|
| **Vue 3** | 3.5 | Framework réactif (Composition API, `<script setup>`) |
| **TypeScript** | 5.9 | Typage statique (build `vue-tsc -b`, réponses API typées dans `src/types/`) |
| **Vite** | 8.x | Build tool & dev server |
| **Tailwind CSS** | 4.x | Styles utilitaires |
| **Pinia** | 3.x | State management (`stores/auth.ts`, `stores/app.ts`) |
| **Vue Router** | 4.x | Navigation SPA (mode hash `#/`) + guards |
| **Vue I18n** | 11.x | Internationalisation FR/EN |
| **Lucide** | — | Icônes |
| **Vitest** | 4.x | Tests unitaires (jsdom + Vue Test Utils) |

## Structure des vues

### Dashboard

La page d'accueil affiche :

- **Statut du daemon** : indicateur basé sur le heartbeat (actif, inactif, inconnu)
- **Services** : carte résumée de chaque service avec état, actions rapides
- **Système** : CPU, mémoire, disque du serveur hôte
- **Alertes récentes** : dernières notifications

### Docker

Gestion des ressources Docker organisée en sous-vues :

- **Containers** : tableau avec filtres, actions en ligne, logs en modal, bouton d'accès à l'assistant de création
- **Assistant de création** : wizard guidé en 6 étapes pour créer un groupe de conteneurs (sélection d'images Docker Hub avec recherche, configuration auto-remplie des ports/volumes/env depuis les métadonnées de l'image, réseau dédié ou existant, orchestrateur Caelix avec health checks, autoscale complet avec proxy dédié)
- **Images** : galerie avec pull, build, suppression
- **Volumes** : liste avec taille et attachements
- **Networks** : topologie des réseaux
- **Stacks** : gestion Docker Compose
- **System Info** : version Docker, storage driver, nombre de conteneurs
- **Events** : flux en direct des événements Docker

### Orchestrator

Interface spécifique Caelix :

- **Services** : état détaillé de chaque service orchestré (santé, replicas, dernière action)
- **Manifest Editor** : édition syntaxique du fichier INI avec validation en direct
- **Autoscale Dashboard** : graphiques de métriques, nombre de replicas, seuils actifs
- **Incidents** : tableau filtrable par date, service, sévérité
- **Audit Journal** : timeline des opérations conteneur avec filtrage

### AppStore

Déploiement simplifié :

- Catalogue de templates (services préconfigurés)
- Sources de templates distantes
- **WizardModal** : assistant multi-étapes pour le déploiement

### Logs

Visionneuse centralisée :

- Logs daemon Caelix (JSON formatté)
- Logs conteneurs (avec suivi temps réel)
- Logs backend UI

### Settings

- Gestion des utilisateurs (admin uniquement)
- Gestion des notifications lues/non lues
- Préférences d'affichage

## Composants réutilisables

| Composant | Description |
|---|---|
| `DataTable` | Tableau avec tri, filtrage, pagination |
| `StatusBadge` | Badge coloré selon l'état (running, stopped, unhealthy) |
| `JsonViewer` | Affichage formatté de JSON |
| `ConfirmModal` | Dialogue de confirmation pour les actions destructives |
| `FeedbackToast` | Notification temporaire (succès, erreur) |
| `DeployProgress` | Barre de progression pour les déploiements |
| `WizardModal` | Assistant multi-étapes |
| `ArrayField` | Champ de formulaire pour les listes (ports, volumes, env) |
| `ContainerWizard` | Formulaire complet de création de conteneur |
| `ServiceAssistant` | Assistant guidé de création multi-conteneurs, découpé en étapes (`ServiceAssistantImageStep`, `…ContainerStep`, `…OrchestratorStep`, `…SummaryStep`) |
| `StackAssistant` | Assistant de déploiement de stacks applicatives (templates) |
| `UsersSettingsPanel` / `WebhooksSettingsPanel` / `BackupSettingsPanel` / `RegistriesSettingsPanel` | Panneaux extraits de `SettingsView` |

!!! note "Découpage des gros composants"
    Les deux assistants `ServiceAssistant` et `StackAssistant` (~1600 lignes chacun à l'origine) ont été découpés en sous-composants d'étape, et `SettingsView` en panneaux dédiés, pour améliorer la lisibilité et la testabilité.

## Authentification et état

- **Session SPA** : aucun token n'est stocké dans `localStorage`. Le JWT est gardé en mémoire pour la durée de l'onglet (`stores/auth.ts`), et le cookie httpOnly `caelix_session` permet de ré-authentifier après un rafraîchissement via `/api/auth/me`.
- **Requêtes** : le client (`src/api/client.ts`) ajoute l'en-tête `Authorization: Bearer` si un token est en mémoire ; le cookie est envoyé automatiquement par le navigateur.
- **Flux SSE** : `src/api/sse.ts` échange d'abord le JWT contre un ticket à usage unique (`/api/auth/sse-ticket`) puis ouvre l'`EventSource` avec `?ticket=`, avec reconnexion automatique.

## Tests

Les tests unitaires (`src/__tests__/`, ~80 cas avec Vitest) couvrent les stores Pinia, les guards du routeur, le client API (gestion 401/429/timeout, token jamais persisté), les utilitaires et les composants partagés (`DataTable`, `StatusBadge`, `ConfirmModal`, `ArrayField`, `DeployProgress`, `WebhookNotificationCard`). Lancer avec `npm run test`.

## Build de production

```bash
cd ui/frontend
npm install
npm run build
```

Les assets sont générés dans `dist/` et servis en tant que fichiers statiques par le backend FastAPI.

## Développement

```bash
cd ui/frontend
npm install
npm run dev
```

Le serveur de développement Vite tourne sur `http://localhost:5173` avec hot-reload. Il proxifie les requêtes `/api` vers le backend FastAPI.
