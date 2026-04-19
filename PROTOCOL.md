# FlowLayer Protocol V1

WebSocket protocol specification for FlowLayer.

## Session Model

- Transport: WebSocket on `ws://<host>:<port>/ws`
- Authentication: HTTP header `Authorization: Bearer <token>` on the upgrade request
  - Missing header → HTTP 401
  - Invalid token → HTTP 403
- Each WebSocket connection is an independent session
- No session persistence — reconnection requires a full handshake
- No automatic replay of commands or events after reconnection

## Message Envelope

Every message follows this structure:

```json
{
  "type": "<message_type>",
  "id": "<correlation_id>",
  "name": "<command_or_event_name>",
  "payload": {}
}
```

| Field | Presence | Description |
|---|---|---|
| `type` | Always required | One of: `command`, `ack`, `result`, `event`, `error` |
| `id` | Required on `command`, `ack`, `result` | Correlation identifier. UUID recommended |
| `name` | Required on `command`, `event` | Command or event name |
| `payload` | Varies by message | Content specific to the message type |

All fields use `omitempty` — absent when not applicable.

## Message Types

| Type | Direction | Scope | Description |
|---|---|---|---|
| `command` | Client → Server | — | Client request |
| `ack` | Server → Client | Private to sender | Immediate acknowledgment |
| `result` | Server → Client | Private to sender | Final command outcome |
| `event` | Server → Client | Varies | Asynchronous notification |
| `error` | Server → Client | Private to sender | Protocol-level error (no correlation) |

## Command Flow

```
Client                    Server
  │                         │
  │── command ─────────────>│
  │<──────────────── ack ───│
  │                         │
  │<────────────── result ──│   (only if ack.accepted = true)
```

1. Client sends a `command` with a unique `id`
2. Server replies with `ack` for that `id`:
   - `accepted: true` — command is being processed, a `result` will follow
   - `accepted: false` — command is rejected, `error` field explains why, no `result` follows
3. If accepted, server sends `result` for that `id`:
   - `ok: true` — success, `data` contains the response
   - `ok: false` — failure, `error` contains the reason

### Ack Payload

```json
{
  "accepted": true,
  "error": null
}
```

When `accepted` is `false`:

```json
{
  "accepted": false,
  "error": {
    "code": "unknown_service",
    "message": "service \"foo\" is not defined"
  }
}
```

### Result Payload

```json
{
  "ok": true,
  "data": {},
  "error": null
}
```

When `ok` is `false`:

```json
{
  "ok": false,
  "error": {
    "code": "internal_error",
    "message": "..."
  }
}
```

## Commands

### `get_snapshot`

Returns the current state of all services.

**Request payload**: none.

**Response** `result.data`:

```json
{
  "services": [
    { "name": "api", "status": "running" },
    { "name": "worker", "status": "ready" }
  ]
}
```

Services are sorted alphabetically. A service with no known state has `status: "unknown"`.

### `get_logs`

Returns log entries stored in memory.

**Request payload** (all fields optional):

| Field | Type | Description |
|---|---|---|
| `service` | string | Filter by service name. Omit to get logs from all services |
| `limit` | integer | Maximum number of entries to return. Omit to let the server decide |
| `after_seq` | integer | Only return entries with `seq` strictly greater than this value |

**Response** `result.data`:

```json
{
  "entries": [
    {
      "seq": 42,
      "service": "api",
      "phase": "running",
      "stream": "stdout",
      "message": "Server started on port 3000",
      "timestamp": "2026-04-18T10:30:00Z"
    }
  ],
  "truncated": false,
  "effective_limit": 500
}
```

| Response field | Type | Description |
|---|---|---|
| `entries` | array | Log entries, ordered by `seq` |
| `truncated` | boolean | `true` if entries were dropped to fit within the limit |
| `effective_limit` | integer | The limit that was actually applied |

#### Log Entry Fields

| Field | Type | Description |
|---|---|---|
| `seq` | integer | Globally unique, monotonically increasing sequence number. Used for ordering, deduplication, and continuity tracking |
| `service` | string | Service that produced this log |
| `phase` | string | Informational lifecycle phase when the log was produced; clients may display or ignore it |
| `stream` | string | `stdout` or `stderr` |
| `message` | string | Log line content |
| `timestamp` | string | RFC 3339 timestamp |

The `seq` field is the only reliable mechanism for ordering and deduplication of log entries across sessions.

#### Limit Resolution

When the client does not provide `limit`, the server applies its default policy based on request scope:

- Without a `service` filter (all-services query):
  1. `logView.all.maxEntries`
  2. `logView.maxEntries`
  3. Built-in fallback: **500**
- With a `service` filter:
  1. `services.<name>.logView.maxEntries`
  2. `logView.maxEntries`
  3. Built-in fallback: **500**

When the client provides `limit`, that explicit value is used as-is for the response.

#### Truncation Behavior

- Without `after_seq`: all matching entries are fetched, then the last `effective_limit` entries are returned (tail truncation)
- With `after_seq`: entries with `seq > after_seq` are fetched, then the last `effective_limit` entries are returned
- `truncated` is `true` when entries were dropped by the limit

### `start_service`

Starts a single service.

**Request payload**:

| Field | Type | Required | Description |
|---|---|---|---|
| `service` | string | Yes | Service name |

### `stop_service`

Stops a single service.

**Request payload**:

| Field | Type | Required | Description |
|---|---|---|---|
| `service` | string | Yes | Service name |

### `restart_service`

Restarts a single service (stop then start).

**Request payload**:

| Field | Type | Required | Description |
|---|---|---|---|
| `service` | string | Yes | Service name |

### `start_all`

Starts all services. No payload.

### `stop_all`

Stops all services. No payload.

## Events

Events are server-initiated messages pushed to clients asynchronously.

### `hello`

Sent immediately when the WebSocket connection is established.

```json
{
  "protocol_version": 1,
  "server": "flowlayer",
  "capabilities": [
    "get_snapshot", "get_logs",
    "start_service", "stop_service", "restart_service",
    "start_all", "stop_all"
  ]
}
```

### `snapshot`

Sent immediately after `hello`. Same structure as `get_snapshot` response.

```json
{
  "services": [
    { "name": "api", "status": "running" }
  ]
}
```

### `service_status`

Broadcast to all connected sessions when a service changes state.

```json
{
  "service": "api",
  "status": "ready",
  "timestamp": "2026-04-18T10:30:05Z"
}
```

### `log`

Broadcast to all connected sessions for each log line produced by a service.

```json
{
  "seq": 43,
  "service": "api",
  "phase": "running",
  "stream": "stdout",
  "message": "Request received",
  "timestamp": "2026-04-18T10:30:06Z"
}
```

Log events are delivered on a best-effort basis and are **not guaranteed**. If a client cannot keep up, log events are silently dropped without notification. A correct client must not rely on log events alone for completeness — use `get_logs` with `after_seq` to ensure continuity. See [BUILDING-A-CLIENT.md](BUILDING-A-CLIENT.md) for the recommended log continuity pattern.

## Service Status Values

| Status | Description |
|---|---|
| `starting` | Service launch in progress |
| `running` | Process spawned |
| `ready` | Readiness probe passed |
| `stopping` | Stop sequence in progress |
| `failed` | Start or stop failed |
| `stopped` | Process terminated |
| `unknown` | No state recorded (appears only in `snapshot`) |

## Message Scope

| Message | Scope |
|---|---|
| `ack`, `result`, `error` | Private — only the command sender |
| `hello`, `snapshot` | Private — on connection only |
| `service_status` | Broadcast — all sessions |
| `log` | Broadcast — all sessions, best-effort |

## Connection Lifecycle

```
1. Client opens WebSocket to /ws with Authorization: Bearer <token>
2. Server sends "hello" event
3. Server sends "snapshot" event
4. Session is operational — client can send commands
5. Server pushes "service_status" and "log" events continuously
6. Disconnect terminates the session — no state is preserved
```

## Error Codes

| Code | Description |
|---|---|
| `invalid_json` | Malformed JSON |
| `missing_type` | Missing or empty `type` field |
| `unknown_type` | Unrecognized `type` value |
| `missing_id` | Missing `id` on a message that requires it |
| `missing_name` | Missing `name` on a command or event |
| `invalid_payload` | Malformed or invalid payload |
| `unsupported_message_type` | Client sent a non-command message type |
| `unknown_command` | Unrecognized command name |
| `unknown_service` | Service name not found in configuration |
| `service_busy` | Service is currently starting or stopping |
| `internal_error` | Server-side error |

## HTTP Endpoints

| Route | Method | Auth | Response |
|---|---|---|---|
| `/ws` | GET | Bearer token | WebSocket upgrade |
| `/health` | GET | Bearer token | `{"ok": true}` |

All routes require a valid `Authorization: Bearer <token>` header.
