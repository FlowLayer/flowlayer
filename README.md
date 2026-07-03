# FlowLayer

**One JSONC file. One deterministic local runtime. Every service in your dev stack, on rails.**

[![Website](https://img.shields.io/badge/site-flowlayer.tech-4d8eff?style=flat-square)](https://flowlayer.tech/)
[![Releases](https://img.shields.io/github/v/release/FlowLayer/flowlayer?style=flat-square&color=4d8eff)](https://github.com/FlowLayer/flowlayer/releases)
[![Protocol](https://img.shields.io/badge/protocol-V1-4d8eff?style=flat-square)](PROTOCOL.md)
[![License](https://img.shields.io/badge/license-proprietary-lightgrey?style=flat-square)](#license)

[Website](https://flowlayer.tech/) · [Docs](https://flowlayer.tech/explore/) · [Releases](https://github.com/FlowLayer/flowlayer/releases) · [Distribution](https://github.com/FlowLayer/distribution) · [TUI](https://github.com/FlowLayer/tui) · [Issues](https://github.com/FlowLayer/flowlayer/issues)

---

FlowLayer is a **local-first service orchestrator** built for the moment your `docker-compose up` stops being enough.

You declare your services once. FlowLayer parses dependencies, computes a wave-based startup plan, gates each wave on real readiness probes (TCP, HTTP, or none), aggregates structured logs, and exposes the whole runtime over a single authenticated WebSocket. Same config in, same plan out — every time.

```text
flowlayer.jsonc  ──►  DAG  ──►  startup waves  ──►  readiness gates  ──►  /ws session truth
```

This repository is the **public release hub** for the project. The server engine source is private; everything you need to **run**, **integrate**, and **extend** FlowLayer lives here:

- pre-built binaries (`flowlayer-server`, `flowlayer-client-tui`)
- Windows bundles + `SHA256SUMS`
- the protocol spec, the config reference, the client-builder guide

For package-manager recipes (Homebrew, Scoop, Chocolatey, Winget, `install.sh`), see the [distribution repo](https://github.com/FlowLayer/distribution). The official terminal client lives in [FlowLayer/tui](https://github.com/FlowLayer/tui).

---

## Why FlowLayer

- **Deterministic by construction.** Same valid config, same DAG, same waves, same outcome — no startup races, no "works on my machine".
- **Readiness-gated, not timing-gated.** Wave N+1 unlocks only when wave N is actually ready. No more `sleep 5 && start-next`.
- **One protocol, any client.** A documented WebSocket V1 contract — build a TUI, a web UI, an IDE extension, a CI runner. The official TUI is just the first.
- **Track-only process model.** FlowLayer signals only what it spawned. It will never touch your other workloads.
- **JSONC, strict-mode.** Comments where they help, errors where they should — unknown fields are rejected loudly, not silently.

---

## Install in 30 seconds

**Linux & macOS** — one-liner:

```bash
curl -fsSL https://raw.githubusercontent.com/FlowLayer/distribution/main/install.sh | sh
```

**Homebrew:**

```bash
brew tap FlowLayer/distribution https://github.com/FlowLayer/distribution.git
brew install flowlayer
```

**Scoop (Windows):**

```powershell
scoop bucket add flowlayer https://github.com/FlowLayer/distribution.git
scoop install flowlayer
```

**Manual** — grab `flowlayer-server` + `flowlayer-client-tui` from [releases](https://github.com/FlowLayer/flowlayer/releases).

> Chocolatey package is approved on Chocolatey Community and can be installed with `choco install flowlayer`. Winget manifests are tracked in the [distribution repo](https://github.com/FlowLayer/distribution); local install is currently blocked by an upstream Winget issue in our test matrix.

### Verify the download

```bash
sha256sum -c SHA256SUMS
```

---

## Quick start

Drop a `flowlayer.jsonc` into your project root:

```jsonc
{
  "session": {
    "bind": "127.0.0.1:6999",
    "token": "my-token"
  },
  "services": {
    "api": {
      "cmd": "npm run dev",
      "port": 3000,
      "ready": {
        "type": "http",
        "url": "http://localhost:3000/health"
      }
    },
    "worker": {
      "cmd": "python worker.py",
      "dependsOn": ["api"]
    }
  }
}
```

Then start the server:

```bash
flowlayer-server
```

FlowLayer auto-discovers the config, computes the dependency-aware launch plan, starts services in parallel waves, gates `worker` on `api`'s HTTP readiness, and starts streaming logs. The server listens on `127.0.0.1:6999` because the config sets `session.bind`.

Open a session from the official TUI:

```bash
flowlayer-client-tui -addr 127.0.0.1:6999 -token my-token
```

The TUI connects to the server address with `-addr`; `my-token` is the same Bearer token configured in `session.token`.

…or talk straight to the protocol:

```text
ws://127.0.0.1:6999/ws
Authorization: Bearer my-token
```

---

## CLI

```text
flowlayer-server [-c path] [path] [-s bind] [-token value] [--no-color] [-h|--help] [--version]
```

| Flag | Description |
|---|---|
| `-c path` / `--config path` | Path to config file |
| `[path]` | Positional alternative to `-c` |
| `-s bind` | Enable session API on `host:port` or `port` |
| `-token value` | Bearer token for API authentication |
| `--no-color` | Disable ANSI colors in terminal output |
| `-h`, `--help` | Print onboarding help and exit |
| `--version` | Print version and exit |

`flowlayer-server` with no arguments prints onboarding help and exits with code `0`. Config or CLI errors print `Error: <message>`, the full help, and exit `2`.

If no config path is given, FlowLayer searches the current directory in order: `flowlayer.jsonc`, `flowlayer.json`, `flowlayer.config.jsonc`, `flowlayer.config.json`.

When `-s` is provided without `-token` and the config defines none, a random token is generated and printed at boot.

---

## Documentation

| Document | What you get |
|---|---|
| [PROTOCOL.md](PROTOCOL.md) | The complete WebSocket V1 contract — message envelopes, command flow, events, errors |
| [CONFIG.md](CONFIG.md) | The JSONC schema — every field, every default, every readiness probe |
| [BUILDING-A-CLIENT.md](BUILDING-A-CLIENT.md) | Step-by-step client implementation guide — handshake, snapshot merging, reconnect strategy |

Full operator and architecture docs: **[flowlayer.tech/explore/](https://flowlayer.tech/explore/)**.

---

## Clients

- **Official TUI** — source: [FlowLayer/tui](https://github.com/FlowLayer/tui), binary: `flowlayer-client-tui` shipped in the global release.
- **Custom clients** — read [BUILDING-A-CLIENT.md](BUILDING-A-CLIENT.md). The protocol is intentionally minimal; a working client fits in a single afternoon.

---

## Issues & roadmap

Bug reports, feature ideas, and roadmap discussion live on [GitHub Issues](https://github.com/FlowLayer/flowlayer/issues).

The server engine is closed-source for now, but the **protocol, the config schema, and the client surface are public and stable** — alternative clients are welcome and supported.

## License

The release artifacts in this repository are distributed under the terms shipped in each release. The protocol, config schema, and client-builder guide are public reference material.
