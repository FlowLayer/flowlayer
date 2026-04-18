# FlowLayer

[Website](https://flowlayer.tech/) · [Releases](https://github.com/FlowLayer/flowlayer/releases) · [Issues](https://github.com/FlowLayer/flowlayer/issues)

FlowLayer is a local service orchestrator. It starts, monitors, and manages multiple processes from a single configuration file, exposes a real-time WebSocket API, and provides structured log aggregation.

The engine is distributed as a single binary. This repository contains the protocol specification, configuration reference, client-building guide, and releases.

**The engine source code is not included in this repository.**

## Installation

Download the latest binary from [Releases](https://github.com/FlowLayer/flowlayer/releases).

Binaries are available for:

| OS | Architecture |
|---|---|
| Linux | amd64, arm64 |
| macOS | amd64, arm64 |
| Windows | amd64, arm64 |

Each release includes GPG-signed checksums.

## Quick Start

1. Create a `flowlayer.jsonc` in your project directory:

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

2. Run FlowLayer:

```
flowlayer
```

FlowLayer auto-discovers the config file, computes a dependency-aware launch plan, starts services in parallel waves, and begins streaming logs.

3. Connect to the WebSocket API at `ws://127.0.0.1:6999/ws` with an `Authorization: Bearer my-token` header.

## CLI

```
flowlayer [-c path] [path] [-s bind] [-token value] [--no-color] [--version]
```

| Flag | Description |
|---|---|
| `-c path` or `--config path` | Path to config file |
| `[path]` | Positional alternative to `-c` |
| `-s bind` | Enable session API on `host:port` or `port` |
| `-token value` | Bearer token for API authentication |
| `--no-color` | Disable ANSI colors in terminal output |
| `--version` | Print version and exit |

If no config path is given, FlowLayer searches the current directory for: `flowlayer.jsonc`, `flowlayer.json`, `flowlayer.config.jsonc`, `flowlayer.config.json`.

When `-s` is provided without `-token` and no token is set in the config, a random token is generated and printed at boot.

## Clients

- TUI reference client: https://github.com/FlowLayer/tui

## Documentation

- [PROTOCOL.md](PROTOCOL.md) — WebSocket protocol V1 specification
- [CONFIG.md](CONFIG.md) — Configuration file reference
- [BUILDING-A-CLIENT.md](BUILDING-A-CLIENT.md) — Guide to building a FlowLayer client

## Issues and Roadmap

Use [GitHub Issues](https://github.com/FlowLayer/flowlayer/issues) for bug reports, feature requests, and roadmap discussion.
