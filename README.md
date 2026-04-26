# MecClaw — Industrial AI Agent for Factory, IoT, MQTT, SCADA & Energy

MecClaw is a self-hosted AI agent built for industrial environments. It runs on your own infrastructure, connects to your existing channels, and operates under your safety rules — not a vendor's.

Forked from [OpenClaw](https://github.com/openclaw/openclaw) and hardened for factory floors, substations, and IoT networks.

---

## What it does

- **Alert routing** — push sensor anomalies, equipment faults, and e-stop events to LINE or Telegram in real time
- **Predictive maintenance** — surface early warning signs before they cascade into unplanned downtime
- **Energy monitoring** — track consumption, flag waste, recommend efficiency actions
- **Shift reporting** — automated summaries at shift changeover via your preferred channel
- **SCADA / IoT integration** — MQTT, webhooks, and REST bridges to your existing control systems
- **Multi-channel support** — LINE, Telegram, Slack, Teams, Discord, and 15+ other platforms, all from one agent

---

## Design priorities

1. **Safety first** — never suggest bypassing interlocks or protective devices
2. **Accuracy** — wrong data in a factory is not a UX problem; verify before stating
3. **Energy efficiency** — default to lower-power options when capability is equivalent
4. **Predictive maintenance** — prefer early warning over reactive response

See [SOUL.md](SOUL.md) for the full behavioral charter.

---

## Quick start

**Requirements:** Node.js 22+, pnpm

```bash
git clone https://github.com/mecclaw/mecclaw.git
cd mecclaw
pnpm install
pnpm mecclaw start
```

Run the setup wizard:

```bash
pnpm mecclaw setup
```

Check system health:

```bash
pnpm mecclaw doctor
```

---

## Channel setup

### Telegram (recommended for alerts)

1. Create a bot via **@BotFather** on Telegram → copy the bot token
2. Set the token:

```bash
export TELEGRAM_BOT_TOKEN="your-token"
```

3. Start MecClaw and send any message to your bot to complete pairing

Full guide: [docs/channels/telegram.md](docs/channels/telegram.md)

### LINE (recommended for SEA/Japan sites)

1. Create a **Messaging API** channel at [LINE Developers Console](https://developers.line.biz/console/)
2. Set credentials:

```bash
export LINE_CHANNEL_ACCESS_TOKEN="your-access-token"
export LINE_CHANNEL_SECRET="your-channel-secret"
```

3. Point your LINE webhook URL to: `https://your-host/webhooks/line`

Full guide: [docs/channels/line.md](docs/channels/line.md)

Industrial-specific setup for both channels: [docs/channels/mecclaw-line-telegram.md](docs/channels/mecclaw-line-telegram.md)

---

## Configuration

MecClaw is configured via `~/.mecclaw/mecclaw.json`.

Minimal example for an industrial site:

```json
{
  "logging": {
    "level": "warn",
    "consoleStyle": "compact",
    "file": "/var/log/mecclaw/mecclaw.log"
  },
  "plugins": {
    "entries": {
      "telegram": { "enabled": true },
      "line": { "enabled": true }
    }
  }
}
```

Full configuration reference: [docs/gateway/configuration.md](docs/gateway/configuration.md)

---

## Logging

Log file (JSONL, rolling daily):

```
/tmp/mecclaw/mecclaw-YYYY-MM-DD.log
```

Live tail:

```bash
mecclaw logs --follow
```

Filter by channel:

```bash
mecclaw channels logs --channel telegram
mecclaw channels logs --channel line
```

Extract only alert events:

```bash
mecclaw logs --json | jq 'select(.alert)'
```

Full logging guide including industrial severity mapping and compliance retention: [docs/logging.md](docs/logging.md)

---

## Plugins

Both LINE and Telegram ship as **bundled plugins** — no separate install needed.

MecClaw has 140+ plugins across channels, AI providers, memory, media, and utilities.

Install additional plugins:

```bash
mecclaw plugins install @openclaw/<plugin-name>
```

Plugin docs: [docs/tools/plugin.md](docs/tools/plugin.md)

---

## Architecture

```
mecclaw/
├── src/              # Core TypeScript (gateway, agent loop, plugin loader)
├── extensions/       # 140+ plugins (channels, providers, tools, memory)
│   ├── line/         # LINE Messaging API — bundled
│   ├── telegram/     # Telegram Bot API — bundled
│   └── ...
├── packages/         # SDK packages (plugin-sdk, memory-host-sdk)
├── ui/               # Web control UI
├── apps/             # Companion apps (iOS, Android, macOS, Linux)
└── docs/             # Full documentation
```

- Core stays extension-agnostic. All channel behavior lives in plugins.
- Plugins communicate with core only through the plugin SDK and manifest contracts.
- No vendor lock-in. Swap AI providers, channels, or memory backends without touching core.

---

## Development

```bash
pnpm install          # install dependencies
pnpm dev              # start in dev mode
pnpm build            # production build
pnpm test             # run all tests
pnpm check:changed    # run checks for changed files only
pnpm format           # format code
```

See [AGENTS.md](AGENTS.md) for full contribution and agent-development rules.

---

## Security

Credentials are stored in `~/.mecclaw/credentials/` — never in the repo.

Security policy and reporting: [SECURITY.md](SECURITY.md)

---

## License

MIT
