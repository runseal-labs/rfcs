# RFC-0001: Codex-style OS-native sandbox abstraction

- Status: Accepted for MVP
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal adopts a Codex-style sandbox model: local agent commands run inside OS-enforced boundaries, while policy decides what is allowed, denied, or escalated. RunSeal does not introduce a cloud VM platform or Docker dependency. It wraps platform-native isolation behind a stable protocol for AI agent frameworks.

## Motivation

AI agents need to run local commands without unrestricted host access. A useful runtime must let routine work proceed autonomously while preserving a hard technical boundary around sensitive files, network access, process capabilities, and resource usage.

OpenAI Codex validates this product shape: sandboxing and approvals are separate controls, spawned commands inherit sandbox boundaries, and each OS uses native enforcement. RunSeal packages that model for general agent frameworks.

## Design goals

- Reuse or track Codex-style platform sandbox behavior where feasible.
- Keep the external protocol stable even if platform backends change.
- Prefer OS-native primitives over Docker, cloud VMs, or microVMs.
- Make the default mode useful for local development: workspace write, controlled network, safe environment handling.
- Keep policy explainable for users and auditable for enterprises.

## Platform backend model

RunSeal exposes a common abstraction:

```text
Execution request
  -> policy resolution
  -> seal preparation
  -> platform backend
  -> command execution
  -> event/audit stream
  -> cleanup
```

Expected backend families:

- macOS: Seatbelt profiles through the native sandbox mechanism.
- Linux/WSL2: bubblewrap/user namespaces as the baseline, with Landlock, seccomp, cgroups, and network namespaces where available.
- Windows: native sandbox primitives such as AppContainer, Job Objects, restricted tokens, integrity levels, ACLs, and process mitigation policies.

## Execution vocabulary

RunSeal does not expose raw process semantics as its public model.

- CLI verb: `runseal exec`
- Protocol method: `execute`
- Domain object: `Execution`
- Internal implementation detail: `spawn_process`

Example:

```bash
runseal exec --policy workspace-write -- pnpm test
```

Protocol sketch:

```json
{
  "method": "execute",
  "params": {
    "command": ["pnpm", "test"],
    "cwd": "/workspace",
    "policy": "workspace-write"
  }
}
```

## Policy and approval separation

A sandbox is the enforcement boundary. A policy is the decision layer.

- The sandbox constrains what the command can technically do.
- The policy decides whether an execution can start, needs approval, or must be denied.
- If an approved command runs, it still runs inside the sandbox.

This avoids treating command classification as a substitute for isolation.

## Non-goals

- No cloud VM sandbox service.
- No Docker Desktop replacement.
- No default microVM runtime.
- No promise of complete protection from kernel-level vulnerabilities.
- No direct secret injection into sandboxed processes.

## Decisions for MVP

- RunSeal mirrors Codex-style product semantics, but keeps implementation independent unless an upstream component has an explicit compatible license and a clean integration boundary.
- Backend-specific policy is not exposed in the public API for MVP. Advanced backend hints may exist only under explicit backend namespaces and must not be required by portable policies.
- A platform is supported only when it can enforce filesystem policy, network `disabled`, structured setup failures, process cleanup, and audit events for the declared sandbox levels. Partial backends must report unsupported capabilities and fail closed.
