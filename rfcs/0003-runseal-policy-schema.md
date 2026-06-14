# RFC-0003: RunSeal policy schema

- Status: Draft
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal policies define what an execution may read, write, execute, connect to, inherit, consume, and emit. The schema should be stable, explainable, versioned, hashable, and portable across platform backends.

## Goals

- Keep common policy portable across macOS, Linux, and Windows.
- Make policy validation strict and fail-closed.
- Separate stable cross-platform fields from backend-specific extensions.
- Support enterprise audit by making the effective policy serializable and hashable.
- Provide useful named profiles for agent runtimes.

## Top-level shape

```json
{
  "version": "runseal.policy/v1",
  "id": "workspace-proxy",
  "description": "Workspace write access with controlled proxy networking",
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

- `none`
- `proxy`
- `direct`

Enterprise default should be `proxy` or `none`.

## Environment policy

```json
{
  "environment": {
    "inherit": "minimal",
    "scrub": ["*_TOKEN", "*_KEY", "AWS_*", "OPENAI_API_KEY", "ANTHROPIC_API_KEY"],
    "set": {
      "CI": "1"
    },
    "proxy": true
  }
}
```

Real enterprise credentials should not be injected into sandbox environment variables. They belong at the proxy boundary.

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
- `workspace-write`
- `workspace-proxy`
- `test-runner`
- `package-manager-proxy`
- `full-access` (unsafe, explicit only)

## Open questions

- Should policy be JSON-first with YAML as a convenience, or both normative?
- How should policy inheritance/imports work without hiding effective permissions?
- Should package manager cache access be a first-class concept?
