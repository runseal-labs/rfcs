# RFC-0020: Linux managed proxy network boundary amendment

- Status: Accepted for experimental implementation
- Created: 2026-07-23
- Amends: RFC-0002, RFC-0007, RFC-0014, RFC-0016, RFC-0019

## Summary

This amendment makes `network.proxy` an experimental capability target for
the Linux backend. The capability combines an isolated network namespace, an
execution-local loopback relay, and a managed proxy outside the sandbox
network namespace.

Windows remains the reference backend and enterprise security baseline.
Linux remains an experimental/community backend.

No public protocol, policy field, event field, audit field, or capability name
changes.

## Linux enforcement contract

When a Linux execution requests `network.proxy`:

- The backend MUST place the command and all descendants in an isolated
  network namespace or an equivalent kernel-enforced boundary.
- The isolated network boundary MUST NOT expose a host network interface,
  route, DNS transport, or unrelated host loopback listener.
- The backend MAY expose an execution-local loopback relay for ordinary
  proxy-aware clients. That relay MUST forward only to the managed proxy
  assigned to the execution.
- Any cross-namespace transport between the relay and managed proxy MUST be a
  private backend-controlled IPC endpoint. Other host IPC endpoints capable
  of providing network access MUST remain unavailable.
- The managed proxy MUST authenticate each execution before forwarding a
  request.
- The backend MUST inject the current proxy address and per-execution
  credential after applying caller-provided environment variables. Caller
  values MUST NOT override the enforced proxy configuration.
- Child processes MUST remain inside the same network and process boundary.
- The execution boundary MUST close or mark close-on-exec every caller-owned
  descriptor other than the command's intended standard input, output, and
  error streams. A socket opened before RunSeal starts MUST NOT bypass the
  isolated network boundary.
- HTTP forwarding and `CONNECT` tunneling MAY be supported, but direct TCP,
  UDP, DNS, and host-local socket access MUST NOT become available as a side
  effect.
- Proxy setup, relay setup, authentication, forwarding, namespace setup, or
  sandbox setup failures MUST fail closed.

The network namespace, relay, IPC endpoint, mount layout, and proxy process
layout are backend-private details. Public output reports only the existing
policy, plan, capability, event, audit, and structured error vocabulary.

## Filesystem and IPC interaction

Network isolation MUST account for host IPC endpoints that are represented in
the filesystem. A filesystem view used with `network.disabled` or
`network.proxy` MUST hide ordinary host runtime and temporary socket
directories unless the active policy explicitly grants a safe endpoint.

The managed proxy bridge MAY be mounted into the sandbox as a dedicated
read-only endpoint. The command MUST NOT be able to replace that endpoint or
use it to reach a service other than the current managed proxy.

Temporary filesystem views introduced for network isolation MUST NOT broaden
the active filesystem policy. In particular, replacing a host temporary
directory with a private view does not permit writes outside the policy's
runtime roots.

## Capability status

- Linux remains an experimental/community backend.
- Linux MAY report `network.proxy: supported` only when the native sandbox,
  isolated network boundary, execution-local relay, and managed proxy are
  available.
- The Linux `network_proxy` and `managed_proxy` feature statuses MUST remain
  `experimental` until a later RFC promotes them.
- Runtime unavailability MUST produce a structured unavailable or unsupported
  result and MUST NOT fall back to unmanaged networking.
- This amendment does not promote Linux to the Windows enterprise security
  baseline.

## Events and audit

The existing controlled-proxy event and audit contract applies on Linux:

- readiness MUST be observable before command execution proceeds;
- each allowed or denied proxy request MUST produce a public-safe decision
  record;
- forwarding failures MUST be observable without exposing credentials or
  backend-private paths;
- execution and proxy records MUST share the execution identity needed for
  correlation.

## Conformance requirements

Linux `network.proxy` support requires executable evidence for at least:

1. capability and plan reporting;
2. successful HTTP traffic through the managed proxy;
3. successful `CONNECT` tunneling through the managed proxy;
4. denial of direct external TCP and HTTP egress;
5. denial of direct UDP and DNS egress;
6. denial of direct child-process egress;
7. denial of unrelated loopback listeners;
8. denial of unapproved host Unix sockets or equivalent local IPC endpoints;
9. denial of inherited pre-opened network sockets;
10. resistance to caller-provided proxy environment overrides;
11. per-execution authorization rejection and credential redaction;
12. structured proxy readiness, request, denial, and failure events;
13. fail-closed behavior when the native sandbox, namespace, relay, or proxy
    is unavailable.

The shared adversarial manifest remains authoritative. A result may be skipped
only when the fixture itself is unavailable and the harness reports that
state explicitly; an unsupported implementation is not a passing result.

WSL2 MAY provide experimental Linux conformance evidence when it exposes the
same kernel primitives used by the backend. A WSL2 pass does not by itself
promote every Linux distribution or kernel configuration.
