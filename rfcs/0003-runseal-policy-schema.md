# RFC-0003: RunSeal policy schema

- Status: Accepted for MVP
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal policies define what an execution may read, write, execute, connect to, inherit, consume, and emit. The schema should be stable, explainable, versioned, hashable, and portable across platform backends.

## Goals

- Keep common policy portable across macOS, Linux, and Windows.
- Make policy validation strict and fail-closed.
- Separate stable cross-platform fields from backend-specific extensions.
- Support enterprise audit by making the effective policy serializable and hashable. The policy hash MUST represent the full effective enforcement state, not only the user-supplied policy JSON.
- Provide useful named profiles for agent runtimes.

## Top-level shape

```json
{
  "version": "runseal.policy/v1",
  "id": "workspace-write",
  "description": "Workspace write access with proxy networking",
  "filesystem": {},
  "process": {},
  "network": {},
  "environment": {},
  "resources": {},
  "audit": {},
  "approval": {},
  "backend": {}
}
```

## Filesystem policy

Default: deny outside explicitly allowed roots.

```json
{
  "filesystem": {
    "read": ["$WORKSPACE"],
    "write": ["$WORKSPACE"],
    "read_only": ["$HOME/.cache/pnpm"],
    "deny": ["$HOME/.ssh", "$HOME/.aws", "$HOME/.config/gh"],
    "protect_vcs": true
  }
}
```

Rules:

- Paths must be normalized before enforcement.
- Traversal components are rejected.
- Broad write roots such as `/` or `$HOME` require explicit unsafe mode.
- Deny rules override allow rules.
- The effective path set is recorded in audit events.

## Process policy

```json
{
  "process": {
    "allow_child_processes": true,
    "kill_on_parent_exit": true,
    "max_processes": 256,
    "interactive": false
  }
}
```

## Network policy

```json
{
  "network": {
    "mode": "proxy",
    "routes": ["github-api", "crm-readonly"],
    "direct_allow_hosts": []
  }
}
```

Modes:

- `disabled`
- `proxy`

Enterprise default should be `proxy` or `disabled`; unmanaged `direct` networking is not part of the MVP policy surface.

## Environment policy

```json
{
  "environment": {
    "inherit": "minimal",
    "scrub": [
      "*_TOKEN",
      "*_KEY",
      "*_SECRET",
      "*_PASSWORD",
      "*_AUTHORIZATION",
      "*_COOKIE",
      "AWS_*",
      "OPENAI_API_KEY",
      "ANTHROPIC_API_KEY",
      "AUTHORIZATION",
      "COOKIE",
      "PASSWORD"
    ],
    "set": {
      "CI": "1"
    },
    "proxy": true
  }
}
```

Real enterprise credentials should not be injected into sandbox environment variables. They belong at the proxy boundary.

The default scrub list MUST deny common credential variable shapes, including token, key, secret, password, authorization header, and cookie names. Implementations MAY add more deny patterns, but MUST NOT remove these defaults unless a policy explicitly replaces the scrub list.

When `network.mode` is `proxy`, `environment.proxy` MUST be `true` so the managed proxy route is advertised to sandboxed tools. A policy MUST fail validation if it requests proxy networking while disabling proxy environment injection.

## Resource policy

```json
{
  "resources": {
    "timeout_ms": 600000,
    "memory_bytes": 2147483648,
    "cpu_percent": 200,
    "max_output_bytes": 10485760
  }
}
```

Platform support varies; unsupported hard requirements must fail closed unless the policy explicitly marks them as best-effort.

## Approval policy

```json
{
  "approval": {
    "on_violation": "deny",
    "on_network_route_missing": "request",
    "on_broad_write": "request"
  }
}
```

Approval is a governance decision, not an isolation substitute.

## Backend extension field

Backend-specific knobs live under explicit namespaces:

```json
{
  "backend": {
    "linux": {
      "landlock": "hard_requirement",
      "bubblewrap": { "unshare_net": true }
    },
    "macos": {
      "seatbelt_profile_mode": "generated"
    },
    "windows": {
      "appcontainer": "preferred"
    }
  }
}
```

Portable policy should not require these fields.

## Standard profiles

Initial profile names:

- `read-only`
- `workspace-contained`
- `workspace-write`
- `danger-full-access` (unsafe, explicit only)

`network.mode` is selected independently as `disabled` or `proxy`. Profiles may provide defaults, but the schema keeps filesystem level and network mode as separate dimensions.

## Effective policy hash

The `policy_hash` MUST represent the effective policy, not only the user-supplied policy JSON. Two executions that share a policy hash MUST have identical enforcement behavior.

The effective policy hash MUST incorporate:

- Normalized sandbox level
- Resolved cwd / workspace root
- Canonical read, write, and deny roots
- Protected subpaths
- Network mode
- Proxy route IDs (when mode=proxy)
- Environment inherit, scrub, and set rules
- Runtime root mode
- Backend-required features
- Policy schema version

A backend MAY also include a backend setup generation identifier when enforcement state depends on out-of-band setup changes.

## Decisions for MVP

- JSON is normative for v1. YAML may be accepted by the CLI as a convenience only after parsing into the same canonical JSON policy.
- Policy inheritance/imports are post-MVP. MVP must materialize one effective policy object and policy hash before execution so permissions are never hidden. The effective policy hash composition is defined in the "Effective policy hash" section above.
- Package-manager cache access is represented as explicit `runtime.cacheRoots` / read-only roots in MVP, not a separate policy dimension. Built-in cache aliases can be added later once conformance tests cover them.
