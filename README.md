# RunSeal RFCs

RunSeal is a reusable Codex-style sandbox layer for AI agents.

It wraps OS-native sandboxing on macOS, Linux, and Windows behind a stable policy protocol. Agent frameworks get safe local command execution without Docker, cloud VMs, or heavyweight infrastructure, while enterprises get controlled proxy networking, credential isolation, and structured audit logs.

RunSeal does **not** aim to be a VM platform, a Docker Desktop replacement, or a cloud multi-tenant sandbox service. It turns local agent execution into a policy-governed, auditable capability.

## Core idea

AI agents increasingly need to run local commands: package managers, tests, linters, code generators, data-processing scripts, internal API clients, and loose community skills. The useful security boundary is not just “ask the user before every command”. The runtime needs a technical boundary that lets low-risk work continue autonomously while forcing sensitive operations through policy.

RunSeal follows the Codex-style model:

- **Sandbox**: the OS-enforced boundary for filesystem, process, network, and resources.
- **Approval/policy**: the governance layer that decides what can run automatically, what is denied, and what requires escalation.
- **Execution**: a single command or tool run inside a seal.
- **Controlled proxy**: the only network path for enterprise use, able to enforce routes, inject auth, redact data, and audit traffic.

## Initial RFC set

1. [RFC-0001: Codex-style OS-native sandbox abstraction](rfcs/0001-codex-style-os-native-sandbox-abstraction.md)
2. [RFC-0002: Controlled proxy networking](rfcs/0002-controlled-proxy-networking.md)
3. [RFC-0003: RunSeal policy schema](rfcs/0003-runseal-policy-schema.md)
4. [RFC-0004: Audit event model](rfcs/0004-audit-event-model.md)
5. [RFC-0005: Workspace/user simplification model](rfcs/0005-workspace-user-simplification-model.md)
6. [RFC-0006: Stable execution protocol](rfcs/0006-stable-execution-protocol.md)
7. [RFC-0007: Platform backend threat model and capability matrix](rfcs/0007-platform-backend-threat-model.md)
8. [RFC-0008: MVP implementation plan](rfcs/0008-mvp-implementation-plan.md)

## CLI vocabulary

The primary CLI verb is `exec`:

```bash
runseal exec --policy workspace-write -- pnpm test
runseal exec --policy workspace-proxy -- python skill.py
```

The protocol method is `execute`; the returned domain object is an `Execution`, not a raw process.

## Non-goals

- No cloud VM sandbox platform.
- No microVM runtime as the default product direction.
- No Docker daemon dependency.
- No direct secret injection into sandboxed processes.
- No unmanaged direct network access as an enterprise default.
- No claim that OS-native sandboxing prevents every kernel-level escape.

## Reference signals

These RFCs intentionally build on public industry signals:

- OpenAI Codex sandboxing: OS-native sandboxing, workspace-write defaults, network approval, and sandbox/approval separation.
- Linux bubblewrap/Flatpak: unprivileged namespace-based isolation, default-limited filesystem and network permissions.
- Enterprise egress proxies such as iron-proxy: default-deny egress, boundary-level secret injection, and per-request structured audit trails.
- OpenTelemetry/structured observability practices for sandbox execution and egress components.

## Status

Draft. This repository is for crystallizing the public protocol and product boundary before implementation work begins in the open.
