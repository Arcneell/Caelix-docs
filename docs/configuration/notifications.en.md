# Notification Configuration

Caelix sends notifications across multiple channels for important events: outages, repairs, autoscaling, automatic updates, backups, manifest errors, etc.

## Supported Channels

| Channel | Webhook Type | Required Configuration |
|---|---|---|
| Discord | Webhook URL | `webhook_url` |
| Slack | Incoming Webhook | `webhook_url` |
| Microsoft Teams | Incoming Webhook | `webhook_url` |
| Telegram | Bot API | `bot_token` + `chat_id` |
| SMTP (email) | SMTP Server | `host`, `user`, `pass`, `from`, `to` |

## Configuration

Edit `etc/notify.ini`. Each channel is an independent section; enable the ones you want:

```ini
[discord]
enabled = 1
webhook_url = https://discord.com/api/webhooks/YOUR_ID/YOUR_TOKEN

[slack]
enabled = 1
webhook_url = https://hooks.slack.com/services/XXX/YYY/ZZZ

[teams]
enabled = 0
webhook_url = https://outlook.webhook.office.com/webhookb2/XXX/IncomingWebhook/YYY/ZZZ

[telegram]
enabled = 0
bot_token = YOUR_BOT_TOKEN
chat_id = YOUR_CHAT_ID

[smtp]
enabled = 0
host = smtp.example.com
port = 587
user = alerts@example.com
pass = secret
from = alerts@example.com
to = admin@example.com
```

### Creating a Discord Webhook

1. Open your Discord server
2. **Server Settings** > **Integrations** > **Webhooks**
3. Click **New Webhook**
4. Choose the destination channel
5. Copy the webhook URL
6. Paste it into `etc/notify.ini`

### Creating a Slack Webhook

1. Go to [api.slack.com/apps](https://api.slack.com/apps)
2. Create an app or select an existing one
3. **Incoming Webhooks** > enable > **Add New Webhook to Workspace**
4. Choose the channel and copy the URL

### Creating a Teams Webhook

1. In Teams, open the target channel
2. **...** > **Connectors** > **Incoming Webhook**
3. Name the connector and copy the URL

### Setting up a Telegram Bot

1. Open [@BotFather](https://t.me/BotFather) on Telegram
2. Send `/newbot` and follow the instructions
3. Copy the bot token
4. Send a message to the bot, then get your `chat_id` via `https://api.telegram.org/bot<TOKEN>/getUpdates`

## Notified Events

| Event | Severity | Description |
|---|---|---|
| Health check failed | `warn` / `critical` | Service outage detected |
| Repair performed | `info` | Restart, recreate, or purge executed |
| Repair failed | `critical` | All repair strategies have failed |
| Blue/green rollout | `info` | Switch to new container |
| Blue/green failed | `critical` | Candidate failed health checks |
| Autoscale up | `info` | Replica added |
| Autoscale down | `info` | Replica removed |
| Manifest error | `critical` | Configuration file is invalid |
| Orphan removed | `warn` | Undeclared caelix-* container removed |
| Service restored | `ok` | A previously failing service is healthy again |
| Proxy event | `info` / `warn` | Load balancer change |
| Update available | `info` | New Docker image detected (notify mode) |
| Update applied | `ok` | Automatic update applied successfully |
| Update failed | `warn` | Update rollout failed |
| Backup completed | `ok` | Scheduled backup succeeded |
| Backup failed | `warn` | Backup creation failed |
| Remote upload failed | `warn` | Remote target upload failed |

## Message Format

Each channel receives an adapted format:

- **Discord**: rich embeds with title, description, severity color, structured fields
- **Slack**: Slack blocks with sections and context
- **Teams**: Adaptive Cards
- **Telegram**: Markdown-formatted messages
- **SMTP**: HTML email with title and details

### Reason Codes

Each notification includes a translated reason code:

| Code | Description |
|---|---|
| `curl_failed` | curl request failed |
| `oom_killed` | Container killed due to out of memory (OOM) |
| `memory_hard` | Critical memory threshold exceeded |
| `memory_soft` | Warning memory threshold exceeded |
| `http_5xx` | HTTP 5xx response code |
| `tcp_refused` | TCP connection refused |
| `high_latency` | Response time too high |
| `high_error_rate` | HTTP error rate too high |
| `disk_full` | Disk usage too high |
| `log_anomaly` | Anomaly detected in logs |

## Anti-Spam

Caelix avoids notification spam:

- Certain incidents are marked to avoid duplicates in tight loops
- Recovery notifications are sent only once per restoration
- The `manifest_load_warn.notified` flag prevents repeated alerts for the same manifest error
- Non-blocking log anomaly notifications have a 10-minute cooldown per service
