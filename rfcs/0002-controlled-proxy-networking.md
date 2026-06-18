# RFC-0002: Controlled proxy networking

- Status: Accepted for MVP
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal routes enterprise network access through a controlled proxy instead of giving sandboxed skills direct network access or raw credentials. The proxy enforces egress policy, injects approved authentication, records audit events, and keeps real secrets outside the sandbox.

## Motivation

Loose skills and agent-generated commands often need to call enterprise systems: CRM, ticketing, Git hosting, package registries, model gateways, databases, or internal APIs. Letting each skill hold credentials and connect directly creates scattered trust, weak auditability, and high exfiltration risk.

RunSeal turns network access from agent-held secrets into policy-governed proxy capabilities.

## Network modes

### `disabled`

No outbound network access.

```json
{
  "network": { "mode": "disabled" }
}
```

### `proxy`

Default enterprise mode. The sandbox receives only proxy configuration. All outbound traffic must pass through RunSeal-controlled egress.

```json
{
  "network": {
    "mode": "proxy",
    "routes": ["crm-readonly", "github-api"]
  }
}
```

## Controlled proxy responsibilities

MVP responsibilities:

- Provide a managed local HTTP proxy endpoint for sandboxed commands.
- Keep direct egress blocked when `network.mode=proxy`.
- Emit proxy lifecycle and request/denial audit events.
- Keep real credentials outside sandbox process environment.
- Fail closed if proxy mode is requested but the backend cannot restrict direct egress or create the proxy guard.

Post-MVP enterprise responsibilities:

- Domain, CIDR, method, path, and port allowlists.
- SSRF and DNS rebinding protection, including denial of loopback, link-local metadata endpoints, and private CIDRs unless explicitly allowed.
- Credential exchange and header injection.
- mTLS or service identity injection.
- Header/body redaction where configured.
- Rate limits and quotas per execution, skill, policy, or route.
- Support for long-lived agent traffic such as SSE and WebSocket where policy permits.

## Auth profile model

Sandboxed code receives placeholders or no credential at all. The proxy owns real credentials.

```json
{
  "routes": [
    {
      "id": "crm-readonly",
      "match": {
        "scheme": "https",
        "host": "crm.internal.example.com",
        "path_prefix": "/api/customers",
        "methods": ["GET"]
      },
      "auth": {
        "type": "header_injection",
        "profile": "crm-readonly"
      }
    }
  ]
}
```

Injected headers are never visible to the sandbox as environment variables.

## Proxy injection: routing hints, not enforcement boundary

Typical implementation sets environment variables:

```bash
HTTP_PROXY=http://127.0.0.1:<session-port>
HTTPS_PROXY=http://127.0.0.1:<session-port>
NO_PROXY=
```

Proxy environment variables are routing hints, not the enforcement boundary.

The enforcement boundary is:

- Direct-egress denial for sandboxed processes.
- Permission to reach only the managed proxy endpoint.

If the platform backend cannot block direct egress, `network.proxy` is unsupported and MUST fail closed.

Where possible, platform backends should prevent bypassing the proxy by denying direct network namespaces/routes. Unix sockets, named pipes, or loopback-only listeners may be used as backend details.

## Audit examples

```json
{
  "type": "execution.network.request",
  "execution_id": "exec_123",
  "route": "crm-readonly",
  "decision": "allowed",
  "method": "GET",
  "host": "crm.internal.example.com",
  "path": "/api/customers/42",
  "auth_injected": true,
  "status_code": 200,
  "duration_ms": 83
}
```

```json
{
  "type": "execution.network.denied",
  "execution_id": "exec_123",
  "decision": "denied",
  "reason": "host_not_allowlisted",
  "host": "169.254.169.254"
}
```

## Reference signals

- Enterprise egress proxy patterns: default-deny, boundary-level secret injection, structured per-request audit.
- Agent workload requirements: package managers, model APIs, internal APIs, SSE/WebSocket streams.
- Zero-trust principle: credentials should remain at the boundary, not inside untrusted workload memory or environment.

## Decisions for MVP

- RunSeal starts with a generic HTTP proxy guard. Database-specific MITM adapters are outside the MVP and belong in later plugins or enterprise extensions.
- Response body inspection is not required for the open-source MVP. The core records metadata and denial events, while body inspection/redaction remains an extension point.
- Organization-wide route composition is post-MVP. MVP policies reference route IDs and emit auditable policy hashes; later admin policy can constrain the allowed route set without changing the execution protocol.
- If direct egress cannot be blocked by the platform backend, `network.proxy` MUST fail closed with `BACKEND_CAPABILITY_MISSING`. Setting proxy environment variables without direct-egress denial is not sufficient enforcement.
