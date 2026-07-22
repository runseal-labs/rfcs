# RFC-0017: Windows workspace-contained boundary amendment

- Status: Accepted for MVP
- Created: 2026-07-22
- Amends: RFC-0007, RFC-0008, RFC-0010

## Summary

This amendment defines the Windows reference behavior for
`workspace-contained`. It replaces the provisional approach of broadly
readable host storage plus a finite list of denied paths. The current boundary
is default-deny for host reads, with per-execution capabilities for the active
workspace, runtime root, approved toolchain roots, and explicit execution
inputs.

No public protocol, policy field, audit field, or capability name changes.

## Decision

For Windows `workspace-contained`:

- The backend MUST create a native Windows application-container execution
  boundary for the command and its descendants.
- Ordinary host files MUST NOT be readable unless they are represented by an
  active execution capability.
- Active read capabilities MAY cover only the normalized workspace, the
  execution-private runtime root, approved toolchain roots, minimum platform
  dependencies, and explicitly materialized execution inputs.
- Active write capabilities MAY cover only the normalized workspace,
  execution-private runtime root, and explicitly approved writable roots.
- Workspace metadata remains readable but MUST remain non-writable when it is
  a protected subpath.
- Shared host temporary directories MUST NOT become general write roots;
  mutable caches and temporary state MUST be redirected to the private runtime
  root.
- The backend MUST NOT substitute a finite deny-list of profile or credential
  directories for the default-deny read boundary.
- Setup, capability preparation, process creation, network guard preparation,
  or cleanup failure MUST fail closed. The backend MUST NOT fall back to a
  less restrictive sandbox mode or unrestricted local execution.

`workspace-write` retains its existing host-read semantics and its bounded
write behavior. It is not an implementation fallback for
`workspace-contained`.

## Network and process requirements

- `network.disabled` MUST deny direct egress for contained executions.
- `network.proxy` MUST allow only the managed proxy path and MUST deny direct
  egress bypasses.
- Process cleanup MUST remain execution-scoped and terminate the contained
  process tree without sweeping unrelated host processes.
- Public diagnostics and `PlatformSandboxPlan` summaries MUST describe only
  policy-level state. They MUST NOT expose backend-private identities,
  capabilities, path grants, handles, or network-rule details.

## Conformance requirements

Windows `workspace-contained` support requires executable evidence for at
least these cases:

1. Read and write the active workspace.
2. Deny reads of an ordinary external host file, including through a junction
   or symlink.
3. Deny writes outside the active write capabilities.
4. Keep protected workspace metadata non-writable.
5. Keep temporary and cache writes inside the private runtime root.
6. Deny direct egress for `network.disabled` and deny proxy bypasses for
   `network.proxy`.
7. Fail closed when the native boundary or its required setup is unavailable.

## Consequences

The contained mode provides a narrower and more durable host-read boundary
than path-specific deny rules. Tool compatibility is intentionally limited to
the approved runtime and toolchain roots. Callers that require broader host
tool access must explicitly select `workspace-write`; the backend MUST NOT
make that change automatically.
