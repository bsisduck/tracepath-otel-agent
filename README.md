<p align="center">
  <a href="https://tracepath.dev"><img src="https://raw.githubusercontent.com/bsisduck/tracepath/main/logo-small-favicon.png" alt="TracePath" width="140" /></a>
</p>

<p align="center">
  <a href="https://github.com/bsisduck/tracepath-otel-agent/blob/main/LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="License"></a>
  <a href="https://github.com/bsisduck/tracepath-otel-agent/releases"><img src="https://img.shields.io/github/v/release/bsisduck/tracepath-otel-agent" alt="Release"></a>
  <a href="https://github.com/bsisduck/tracepath-otel-agent/actions"><img src="https://img.shields.io/github/actions/workflow/status/bsisduck/tracepath-otel-agent/ci.yml?branch=main" alt="CI"></a>
</p>

<p align="center">
  <a href="https://tracepath.dev">Website</a> · <a href="https://docs.tracepath.dev">Docs</a>
</p>

# TracePath OTel Agent

A simple, pre-configured [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
distribution (built with [OCB](https://opentelemetry.io/docs/collector/custom-collector/))
that pulls host metrics every 60s and tails log files, then ships both to
**any OTLP/HTTP-compatible backend**. Runs as a background service via systemd,
launchd, or Windows Service.

**Built for** sysadmins, SREs, and platform engineers who want host
observability without compiling and tuning the upstream OpenTelemetry
Collector themselves. Vendor-agnostic on the receiving side — point it at
TracePath, a self-hosted Jaeger / Grafana stack, or any other OTLP/HTTP
endpoint by setting `TRACEPATH_ENDPOINT`.

> **Auth is Bearer-only.** Every request goes out with
> `Authorization: Bearer ${TRACEPATH_TOKEN}` and nothing else. Compatible
> with TracePath, Grafana Cloud (bearer token), Honeycomb, New Relic OTLP,
> and most other OpenTelemetry collectors. **Not** compatible out of the
> box with backends expecting Basic auth, custom API-key headers
> (`X-Api-Key`, `DD-API-KEY`, …), HMAC-signed requests, mTLS client certs,
> or AWS sigv4.

**Design goals**

- **Easy to install** — one `curl | bash` line, no YAML to write.
- **Configured by default** — sane scrape interval, sane batching, sane retries, crash-safe export queue.
- **Small surface area** — only the receivers/processors/exporters needed for host metrics + tailed logs are compiled in. Auditable in one sitting.

```bash
curl -fsSL https://install.tracepath.dev/install.sh | \
  TRACEPATH_TOKEN=<your-token> \
  TRACEPATH_ENDPOINT=https://cloud.tracepath.dev/api/otel \
  TRACEPATH_SERVICE_NAME=<host-label> \
  bash
```

This is a **host agent**, not an application SDK — for app traces and
in-process runtime metrics, instrument your application with
[OpenTelemetry](https://docs.tracepath.dev/client/otel) or the per-language
TracePath clients ([js-client](https://github.com/bsisduck/js-client),
[Go SDK](https://github.com/bsisduck/tracepath/tree/main/sdks/go)).

## Install

All installers read the same env vars:

| Var | Default | Purpose |
| --- | --- | --- |
| `TRACEPATH_TOKEN` | _(required)_ | Project token. Sent verbatim as `Authorization: Bearer <token>` — the only auth mode this agent supports |
| `TRACEPATH_ENDPOINT` | `https://cloud.tracepath.dev/api/otel` | Override for self-hosted TracePath |
| `TRACEPATH_SERVICE_NAME` | `$(hostname)` | `service.name` resource attribute |
| `TRACEPATH_LOG_PATHS` | _(unset)_ | Comma-separated globs to tail. Enables logs pipeline when set |
| `TRACEPATH_PROCESS_NAMES` | _(unset)_ | Comma-separated process names (e.g. `myapp,postgres`) or `*` for all processes. Off by default — the OTel `process` scraper emits one data point per running process per scrape, which scales linearly with the host's process count. Set this when you want per-process CPU / memory metrics for specific binaries |
| `TRACEPATH_PERSISTENT_QUEUE` | `on` | Set `off` to keep the exporter queue in-memory. When on (the default), queued batches are stored on disk with a 64 MiB hard cap and survive agent restarts |

### Linux (systemd) / macOS (launchd)

Requires `curl`, `tar`, `sudo` (or root).

```bash
curl -fsSL https://install.tracepath.dev/install.sh | \
  TRACEPATH_TOKEN=<your-token> \
  TRACEPATH_SERVICE_NAME=api-prod-eu-1 \
  TRACEPATH_LOG_PATHS="/var/log/app/*.log,/var/log/nginx/access.log" \
  TRACEPATH_PROCESS_NAMES=myapp \
  bash
```

Metrics arrive within ~60s under their hostmetrics names (`system.cpu.utilization`, `system.memory.usage`, …). Each host is identified by the `server_name` tag from `TRACEPATH_SERVICE_NAME` — give every host a distinct value.

### Windows (PowerShell, admin)

```powershell
$env:TRACEPATH_TOKEN = "<your-token>"
iwr -useb https://install.tracepath.dev/install.ps1 | iex
```

> The `install.tracepath.dev` installer endpoints are served by the TracePath project. Until the installer CDN is published, you can run the agent from source (below) or download a binary from [Releases](https://github.com/bsisduck/tracepath-otel-agent/releases).

## Building from source

The agent is a thin, pre-configured distribution of the upstream OpenTelemetry Collector — it does **not** vendor a fork of the collector. The full build (OCB config, collector config overlays, install scripts) lives in the main [TracePath monorepo](https://github.com/bsisduck/tracepath) and in the upstream [tracewayapp/traceway-otel-agent](https://github.com/tracewayapp/traceway-otel-agent) (v1.0.0), from which this project is derived (MIT).

```bash
# 1. Install the OpenTelemetry Collector Builder
go install go.opentelemetry.io/collector/cmd/builder@v0.159.0

# 2. Build with the upstream builder config (rebrand endpoints as needed)
git clone https://github.com/tracewayapp/traceway-otel-agent
cd traceway-otel-agent
builder --config=builder-config.yaml   # → dist/traceway-otel-agent

# 3. Validate the shipped config against the binary
TRACEPATH_TOKEN=placeholder \
TRACEPATH_ENDPOINT=https://cloud.tracepath.dev/api/otel \
TRACEPATH_SERVICE_NAME=local-validate \
./dist/traceway-otel-agent validate --config=config/default.yaml
```

This repo carries the TracePath-branded [config](config/default.yaml), release workflow skeleton, and documentation. Full build instructions: [upstream README](https://github.com/tracewayapp/traceway-otel-agent#building-from-source) and the [TracePath docs](https://docs.tracepath.dev).

## Repository layout

```
tracepath-otel-agent/
├── config/default.yaml          # TracePath-branded collector config (hostmetrics + logs)
├── .github/workflows/
│   ├── ci.yml                   # lint: shell/yaml checks on every push/PR
│   └── release.yml              # tag → build + GitHub Release (gated)
├── LICENSE
└── README.md
```

## Links

- [TracePath](https://tracepath.dev) — main product site
- [Documentation](https://docs.tracepath.dev)
- [Main monorepo](https://github.com/bsisduck/tracepath)
- [js-client SDKs](https://github.com/bsisduck/js-client)
- [Agent skills](https://github.com/bsisduck/skills)
- [Upstream Traceway OTel Agent](https://github.com/tracewayapp/traceway-otel-agent) (MIT)

## License

MIT — see [LICENSE](LICENSE). Derived from the upstream Traceway OTel Agent (© dusanstanojeviccs, MIT).
