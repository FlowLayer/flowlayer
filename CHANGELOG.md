# Changelog

All notable changes to FlowLayer are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and FlowLayer adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Releases are published at <https://github.com/FlowLayer/flowlayer/releases> and contain both binaries (`flowlayer-server` and `flowlayer-client-tui`) plus a global `SHA256SUMS` covering every asset.

## [1.1.1] - 2026-05-05

### Changed

- TUI: logs follow mode restored on initial load.
- TUI: older logs pagination stabilized.
- TUI: WebSocket client read limit increased to handle larger `get_logs` responses.
- TUI: clearer fetch/history diagnostics.
- Server: default `get_logs` limit now aligns with `logs.bufferSize`.
- Distribution: Homebrew and Scoop updated for v1.1.1.

## [1.1.0] - 2026-04-29

### Added — server

- Bounded per-service in-memory log ring buffer with configurable capacity via `logs.bufferSize` (default `5000`). Older entries are evicted silently when `logs.dir` is configured, otherwise a one-shot `WARN` is emitted at boot.
- Optional JSONL projection on disk via `logs.dir`. The server writes `all.jsonl` and one file per service (`<service>.jsonl`), append-only, durably retaining history beyond the ring buffer.
- `get_logs` payload accepts a new optional field `before_seq` for backward pagination. Responses contain entries with strictly lower sequence numbers than the cursor.
- When `before_seq` is supplied, the server first answers from the in-memory ring; if the requested page is older than the oldest entry still in the ring and `logs.dir` is configured, it transparently falls back to a chunked reverse-tail read of the JSONL projection. The fallback is invisible to clients.
- `get_logs` without any cursor returns a tail-bounded slice (`limit + 1`) so clients can detect truncation via `truncated: true`.

### Added — TUI

- Scrolling past the top of the log buffer (`up`, `pgup`, `k`) automatically issues `get_logs` with `before_seq` set to the lowest sequence currently displayed. Older entries are prepended in order, with deduplication by sequence number.
- Viewport position is preserved across prepend operations: the user's anchor line stays visible. Auto-follow disengages on prepend so the view does not jump back to the live tail.
- In-flight backward requests are coalesced. When the server returns nothing new, a `noOlderLogsAvailable` latch is set and further attempts are suppressed until the selection changes.

### Changed

- `get_logs` enforces mutual exclusion between `after_seq` and `before_seq`; supplying both is rejected with a clean error.
- The server's `LogsTail`, `LogsAfter` and `LogsBefore` API surface is now bounded and never serialises the full buffer, regardless of how large it grows.
- `get_logs` default limit resolution now falls back to `logs.bufferSize` (default `5000`) when no explicit `limit` and no applicable `logView` policy are provided, replacing the previous internal `500` fallback.

### Fixed

- TUI no longer drops history on transient `get_logs` failures. Failed requests retain the visible log buffer so brief network blips no longer wipe the panel.

### Documentation

- New sections on backward pagination and the JSONL disk projection in `PROTOCOL.md`, `BUILDING-A-CLIENT.md`, and the public site (`/api-reference/logs/`, `/protocol/`, `/tui/`).

[1.1.1]: https://github.com/FlowLayer/flowlayer/releases/tag/v1.1.1
[1.1.0]: https://github.com/FlowLayer/flowlayer/releases/tag/v1.1.0
[1.0.0]: https://github.com/FlowLayer/flowlayer/releases/tag/v1.0.0
