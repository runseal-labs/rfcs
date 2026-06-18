# RFC-0010: RFC/implementation boundary and Windows reference extraction

- Status: Accepted for MVP
- Created: 2026-06-18
- Project: RunSeal

## Summary

RunSeal is split across two public repositories:

- `runseal-labs/rfcs` owns the public contract: protocol, policy schema, threat model, implementation plan, conformance requirements, and acceptance criteria.
- `runseal-labs/runseal` owns the concrete Rust implementation: CLI, JSON-RPC server, core models, backend implementations, conformance tests, and release artifacts.

The Windows reference backend may be informed by prior Windows sandbox engineering experience, but the public implementation must be abstracted, renamed, and sanitized. Public code, comments, tests, commit messages, README content, issues, and RFCs must use RunSeal terminology only.

## Goals

1. Keep RFCs as the durable public source of truth for protocol and implementation requirements.
2. Keep the implementation repository focused on executable code and conformance evidence.
3. Allow the Windows reference backend to reuse proven engineering patterns without exposing private product context.
4. Define redaction and abstraction rules before implementation work begins.
5. Give future macOS/Linux contributors a clean backend contract that does not require understanding Windows internals.

## Repository responsibilities

| Area | `runseal-labs/rfcs` | `runseal-labs/runseal` |
| --- | --- | --- |
| Product boundary | Defines scope, non-goals, threat model, and platform status | Implements behavior without redefining public scope |
| Policy schema | Defines stable policy shape, profiles, canonical JSON, and hashing rules | Parses, validates, normalizes, and hashes policies |
| Protocol | Defines JSON-RPC methods, CLI vocabulary, events, errors, and result objects | Implements CLI/RPC transport and event streaming |
| Backend contract | Defines platform-neutral capability model and conformance expectations | Provides backend traits, platform implementations, and capability reports |
| Windows backend | Defines required semantics, fail-closed posture, and acceptance tests | Implements the Windows reference backend |
| macOS/Linux | Defines experimental/future status and promotion criteria | Provides skeletons or contributed implementations |
| Audit | Defines event model and redaction requirements | Writes JSONL audit events and enforces redaction |
| Tests | Defines categories and public semantics | Hosts executable black-box and backend conformance tests |

Implementation details may feed back into RFC amendments when they change public behavior. Implementation details that do not change the public contract should stay in the implementation repository.

## Public terminology

All public artifacts must use RunSeal terminology:

- `RunSeal`
- `Execution`
- `SandboxPolicy`
- `SandboxLevel`
- `NetworkPolicy`
- `BackendCapabilities`
- `SandboxBackend`
- `PlatformSandboxPlan`
- `AuditEvent`
- `runtime root`
- `synthetic home`
- `protected subpath`
- `managed proxy`

Do not expose private product names, internal repository names, private issue IDs, private MR IDs, customer names, internal codenames, internal filesystem paths, or chat-only context.

## Windows reference extraction rules

The Windows backend should be extracted as a public reference implementation, not copied as a product-specific subsystem.

Required abstraction steps:

1. Start from the public policy and protocol model, not from an existing product session or agent model.
2. Rebuild module boundaries around RunSeal crates and traits.
3. Map implementation concepts to platform-neutral RunSeal concepts before writing public APIs.
4. Replace private names, paths, logs, metrics, and configuration keys with public RunSeal names.
5. Remove assumptions tied to a specific host application, agent loop, UI, permission dialog, or session store.
6. Add conformance tests for every public behavior that is migrated.
7. Treat unsupported or partially migrated behavior as fail-closed until tests prove support.

Suggested migration map:

| Private implementation concept | Public RunSeal concept |
| --- | --- |
| Host application session | `Execution` / implicit execution session |
| Tool call or command bridge | `execute` request |
| Permission profile | `SandboxPolicy` / `SandboxLevel` |
| Workspace root | `cwd` plus normalized runtime roots |
| Temporary app profile | synthetic home/profile/runtime root |
| Tool-specific allowlists | toolchain read roots and runtime roots |
| Internal event/log stream | `AuditEvent` and execution event stream |
| Internal rollback/checkpoint feature | Out of MVP security boundary unless separately specified |

## Windows backend public shape

The implementation repository should expose the Windows backend through the same public backend trait used by all platforms.

Expected public crates or modules:

- `runseal-core`: policy-independent execution model, IDs, results, events, and errors.
- `runseal-policy`: policy schema, profiles, normalization, validation, canonical JSON, and hashing.
- `runseal-backend`: backend trait, capability model, compiled plan, and conformance helpers.
- `runseal-backend-windows`: Windows reference backend.
- `runseal-audit`: JSONL audit writer and redaction helpers.
- `runseal-protocol`: JSON-RPC request/response and event schema.
- `runseal-cli`: CLI entrypoint and stdio server.

The Windows backend may internally use Windows-specific primitives, but public APIs must not expose backend-private handles or policy knobs such as raw ACLs, SIDs, token attributes, firewall rule names, WFP callouts, internal helper identities, or host-specific profile names.

## Windows backend identity boundary

The Windows reference backend MUST present backend-private identity handling as a single internal sandbox identity at the RunSeal boundary. Public protocol, audit output, capability reports, and conformance fixtures MUST NOT expose account counts, local account names, SIDs, group names, or helper-specific identities.

Windows setup, readiness checks, filesystem enforcement, restricted process setup, network guard/proxy setup, command-runner IPC, cleanup, and diagnostics MUST derive from that same internal sandbox identity. The implementation MUST NOT add compatibility readers, migrations, marker fields, or setup schemas for split sandbox identities unless a later accepted RFC explicitly changes the Windows identity model.

## Redaction requirements

Before pushing public implementation work, contributors must check:

1. No private product names or internal codenames appear in tracked files.
2. No internal repository paths, local user paths, private issue IDs, private MR IDs, or customer names appear in code, tests, docs, comments, or fixtures.
3. No private environment variable names, telemetry stream names, or host application config keys appear in public schemas.
4. No private screenshots, logs, crash dumps, or trace output are committed.
5. Commit messages and pull request text use public RunSeal terminology only.

The implementation repository should include a lightweight redaction check in CI before accepting Windows backend code.

## Implementation handoff sequence

The implementation repository should proceed in this order:

1. Build a minimal CLI/RPC shell that satisfies version and protocol discovery tests.
2. Implement policy parsing, normalization, profiles, canonical JSON, and hashing.
3. Implement explicit `danger-full-access` as local execution with no sandbox guarantee.
4. Add audit event writer and execution event streaming.
5. Add backend trait and capability reporting.
6. Add Windows backend scaffolding that fails closed for unsupported capabilities.
7. Implement Windows filesystem enforcement for `read-only`, `workspace-write`, and `workspace-contained`.
8. Implement synthetic home/profile/runtime roots on Windows.
9. Implement Windows network `disabled` and `proxy` enforcement.
10. Harden cleanup, cancellation, and setup/repair failure handling.
11. Add macOS/Linux backend skeletons that report unsupported or experimental capabilities.

Each step should add or update conformance tests before broadening the implementation.

## Conformance gate

A backend can claim support for a capability only when the implementation repository has executable evidence for that capability.

Minimum evidence for Windows MVP support:

- CLI and JSON-RPC version discovery pass.
- Policy parsing and policy hash tests pass.
- `danger-full-access` is explicit and marked as non-sandboxed local execution.
- `read-only` denies writes.
- `workspace-write` permits workspace writes and denies workspace-external writes.
- `workspace-contained` denies common real-profile and credential path reads unless explicitly allowed.
- Protected workspace metadata writes are denied by default.
- `network.disabled` denies direct outbound access.
- `network.proxy` either enforces proxy-only egress or fails closed as unsupported.
- Audit events are emitted for success, denial, setup failure, cancellation, and final result.

macOS/Linux backends remain experimental or future until they pass the same relevant conformance categories and update RFC-0007/RFC-0009 through a public amendment.

## Non-goals

- This RFC does not define the low-level Windows implementation algorithm.
- This RFC does not require publishing private implementation history.
- This RFC does not make macOS or Linux part of the enterprise security baseline.
- This RFC does not place concrete runtime code in the RFC repository.
- This RFC does not make rollback/checkpoint behavior part of the MVP security boundary.

## Review decision

The RFC repository owns the public protocol and implementation plan. The implementation repository owns concrete code. Windows may be implemented from prior engineering experience only after the implementation has been abstracted into RunSeal concepts, sanitized for public release, and covered by conformance tests.
