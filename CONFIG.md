# FlowLayer Configuration

FlowLayer is configured with a JSONC file (JSON with comments and trailing commas).

## File Discovery

When no config path is specified, FlowLayer searches the current directory in this order:

1. `flowlayer.jsonc`
2. `flowlayer.json`
3. `flowlayer.config.jsonc`
4. `flowlayer.config.json`

Unknown fields are rejected. The parser uses strict mode — typos in field names produce an error.

## Schema

```jsonc
{
  // Session API configuration
  "session": {
    "bind": "127.0.0.1:6999",       // host:port or port
    "token": "my-secret-token"       // bearer token for API auth
  },

  // Log persistence to disk (JSONL files)
  "logs": {
    "dir": "./logs"                  // directory for JSONL projection
  },

  // Default log retrieval limits for get_logs
  "logView": {
    "maxEntries": 500,               // global default limit
    "all": {
      "maxEntries": 800              // limit when querying all services
    }
  },

  // Service definitions
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
      "cmd": ["python", "worker.py"],
      "kind": "oneshot",
      "dependsOn": ["api"],
      "env": {
        "API_URL": "http://localhost:3000"
      }
    }
  }
}
```

## CLI Precedence

CLI flags always take precedence over config file values. When both are specified, the CLI flag wins. See the [README](README.md) for available flags.

## Top-Level Fields

### `session`

Controls the WebSocket/HTTP API server.

| Field | Type | Default | Description |
|---|---|---|---|
| `bind` | string | — | Listen address. Accepts `host:port`, `:port`, or a bare port number. A bare port binds to `127.0.0.1` |
| `token` | string | — | Bearer token for API authentication. If the key is present, the value must not be empty |

The session API is only started when `bind` is configured or the `-s` CLI flag is used. CLI flags (`-s`, `-token`) take precedence over config values.

When the API is enabled without a token (neither in config nor CLI), a random token (`fl_<uuid>`) is generated and printed at boot.

### `logs`

Controls log persistence to disk.

| Field | Type | Default | Description |
|---|---|---|---|
| `dir` | string | — | Directory for JSONL log files. When set, FlowLayer writes `all.jsonl` and `<service>.jsonl` files |

### `logView`

Controls default limits for `get_logs` responses. See [PROTOCOL.md](PROTOCOL.md) for the full limit resolution order.
These defaults apply when the client does not provide `limit`; an explicit client `limit` is used as-is for that response.

| Field | Type | Default | Description |
|---|---|---|---|
| `maxEntries` | integer | — | Global default limit. Must be > 0 when provided |
| `all.maxEntries` | integer | — | Override when querying all services (no `service` filter). Must be > 0 |

When no limit is configured and the client does not specify one, the built-in fallback is **500**.

## Service Fields

Each key in `services` is the service name. Names must not be empty.

| Field | Type | Required | Default | Description |
|---|---|---|---|---|
| `cmd` | string or string[] | Yes | — | Command to run. A string is split on whitespace with no shell interpretation. An array is passed directly to the process. Prefer the array form for precise control over arguments |
| `stopCmd` | string or string[] | No | — | Custom stop command. When absent, FlowLayer sends SIGTERM to the process group |
| `kind` | string | No | `"daemon"` | `"daemon"` (long-running) or `"oneshot"` (run to completion) |
| `port` | integer | No | `0` | Port for preflight checks. Must be ≥ 0 |
| `ready` | object | No | — | Readiness probe configuration |
| `dependsOn` | string[] | No | `[]` | Services that must be ready before this one starts |
| `env` | object | No | `{}` | Environment variables merged into the process environment |
| `logView.maxEntries` | integer | No | — | Per-service log limit override. Must be > 0 |

### `ready`

Readiness probe configuration. When present, `type` is required.

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | string | Yes | `"http"`, `"tcp"`, or `"none"` |
| `url` | string | For `http` | URL to poll for HTTP 200 |
| `port` | integer | For `tcp` | TCP port to probe. Falls back to the service `port` if not specified |

### `dependsOn`

Declares dependencies between services. FlowLayer computes a launch plan from the dependency graph and starts services in parallel waves.

- Self-references are rejected
- References to undefined services are rejected
- Duplicates are automatically removed
- Order is normalized alphabetically

## Example

A full configuration managing a Node.js API, a dependent frontend, and a Docker-based service:

```jsonc
{
  "session": {
    "bind": "127.0.0.1:6999",
    "token": "flowlayer-dev-token"
  },
  "logView": {
    "maxEntries": 500,
    "all": {
      "maxEntries": 800
    }
  },
  "services": {
    "billing": {
      "cmd": "npm --prefix ./billing run start:dev",
      "port": 3002,
      "ready": {
        "type": "http",
        "url": "http://localhost:3002/healthz"
      },
      "logView": {
        "maxEntries": 5
      }
    },
    "users": {
      "cmd": "npm --prefix ./users run start:dev",
      "port": 3003,
      "dependsOn": ["billing"],
      "ready": {
        "type": "http",
        "url": "http://localhost:3003/healthz"
      }
    },
    "ping": {
      "cmd": "docker compose -f ./docker/python.compose.yml up",
      "stopCmd": "docker compose -f ./docker/python.compose.yml down",
      "port": 8088,
      "ready": {
        "type": "tcp",
        "port": 8088
      }
    }
  }
}
```

## Validation Summary

| Rule | Error |
|---|---|
| Unknown fields in any object | Parse error |
| Empty service name | Rejected |
| `kind` not `daemon` or `oneshot` | Rejected |
| `port` < 0 | Rejected |
| `logView.maxEntries` ≤ 0 | Rejected |
| `session.token` set to empty string | Rejected |
| `session.bind` port outside 1–65535 | Rejected |
| `dependsOn` references itself | Rejected |
| `dependsOn` references unknown service | Rejected |
| `ready` without `type` | Rejected |
| `ready.type=http` without `url` | Rejected |
| `ready.type=tcp` without `port` (and no service `port`) | Rejected |
| Circular dependencies | Rejected at plan computation |
