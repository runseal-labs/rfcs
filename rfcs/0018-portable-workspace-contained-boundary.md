# RFC-0018: Portable workspace-contained boundary amendment

- Status: Accepted for experimental implementation
- Created: 2026-07-23
- Amends: RFC-0007, RFC-0008, RFC-0009, RFC-0014

## Summary

This amendment makes `workspace-contained` an experimental capability target
for the macOS and Linux backends. It replaces the earlier decision to keep the
level permanently Windows-only.

Windows remains the reference backend and enterprise security baseline.
Portable implementations MUST pass the relevant shared conformance and
adversarial cases before reporting the capability as supported.

No public protocol, policy field, audit field, or capability name changes.

## Portable contained contract

For macOS and Linux `workspace-contained`:

- Ordinary host files MUST be unreadable unless the active policy grants a
  read or write root or the path belongs to the minimum platform execution
  baseline.
- The minimum platform execution baseline MAY expose OS-owned executables,
  dynamic loaders, system libraries, certificate stores, time-zone data,
  resolver configuration, and equivalent files required to start ordinary
  command-line programs.
- The platform baseline MUST NOT include the real user profile, credential
  stores, user session sockets, arbitrary local installation roots, removable
  media, or broad runtime-state directories.
- The normalized workspace and execution-private runtime roots MUST remain
  visible. Write access MUST follow `filesystem.write`; protected subpaths
  MUST remain non-writable.
- Explicit `filesystem.read` and `filesystem.read_only` roots MAY extend the
  default readable set. Implementations MUST normalize these paths and MUST
  fail closed when a requested root cannot be safely represented.
- Symbolic links, path normalization differences, mount ordering, and path
  replacement MUST NOT turn an allowed root into access to an unapproved host
  root.
- The backend MUST NOT fall back to `workspace-write`, unrestricted host
  reads, or local execution when contained setup or execution fails.

The platform execution baseline is an implementation detail. Public
`PlatformSandboxPlan`, error, event, and audit output MUST describe only
policy-level roots and status.

## Linux requirements

The Linux backend SHOULD use an unprivileged mount namespace or an equivalent
kernel-enforced boundary. A contained filesystem view MUST start from a
deny-by-default root and add:

1. a minimal read-only system execution baseline;
2. explicit policy read roots;
3. explicit policy write roots;
4. private runtime roots;
5. fresh process and device views required by the command.

Binding the complete host root read-only does not satisfy
`workspace-contained`, because it preserves unrestricted host reads.

When `network.disabled` is requested, the contained execution MUST also enter
a network boundary that denies direct egress.

## macOS requirements

The macOS backend SHOULD use the native per-process sandbox mechanism or an
equivalent kernel-enforced boundary. The generated profile MUST begin from
default deny and add:

1. the minimum process and system-service operations required for command-line
   execution;
2. the minimum read-only system execution baseline;
3. explicit policy read and write roots;
4. private runtime roots;
5. the requested network behavior.

A profile that grants unrestricted `file-read` and then denies a finite list
of sensitive paths does not satisfy `workspace-contained`.

## Capability status

- macOS remains an experimental backend.
- Linux remains an experimental/community backend.
- A backend MAY report `workspace-contained: supported` only when its runtime
  guard is available and the relevant conformance cases pass on that platform.
- Missing runtime primitives MUST produce a structured unavailable or
  unsupported result and MUST NOT weaken the policy.
- This amendment does not promote either portable backend to the Windows
  enterprise security baseline.

## Conformance requirements

Portable `workspace-contained` promotion requires executable evidence for at
least:

1. executing a platform-baseline command;
2. reading and writing the active workspace as allowed by policy;
3. denying an ordinary external host-file read;
4. denying traversal through symbolic links or equivalent path indirection;
5. denying writes outside explicit write roots;
6. keeping protected workspace metadata non-writable;
7. redirecting home and temporary state into the private runtime root;
8. denying direct egress when `network.disabled` is requested;
9. failing closed when the platform runtime guard is unavailable.

The shared adversarial manifest remains authoritative. Platform-specific
fixtures MAY be skipped only when the result is explicitly reported as an
unsupported fixture rather than a pass.

## Reference signals

The implementation direction is informed by public OS-native agent sandbox
work:

- Anthropic Sandbox Runtime:
  https://github.com/anthropic-experimental/sandbox-runtime
- Microsoft MXC:
  https://github.com/microsoft/mxc

These projects validate deny-by-default platform profiles and minimal
filesystem views as practical local-agent execution boundaries. RunSeal keeps
its own protocol, policy, audit, fail-closed, and managed-network contracts.
