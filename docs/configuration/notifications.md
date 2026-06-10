# Configuration des notifications

Caelix envoie des notifications sur plusieurs canaux pour les événements importants : pannes, réparations, autoscaling, mises à jour automatiques, sauvegardes, erreurs de manifest, etc.

## Canaux supportés

| Canal | Type de webhook | Configuration requise |
|---|---|---|
| Discord | Webhook URL | `webhook_url` |
| Slack | Incoming Webhook | `webhook_url` |
| Microsoft Teams | Incoming Webhook | `webhook_url` |
| Telegram | Bot API | `bot_token` + `chat_id` |
| SMTP (email) | Serveur SMTP | `host`, `user`, `pass`, `from`, `to` |

## Configuration

Éditez `etc/notify.ini`. Chaque canal est une section indépendante ; activez ceux que vous souhaitez :

```ini
[discord]
enabled = 1
webhook_url = https://discord.com/api/webhooks/VOTRE_ID/VOTRE_TOKEN

[slack]
enabled = 1
webhook_url = https://hooks.slack.com/services/XXX/YYY/ZZZ

[teams]
enabled = 0
webhook_url = https://outlook.webhook.office.com/webhookb2/XXX/IncomingWebhook/YYY/ZZZ

[telegram]
enabled = 0
bot_token = VOTRE_BOT_TOKEN
chat_id = VOTRE_CHAT_ID

[smtp]
enabled = 0
host = smtp.example.com
port = 587
user = alerts@example.com
pass = secret
from = alerts@example.com
to = admin@example.com
```

### Créer un webhook Discord

1. Ouvrez votre serveur Discord
2. **Paramètres du serveur** > **Intégrations** > **Webhooks**
3. Cliquez **Nouveau webhook**
4. Choisissez le canal de destination
5. Copiez l'URL du webhook
6. Collez-la dans `etc/notify.ini`

### Créer un webhook Slack

1. Allez sur [api.slack.com/apps](https://api.slack.com/apps)
2. Créez une application ou sélectionnez-en une existante
3. **Incoming Webhooks** > activez > **Add New Webhook to Workspace**
4. Choisissez le canal et copiez l'URL

### Créer un webhook Teams

1. Dans Teams, ouvrez le canal cible
2. **...** > **Connecteurs** > **Incoming Webhook**
3. Nommez le connecteur et copiez l'URL

### Configurer un bot Telegram

1. Ouvrez [@BotFather](https://t.me/BotFather) sur Telegram
2. Envoyez `/newbot` et suivez les instructions
3. Copiez le token du bot
4. Envoyez un message au bot, puis récupérez votre `chat_id` via `https://api.telegram.org/bot<TOKEN>/getUpdates`

## Événements notifiés

| Événement | Sévérité | Description |
|---|---|---|
| Health check échoué | `warn` / `critical` | Service en panne détecté |
| Réparation effectuée | `info` | Restart, recreate ou purge exécuté |
| Réparation échouée | `critical` | Toutes les stratégies de réparation ont échoué |
| Blue/green rollout | `info` | Bascule vers le nouveau conteneur |
| Blue/green échoué | `critical` | Le candidat a échoué les health checks |
| Autoscale up | `info` | Ajout d'une replica |
| Autoscale down | `info` | Suppression d'une replica |
| Erreur manifest | `critical` | Le fichier de configuration est invalide |
| Orphelin supprimé | `warn` | Conteneur caelix-* non déclaré supprimé |
| Service rétabli | `ok` | Un service précédemment en panne est redevenu sain |
| Événement proxy | `info` / `warn` | Changement dans le load balancer |
| Mise à jour disponible | `info` | Nouvelle image Docker détectée (mode notify) |
| Mise à jour appliquée | `ok` | Mise à jour automatique appliquée avec succès |
| Mise à jour échouée | `warn` | Échec du rollout de mise à jour |
| Sauvegarde terminée | `ok` | Sauvegarde planifiée réussie |
| Sauvegarde échouée | `warn` | Échec de la sauvegarde |
| Envoi distant échoué | `warn` | Échec de l'envoi vers la cible distante |

## Format des messages

Chaque canal reçoit un format adapté :

- **Discord** : embeds riches avec titre, description, couleur par sévérité, champs structurés
- **Slack** : blocs Slack avec sections et contexte
- **Teams** : cartes adaptatives (Adaptive Cards)
- **Telegram** : messages Markdown avec mise en forme
- **SMTP** : email HTML avec titre et détails

### Codes de raison

Chaque notification inclut un code de raison traduit en français :

| Code | Traduction |
|---|---|
| `curl_failed` | Échec de la requête curl |
| `oom_killed` | Conteneur tué par manque de mémoire (OOM) |
| `memory_hard` | Seuil mémoire critique dépassé |
| `memory_soft` | Seuil mémoire d'avertissement dépassé |
| `http_5xx` | Code de réponse HTTP 5xx |
| `tcp_refused` | Connexion TCP refusée |
| `high_latency` | Temps de réponse trop élevé |
| `high_error_rate` | Taux d'erreur HTTP trop élevé |
| `disk_full` | Utilisation disque trop élevée |
| `log_anomaly` | Anomalie détectée dans les logs |

## Anti-spam

Caelix évite le spam de notifications :

- Certains incidents sont marqués pour éviter les doublons en boucle serrée
- Les notifications de recovery ne sont envoyées qu'une seule fois par rétablissement
- Le flag `manifest_load_warn.notified` empêche les alertes répétées sur la même erreur de manifest
- Les anomalies de logs non bloquantes ont un cooldown de 10 minutes par service
