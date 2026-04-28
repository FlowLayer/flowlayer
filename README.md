# FlowLayer

[Website](https://flowlayer.tech/) · [Official Releases](https://github.com/FlowLayer/flowlayer/releases) · [Distribution Repo](https://github.com/FlowLayer/distribution) · [TUI Source Repo](https://github.com/FlowLayer/tui) · [Issues](https://github.com/FlowLayer/flowlayer/issues)

FlowLayer is a local service orchestrator. It starts, monitors, and manages multiple processes from a single configuration file, exposes a real-time WebSocket API, and provides structured log aggregation.

This repository is the main public FlowLayer repository and the official release hub.
Official release assets include:

- `flowlayer-server`
- `flowlayer-client-tui`
- global Windows bundles
- `SHA256SUMS`

Package-manager install methods are maintained in the distribution repository: https://github.com/FlowLayer/distribution.
The TUI source code repository is: https://github.com/FlowLayer/tui.

**The server source code is not included in this repository.**

## Installation

- Official binaries are published in the global FlowLayer releases: https://github.com/FlowLayer/flowlayer/releases
- Linux/macOS install script:

```bash
curl -fsSL https://raw.githubusercontent.com/FlowLayer/distribution/main/install.sh | sh
```

- Homebrew:

```bash
brew tap FlowLayer/distribution https://github.com/FlowLayer/distribution.git
brew install flowlayer
```

- Scoop:

```powershell
scoop bucket add flowlayer https://github.com/FlowLayer/distribution.git
scoop install flowlayer
```

- Chocolatey package has been submitted and is pending Chocolatey Community moderation.
- Winget manifests are tracked in https://github.com/FlowLayer/distribution and are valid, but local installation remains blocked by a Winget internal error in the test environment.

## Verification

1. Download your binary and `SHA256SUMS` from https://github.com/FlowLayer/flowlayer/releases.
2. Verify checksums with:

```bash
sha256sum -c SHA256SUMS
```

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
flowlayer-server
```

FlowLayer auto-discovers the config file, computes a dependency-aware launch plan, starts services in parallel waves, and begins streaming logs.

3. Connect to the WebSocket API at `ws://127.0.0.1:6999/ws` with an `Authorization: Bearer my-token` header.

## CLI

```
flowlayer-server [-c path] [path] [-s bind] [-token value] [--no-color] [-h|--help] [--version]
```

| Flag | Description |
|---|---|
| `-c path` or `--config path` | Path to config file |
| `[path]` | Positional alternative to `-c` |
| `-s bind` | Enable session API on `host:port` or `port` |
| `-token value` | Bearer token for API authentication |
| `--no-color` | Disable ANSI colors in terminal output |
| `-h`, `--help` | Print onboarding help and exit |
| `--version` | Print version and exit |

`flowlayer-server` with no arguments prints the same onboarding help and exits with code `0`.

CLI/config errors print `Error: <message>`, then the full help, and exit with code `2`.

If no config path is given, FlowLayer searches the current directory for: `flowlayer.jsonc`, `flowlayer.json`, `flowlayer.config.jsonc`, `flowlayer.config.json`.

When `-s` is provided without `-token` and no token is set in the config, a random token is generated and printed at boot.

## Clients

- TUI source repository: https://github.com/FlowLayer/tui
- Official TUI binaries are published with the global FlowLayer releases: https://github.com/FlowLayer/flowlayer/releases

## Documentation

Full documentation: https://flowlayer.tech

- [PROTOCOL.md](PROTOCOL.md) — WebSocket protocol V1 specification
- [CONFIG.md](CONFIG.md) — Configuration file reference
- [BUILDING-A-CLIENT.md](BUILDING-A-CLIENT.md) — Guide to building a FlowLayer client

## Issues and Roadmap

Use [GitHub Issues](https://github.com/FlowLayer/flowlayer/issues) for bug reports, feature requests, and roadmap discussion.
