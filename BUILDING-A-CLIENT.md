# Building a FlowLayer Client

> **Documentation:** https://flowlayer.tech  
> **Entry point:** [README.md](README.md)

This document describes the expected behavior of a correct FlowLayer client. It covers everything needed to build one in any language. The only requirements are a WebSocket library and a JSON parser.

For the complete protocol specification, see [PROTOCOL.md](PROTOCOL.md).

## Connecting

Open a WebSocket connection to `ws://<host>:<port>/ws` with the authentication header:

```
Authorization: Bearer <token>
```

Missing or invalid tokens are rejected at the HTTP level (401/403) before the WebSocket upgrade.

## Handshake

After connecting, the server sends two events in order:

1. **`hello`** — protocol version and supported commands
2. **`snapshot`** — current state of all services

Wait for both before sending any commands. The session is not operational until the snapshot is received.

```json
{"type": "event", "name": "hello", "payload": {
  "protocol_version": 1,
  "server": "flowlayer",
  "capabilities": ["get_snapshot", "get_logs", "start_service", "stop_service", "restart_service", "start_all", "stop_all"]
}}

{"type": "event", "name": "snapshot", "payload": {
  "services": [
    {"name": "api", "status": "running"},
    {"name": "worker", "status": "ready"}
  ]
}}
```

## Event Loop

After the handshake, read messages continuously from the WebSocket. Dispatch by `type` first, then by `name` for events:

```
message.type == "event"   → dispatch by message.name (service_status, log)
message.type == "ack"     → match to pending command by message.id
message.type == "result"  → match to pending command by message.id
message.type == "error"   → protocol-level error, no correlation
```

### Service State

Use `snapshot` for the full initial state. Then apply `service_status` events as incremental updates. The server is the single authority on service state.

`running` does not always mean full operational readiness. For services without a readiness probe, `running` can be terminal; for services with a readiness probe, only `ready` indicates full availability.

If a `service_status` event references a service not in your current list, add it. Services are never removed.

### Log Events

Each `log` event carries a `seq` (globally unique, monotonically increasing). Track the highest `seq` you have seen — this is your high-water mark for log continuity.

Log events are best-effort. If your client is slow, the server drops log events silently. Use `get_logs` with `after_seq` to recover gaps.

## Sending Commands

Build a command envelope with a unique `id` (UUID recommended), send it over the WebSocket, and register it as pending:

```json
{"type": "command", "id": "550e8400-e29b-41d4-a716-446655440000", "name": "get_logs", "payload": {"service": "api"}}
```

### Correlating Responses

Match `ack` and `result` messages to pending commands using the `id` field.

**Flow:**
1. Receive `ack` with matching `id`
2. If `ack.payload.accepted == false` → command rejected, done
3. If `ack.payload.accepted == true` → wait for `result`
4. Receive `result` with matching `id` → command complete

Set a client-side timeout (5 seconds is reasonable). The server does not impose command timeouts — this is entirely a client responsibility. If no response arrives within the timeout, treat the command as failed.

## Log Management

This is the most complex part of a FlowLayer client. Getting it wrong leads to missing logs, duplicates, or unbounded memory growth.

### Fetching Logs

Send `get_logs` to retrieve stored log entries:

```json
{"type": "command", "id": "...", "name": "get_logs", "payload": {"service": "api"}}
```

The response includes:

```json
{
  "entries": [...],
  "truncated": false,
  "effective_limit": 500
}
```

- The `effective_limit: 500` value in this example reflects the current built-in fallback and is not a universal fixed value outside the protocol contract.
- `effective_limit` is the limit the server actually applied. Use this value for your local buffer size — do not compute limits locally.
- `truncated` tells you if older entries were dropped.

### Using `after_seq`

To fetch only new logs (for example, after reconnection), include `after_seq` set to your high-water mark:

```json
{"type": "command", "id": "...", "name": "get_logs", "payload": {"after_seq": 42}}
```

This returns only entries with `seq > 42`.

### Deduplication

The same log entry can arrive through two paths:
1. As a real-time `log` event
2. In a `get_logs` response

Always deduplicate by `seq`. Maintain a set of seen sequence numbers and discard any entry you have already processed.

### Buffer Management

Use the `effective_limit` from `get_logs` responses to cap your local log buffer. When the buffer exceeds this size, trim from the beginning (oldest entries first).

Before the first successful `get_logs`, a client may not know `effective_limit` yet. Do not assume an arbitrary local trim limit before receiving it.

### Log Continuity Pattern

The recommended pattern for maintaining log continuity:

1. On initial load, send `get_logs` (optionally with `service`)
2. Store `effective_limit` for buffer management (it is only known after a successful `get_logs`)
3. Track `lastSeq` as the highest `seq` seen (from both `get_logs` responses and live `log` events)
4. Append live `log` events as they arrive, deduplicating by `seq`
5. On reconnection, after receiving `snapshot`, send `get_logs` with `after_seq = lastSeq`
6. Merge the replayed entries with any live events received in the meantime, deduplicating by `seq`
7. Trim buffer to `effective_limit`

## Reconnection

The server does not persist sessions. A reconnection is a completely new session.

### Reconnection Strategy

1. Implement exponential backoff (recommended: 500ms → 1s → 2s → 5s max)
2. On reconnect, go through the full handshake (wait for `hello` → `snapshot`)
3. After `snapshot`, replay logs with `get_logs` using `after_seq = lastSeq`
4. Invalidate all pending commands — they will never receive a response

### What Happens to Pending Commands

When the connection drops, any commands awaiting `ack` or `result` are orphaned. The server has no mechanism to deliver responses across sessions. Mark all pending commands as failed and let the caller decide whether to retry.

Do not automatically retry commands after reconnection. The service state may have changed.

## Service Control

To start, stop, or restart services:

```json
{"type": "command", "id": "...", "name": "start_service", "payload": {"service": "api"}}
{"type": "command", "id": "...", "name": "stop_service", "payload": {"service": "worker"}}
{"type": "command", "id": "...", "name": "restart_service", "payload": {"service": "api"}}
{"type": "command", "id": "...", "name": "start_all"}
{"type": "command", "id": "...", "name": "stop_all"}
```

These commands may be rejected with `unknown_service` (service not in config) or `service_busy` (service is mid-start or mid-stop). Both rejections come as `ack.accepted = false`.

## Summary

A minimal client needs to:

1. Connect with `Authorization: Bearer <token>`
2. Wait for `hello` and `snapshot`
3. Read events in a loop, dispatch by `type` and `name`
4. Correlate `ack`/`result` to commands by `id`
5. Track `lastSeq` for log continuity
6. Deduplicate logs by `seq`
7. Handle disconnection with backoff and full handshake replay
