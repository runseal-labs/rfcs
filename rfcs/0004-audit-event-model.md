# RFC-0004: Audit event model

- Status: Accepted for MVP
- Created: 2026-06-14
- Project: RunSeal

## Summary

RunSeal emits structured audit events for executions, policy decisions, sandbox backend setup, network proxy activity, resource usage, and denials. Auditability is a core product feature, not an afterthought.

## Goals

- Make every execution explainable after the fact.
- Correlate command activity, policy, network requests, and resource usage.
- Keep secrets out of logs by default.
- Support JSONL locally and OpenTelemetry-compatible export paths.
- Let enterprises feed events into SIEM and observability stacks.

## Event envelope

```json
{
  "type": "execution.started",
  "time": "2026-06-14T00:00:00Z",
  "runseal_version": "0.1.0",
  "execution_id": "exec_123",
  "session_id": "sess_123",
  "policy_id": "workspace-write",
  "policy_hash": "sha256:...",
  "policy_epoch": "epoch_01J...",
  "actor": {
    "type": "agent",
    "id": "agent_123"
  },
  "workspace": "/workspace",
  "backend": "linux-bubblewrap"
}
```

## Event families

### Execution lifecycle

- `execution.requested`
- `execution.started`
- `execution.stdout`
- `execution.stderr`
- `execution.finished`
- `execution.failed`
- `execution.killed`

### Policy decisions

- `policy.resolved`
- `policy.allowed`
- `policy.denied`
- `policy.requires_approval`
- `policy.violation`

### Sandbox backend

- `sandbox.prepared`
- `sandbox.setup_failed`
- `sandbox.backend_capability`
- `sandbox.backend_warning`
- `sandbox.cleanup`

### Network proxy

- `execution.network.proxy_ready`
- `execution.network.request`
- `execution.network.denied`
- `execution.network.auth_injected`
- `execution.network.redacted`
- `execution.network.error`

### Resources

- `execution.resource.sample`
- `execution.resource.limit_exceeded`
- `execution.output.truncated`

## Network event example

```json
{
  "type": "execution.network.request",
  "time": "2026-06-14T00:00:01Z",
  "execution_id": "exec_123",
  "route": "crm-readonly",
  "decision": "allowed",
  "method": "GET",
  "scheme": "https",
  "host": "crm.internal.example.com",
  "path": "/api/customers/42",
  "auth_profile": "crm-readonly",
  "auth_injected": true,
  "status_code": 200,
  "request_bytes": 0,
  "response_bytes": 2048,
  "duration_ms": 83
}
```

## Denial example

```json
{
  "type": "policy.denied",
  "time": "2026-06-14T00:00:02Z",
  "execution_id": "exec_123",
  "reason": "write_outside_workspace",
  "path": "/Users/alice/.ssh/config",
  "policy_id": "workspace-write"
}
```

## Secret handling

Audit events must not include raw secrets.

- Authorization, Cookie, proxy credentials, API keys, and configured sensitive headers are redacted.
- Environment values matching secret patterns are not logged.
- Auth profile IDs may be logged; underlying credential material must not be logged.
- Response bodies are not logged by default.

## Storage and export

Initial local format:

```text
.runseal/audit/<session_id>.jsonl
```

Expected export paths:

- stdout JSONL for local debugging.
- file JSONL for replay and support bundles.
- OpenTelemetry logs/metrics for production deployments.
- SIEM integration through collector pipelines.

## Metrics

Metrics should complement audit logs:

- executions started/finished/failed
- policy denies by reason
- network requests by route/decision/status
- resource limit exceedances
- execution duration and output size histograms

## Decisions for MVP

- Store stdout/stderr byte counts and bounded stream chunks in event streams. Durable JSONL audit stores output metadata by default; full output body retention is opt-in because logs can contain secrets.
- Mandatory MVP fields for execution-scoped events are `type`, `time`, `runseal_version`, `execution_id`, `policy_id`, `policy_hash`, `policy_epoch`, `backend`, and a family-specific `decision` or result payload when applicable. OpenTelemetry export can map from these fields later.
- Backend setup events that happen before an execution exists MUST still include `type`, `time`, `runseal_version`, `backend`, a structured error or decision payload, and any resolved policy fields available at the time of failure.
- `execution.failed` events MAY include coarse setup readiness diagnostics such as platform support, setup requirement, broker readiness, and repairability when backend setup is unavailable. They MUST NOT expose backend-private helper paths, account names, SIDs, ACLs, token attributes, firewall/WFP details, or profile paths.
- Retention is local and workspace/session scoped in MVP. Enterprise retention policy is an extension point over the same JSONL event format.
