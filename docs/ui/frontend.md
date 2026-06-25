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

## Shell de l'application

La v2.0 introduit une navigation plate (style NetBird / Portainer), pensée pour le cluster. Le shell est défini dans `src/App.vue` :

- une barre latérale unique rend la liste des sections (boutons à plat, sans groupes repliables) ;
- un en-tête affiche le titre de la page, la bande de statut cluster (`components/cluster/ClusterStatusStrip.vue`), le bascule de langue, le bascule de thème clair/sombre, la cloche de notifications et le menu utilisateur ;
- une barre d'onglets (`components/layout/SectionTabs.vue`) pour les sections à plusieurs facettes.

### Configuration des sections

Les sections et leurs onglets sont déclarés de façon centralisée dans `src/config/sections.ts` (`SECTIONS: NavSection[]`). Chaque section porte un `id`, un `labelKey` i18n, une icône Lucide, une route `to`, des `prefixes` de route (état actif et résolution des onglets) et, éventuellement, des `tabs` et un drapeau `clusterOnly`. La fonction `sectionForPath()` fait un *longest-prefix match* pour résoudre la section active. Les routes de vues existantes sont conservées ; seul le *chrome* est nouveau.

| Section | Route | Onglets |
|---|---|---|
| **Overview** | `/` | — (tableau de bord topologie du cluster) |
| **Nodes** *(`clusterOnly`)* | `/nodes` | — |
| **Containers** | `/containers` | `/containers` · `/images` · `/volumes` · `/networks` · `/system` |
| **Services** | `/services` | `/services` · `/autoscale` |
| **Stacks** | `/stacks` | `/stacks` · `/apps` · `/app-store` |
| **Ingress** | `/apps/domains` | `/apps/domains` · `/apps/certificates` |
| **Activity** | `/logs` | `/logs` · `/events` · `/incidents` · `/journal` |
| **Settings** | `/settings` | `/settings` · `/cluster` |

Les sections marquées `clusterOnly` (Nodes) sont masquées en mono-hôte ; la console retire alors aussi la colonne « Node » des listes et la bande de statut cluster de l'en-tête.

### Vues notables

- **Overview** : cartes de nœud (rôle / leader / VIP, santé, CPU·RAM, comptes de ressources par nœud), KPI cluster, quorum et incidents récents.
- **Containers** : tableau avec filtres et colonne « Node » (cluster), actions en ligne ciblant le nœud de la ligne, logs en modal, accès à l'assistant de création (wizard guidé : recherche d'images Docker Hub, ports/volumes/env auto-remplis depuis les métadonnées, réseau dédié ou existant, orchestrateur Caelix avec health checks, autoscale avec proxy dédié).
- **Services / Autoscale** : état détaillé des services orchestrés, métriques, replicas, seuils, scale manuel.
- **Stacks** : Compose, applications déployées et catalogue de templates (assistant de déploiement).
- **Activity** : logs centralisés (daemon, conteneurs en temps réel, backend UI), événements Docker, incidents filtrables, journal d'audit.
- **Settings** : gestion des utilisateurs (admin), notifications, préférences, et paramètres du cluster.

Les longues listes sont virtualisées et rendues progressivement : un nœud lent ne bloque pas l'affichage.

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
