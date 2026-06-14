# RFC-0002: Controlled proxy networking

- Status: Draft
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal routes enterprise network access through a controlled proxy instead of giving sandboxed skills direct network access or raw credentials. The proxy enforces egress policy, injects approved authentication, records audit events, and keeps real secrets outside the sandbox.

## Motivation

Loose skills and agent-generated commands often need to call enterprise systems: CRM, ticketing, Git hosting, package registries, model gateways, databases, or internal APIs. Letting each skill hold credentials and connect directly creates scattered trust, weak auditability, and high exfiltration risk.

RunSeal turns network access from agent-held secrets into policy-governed proxy capabilities.

## Network modes

### `none`

No outbound network access.

```json
{
  "network": { "mode": "none" }
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

### `direct`

Explicit development/debug mode. Direct network should not be the enterprise default.

```json
{
  "network": {
    "mode": "direct",
    "allow_hosts": ["api.github.com"]
  }
}
```

## Controlled proxy responsibilities

- Default-deny egress.
- Domain, CIDR, method, path, and port allowlists.
- SSRF and DNS rebinding protection, including denial of loopback, link-local metadata endpoints, and private CIDRs unless explicitly allowed.
- Credential exchange and header injection.
- mTLS or service identity injection.
- Request and response metadata audit.
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

## Proxy injection into sandbox

Typical implementation:

```bash
HTTP_PROXY=http://127.0.0.1:<session-port>
HTTPS_PROXY=http://127.0.0.1:<session-port>
NO_PROXY=
```

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

## Open questions

- Should RunSeal define a generic HTTP-only proxy first, then add database-specific MITM adapters later?
- Should response body inspection be in scope for the open-source core or left to enterprise plugins?
- How should proxy route definitions compose with organization-wide policy?
