# RFC-0007: Platform backend threat model and capability matrix

## Summary

RunSeal prioritizes OS-native local sandboxing on Windows and macOS before adding Linux support. The product contract is platform-neutral, but each backend has different enforcement primitives and known gaps. This RFC defines the initial threat model, backend capability matrix, fail-closed requirements, and platform-specific MVP boundaries.

The implementation should expose one stable policy model to clients. It must not leak Windows ACL/SID details, macOS Seatbelt profile details, or future Linux namespace details into public client APIs.

## Goals

1. Treat Windows and macOS as first-class MVP platforms.
2. Keep Linux as an explicit future backend behind the same abstraction.
3. Define what the sandbox is intended to prevent and what it does not claim to prevent.
4. Require fail-closed behavior when a requested policy cannot be enforced.
5. Align file-system and network policy semantics across platforms even when backend mechanisms differ.

## Non-goals

- No VM, microVM, or container daemon dependency in the MVP.
- No claim of kernel exploit resistance.
- No transparent sandboxing of every helper process created by the host application.
- No browser, keychain, credential vault, or desktop automation isolation guarantee beyond explicit file/network policy.
- No domain allowlist or denylist enforcement in the first proxy iteration.

## Threat model

RunSeal is designed to reduce risk from untrusted or semi-trusted agent commands running on a local developer or enterprise workstation.

In scope:

- Accidental or malicious writes outside the allowed workspace/runtime roots.
- Destructive writes to protected workspace metadata such as `.git` and agent configuration directories.
- Direct network access when policy disables networking.
- Direct egress bypass when policy requires managed proxy egress.
- Environment-variable leakage through inherited process state.
- Persistent pollution of the real user profile through HOME/AppData/tmp paths.
- Ambiguous partial setup states that look sandboxed but are not actually enforceable.

Out of scope:

- Kernel privilege escalation.
- Attacks against trusted host-side helpers or the parent application.
- Exfiltration through already-authorized read roots.
- Side channels such as timing, CPU cache behavior, UI automation, clipboard, screen capture, or accessibility APIs unless explicitly restricted by a future backend.
- Network governance finer than disabled/proxy in the MVP.
- Complete automatic discovery of every local language runtime and package-manager cache.

## Common policy contract

The public policy surface is shared across all platforms:

- Sandbox levels: `read-only`, `workspace-contained`, `workspace-write`, `danger-full-access`.
- Network modes: `disabled`, `proxy`.
- Runtime roots: synthetic HOME/profile/tmp roots for sandboxed commands.
- Toolchain roots: explicit read allowlist for executable and runtime dependencies.
- Protected subpaths: workspace metadata that remains non-writable even when a broader writable root covers the workspace.

`danger-full-access` is local execution. It does not claim filesystem or network hard boundaries.

All other sandbox levels must use the platform backend when supported. If RunSeal cannot enforce the requested policy, it must fail closed with a structured error.

## Capability matrix

| Capability | Windows MVP | macOS MVP | Linux future |
| --- | --- | --- | --- |
| Backend priority | First-class | First-class | Deferred |
| Execution isolation | Restricted local process | Seatbelt-wrapped process | bubblewrap / namespaces |
| `read-only` | No writes | No writes | Read-only binds |
| `workspace-contained` | Workspace/runtime/toolchain/platform roots only | Workspace/runtime/toolchain/platform roots only | Explicit ro/rw binds |
| `workspace-write` | Default readable surface, workspace/runtime writable | Default readable surface, workspace/runtime/temp writable | Default readable surface, workspace/runtime writable |
| `danger-full-access` | Local execution | Local execution | Local execution |
| Synthetic HOME/profile | Required | Required | Required |
| Protected workspace metadata | Required | Required | Required |
| Network `disabled` | Required | Required | Required |
| Network `proxy` | Proxy-only egress required | Proxy-only egress required | Future |
| Domain rules | Not MVP | Not MVP | Future |
| Fail closed on setup failure | Required | Required | Required |

## Windows backend model

Windows is an MVP platform. The implementation should use a restricted local execution identity and OS policy controls to enforce file and network boundaries.

Required behavior:

- Use one low-privilege sandbox identity/group model for sandboxed commands.
- Use ACLs and restricted process tokens to enforce read/write roots.
- Redirect HOME, AppData, LocalAppData, Temp, and Tmp to the sandbox runtime root.
- Protect the real user profile and common credential directories from sandboxed reads in `workspace-contained`.
- Deny writes outside allowed roots in `read-only`, `workspace-contained`, and `workspace-write`.
- Keep workspace metadata such as `.git`, `.agents`, and `.codex` non-writable by default.
- Enforce `network.disabled` through OS-level network restrictions.
- Enforce `network.proxy` as proxy-only egress: sandboxed processes may reach the managed proxy endpoint but must not bypass it with direct outbound connections.
- Treat helper/setup/repair failures as hard failures, not soft warnings.

Known gaps and constraints:

- Local toolchain compatibility depends on explicit read roots and cache roots.
- Some developer tools may require additional runtime directories; these should be added through explicit policy expansion, not broad profile reads.
- The backend does not defend against administrator-level tampering or kernel compromise.

## macOS backend model

macOS is an MVP platform. The implementation should use `/usr/bin/sandbox-exec` with generated Seatbelt profiles and dynamic path parameters.

Required behavior:

- Use the fixed absolute path `/usr/bin/sandbox-exec`.
- Generate a per-execution Seatbelt profile from the platform-neutral policy.
- Pass dynamic paths through `-DKEY=value` style parameters or an equivalent escaping-safe mechanism.
- Canonicalize paths before policy compilation where possible, while preserving safe absolute fallbacks for paths that do not exist yet.
- Model raw and canonical paths for `/tmp`, `/var`, and `/private/var` style symlinked roots.
- Use literal-only reads for path traversal nodes when needed, avoiding broad subtree reads.
- Keep `.git`, `.agents`, and `.codex` non-writable inside the workspace by default.
- In `workspace-contained`, avoid global `file-read*`; allow only workspace/runtime/toolchain/platform roots needed for execution.
- In `workspace-write`, allow broad reads but limit writes to workspace/runtime/temp plus explicitly allowed roots.
- Enforce `network.disabled` by omitting outbound network permissions.
- Enforce `network.proxy` by starting a managed local HTTP proxy, injecting proxy environment variables, and only allowing the sandboxed process to reach that loopback proxy endpoint.
- Bind proxy lifetime to the command/policy guard; when the command ends or is cancelled, proxy listeners and forwarding tasks must be stopped.

Known gaps and constraints:

- `sandbox-exec` is deprecated on modern macOS but remains the practical OS-native mechanism for dynamic CLI process sandboxing.
- App Sandbox entitlements are not an MVP replacement for arbitrary per-command CLI execution.
- Some tools require additional Seatbelt allowances for sysctl, Mach lookup, pty, IOKit, trustd/networkd, or language-runtime shared memory.
- The backend does not claim to isolate Keychain, UI automation, accessibility APIs, or side channels.

## Linux future backend model

Linux is intentionally deferred for the MVP, but the policy model should remain compatible with a later bubblewrap/namespace backend.

Expected future mapping:

- `readRoots` and `writeRoots` map to read-only and writable bind mounts.
- Deny roots are omitted or shadowed with empty dirs/tmpfs where needed.
- `network.disabled` maps to network namespace isolation.
- Resource policy can later map to cgroups/seccomp where appropriate.
- Toolchain compatibility should remain explicit rather than defaulting to host-wide access.

## Fail-closed requirements

The backend must refuse execution when:

- The platform backend is unavailable for a non-`danger-full-access` request.
- The backend cannot enforce a requested sandbox level.
- Network proxy mode is requested but no managed proxy endpoint can be created or restricted.
- Required ACL, token, firewall, WFP, Seatbelt, or path-translation setup fails.
- A policy includes unknown mandatory fields.
- A requested path cannot be safely normalized or represented in the backend plan.

Structured failures should include:

- `code`
- `source`
- `platform`
- `retryable`
- `terminal`
- human-readable `message`

Hard policy denials should be terminal and non-retryable by default.

## Acceptance criteria

1. Windows and macOS can report backend capabilities through the stable protocol.
2. The same sandbox level and network mode produce the same product semantics on both platforms.
3. Unsupported Linux sandbox execution returns a clear fail-closed result in the MVP.
4. Proxy mode cannot silently fall back to unrestricted direct egress.
5. `danger-full-access` is explicitly represented as local execution with no sandbox guarantee.
6. Public docs and client APIs describe platform-neutral policy semantics instead of backend-private implementation knobs.
