---
summary: "Set up LINE and Telegram as industrial alert channels in MecClaw"
read_when:
  - You want to receive factory alerts, SCADA events, or maintenance notifications on LINE or Telegram
  - You are configuring MecClaw for an industrial site
  - You need push alerts for sensor anomalies or equipment faults
title: "LINE & Telegram — Industrial Alert Setup"
---

Both LINE and Telegram ship as **bundled plugins** in MecClaw. No separate install is needed.

Use these channels to receive:
- Equipment fault alerts
- Sensor anomaly notifications
- Predictive maintenance reminders
- Energy usage threshold warnings
- Shift reports and status summaries

---

## Telegram setup

Telegram is the recommended channel for industrial environments: low latency, reliable delivery, and no regional restrictions.

### 1. Create a bot token

Open Telegram and chat with **@BotFather** (verify the handle exactly — phishing bots exist).

```
/newbot
```

Follow the prompts. Save the token — it looks like `123456789:ABCdef...`.

### 2. Set the environment variable

```bash
export TELEGRAM_BOT_TOKEN="123456789:ABCdef..."
```

Or add it to your MecClaw credentials store:

```bash
mecclaw credentials set TELEGRAM_BOT_TOKEN
```

### 3. Add the channel to your config

In your MecClaw config (`~/.mecclaw/config.json` or equivalent):

```json
{
  "plugins": {
    "entries": {
      "telegram": {
        "enabled": true
      }
    }
  }
}
```

### 4. Pair the bot

Start MecClaw and send a message to your bot. The pairing flow will authenticate your Telegram user ID.

```bash
mecclaw start
```

### Industrial-specific recommendations

- Create a **dedicated Telegram group** per plant area (e.g., `Compressor Hall Alerts`, `Substation B`).
- Use **topic threads** (Telegram Supergroups) to separate alert types: faults, warnings, maintenance, energy.
- Set bot commands for quick queries: `/status`, `/lastfault`, `/energy`.
- Enable `nativeCommandsAutoEnabled` (already on by default) for slash-command support.

---

## LINE setup

LINE is the recommended channel for sites in Thailand, Japan, and Southeast Asia where LINE is the dominant enterprise messaging platform.

### 1. Create a LINE Messaging API channel

Go to the [LINE Developers Console](https://developers.line.biz/console/) and create a new **Messaging API** channel.

Collect two credentials from the channel settings:
- **Channel Access Token** (long-lived, under Messaging API tab)
- **Channel Secret** (under Basic Settings tab)

### 2. Set the environment variables

```bash
export LINE_CHANNEL_ACCESS_TOKEN="your-long-lived-token"
export LINE_CHANNEL_SECRET="your-channel-secret"
```

Or use the credentials store:

```bash
mecclaw credentials set LINE_CHANNEL_ACCESS_TOKEN
mecclaw credentials set LINE_CHANNEL_SECRET
```

### 3. Configure the webhook URL

In the LINE Developers Console, set the webhook URL to your MecClaw gateway:

```
https://your-mecclaw-host/webhooks/line
```

Enable **Use webhook** and disable **Auto-reply messages** (MecClaw handles replies).

### 4. Add the channel to your config

```json
{
  "plugins": {
    "entries": {
      "line": {
        "enabled": true
      }
    }
  }
}
```

### Industrial-specific recommendations

- Use **LINE Flex Messages** for structured alerts: equipment ID, fault code, severity, recommended action.
- Use **Quick Reply buttons** for one-tap acknowledgement (e.g., "ACK", "Escalate", "Silence 1h").
- Group chats work for team-wide alerts; DMs work for assigned maintenance technicians.
- LINE does not support topic threads — use separate groups per area or severity level.

---

## Alert routing example

Route critical faults to Telegram (faster) and daily summaries to LINE (better for shift handover):

```json
{
  "agents": [
    {
      "id": "fault-monitor",
      "channel": "telegram",
      "triggers": ["fault", "e-stop", "anomaly"]
    },
    {
      "id": "shift-report",
      "channel": "line",
      "schedule": "0 6,14,22 * * *"
    }
  ]
}
```

---

## Troubleshooting

Run the built-in diagnostics:

```bash
mecclaw doctor
```

Check channel status:

```bash
mecclaw status --channel telegram
mecclaw status --channel line
```

For deeper diagnostics: [Channel troubleshooting](/channels/troubleshooting)
