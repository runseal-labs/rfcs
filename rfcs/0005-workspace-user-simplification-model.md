# RFC-0005: Workspace/user simplification model

- Status: Draft
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal should hide platform-specific user, UID, ACL, and namespace complexity behind a simple workspace model. For local agent use, the developer should experience one workspace and one effective user, even when a backend internally uses user namespaces, restricted tokens, AppContainers, ACLs, or helper processes.

## Motivation

Many sandbox systems leak isolation implementation details into the developer workflow:

- Files created by the sandbox have surprising ownership.
- Package caches stop working because UID mappings differ.
- Git operations fail because credentials and config are mounted inconsistently.
- Node/Python dependency directories become unwritable.
- Users need to reason about host user versus sandbox user.

RunSeal should simplify this. The external contract is a workspace with policy-governed read/write behavior. Backend-specific identity mechanisms are implementation details.

## Goals

- Preserve host-developer ergonomics for local workspaces.
- Avoid exposing two-user models unless required for advanced isolation.
- Ensure files created in allowed write roots remain usable by the host user.
- Keep sensitive home-directory material out of the sandbox by default.
- Make cache/toolchain access explicit and policy-governed.

## Conceptual model

```text
Host user
  -> RunSeal execution request
  -> workspace seal
  -> backend-specific identity/isolation
  -> files appear usable to host user
```

The policy talks about workspaces and capabilities, not platform UID details.

## Workspace roots

RunSeal recognizes these logical roots:

- `$WORKSPACE`: project directory.
- `$RUNSEAL_TMP`: per-execution temporary directory.
- `$RUNSEAL_CACHE`: controlled cache area.
- `$RUNSEAL_HOME`: synthetic minimal home directory, when needed.

Example:

```json
{
  "filesystem": {
    "read": ["$WORKSPACE"],
    "write": ["$WORKSPACE", "$RUNSEAL_TMP"],
    "read_only": ["$RUNSEAL_CACHE/pnpm"]
  }
}
```

## Synthetic home

RunSeal should avoid mounting the real home directory by default. Instead, it can construct a synthetic home with only explicitly allowed files:

- minimal shell config if needed
- package manager config without secrets
- Git identity if policy permits
- tool caches by explicit mount

Sensitive locations are denied by default:

- `~/.ssh`
- `~/.aws`
- `~/.kube`
- `~/.config/gh`
- cloud and model provider credential files

## File ownership behavior

Desired external behavior:

- Files written in `$WORKSPACE` are editable by the host user.
- Temporary files are cleaned up unless retained by policy.
- Backend UID/GID mappings do not leak into user-visible artifacts.

Backend strategies may include:

- Same-user execution plus OS sandbox restrictions.
- User namespaces with ID mapping.
- Post-execution ownership repair where safe.
- Windows ACL grants with rollback.
- AppContainer capability grants scoped to allowed paths.

## Cache strategy

Toolchain caches can be valuable but risky. RunSeal should make cache access explicit:

```json
{
  "caches": {
    "pnpm": "read_only",
    "pip": "read_write_scoped",
    "cargo": "read_only"
  }
}
```

Cache mounts should never imply broad home access.

## Git and package managers

Common local-agent workflows need Git and package managers. RunSeal should support them through explicit capabilities:

- Git read-only config without exposing SSH keys by default.
- Git network access through controlled proxy routes.
- Package registry access through controlled proxy routes.
- Credential injection at proxy boundary, not inside Git/npm/pip environment variables when possible.

## Non-goals

- No promise that every tool sees an identical environment to the unsandboxed host.
- No automatic mounting of the full home directory.
- No hidden credential forwarding.
- No exposing backend user complexity in the default API.

## Open questions

- Which caches deserve built-in first-class names in v1?
- How should RunSeal handle tools that require real-home assumptions?
- Should same-user sandboxing be preferred over user namespaces when platform primitives are strong enough?
