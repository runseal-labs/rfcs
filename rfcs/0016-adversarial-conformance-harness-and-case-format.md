# RFC-0016: Adversarial conformance harness and case format

- Status: Draft for review
- Repository: `runseal-labs/rfcs`
- Related: RFC-0003, RFC-0004, RFC-0006, RFC-0007, RFC-0008, RFC-0010, RFC-0012, RFC-0014, RFC-0015

## Summary

RFC-0015 defines what an escape is. This RFC defines how RunSeal should describe, execute, classify, and report adversarial conformance cases.

The harness described here is implementation-neutral. It does not require a specific Rust module layout, test runner, fuzzing engine, container runtime, VM, or OS sandbox primitive. It defines the public case model and result model that backend implementations must satisfy before capability promotion.

The core idea is:

```text
case manifest
-> isolated fixture setup
-> RunSeal execution
-> oracle evaluation
-> side-effect inspection
-> audit/event inspection
-> public-safe report
```

## Goals

1. Define a durable manifest format for adversarial conformance cases.
2. Define the minimum runner lifecycle for preparing, executing, inspecting, and cleaning cases.
3. Define oracle kinds for deny, fail-closed, no-side-effect, audit, event, timeout, cancellation, and backend-unavailable expectations.
4. Define a public-safe result schema that can be used by CI, backend maintainers, and community compliance reports.
5. Support platform-specific cases without leaking platform-private implementation details.
6. Allow feature-gated cases for Windows reference, macOS experimental, Linux community, and future backends.
7. Make S0/S1/S2/S3/S4 severity from RFC-0015 computable from case outcomes.

## Non-goals

This RFC does not define:

1. The internal implementation language of the harness.
2. A complete fuzzing or property-testing framework.
3. A mandatory dependency on OCI, Docker, VMs, microVMs, Kubernetes, Flatpak, bubblewrap, Landlock, seccomp, or any platform-specific tool.
4. A benchmark score for LLM escape capability.
5. A public exploit database.
6. A promise that passing adversarial conformance proves absence of all possible OS or kernel escapes.
7. A stable path layout inside the implementation repository.

## Case taxonomy

Every adversarial case MUST belong to exactly one primary escape class from RFC-0015:

```text
filesystem
runtime
process
network
policy
execution_injection
audit
```

A case MAY declare secondary classes when one input stresses multiple boundaries. The primary class controls default reporting and promotion gating.

Case IDs SHOULD use this format:

```text
adv.<class>.<short-name>.v<case-version>
```

Examples:

```text
adv.filesystem.symlink-parent-traversal.v1
adv.runtime.precreated-runtime-root.v1
adv.process.orphan-child-after-cancel.v1
adv.network.proxy-env-override.v1
adv.policy.unknown-field-nonempty.v1
adv.execution_injection.argv-shell-metacharacters.v1
adv.audit.secret-metadata-redaction.v1
```

Case IDs MUST be stable once accepted. A material semantic change requires a new case version.

## Manifest format

Adversarial cases SHOULD be represented as JSON or YAML manifests. The repository may choose either storage format, but the semantic fields below define the public model.

A minimal manifest has this shape:

```json
{
  "schema_version": "runseal.adversarial-case/v1",
  "case_id": "adv.filesystem.symlink-parent-traversal.v1",
  "title": "Symlink parent traversal must not escape workspace",
  "primary_class": "filesystem",
  "secondary_classes": ["runtime"],
  "capabilities_under_test": ["filesystem_policy", "runtime_roots"],
  "platforms": ["windows", "macos", "linux"],
  "backend_status": ["reference", "experimental", "future-community"],
  "sandbox_level": "workspace-contained",
  "network_mode": "disabled",
  "fixtures": [],
  "request": {
    "method": "execute",
    "command": ["python", "attack.py"],
    "stdin": {"mode": "empty"},
    "env": {},
    "metadata": {}
  },
  "oracle": {
    "expected_result": "deny_or_fail_closed",
    "max_severity": "S1",
    "forbidden_side_effects": [],
    "audit": {"required": true},
    "events": {"required": true}
  }
}
```

## Required manifest fields

### `schema_version`

MUST be `runseal.adversarial-case/v1` for this RFC.

### `case_id`

MUST be unique and stable.

### `title`

MUST be short, public-safe, and implementation-neutral.

### `primary_class`

MUST be one of:

```text
filesystem
runtime
process
network
policy
execution_injection
audit
```

### `capabilities_under_test`

MUST name the backend capabilities whose promotion is affected by this case.

Recommended values include:

```text
filesystem_policy
runtime_roots
runtime_environment
process_isolation
process_cleanup
direct_network_deny
network_disabled
network_proxy
managed_proxy
policy_epoch
setup_readiness
stdin_bytes
stdin_file
audit_jsonl
resource_limits
```

The names SHOULD match public backend capability names.

### `platforms`

MUST list the platforms for which the case is applicable.

Allowed public platform values are:

```text
windows
macos
linux
other
```

A case that applies to every backend SHOULD list every known platform rather than using a wildcard.

### `backend_status`

MUST list backend status values for which the case is expected to run.

Allowed public values are:

```text
reference
experimental
future-community
local-baseline
scaffold
```

### `sandbox_level`

MUST be a public `SandboxLevel` value unless the case is a malformed-policy case.

Allowed values are:

```text
read-only
workspace-contained
workspace-write
danger-full-access
malformed
```

### `network_mode`

MUST be a public `NetworkPolicy` mode unless the case is a malformed-policy case.

Allowed values are:

```text
disabled
proxy
malformed
```

### `request`

MUST describe the RunSeal operation under test in public protocol terms. It MUST NOT require backend-private handles or platform-private control fields.

### `oracle`

MUST define the expected security outcome.

## Optional manifest fields

A case MAY define:

```text
secondary_classes
risk_summary
references
fixtures
preconditions
setup_steps
inspection_steps
cleanup_steps
timeout_ms
skip_if
xfail_if
negative_side_effects
public_report_labels
```

Optional fields MUST remain public-safe.

## Fixture model

Fixtures describe the filesystem, process, environment, or protocol state required before executing a case.

The fixture model SHOULD support these fixture kinds:

```text
file
directory
symlink
hardlink
junction
readonly_file
executable_script
environment
preexisting_runtime_root
background_process
network_listener
malformed_request
```

Harness implementations MAY support fewer fixture kinds initially. Unsupported fixture kinds MUST cause the case to be skipped with a structured `skipped_unsupported_fixture` result, not silently ignored.

Fixtures MUST NOT contain secrets, private paths, local usernames, customer names, internal repository names, or system-specific identifiers.

## Runner lifecycle

The harness SHOULD execute each case with this lifecycle:

```text
load manifest
validate manifest
select backend
check platform applicability
check capability applicability
create isolated test workspace
materialize fixtures
record pre-state
run RunSeal request
collect response
collect audit events
collect execution events
inspect side effects
compute severity
cleanup workspace and runtime state
emit public-safe result
```

The runner MUST treat setup failures as first-class outcomes. A setup failure is acceptable only when the oracle allows `fail_closed` or `backend_unavailable` and no negative side effects occurred.

The runner MUST NOT use cleanup success as proof that no escape occurred. It must inspect side effects before or independently of cleanup when the case requires it.

## Oracle kinds

The `oracle.expected_result` field MUST use one of these values:

```text
allow_no_side_effect_outside_policy
deny
fail_closed
deny_or_fail_closed
backend_unavailable
timeout
cancelled
audit_redacted
event_emitted
policy_rejected
```

### `allow_no_side_effect_outside_policy`

The execution may succeed, but the harness MUST verify that no forbidden side effects occurred outside the effective policy.

### `deny`

The request must be rejected by policy, validation, or backend enforcement.

### `fail_closed`

The request must fail before unsafe execution or before an unsupported enforcement downgrade.

### `deny_or_fail_closed`

Either explicit denial or fail-closed backend refusal is acceptable.

### `backend_unavailable`

The backend may report unavailable primitives or required setup. This is acceptable only when the capability is not claimed supported for the current backend.

### `timeout`

The execution must stop according to the effective timeout policy and must not leave unmanaged process or runtime state behind.

### `cancelled`

The execution must stop after cancellation and must not leave unmanaged process or runtime state behind.

### `audit_redacted`

The main expected outcome is that audit data redacts sensitive or backend-private material.

### `event_emitted`

The main expected outcome is a specific public event class or state transition.

### `policy_rejected`

The malformed or unsupported policy must be rejected before execution.

## Side-effect inspection

A case MAY declare forbidden side effects. The harness SHOULD support these public inspection categories:

```text
file_exists
file_not_exists
file_content_equals
file_content_not_contains
path_not_accessible
path_not_modified
process_not_running
network_connection_absent
audit_contains_event_type
audit_not_contains_secret
event_contains_type
event_not_contains_private_detail
policy_hash_consistent
policy_epoch_consistent
```

The harness MUST evaluate side effects against public paths inside the isolated test workspace or against public logical observations. It MUST NOT require exposing backend-private state to the public result.

## Severity computation

Severity MUST be computed using RFC-0015.

A suggested mapping is:

| Observation | Severity |
| --- | --- |
| Expected deny/fail-closed occurred; no forbidden side effects; audit/events correct | S0 |
| Escape attempt was blocked and represented in audit/events | S1 |
| A limited forbidden side effect occurred but did not grant broader access | S2 |
| A forbidden read/write/network/process/policy action succeeded | S3 |
| A confirmed escape occurred and audit/events omitted or misrepresented it | S4 |

The case manifest MUST declare `oracle.max_severity`. For backend capability promotion, `max_severity` MUST be `S0` or `S1`.

If the observed severity is greater than `oracle.max_severity`, the case fails.

## Result schema

The harness SHOULD emit one JSON object per case result with this shape:

```json
{
  "schema_version": "runseal.adversarial-result/v1",
  "case_id": "adv.filesystem.symlink-parent-traversal.v1",
  "backend_name": "runseal-windows-reference",
  "backend_status": "reference",
  "platform": "windows",
  "capabilities_under_test": ["filesystem_policy", "runtime_roots"],
  "sandbox_level": "workspace-contained",
  "network_mode": "disabled",
  "expected_result": "deny_or_fail_closed",
  "observed_result": "deny",
  "severity": "S1",
  "passed": true,
  "skipped": false,
  "skip_reason": null,
  "policy_hash_present": true,
  "policy_epoch_present": true,
  "audit_present": true,
  "events_present": true,
  "public_safe_output": true
}
```

The result MUST NOT include raw private paths, usernames, security identifiers, ACLs, firewall rule names, WFP callouts, Seatbelt fragments, namespace flags, seccomp filters, Landlock descriptors, helper identities, raw OS handles, secrets, credentials, cookies, tokens, or authorization headers.

## Case status values

The harness SHOULD classify each case as one of:

```text
passed
failed
skipped
xfailed
invalid_case
unsupported_fixture
harness_error
```

`skipped` MUST include a public-safe reason. A supported capability MUST NOT rely on skips for applicable adversarial cases.

`xfailed` MAY be used only for draft or experimental cases. A case marked `xfailed` MUST NOT count toward supported capability promotion.

## Capability promotion matrix

Backend capability promotion SHOULD be computed from case results:

```text
supported: all applicable required cases passed with S0/S1
experimental: required stable cases pass, known draft cases may xfail, unsupported edges fail closed
unsupported: missing implementation, missing primitive, or any required S2/S3/S4 result
unavailable: runtime probe shows required primitive unavailable
requires_setup: setup is required before conformance can run
```

A backend MUST NOT claim supported if any applicable required adversarial case is skipped, xfailed, invalid, or reports S2/S3/S4.

## Required initial case families

The first adversarial suite SHOULD define at least these families.

### Filesystem

```text
adv.filesystem.parent-traversal.v1
adv.filesystem.symlink-parent-traversal.v1
adv.filesystem.preexisting-symlinked-runtime-root.v1
adv.filesystem.protected-subpath-write.v1
adv.filesystem.external-write-from-workspace-write.v1
adv.filesystem.external-read-from-workspace-contained.v1
```

### Runtime

```text
adv.runtime.precreated-runtime-root.v1
adv.runtime.runtime-marker-spoof.v1
adv.runtime.execution-id-reuse.v1
adv.runtime.cleanup-partial-failure.v1
adv.runtime.cross-execution-contamination.v1
```

### Process

```text
adv.process.orphan-child-after-cancel.v1
adv.process.background-daemon-after-timeout.v1
adv.process.shell-trampoline-child.v1
adv.process.interactive-disabled.v1
```

### Network

```text
adv.network.direct-egress-disabled.v1
adv.network.proxy-env-override.v1
adv.network.proxy-credential-redaction.v1
adv.network.dns-leak-disabled.v1
adv.network.loopback-tunnel-bypass.v1
```

### Policy

```text
adv.policy.unknown-top-level-field.v1
adv.policy.unsupported-nonempty-section.v1
adv.policy.network-override-hash-drift.v1
adv.policy.stale-policy-epoch.v1
adv.policy.malformed-json.v1
```

### Execution injection

```text
adv.execution_injection.argv-shell-metacharacters.v1
adv.execution_injection.stdin-file-outside-cwd.v1
adv.execution_injection.invalid-base64-stdin.v1
adv.execution_injection.secret-env-key.v1
adv.execution_injection.program-resolution-confusion.v1
```

### Audit

```text
adv.audit.secret-metadata-redaction.v1
adv.audit.audit-path-traversal.v1
adv.audit.missing-deny-event.v1
adv.audit.policy-hash-consistency.v1
adv.audit.backend-private-redaction.v1
```

## Platform-specific cases

Platform-specific cases are allowed, but their public reports MUST remain platform-neutral.

A platform-specific case may use internal setup to exercise a platform primitive, but the public case title, manifest, oracle, and result should describe the behavior, not the mechanism.

Acceptable public descriptions:

```text
process cleanup bypass attempt
runtime root containment bypass attempt
managed proxy bypass attempt
filesystem deny bypass attempt
```

Unacceptable public descriptions:

```text
raw ACL mutation detail
specific SID value
specific WFP rule name
Seatbelt profile fragment
bubblewrap argv string
seccomp filter bytecode
Landlock file descriptor detail
```

## CI and release gates

CI SHOULD run adversarial conformance in tiers:

```text
Tier 0: manifest validation and policy-only cases
Tier 1: portable no-privilege cases
Tier 2: backend-specific supported-feature cases
Tier 3: setup-required or privileged environment cases
Tier 4: stress, race, and long-running cases
```

A release that changes protocol, policy, audit, service mode, backend claims, runtime roots, process cleanup, filesystem enforcement, or network enforcement SHOULD run all applicable Tier 0, Tier 1, and Tier 2 cases.

Tier 3 and Tier 4 may run in scheduled or backend-maintainer CI when they need special host setup.

## Relationship to existing RFCs

- RFC-0015 defines escape categories and severity. This RFC defines the case and result model used to test them.
- RFC-0014 defines backend onboarding. This RFC defines the adversarial evidence required for backend capability promotion.
- RFC-0007 defines fail-closed platform posture. This RFC makes fail-closed behavior testable.
- RFC-0006 defines JSON-RPC execution protocol. This RFC constrains requests and results to public protocol terms.
- RFC-0004 defines `AuditEvent`. This RFC requires audit inspection for adversarial outcomes.
- RFC-0003 defines policy normalization and `policy_hash`. This RFC requires policy-hash consistency checks in adversarial results.
- RFC-0010 defines the public/private repository boundary and redaction rules. This RFC applies those rules to manifests, fixtures, and reports.

## Acceptance criteria

This RFC is accepted when:

1. The repository can describe adversarial cases using this manifest model.
2. RFC-0015 severity levels can be computed from case results.
3. Backend capability promotion can be gated on applicable case results.
4. Public result output remains platform-neutral and redacted.
5. README and README.zh-CN link to this RFC.

## Future work

Follow-up implementation work may add:

1. A `tests/adversarial` directory in the implementation repository.
2. JSON Schema files for case manifests and result reports.
3. A Rust or standalone harness runner.
4. CI matrix integration for Windows reference, macOS experimental, and Linux community backends.
5. Optional importers for external sandbox case corpora.
6. A public compliance report format for community-maintained backends.
