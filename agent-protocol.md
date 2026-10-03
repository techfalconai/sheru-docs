# Agent-to-server protocol

This document defines the wire protocol between the Sheru agent and the backend
API. It is versioned so the backend can evolve without breaking older agents.

## Transport

- **Control channel**: persistent WebSocket (`/api/v1/agent/ws`) over TLS 1.2+.
  The agent authenticates with its device credential (query param `token`).
- **Bulk upload**: plain HTTPS (`httpx`) for large payloads (inventory, log
  bundles) so large transfers do not block the control channel.
- **TLS**: strict CA validation in production; the agent must not silently
  accept invalid/self-signed certificates. No plaintext fallback.

## Message envelope

Every agent→server message is wrapped in a versioned envelope:

```json
{
  "version": "1.0",
  "type": "telemetry|heartbeat|command_result|alert|inventory|...",
  "agent_id": "uuid",
  "tenant_id": "uuid",
  "timestamp": "ISO8601",
  "payload": { }
}
```

The server unwraps the envelope and dispatches on `type`. Legacy flat frames
(without `version`/`payload`) are still accepted for backward compatibility.

## Message types

| Type            | Direction | Purpose                                        |
| --------------- | --------- | ---------------------------------------------- |
| `heartbeat`     | agent →   | lightweight liveness (version, uptime)        |
| `telemetry`     | agent →   | metric samples (CPU/mem/disk/network)          |
| `inventory`     | agent →   | hardware + software inventory                  |
| `alert`         | agent →   | policy-threshold alert                         |
| `command_result`| agent →   | result of a pushed command (correlated by ID)  |
| `patch.scan`    | agent →   | reported patch inventory                       |
| `result`        | agent →   | reply to a request frame (correlated by req_id)|

## Command dispatch

The backend pushes a request frame over the WebSocket with a `req_id`:

```json
{ "type": "exec", "req_id": "uuid", "command": "...", "timeout": 60 }
```

The agent acknowledges immediately, executes asynchronously, and replies with a
correlated `result` frame:

```json
{ "type": "result", "req_id": "uuid", "ok": true, "data": { "exit_code": 0, "stdout": "...", "stderr": "" } }
```

Request types: `exec`, `script`, `file.list`, `file.read`, `file.write`,
`services.list`, `services.control`, `inventory`, `events`, `install`, `check`,
`registry.read`, `registry.list`, `registry.write`, `registry.delete`,
`screenshot`, `input`, `osquery.query`, `osquery.status`.

## Idempotency

Commands and config updates carry an ID (`req_id` / command ID) so re-delivery
after a reconnect does not double-execute. The agent de-duplicates by ID.

## Backpressure

The backend may signal overload by closing the connection or sending a
`backpressure` frame; the agent throttles its telemetry volume in response and
caps its local offline queue (default 50 MB / 10 000 items), dropping oldest
messages with a counter rather than growing unbounded.

## Offline queue

When the backend is unreachable, outbound messages are persisted to a
size-capped SQLite queue and flushed on reconnect. The queue survives process
restarts and endpoint reboots.

## Reconnect

The agent reconnects with exponential backoff + full jitter (base 1s, max 300s),
resetting the backoff on a successful connection.

## Multi-tenancy

Every message carries `tenant_id` (and the agent enrolls against a specific
`site_id`), so backend-side isolation is straightforward. The enrollment token
embeds tenant/site scope, so an agent can never register into the wrong tenant.
