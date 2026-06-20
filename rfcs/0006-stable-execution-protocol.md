# RFC-0006: Stable execution protocol

- Status: Accepted for MVP
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal exposes a stable protocol for policy-governed local execution. The protocol is intentionally higher-level than raw process spawning: clients request an `Execution`, RunSeal prepares a `Seal`, applies policy, streams events, routes network access through controlled proxy when configured, and returns structured results.

The initial transport should be JSON-RPC 2.0 over stdio or a local Unix socket / named pipe. HTTP can be added later without changing method semantics.

## Goals

- Give agent frameworks a small, stable API surface.
- Keep platform backend details out of client integrations.
- Make streaming stdout/stderr, policy denials, network proxy events, and final results first-class.
- Support one-shot CLI use and long-lived daemon/session use.
- Make the protocol easy to implement from Node, Python, Rust, Go, and shell wrappers.

## Non-goals

- No remote multi-tenant cloud control plane in v1.
- No generic workflow scheduler.
- No attempt to expose every OS-specific sandbox knob through the protocol.
- No raw credential delivery to clients or sandboxed executions.

## Transport

### JSON-RPC 2.0

Baseline transport:

```text
client <-> runseal daemon/helper
JSON-RPC 2.0 messages over stdio, Unix socket, named pipe, or localhost HTTP bridge
```

JSON-RPC gives simple request/response semantics and notifications for event streams. The protocol should not depend on a specific transport.

### Message conventions

- Method names use lower camel case: `execute`, `cancelExecution`.
- Object fields use snake_case for JSON payloads: `execution_id`, `policy_id`.
- IDs are opaque strings with stable prefixes: `exec_`, `sess_`, `seal_`, `pol_`.
- Timestamps are RFC 3339 UTC strings.
- Byte fields are integers, not human-readable strings.

## Core objects

### Agent app global policy model

RunSeal supports an app-level global sandbox policy mode. A host application typically sets one sandbox policy for all agent executions; it does not pass a different policy on every `execute` call.

A backend MAY maintain one active effective policy cohort per sandbox identity. Concurrent executions are supported when they share the same active policy hash and policy epoch.

Changing the global sandbox policy is a **policy epoch transition**:

- A policy transition MUST occur only when there are no active sandboxed executions, or it MUST fail closed.
- A backend MAY terminate old-epoch executions at transition time rather than blocking the transition.
- Mixed effective-policy concurrency under the same sandbox identity is outside the MVP.
- `policy_hash` and `policy_epoch` are bound before process start and MUST remain immutable for the lifetime of the execution.
- Every execution-scoped event and final result MUST echo the same `policy_hash` and `policy_epoch` that were bound at start.

**Design invariants:**

1. A single sandbox identity has at most one active effective policy cohort at any time.
2. All concurrent sandbox executions share the active `policy_hash` and `policy_epoch`.
3. A policy transition MUST NOT modify SID-scoped or global enforcement state while old-epoch executions are still active.
4. Global enforcement state MUST be refcounted or generation-bound.
5. Execution cleanup is per-execution (job/process tree), not per sandbox user (no mass kill).
6. Runtime roots, synthetic home, and temporary directories remain per-execution even when policy enforcement is shared.

### ExecutionRequest

```json
{
  "command": ["pnpm", "test"],
  "cwd": "/workspace",
  "policy": "workspace-write-proxy",
  "env": {
    "CI": "1"
  },
  "stdin": {
    "mode": "empty"
  },
  "timeout_ms": 600000,
  "metadata": {
    "agent_id": "agent_123",
    "skill_id": "skill_test_runner"
  }
}
```

Fields:

- `command`: argv array. Shell strings are allowed only through explicit shell mode in a future extension.
- `cwd`: requested working directory, resolved against policy.
- `policy`: named policy ID or inline policy object.
- `env`: requested non-secret environment additions. Policy may scrub or deny entries.
- `stdin`: stdin mode: `empty`, `inherit`, `bytes`, or `stream`. `bytes` payload encoding is defined by RFC-0011.
- `timeout_ms`: optional request-level timeout.
- `metadata`: client-provided opaque metadata copied into audit events after size limits and audit redaction rules.

### Execution

```json
{
  "execution_id": "exec_01J...",
  "session_id": "sess_01J...",
  "seal_id": "seal_01J...",
  "policy_id": "workspace-write-proxy",
  "policy_hash": "sha256:...",
  "policy_epoch": "epoch_01J...",
  "status": "running"
}
```

`policy_epoch` is an opaque global enforcement generation marker for the sandbox identity. The Windows reference backend MAY use the effective policy hash as the epoch identifier for the stdio MVP, but clients MUST treat the field as opaque.

Statuses:

- `queued`
- `preparing`
- `running`
- `canceling`
- `finished`
- `failed`
- `denied`

### ExecutionResult

```json
{
  "execution_id": "exec_01J...",
  "status": "finished",
  "policy_id": "workspace-write-proxy",
  "policy_hash": "sha256:...",
  "policy_epoch": "epoch_01J...",
  "exit_code": 0,
  "signal": null,
  "started_at": "2026-06-14T00:00:00Z",
  "finished_at": "2026-06-14T00:00:03Z",
  "stdout_bytes": 1024,
  "stderr_bytes": 128,
  "output_truncated": false,
  "resource_usage": {
    "duration_ms": 3000,
    "max_rss_bytes": 268435456,
    "cpu_ms": 1420
  },
  "audit_path": ".runseal/audit/sess_01J.jsonl"
}
```

### PolicyRef

A request may pass a named policy:

```json
"workspace-write-proxy"
```

Or an inline policy:

```json
{
  "version": "runseal.policy/v1",
  "filesystem": {},
  "network": {},
  "audit": {}
}
```

Inline policies must still resolve to an effective `policy_id` and `policy_hash` for audit.

## Methods

### `getVersion`

Returns implementation and protocol versions.

Request:

```json
{ "jsonrpc": "2.0", "id": 1, "method": "getVersion", "params": {} }
```

Response:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "runseal_version": "0.1.0",
    "protocol_version": "runseal.protocol/v1",
    "policy_versions": ["runseal.policy/v1"]
  }
}
```

### `getCapabilities`

Returns backend and feature capabilities for the current host.

```json
{
  "backend": "macos-seatbelt",
  "platform": "macos",
  "features": {
    "filesystem_policy": true,
    "runtime_roots": true,
    "runtime_environment": true,
    "process_isolation": true,
    "process_cleanup": true,
    "network_proxy": true,
    "managed_proxy": true,
    "direct_network_deny": true,
    "resource_limits": true,
    "audit_jsonl": true,
    "otel_export": false
  }
}
```

### `explainPolicy`

Resolves and validates a policy without running a command.

```json
{
  "method": "explainPolicy",
  "params": {
    "policy": "workspace-write-proxy",
    "cwd": "/workspace"
  }
}
```

Result includes effective filesystem roots, network mode, denied defaults, backend requirements, warnings, and the policy hash.

### `execute`

Starts an execution.

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "method": "execute",
  "params": {
    "command": ["python", "skill.py"],
    "cwd": "/workspace",
    "policy": "workspace-write-proxy"
  }
}
```

Response returns an `Execution` object. Output and audit events are streamed as notifications.

### `cancelExecution`

Requests cancellation. RunSeal should terminate the process tree and clean up the seal.
The stdio MVP validates the method and returns `EXECUTION_NOT_FOUND` for unknown
or already-finished executions. Cancelling an active execution requires a
concurrent daemon transport such as a named pipe or Unix socket; that transport
can be added later without changing the method shape.

```json
{
  "method": "cancelExecution",
  "params": {
    "execution_id": "exec_01J...",
    "reason": "user_requested"
  }
}
```

### `getExecution`

Returns current execution status and summary.

```json
{
  "method": "getExecution",
  "params": {
    "execution_id": "exec_01J..."
  }
}
```

### `subscribeEvents`

Subscribes to events for a session or execution. In stdio mode, clients may receive events automatically after `execute`; explicit subscription is useful for daemon transports.

```json
{
  "method": "subscribeEvents",
  "params": {
    "execution_id": "exec_01J...",
    "types": ["execution.*", "policy.*", "execution.network.*"]
  }
}
```

### `disposeSession`

Releases session-scoped resources such as proxy listeners, temporary directories, and audit handles.

```json
{
  "method": "disposeSession",
  "params": {
    "session_id": "sess_01J..."
  }
}
```

## Event notifications

Events use JSON-RPC notifications. They do not have an `id`.

```json
{
  "jsonrpc": "2.0",
  "method": "event",
  "params": {
    "type": "execution.stdout",
    "time": "2026-06-14T00:00:01Z",
    "runseal_version": "0.1.0",
    "execution_id": "exec_01J...",
    "policy_id": "workspace-write-proxy",
    "policy_hash": "sha256:...",
    "policy_epoch": "epoch_01J...",
    "data": "base64:SGVsbG8K",
    "encoding": "base64",
    "stream_offset": 0
  }
}
```

Important event types:

- `execution.started`
- `execution.stdout`
- `execution.stderr`
- `execution.finished`
- `execution.failed`
- `policy.denied`
- `policy.requires_approval`
- `sandbox.prepared`
- `execution.network.request`
- `execution.network.denied`
- `execution.network.proxy_ready`
- `execution.resource.limit_exceeded`

Event schemas should align with RFC-0004.

## Error model

JSON-RPC errors use stable RunSeal codes in `error.data.code`.

Malformed JSON-RPC frames on stream transports MUST return a structured error with `id: null`, JSON-RPC error code `-32700`, and `error.data.code: "INVALID_REQUEST"`. A malformed frame MUST NOT terminate the stdio service loop or mutate execution state.

JSON values that parse successfully but are not valid JSON-RPC request envelopes MUST return JSON-RPC error code `-32600` and `error.data.code: "INVALID_REQUEST"`.

If `id` is present, it MUST be a string, number, or `null`. Invalid `id` values MUST be rejected as invalid request envelopes and the error response `id` MUST be `null`.

Unknown JSON-RPC methods MUST return JSON-RPC error code `-32601` and `error.data.code: "METHOD_NOT_FOUND"`.

Method parameter validation failures MUST return JSON-RPC error code `-32602` and a stable RunSeal code in `error.data.code`.

```json
{
  "jsonrpc": "2.0",
  "id": 10,
  "error": {
    "code": -32010,
    "message": "Policy denied execution",
    "data": {
      "code": "POLICY_DENIED",
      "reason": "write_outside_workspace",
      "policy_id": "workspace-write"
    }
  }
}
```

Initial stable error codes:

| Code | Meaning |
|---|---|
| `INVALID_REQUEST` | Request payload is malformed or unsupported. |
| `METHOD_NOT_FOUND` | JSON-RPC method is unknown. |
| `POLICY_INVALID` | Policy failed validation. |
| `POLICY_DENIED` | Policy denied the request. |
| `POLICY_TRANSITION_BUSY` | A policy epoch transition or mixed-policy execution was rejected because sandboxed executions are still active. |
| `APPROVAL_REQUIRED` | The request needs approval not available in this protocol call. |
| `BACKEND_UNAVAILABLE` | No suitable backend exists on this host, or the selected backend cannot be initialized. |
| `BACKEND_CAPABILITY_MISSING` | Required backend capability is unavailable. |
| `EXECUTION_NOT_FOUND` | Execution ID is unknown. |
| `EXECUTION_FAILED_TO_START` | Backend prepared but command could not start. |
| `EXECUTION_TIMEOUT` | Execution exceeded timeout. |
| `OUTPUT_LIMIT_EXCEEDED` | Output exceeded configured limits. |
| `PROXY_ROUTE_DENIED` | Network proxy denied a route. |
| `INTERNAL_ERROR` | Unexpected implementation failure. |

## CLI mapping

The CLI is a thin protocol client:

```bash
runseal exec --policy workspace-write-proxy -- python skill.py
```

Maps to:

```json
{
  "method": "execute",
  "params": {
    "command": ["python", "skill.py"],
    "policy": "workspace-write-proxy"
  }
}
```

CLI output modes:

- default: passthrough stdout/stderr with concise final status
- `--json`: emit final `ExecutionResult` on success, or a structured `error` object on failure
- `--events`: emit JSONL event stream on success, or one structured `error` object line on failure before the stream can complete
- `--explain`: call `explainPolicy` before execution

## Compatibility rules

- New optional fields may be added to objects.
- Existing field meanings must not change within `runseal.protocol/v1`.
- Clients must ignore unknown fields.
- Servers must reject unknown required feature flags.
- Error `data.code` values are stable within v1.

## Decisions for MVP

- Approval remains a host-application concern in v1. RunSeal can return `APPROVAL_REQUIRED`, but it does not own interactive approval UI or retry orchestration in the MVP.
- Event stream chunks use base64 in v1. Clients may render UTF-8 after decoding, but wire encoding stays unambiguous for binary-safe logs.
- Sessions are implicit per execution for CLI/stdio MVP. `session_id` exists for audit and cleanup, while explicit `createSession` can be added later for long-lived daemon use cases.
- MCP is a separate adapter over the stable protocol, not a first-class transport profile in v1. This keeps the core protocol small and avoids coupling RunSeal to one agent tool ecosystem.
- Policy epoch transitions and the invariants in "Agent app global policy model" are MVP requirements for the Windows reference backend. macOS/Linux backends may adopt them as they mature.
