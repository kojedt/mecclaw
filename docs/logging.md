---
summary: "File logs, console output, CLI tailing, and the Control UI Logs tab"
read_when:
  - You need a beginner-friendly overview of MecClaw logging
  - You want to configure log levels, formats, or redaction
  - You are troubleshooting and need to find logs quickly
  - You want to route industrial alerts (sensor faults, e-stops, anomalies) to LINE or Telegram
title: "Logging"
---

MecClaw has two main log surfaces:

- **File logs** (JSON lines) written by the Gateway.
- **Console output** shown in terminals and the Gateway Debug UI.

The Control UI **Logs** tab tails the gateway file log. This page explains where
logs live, how to read them, and how to configure log levels and formats.

## Where logs live

By default, the Gateway writes a rolling log file under:

`/tmp/mecclaw/mecclaw-YYYY-MM-DD.log`

The date uses the gateway host's local timezone.

Each file rotates when it reaches `logging.maxFileBytes` (default: 100 MB).
OpenClaw keeps up to five numbered archives beside the active file, such as
`openclaw-YYYY-MM-DD.1.log`, and keeps writing to a fresh active log instead of
suppressing diagnostics.

You can override this in `~/.mecclaw/mecclaw.json`:

```json
{
  "logging": {
    "file": "/path/to/mecclaw.log"
  }
}
```

## How to read logs

### CLI: live tail (recommended)

Use the CLI to tail the gateway log file via RPC:

```bash
mecclaw logs --follow
```

Useful current options:

- `--local-time`: render timestamps in your local timezone
- `--url <url>` / `--token <token>` / `--timeout <ms>`: standard Gateway RPC flags
- `--expect-final`: agent-backed RPC final-response wait flag (accepted here via the shared client layer)

Output modes:

- **TTY sessions**: pretty, colorized, structured log lines.
- **Non-TTY sessions**: plain text.
- `--json`: line-delimited JSON (one log event per line).
- `--plain`: force plain text in TTY sessions.
- `--no-color`: disable ANSI colors.

When you pass an explicit `--url`, the CLI does not auto-apply config or
environment credentials; include `--token` yourself if the target Gateway
requires auth.

In JSON mode, the CLI emits `type`-tagged objects:

- `meta`: stream metadata (file, cursor, size)
- `log`: parsed log entry
- `notice`: truncation / rotation hints
- `raw`: unparsed log line

If the local loopback Gateway asks for pairing, `openclaw logs` falls back to
the configured local log file automatically. Explicit `--url` targets do not
use this fallback.

If the Gateway is unreachable, the CLI prints a short hint to run:

```bash
openclaw doctor
```

### Control UI (web)

The Control UI’s **Logs** tab tails the same file using `logs.tail`.
See [/web/control-ui](/web/control-ui) for how to open it.

### Channel-only logs

To filter channel activity (WhatsApp/Telegram/etc), use:

```bash
mecclaw channels logs --channel whatsapp
```

## Log formats

### File logs (JSONL)

Each line in the log file is a JSON object. The CLI and Control UI parse these
entries to render structured output (time, level, subsystem, message).

### Console output

Console logs are **TTY-aware** and formatted for readability:

- Subsystem prefixes (e.g. `gateway/channels/whatsapp`)
- Level coloring (info/warn/error)
- Optional compact or JSON mode

Console formatting is controlled by `logging.consoleStyle`.

### Gateway WebSocket logs

`openclaw gateway` also has WebSocket protocol logging for RPC traffic:

- normal mode: only interesting results (errors, parse errors, slow calls)
- `--verbose`: all request/response traffic
- `--ws-log auto|compact|full`: pick the verbose rendering style
- `--compact`: alias for `--ws-log compact`

Examples:

```bash
openclaw gateway
openclaw gateway --verbose --ws-log compact
openclaw gateway --verbose --ws-log full
```

## Configuring logging

All logging configuration lives under `logging` in `~/.mecclaw/mecclaw.json`.

```json
{
  "logging": {
    "level": "info",
    "file": "/tmp/openclaw/openclaw-YYYY-MM-DD.log",
    "consoleLevel": "info",
    "consoleStyle": "pretty",
    "redactSensitive": "tools",
    "redactPatterns": ["sk-.*"]
  }
}
```

### Log levels

- `logging.level`: **file logs** (JSONL) level.
- `logging.consoleLevel`: **console** verbosity level.

You can override both via the **`MECCLAW_LOG_LEVEL`** environment variable (e.g. `MECCLAW_LOG_LEVEL=debug`). The env var takes precedence over the config file, so you can raise verbosity for a single run without editing `openclaw.json`. You can also pass the global CLI option **`--log-level <level>`** (for example, `mecclaw --log-level debug gateway run`), which overrides the environment variable for that command.

`--verbose` only affects console output and WS log verbosity; it does not change
file log levels.

### Console styles

`logging.consoleStyle`:

- `pretty`: human-friendly, colored, with timestamps.
- `compact`: tighter output (best for long sessions).
- `json`: JSON per line (for log processors).

### Redaction

Tool summaries can redact sensitive tokens before they hit the console:

- `logging.redactSensitive`: `off` | `tools` (default: `tools`)
- `logging.redactPatterns`: list of regex strings to override the default set

Redaction applies at the logging sinks for **console output**, **stderr-routed
console diagnostics**, and **file logs**. File logs stay JSONL, but matching
secret values are masked before the line is written to disk.

## Diagnostics and OpenTelemetry

Diagnostics are structured, machine-readable events for model runs and
message-flow telemetry (webhooks, queueing, session state). They do **not**
replace logs — they feed metrics, traces, and exporters. Events are emitted
in-process whether or not you export them.

Two adjacent surfaces:

- **OpenTelemetry export** — send metrics, traces, and logs over OTLP/HTTP to
  any OpenTelemetry-compatible collector or backend (Grafana, Datadog,
  Honeycomb, New Relic, Tempo, etc.). Full configuration, signal catalog,
  metric/span names, env vars, and privacy model live on a dedicated page:
  [OpenTelemetry export](/gateway/opentelemetry).
- **Diagnostics flags** — targeted debug-log flags that route extra logs to
  `logging.file` without raising `logging.level`. Flags are case-insensitive
  and support wildcards (`telegram.*`, `*`). Configure under `diagnostics.flags`
  or via the `MECCLAW_DIAGNOSTICS=...` env override. Full guide:
  [Diagnostics flags](/diagnostics/flags).

To enable diagnostics events for plugins or custom sinks without OTLP export:

```json5
{
  diagnostics: { enabled: true },
}
```

For OTLP export to a collector, see [OpenTelemetry export](/gateway/opentelemetry).

## Troubleshooting tips

- **Gateway not reachable?** Run `mecclaw doctor` first.
- **Logs empty?** Check that the Gateway is running and writing to the file path
  in `logging.file`.
- **Need more detail?** Set `logging.level` to `debug` or `trace` and retry.

## Industrial logging — MecClaw

MecClaw operates in factory, SCADA, IoT, and energy environments where logs carry
operational weight. This section covers how to configure logging for industrial use
and how to route critical events to LINE and Telegram.

### Recommended config for factory environments

```json
{
  "logging": {
    "level": "warn",
    "consoleLevel": "info",
    "consoleStyle": "compact",
    "file": "/var/log/mecclaw/mecclaw.log",
    "maxFileBytes": 52428800,
    "redactSensitive": "tools"
  },
  "diagnostics": {
    "enabled": true,
    "flags": ["line.*", "telegram.*", "gateway/channels/*"]
  }
}
```

- `level: "warn"` — file logs only capture warnings and errors in steady state; avoids log bloat on high-frequency sensor traffic.
- `consoleStyle: "compact"` — tighter output for small operator terminals and HMI screens.
- `maxFileBytes: 52428800` — 50 MB rotation keeps disk usage predictable on embedded or edge-mounted hosts.
- Diagnostics flags for `line.*` and `telegram.*` — route extra channel-level debug events to file without raising the global level.

### Routing alerts to LINE and Telegram

MecClaw can push structured log events to LINE or Telegram as operational alerts.
Set up the channels first: [LINE & Telegram setup](/channels/mecclaw-line-telegram).

Filter channel-specific activity:

```bash
mecclaw channels logs --channel line
mecclaw channels logs --channel telegram
```

For real-time alert tailing from a control room terminal:

```bash
mecclaw logs --follow --plain
```

### Industrial event severity mapping

| MecClaw log level | Industrial meaning | Recommended action |
|---|---|---|
| `error` | Equipment fault, communication loss, safety interlock trip | Alert immediately via Telegram/LINE |
| `warn` | Sensor reading outside normal envelope, degraded mode | Log to file + periodic LINE summary |
| `info` | Normal operational events, shift changeover, connection status | File log only |
| `debug` | Detailed protocol traffic, message parsing, polling intervals | Enable only for active troubleshooting |
| `trace` | Raw byte-level I/O, internal state transitions | Never in production |

Set severity routing per channel using `MECCLAW_LOG_LEVEL` per process, or use the `diagnostics.flags` wildcard to target specific subsystems:

```bash
# Raise verbosity only for LINE channel during debugging
MECCLAW_DIAGNOSTICS=line.* mecclaw gateway
```

### Log retention for compliance

Some industrial sites (IEC 62443, ISO 50001, OSHA) require log retention of 30–90 days.
MecClaw keeps up to five rolling archives by default. For longer retention, pipe JSONL output to an external log manager:

```bash
mecclaw logs --json | tee -a /mnt/nas/mecclaw-audit.jsonl
```

Or use the [OpenTelemetry export](/gateway/opentelemetry) to forward logs to a SIEM or historian (Grafana Loki, Splunk, OSIsoft PI, etc.).

### Sensor anomaly log pattern

When a sensor anomaly is detected and reported via LINE or Telegram, MecClaw emits a structured log entry:

```json
{
  "level": "warn",
  "subsystem": "gateway/channels/telegram",
  "msg": "alert dispatched",
  "alert": {
    "type": "sensor_anomaly",
    "asset": "COMPRESSOR-04",
    "tag": "VIBRATION_X",
    "value": 14.7,
    "unit": "mm/s",
    "threshold": 10.0,
    "severity": "warn"
  },
  "channel": "telegram",
  "ts": "2026-04-26T03:17:44.201Z"
}
```

Use `mecclaw logs --json | jq 'select(.alert)'` to extract only alert events from the log stream.

## Related

- [OpenTelemetry export](/gateway/opentelemetry) — OTLP/HTTP export, metric/span catalog, privacy model
- [Diagnostics flags](/diagnostics/flags) — targeted debug-log flags
- [Gateway logging internals](/gateway/logging) — WS log styles, subsystem prefixes, and console capture
- [Configuration reference](/gateway/configuration-reference#diagnostics) — full `diagnostics.*` field reference
- [LINE & Telegram alert setup](/channels/mecclaw-line-telegram) — industrial alert channel configuration
