# RFC-0013: RunSeal service mode

- Status: Draft for review
- Repository: `runseal-labs/rfcs`
- Related: RFC-0006, RFC-0007, RFC-0008, RFC-0010, RFC-0012

## Summary

RunSeal should evolve from a CLI and stdio JSON-RPC executor into an optional local service runtime. The service owns long-lived runtime state such as sessions, executions, event subscriptions, policy epoch state, managed proxy leases, setup readiness, and audit indexing.

The CLI remains supported. Service mode adds a long-running local control plane for richer integrations without changing the public `Execution`, `SandboxPolicy`, `BackendCapabilities`, `PlatformSandboxPlan`, or `AuditEvent` vocabulary.

The first service-mode milestone MUST be local-only and SHOULD start with stdio or same-user local IPC. It MUST NOT expose a remote HTTP API, multi-user daemon surface, or network listener by default.

## Operating mode stance

RunSeal MUST NOT become daemon-only. Direct mode remains a first-class operating mode for CLI use, CI, debugging, bootstrap, and environments where no service is running.

Service mode is the long-term stateful integration path. It is expected to evolve toward daemon-capable local operation, but daemon capability does not imply a default remote server, a network listener, a multi-user service, or a privileged system service.

The intended long-term model is:

```text
direct mode
  runseal exec ...
  runseal rpc --stdio

service mode
  runseal service --stdio
  runseal service --pipe / --socket
  runseal --service exec ...
```

Implementations MUST preserve direct mode even after service mode becomes stable.

## Goals

1. Add a local service control plane for long-lived RunSeal state.
2. Preserve current CLI and JSON-RPC behavior as thin client entry points.
3. Make `execute`, `getExecution`, `cancelExecution`, `subscribeEvents`, and `disposeSession` backed by real service state.
4. Let service mode own sessions, execution records, event subscriptions, audit handles, managed proxy leases, runtime cleanup, backend status, and policy epoch state.
5. Keep `SandboxBackend` implementations independent of service internals.
6. Keep unsupported or partially enforceable requests fail-closed.
7. Keep private platform details out of public protocol, audit output, setup status, and conformance fixtures.

## Non-goals

This RFC does not require:

1. A remote network API.
2. A cloud service.
3. A multi-user daemon.
4. A system service installed at boot.
5. A database-backed history store.
6. A policy migration system.
7. A privilege-escalation path for setup or repair.
8. A replacement for direct CLI execution.

Service mode is optional. Direct CLI execution MUST remain available for debugging, CI, bootstrap, and environments where no service is running.

## Architecture

The intended dependency direction is:

```text
commands -> execution OR service client
protocol -> service
service -> execution
execution -> backend
backend -> platform modules
```

Backend modules MUST NOT depend on service modules.

A service-oriented layout in the implementation repository may look like:

```text
src/service/mod.rs
src/service/daemon.rs
src/service/state.rs
src/service/sessions.rs
src/service/executions.rs
src/service/event_bus.rs
src/service/transport.rs
src/service/shutdown.rs
```

This layout is illustrative. The public requirement is the ownership boundary, not exact file names.

## Service state model

The service owns mutable runtime state that one-shot CLI mode cannot safely or efficiently hold:

```text
ServiceState
  SessionStore
  ExecutionStore
  EventBus
  BackendRuntime
  PolicyEpochRuntime
  ManagedProxyRuntime
  AuditIndex
  SetupReadinessCache
```

The implementation MAY use in-memory state for the MVP. Persistent storage is not required by this RFC.

## Sessions

A `Session` is the service-scoped owner for resources that may outlive a single request:

```text
session_id
active executions
subscriptions
managed proxy handles
runtime cleanup leases
audit handles or audit index cursors
```

The service MAY create implicit sessions for CLI and stdio clients. Explicit session creation can be added later without changing the core `Execution` object.

`disposeSession` MUST release session-scoped resources that are safe to release. It MUST NOT delete audit records that are already committed.

## Executions

Service mode should make execution lifecycle methods stateful:

```text
execute            creates and starts an Execution
getExecution       returns current or final Execution state
cancelExecution    requests cancellation of a running Execution
subscribeEvents    streams matching events
disposeSession     releases session resources
```

An `Execution` record should track at least:

```text
execution_id
session_id
seal_id
status
policy_id
policy_hash
policy_epoch
backend
platform_plan summary
audit_path
start and finish timestamps
exit status or structured failure
```

The service MUST preserve existing audit and event semantics. Service state is an index and lifecycle owner, not a replacement for the audit event model.

## Event bus

Service mode should provide a first-class event bus for execution lifecycle, policy decisions, backend setup failures, managed proxy activity, resource limits, cancellation, and final result events.

Event notifications MUST preserve the JSON-RPC event envelope and `AuditEvent` shape defined by accepted RFCs. Service mode MAY add subscription filtering, but it MUST NOT change event names or event payload shape without a separate RFC.

## Policy epoch runtime

Service mode should move policy epoch handling from per-request derivation toward explicit runtime ownership.

For the Windows reference backend, the service SHOULD own:

```text
active policy cohort
active execution count
policy epoch transition lock
same-policy concurrency tracking
post-drain epoch activation
```

Policy transitions MUST follow RFC-0006 and RFC-0012:

- A transition MUST occur only when no active sandboxed executions require old global enforcement state, or it MUST fail closed.
- A started execution's `policy_hash` and `policy_epoch` MUST remain immutable.
- Every execution-scoped event and final result MUST echo the bound values.

## Managed proxy runtime

Service mode can improve managed proxy behavior by pooling listeners and using per-execution authorization tokens.

The service MAY hold:

```text
managed proxy listener pool
per-execution proxy token leases
proxy event routing
proxy cleanup leases
```

Managed proxy pooling MUST NOT weaken proxy-only enforcement. A direct socket bypass MUST still fail where proxy mode is claimed supported.

## Audit indexing

Service mode MAY maintain an in-memory or lightweight persistent audit index for queries such as:

```text
listExecutions
getAuditEvents
tailAudit
queryAuditBySession
queryAuditByPolicyHash
```

The JSONL audit log remains the durable event artifact unless a future RFC replaces or extends storage requirements.

Audit indexing MUST NOT store raw secrets, raw stdin bytes, encoded stdin payloads, decoded stdin text, credential material, Authorization headers, cookies, or backend-private platform details.

## Setup readiness

Service mode MAY expose setup status as a service method:

```text
getSetupStatus
watchSetupStatus
explainSetupFailure
```

A repair or setup action MUST NOT become an implicit privilege-escalation path. Unsupported or stale setup states MUST remain structured fail-closed conditions.

## Transport model

The service transport should be introduced in phases.

### Phase 1: stdio service mode

The first implementation SHOULD support a stdio service mode:

```text
runseal service --stdio
```

This mode is suitable for tests, embedded clients, and development. It avoids local IPC ACL questions while proving stateful lifecycle behavior.

### Phase 2: same-user local IPC

After stdio service mode is proven, RunSeal MAY add same-user local IPC:

```text
Windows: named pipe
macOS/Linux: Unix domain socket
```

The IPC transport MUST be local-only. It MUST authenticate or restrict access to the intended local user boundary. It MUST NOT expose a TCP listener by default.

The minimum IPC security floor is:

1. A same-user IPC transport MUST be enabled only when the platform can prove the peer is inside the intended local user or logon-session boundary.
2. If the implementation cannot reliably authenticate the peer, the transport MUST fail closed and direct CLI or stdio service mode MUST remain the fallback.
3. IPC authorization failure MUST happen before executing a request, mutating service state, allocating a managed proxy lease, or exposing execution/audit data.
4. Transport diagnostics MUST use public RunSeal terminology and MUST NOT expose backend-private ACLs, SIDs, token attributes, socket inode details, runtime paths, or credential material.

Windows named pipe requirements:

1. The server MUST create the named pipe with an explicit security descriptor; it MUST NOT rely on the Windows default named pipe DACL.
2. The DACL MUST restrict access to the service owner and intended logon-session boundary, and MUST reject remote network clients.
3. The server MUST validate the connected client token before honoring a request, using impersonation/access-check behavior or an equivalent Windows peer-authentication primitive.
4. Pipe creation MUST fail closed if the implementation cannot prevent a preexisting or less-restricted pipe instance from being used for RunSeal service traffic.

macOS/Linux Unix domain socket requirements:

1. The socket MUST live under a service-owned runtime directory that is not writable by other users.
2. The implementation SHOULD set restrictive socket permissions where the platform honors them, but MUST NOT rely on socket file permissions as the only security boundary.
3. The server MUST verify peer credentials with the platform's supported primitive, such as `SO_PEERCRED` on Linux or `getpeereid` / `LOCAL_PEERCRED` on macOS/BSD-derived systems.
4. Linux abstract sockets MUST NOT be used as the portable default because their access cannot be constrained by filesystem permissions.

A future TCP, HTTP, or browser-reachable local transport requires a separate RFC. That RFC must define authentication, request origin handling, CORS behavior, DNS rebinding defenses, CSRF defenses, listener binding, token storage, and audit-safe failure semantics.

### Phase 3: installable service

An installable Windows service, launchd service, or systemd user service MAY be added later through a separate RFC.

That RFC must define:

```text
service identity
IPC ACLs
upgrade behavior
shutdown behavior
setup/repair permissions
log locations
crash recovery
uninstall behavior
```

## CLI behavior

Direct mode remains available:

```text
runseal exec ...
```

Service client mode MAY be explicit:

```text
runseal --service exec ...
```

or may auto-detect a running service later. If auto-detection is added, direct mode MUST remain available as an explicit fallback.

## Protocol compatibility

The MVP service should reuse existing JSON-RPC methods where possible:

```text
getVersion
getCapabilities
explainPolicy
execute
getExecution
cancelExecution
subscribeEvents
disposeSession
```

Future service methods MAY include:

```text
getServiceStatus
getSetupStatus
listExecutions
getAuditEvents
tailAudit
drainExecutions
cleanupRuntimeRoots
getPolicyEpoch
```

New methods that change public protocol behavior require an RFC update.

### `getServiceStatus`

`getServiceStatus` reports the current local control-plane mode without starting setup, opening a listener, or mutating service state.

The response MUST remain public-safe and SHOULD include:

```json
{
  "status": "running",
  "mode": "service",
  "transport": "stdio",
  "stateful": true,
  "local_only": true,
  "remote_listener": false
}
```

Allowed `mode` values are `direct` and `service`.
Allowed `transport` values are `stdio`, `pipe`, `socket`, and `none`.
`remote_listener` MUST be `false` unless a later RFC explicitly defines a remote-capable transport.

### `listExecutions`

`listExecutions` returns an in-memory summary of executions known to the current service process. Direct mode MAY return an empty list because it does not own cross-request service state.

The response MUST remain public-safe and MUST NOT include stdout, stderr, raw stdin, metadata payloads, raw audit events, credential material, or backend-private platform details.

The response SHOULD include:

```json
{
  "executions": [
    {
      "execution_id": "exec_01J...",
      "session_id": "sess_01J...",
      "status": "finished",
      "policy_id": "danger-full-access",
      "policy_hash": "sha256:...",
      "policy_epoch": "sha256:...",
      "backend": {
        "name": "runseal-windows-reference",
        "status": "reference",
        "platform": "windows"
      },
      "audit_path": ".runseal/audit/sess_01J.jsonl",
      "started_at": "2026-06-14T00:00:00Z",
      "finished_at": "2026-06-14T00:00:03Z"
    }
  ],
  "count": 1
}
```

### `getAuditEvents`

`getAuditEvents` returns service-known audit/event records for one execution. The MVP service MAY serve only events retained in the current service process; it MUST NOT accept arbitrary filesystem paths or read an untrusted `audit_path` from the request.

Request parameters SHOULD be:

```json
{
  "execution_id": "exec_01J...",
  "types": ["execution.*", "policy.*"]
}
```

`types` follows the same filtering semantics as `subscribeEvents`. Omitting `types` returns all retained events for that execution.

The response SHOULD include:

```json
{
  "execution_id": "exec_01J...",
  "events": [],
  "count": 0
}
```

The response MUST preserve `AuditEvent` shape and MUST NOT include raw stdin, credential material, Authorization headers, cookies, backend-private details, or metadata payloads that were not already part of the public event stream.

## Security requirements

Service mode MUST preserve these requirements:

1. Unsupported sandboxed requests fail closed.
2. Setup unavailable states fail closed.
3. Policy transition conflicts fail closed.
4. Backend-private details are not exposed publicly.
5. Service transport is local-only by default.
6. Service mode does not grant hidden privilege escalation.
7. CLI direct mode remains available.
8. Audit/event shape remains stable unless changed by a later RFC.
9. Same-user IPC is not supported unless peer authentication is reliable on that platform.

## Implementation phases

Recommended implementation sequence:

```text
1. Complete execution layer extraction from lib.rs.
2. Add service-neutral state types without opening a socket.
3. Route JSON-RPC methods through a Service object in stdio mode.
4. Implement real getExecution, cancelExecution, subscribeEvents, and disposeSession semantics.
5. Add stdio service mode.
6. Add same-user local IPC.
7. Add installable service only after a separate design review.
```

## Acceptance criteria

A service-mode implementation is acceptable only when:

1. Existing CLI and protocol contract tests still pass.
2. Service-backed `execute` emits the same event and audit shapes as direct execution.
3. `getExecution` returns correct state for active and completed executions.
4. `cancelExecution` either cancels an active execution or returns a stable not-found / not-cancellable result.
5. `subscribeEvents` streams existing event shapes.
6. `disposeSession` releases service-owned resources without deleting committed audit records.
7. Unsupported requests remain fail-closed.
8. Windows policy epoch tests still pass.
9. Windows smoke still passes.
10. No remote listener is exposed by default.
11. Same-user IPC tests prove unauthorized peers and remote clients are rejected before any request is executed.

## Review decision

Service mode is approved as the long-term stateful integration direction after the execution layer is extracted. RunSeal is expected to become daemon-capable over time, but it MUST NOT become daemon-only. The first implementations should remain local-only, same-user, stateful, and conservative. They should add service ownership for lifecycle and resources without changing public policy, audit, backend, or platform semantics.
