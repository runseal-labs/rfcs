# RFC-0019: macOS managed proxy network boundary amendment

- Status: Accepted for experimental implementation
- Created: 2026-07-23
- Amends: RFC-0002, RFC-0007, RFC-0014, RFC-0016

## Summary

This amendment makes `network.proxy` an experimental capability target for
the macOS backend. The capability combines a per-execution managed proxy with
an OS-native default-deny network boundary that permits only the managed
proxy endpoint.

Windows remains the reference backend and enterprise security baseline.
Linux `network.proxy` remains unsupported until a separate amendment and
conformance evidence define an enforceable implementation.

No public protocol, policy field, event field, audit field, or capability name
changes.

## macOS enforcement contract

When a macOS execution requests `network.proxy`:

- The backend MUST start or attach to a managed proxy before launching the
  command.
- The OS-native sandbox MUST deny direct outbound networking by default and
  MUST allow outbound connections only to the managed proxy endpoint assigned
  to that execution.
- Other loopback listeners, direct external addresses, raw TCP destinations,
  and direct DNS fallback MUST remain unavailable.
- The backend MUST inject the current proxy address and per-execution
  credential after applying caller-provided environment variables. Caller
  values MUST NOT override the enforced proxy configuration.
- Proxy authorization MUST be unique to the execution, MUST be checked by the
  managed proxy, and MUST NOT appear in public errors, events, audit records,
  or command output produced by RunSeal.
- Child processes MUST remain inside the same network boundary.
- HTTP forwarding and `CONNECT` tunneling through the managed proxy MAY be
  supported, but direct sockets MUST NOT become available as a side effect.
- Proxy setup, authentication, forwarding, or native sandbox setup failures
  MUST fail closed with structured public errors.

The proxy endpoint, native sandbox representation, and process-local relay
mechanism are backend-private details. Public capability output reports only
the existing `network.proxy` status and managed-proxy feature status.

## Capability status

- macOS remains an experimental backend.
- macOS MAY report `network.proxy: supported` only when the native network
  boundary and managed proxy are both available.
- The macOS `network_proxy` and `managed_proxy` feature statuses MUST remain
  `experimental` until a later RFC promotes them.
- Runtime unavailability MUST produce a structured unavailable or unsupported
  result and MUST NOT fall back to unmanaged networking.
- This amendment does not promote macOS to the Windows enterprise security
  baseline.
- Linux MUST continue to report `network.proxy: unsupported`.

## Events and audit

The existing controlled-proxy event and audit contract applies on macOS:

- readiness MUST be observable before command execution proceeds;
- each allowed or denied proxy request MUST produce a public-safe decision
  record;
- forwarding failures MUST be observable without exposing credentials or
  backend-private details;
- execution and proxy records MUST share the execution identity needed for
  correlation.

## Conformance requirements

macOS `network.proxy` support requires executable evidence for at least:

1. capability and plan reporting;
2. successful HTTP traffic through the managed proxy;
3. successful `CONNECT` tunneling through the managed proxy;
4. denial of direct external TCP and HTTP egress;
5. denial of direct child-process egress;
6. denial of unapproved loopback listeners;
7. resistance to caller-provided proxy environment overrides;
8. per-execution authorization rejection and credential redaction;
9. structured proxy readiness, request, denial, and failure events;
10. fail-closed behavior when the native sandbox or proxy is unavailable.

The shared adversarial manifest remains authoritative. A result may be skipped
only when the fixture itself is unavailable and the harness reports that
state explicitly; an unsupported implementation is not a passing result.

## Security notes

Allowing the whole loopback interface does not satisfy this amendment. The
native boundary must identify only the current managed proxy endpoint so an
execution cannot reach unrelated host-local services.

Environment variables are routing hints for compatible tools, not the
security boundary. The OS-native deny rule is authoritative when a command
ignores or replaces those variables.
