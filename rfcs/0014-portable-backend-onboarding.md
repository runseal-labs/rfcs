# RFC-0014: Portable backend onboarding for macOS and Linux

- Status: Draft for review
- Repository: `runseal-labs/rfcs`
- Related: RFC-0003, RFC-0006, RFC-0007, RFC-0008, RFC-0010, RFC-0011, RFC-0012

## Summary

RunSeal should onboard macOS and Linux through a capability-driven `SandboxBackend` framework. New platforms must implement the shared backend contract, report conservative capabilities, pass conformance tests, and fail closed for unsupported requests.

The public protocol, policy schema, `AuditEvent` shape, `PlatformSandboxPlan` shape, and CLI behavior must remain platform-neutral. Platform-private details such as Windows ACLs, macOS Seatbelt internals, Linux namespace flags, seccomp filters, Landlock descriptors, or bubblewrap command lines must not leak into public output.

## Goals

1. Make future macOS and Linux backends plug into the backend contract without rewriting CLI, JSON-RPC, execution, or audit code.
2. Keep Windows as the MVP reference backend and enterprise security baseline.
3. Keep macOS experimental until specific capabilities pass conformance tests.
4. Keep Linux future/community or experimental until runtime probes and conformance tests prove specific capabilities.
5. Make capability claims granular and evidence-based.
6. Keep unsupported capability requests structured fail-closed.
7. Preserve platform-neutral policy, audit, event, and protocol shapes.
8. Define promotion criteria for moving a backend feature from unsupported or experimental to supported.

## Non-goals

This RFC does not require:

1. Making macOS or Linux equivalent to the Windows reference backend immediately.
2. Adding a VM, microVM, container daemon, or root-required firewall dependency for MVP.
3. Exposing platform-private enforcement controls in public protocol output.
4. Supporting remote execution.
5. Adding network support before filesystem and runtime isolation are proven.
6. Promoting any backend feature without conformance evidence.

## Backend onboarding model

Every platform backend must implement the same conceptual contract:

```text
name
status
platform
supported_features
compile_plan
execute_plan
capabilities_json
```

Platform onboarding should follow this order:

1. Add a backend type.
2. Add backend registry selection.
3. Report unsupported or experimental capabilities first.
4. Implement `compile_plan` with platform-neutral public output.
5. Implement `execute_plan` only when enforcement is proven.
6. Add conformance tests.
7. Promote individual capability claims only when tests pass.
8. Keep unsupported edge cases fail-closed.
9. Keep JSON-RPC, CLI, and audit output shapes unchanged.
10. Keep backend-private details out of public output.

## Capability model

Backend capability reporting should be granular enough to describe partial progress.

Recommended capability categories:

```text
FilesystemPolicy
RuntimeRoots
RuntimeEnvironmentRedirects
ProcessIsolation
ProcessCleanup
DirectNetworkDeny
ManagedProxy
PolicyEpoch
SetupReadiness
StdinBytes
StdinFile
AuditJsonl
```

Capability status should distinguish:

```text
supported
experimental
unsupported
unavailable
requires_setup
```

Diagnostics MAY be reported, but they must remain public-safe and coarse. They must not expose local private paths, account names, raw rule names, internal helper identities, or system-specific secrets.

## Public and private plan boundary

`PlatformSandboxPlan` must remain platform-neutral.

Allowed public plan concepts:

```text
backend name
backend status
platform
execution_id
policy_id
policy_hash
sandbox level
network mode
runtime root
synthetic home
profile root
temp root
logical setup requirements
logical filesystem labels
fail-closed setup requirement
```

Disallowed public plan concepts:

```text
raw ACLs
SIDs
token attributes
Job Object handles
firewall or WFP rule names
helper identities
Seatbelt profile fragments
sandbox-exec flags
namespace flags
bubblewrap argv
Landlock ruleset file descriptors
seccomp filters
private helper paths
```

Backends MAY maintain private compiled plans internally, but those details must not cross into JSON-RPC results, audit events, setup status, README examples, or conformance fixtures.

## macOS backend strategy

macOS remains a local-development backend. `read-only` and `workspace-write` with `network.disabled` are supported when the runtime guard is available; other sandbox levels and network modes still fail closed. `workspace-contained` is not a macOS target because endpoint agents need practical host-side tool and desktop integration.

A future implementation may use a structure like:

```text
src/backend/macos.rs
src/macos/capability_probe.rs
src/macos/plan.rs
src/macos/process.rs
src/macos/profile.rs
```

This layout is illustrative. The implementation should use the smallest structure that keeps macOS-specific logic out of generic backend modules.

### macOS phase 0: capability-only backend

The initial macOS backend should:

```text
report platform = macos
report status = experimental
allow danger-full-access as explicit local execution
fail closed for sandboxed policies
```

### macOS phase 1: plan preview

The backend may compile logical public plans for read-only or workspace-write policies without claiming execution support.

Plan preview must remain platform-neutral and must not expose private profile fragments or sandbox mechanism details.

### macOS phase 2: read-only execution

Read-only execution may be reported supported after tests prove:

```text
workspace write is denied
expected read roots are accessible
runtime roots are prepared safely
audit and event shapes remain unchanged
unsupported network modes fail closed
```

### macOS phase 3: workspace-write

`workspace-write` may be reported supported after tests prove:

```text
workspace writes are limited to approved roots
external writes are denied
synthetic home and profile roots are isolated
temp roots are isolated
protected subpaths remain protected
```

`workspace-contained` remains unsupported on macOS by design.

### macOS phase 4: network

Network features should be added after filesystem and runtime isolation are stable.

No macOS network mode should be claimed supported until direct bypass tests pass for that mode.

## Linux backend strategy

Linux support should use runtime feature detection. A Linux build target is not by itself evidence that sandbox capabilities are enforceable.

A future implementation may use a structure like:

```text
src/backend/linux.rs
src/linux/capability_probe.rs
src/linux/plan.rs
src/linux/bubblewrap.rs
src/linux/landlock.rs
src/linux/seccomp.rs
src/linux/process.rs
```

This layout is illustrative and should start smaller if possible.

## Linux capability probe

Linux backends should probe runtime support for:

```text
user namespace
user namespace quota
mount namespace
PID namespace
network namespace
seccomp
Landlock
Landlock ABI version
bubblewrap availability
unprivileged namespace availability
```

The probe should produce public-safe diagnostics. The probe alone does not promote sandbox execution support.

Probe entries may include optional public-safe `details` when a mechanism needs
version-gated follow-up decisions, such as a Linux Landlock ABI version. These
details remain diagnostic-only and MUST NOT expose backend-private handles,
paths, descriptors, or command lines.

## Linux phases

### Linux phase 0: runtime probe only

The first Linux step should add capability probes and public-safe diagnostics while keeping sandboxed execution unsupported and fail-closed.

### Linux phase 1: danger-full-access only

`danger-full-access` remains explicit local execution. All other sandboxed policies must fail closed unless a feature is implemented and tested.

### Linux phase 2: bubblewrap read-only execution

A Linux backend may use bubblewrap or an equivalent low-level unprivileged sandboxing mechanism for initial read-only execution.

Initial experimental targets:

```text
read-only
runtime root bind or equivalent
synthetic home
isolated temp root
process cleanup
```

### Linux phase 3: workspace-write

`workspace-write` with `network.disabled` may become supported after conformance tests prove:

```text
workspace-write allows approved workspace writes
workspace-write denies external writes
runtime roots are per execution
protected subpaths remain protected
```

`workspace-contained` remains unsupported on Linux by design.

### Linux phase 4: Landlock augmentation

Landlock can augment filesystem policy, but must be gated by runtime ABI detection.

The backend must use only access rights supported by the running kernel's Landlock ABI. Missing or insufficient Landlock support must produce an unsupported or unavailable status, not a weak fallback that claims enforcement.

### Linux phase 5: network

Network support should come after filesystem behavior is stable.

Potential implementation routes include network namespaces or equivalent direct egress denial, and proxy-only routing with managed proxy token guards.

Root-required system firewall mutation is out of MVP scope for Linux.

## Conformance promotion criteria

A backend feature can move from unsupported or experimental to supported only when all of these are true:

1. Implementation exists.
2. Capability reporting claims the feature.
3. Black-box conformance tests pass for the feature.
4. Unsupported edge cases fail closed.
5. Audit and event shape remains platform-neutral.
6. JSON-RPC response shape remains unchanged.
7. Backend-private details do not appear in public output.
8. Windows reference behavior remains unchanged.

## Required conformance matrix

Future conformance tests should preserve the existing `when_supported_or_fails_closed` style.

Required platform-agnostic tests include:

```text
read_only_denies_workspace_write_when_supported_or_fails_closed
workspace_write_allows_workspace_write_when_supported_or_fails_closed
workspace_write_denies_external_write_when_supported_or_fails_closed
workspace_contained_denies_external_read_when_supported_or_fails_closed
runtime_environment_roots_are_per_execution_when_supported_or_fails_closed
workspace_write_protects_workspace_metadata_when_supported_or_fails_closed
network_disabled_blocks_direct_egress_when_supported_or_fails_closed
network_proxy_blocks_direct_egress_when_supported_or_fails_closed
network_proxy_allows_http_through_managed_proxy_when_supported_or_fails_closed
network_proxy_credentials_are_redacted_when_supported_or_fails_closed
workspace_write_accepts_bytes_stdin_when_supported_or_fails_closed
workspace_write_accepts_file_stdin_when_supported_or_fails_closed
```

Platform-specific tests may be added, but they should verify behavior or public-safe boundaries rather than private implementation details.

## Backend registry evolution

Current registry shape may remain simple:

```text
active_backend() -> ActiveBackend
```

Future runtime probing should stay inside backend registry or platform modules. CLI and protocol layers should not know platform internals.

A future internal shape may be:

```text
BackendRegistry
  probe cache
  active backend selection
  public-safe capability diagnostics
```

The public entrypoint can remain stable while internals become more capable.

## Service-mode compatibility

Portable backend onboarding must be compatible with service mode.

Dependency direction should remain:

```text
commands -> execution OR service client
protocol -> service
service -> execution
execution -> backend
backend -> platform modules
```

Backend modules MUST NOT depend on service modules.

## Documentation and RFC alignment

If macOS or Linux support changes public platform status, protocol fields, policy semantics, event shape, error codes, or conformance semantics, update RFCs before or in the same change.

Implementation-only details should stay in the implementation repository.

## Stop conditions

Stop and create a design follow-up if platform implementation requires:

```text
public protocol shape changes
public audit/event shape changes
policy hash changes
weaker fail-closed behavior
backend-private details in public output
system-wide root-required mutation for MVP
weak fallback while claiming supported enforcement
test weakening
```

## Reference signals

This RFC is aligned with these public technical signals:

```text
Apple App Sandbox is app-oriented and is not automatic proof of per-command CLI sandbox equivalence.
Linux Landlock supports unprivileged self-restriction and requires runtime ABI-aware handling.
Linux namespaces partition kernel resources and are relevant to filesystem, process, PID, and network isolation.
Linux seccomp is relevant to syscall filtering and attack-surface reduction.
bubblewrap is a low-level unprivileged sandboxing tool used by Flatpak and related projects.
```

These are guidance signals, not support claims. RunSeal support claims require conformance evidence.

## Recommended next implementation task

Do not implement macOS or Linux enforcement first.

The recommended next implementation task is:

```text
Add Linux and macOS capability probe scaffolding without changing execution behavior.
```

Expected behavior:

```text
capability diagnostics remain public-safe
sandboxed execution behavior remains unchanged
Windows reference behavior remains unchanged
all tests pass
```

## Review decision

Portable backend onboarding is approved as a future direction. Backends should be capability-driven, conformance-gated, and fail-closed by default. macOS and Linux should be added incrementally without changing public CLI, JSON-RPC, policy, audit, or Windows reference behavior.
