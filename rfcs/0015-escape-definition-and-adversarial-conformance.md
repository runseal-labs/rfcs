# RFC-0015: Escape definition and adversarial conformance model

- Status: Draft for review
- Repository: `runseal-labs/rfcs`
- Related: RFC-0003, RFC-0004, RFC-0006, RFC-0007, RFC-0008, RFC-0010, RFC-0012, RFC-0014

## Summary

RunSeal needs a precise public definition of an execution boundary escape. This RFC defines escape categories, severity levels, detection rules, and adversarial conformance requirements for all `SandboxBackend` implementations.

The goal is to make backend promotion depend on adversarial evidence, not only feature correctness. A backend may implement the right API and still be unsafe if adversarial inputs can cross filesystem, process, network, runtime, policy, or audit boundaries.

## Goals

1. Define what counts as an escape in a RunSeal `Execution`.
2. Make escape detection part of the shared conformance model.
3. Require every backend capability claim to pass adversarial tests before promotion.
4. Keep public escape reporting platform-neutral and free of backend-private details.
5. Preserve fail-closed semantics for unsupported or partially enforceable behavior.
6. Define severity levels that can be used by conformance reports and release gates.
7. Establish an implementation-neutral taxonomy for future adversarial test suites.

## Non-goals

This RFC does not define:

1. A specific Windows, macOS, Linux, VM, microVM, container, or kernel mechanism.
2. A complete executable test harness format.
3. A fuzzing engine or property-test framework.
4. A remote execution model.
5. A rollback or checkpoint feature.
6. A claim that RunSeal prevents kernel-level, firmware-level, or administrator-level compromise.
7. Public exposure of backend-private controls such as ACLs, SIDs, token attributes, firewall rule names, WFP callouts, Seatbelt fragments, sandbox-exec flags, namespace flags, seccomp filters, Landlock descriptors, helper identities, or raw OS handles.

## Terms

An **escape** is an observed behavior where a sandboxed `Execution` exceeds the authority granted by its effective `SandboxPolicy` or mutates RunSeal runtime state outside the boundary represented by its `PlatformSandboxPlan`.

An **adversarial case** is a conformance case designed to cross or confuse a RunSeal boundary by using hostile inputs, hostile filesystem state, hostile environment values, hostile timing, or malformed protocol payloads.

An **escape oracle** is the expected decision rule for an adversarial case. The oracle states whether the case must be denied, fail closed, complete without side effects outside policy, or produce a specific public error class.

A **negative side effect** is any persistent mutation outside the allowed policy scope, including file changes, process persistence, network activity, runtime contamination, policy drift, or audit corruption.

## Escape definition

A RunSeal escape occurs when all of the following are true:

1. A RunSeal `Execution` is started or attempted with an effective `SandboxPolicy`.
2. The effective policy, protocol request, or backend capability claim establishes a boundary.
3. An observable side effect or access crosses that boundary.
4. The crossing is not explicitly allowed by the effective policy.

The following outcomes also count as escapes even if no user workspace file is modified:

1. A sandboxed process continues running after RunSeal is required to clean it up.
2. A sandboxed process performs network activity outside the declared `NetworkPolicy`.
3. A backend silently downgrades enforcement instead of returning a structured fail-closed error.
4. A `policy_hash` or policy epoch no longer describes the enforcement state used by the backend.
5. An `AuditEvent` omits or misrepresents a security-relevant deny, failure, setup requirement, or backend decision.
6. Runtime state leaks between executions, sessions, or policy epochs.

## Escape classes

### Filesystem escape

A filesystem escape occurs when an `Execution` reads, writes, creates, deletes, renames, links, or observes filesystem content outside the effective filesystem authority.

Conformance suites SHOULD include adversarial cases for:

```text
parent traversal
absolute path confusion
relative cwd confusion
symlink and junction traversal
symlink swap races
case-folding bypasses
path normalization differences
reserved device names
UNC or network path injection where relevant
pre-existing runtime root tampering
protected subpath bypass
```

A backend MUST NOT claim filesystem policy support unless it proves that allowed roots, denied roots, protected subpaths, runtime roots, synthetic home, profile roots, and temp roots behave according to the effective policy.

### Runtime boundary escape

A runtime boundary escape occurs when RunSeal runtime state for one execution can affect or be affected by another execution outside the effective policy.

Conformance suites SHOULD include adversarial cases for:

```text
execution_id reuse attempts
runtime root pre-creation
runtime root symlink or junction replacement
runtime marker spoofing
partial setup continuation
partial cleanup reuse
cross-session runtime contamination
policy epoch reuse with incompatible enforcement state
```

Runtime root preparation and cleanup MUST fail closed when containment cannot be proven.

### Process escape

A process escape occurs when a sandboxed command creates or preserves process state outside the process authority granted by the effective policy.

Conformance suites SHOULD include adversarial cases for:

```text
background child persistence
orphaned process survival after cancellation
process tree cleanup bypass
shell trampoline process creation
helper process reuse across executions
interactive process misuse when interactive execution is disabled
```

If process cleanup is claimed as supported, the backend MUST prove that sandboxed process trees are controlled according to the effective policy and that cancellation does not leave unmanaged child processes behind.

### Network escape

A network escape occurs when an `Execution` performs direct or indirect network activity outside the effective `NetworkPolicy`.

Conformance suites SHOULD include adversarial cases for:

```text
direct egress while network is disabled
managed proxy bypass
proxy environment override
credential leakage through proxy URLs
DNS fallback leakage
localhost tunnel abuse
pre-opened socket inheritance
route allowlist bypass
```

A backend MUST NOT claim `network.disabled`, `network.proxy`, or managed proxy support unless direct bypass cases fail closed or are denied in a way visible to audit and events.

### Policy escape

A policy escape occurs when the enforcement state differs from the effective `SandboxPolicy` or when a request changes semantics without changing the public policy binding.

Conformance suites SHOULD include adversarial cases for:

```text
unknown policy fields
malformed policy JSON
unsupported fields with non-empty values
policy merge ambiguity
network override drift
stale policy epoch reuse
policy_hash spoofing
policy_hash mismatch between plan, execution, event, and audit
```

Policy normalization MUST be strict. Unknown fields, ambiguous aliases, unsupported non-empty sections, and policy shapes that cannot be represented in the canonical policy MUST be rejected rather than approximated.

### Execution injection escape

An execution injection escape occurs when command, stdin, environment, or encoding handling causes RunSeal to execute a different program or authority path than the request represents.

Conformance suites SHOULD include adversarial cases for:

```text
argv versus shell ambiguity
shell metacharacter injection
program resolution confusion
stdin mode confusion
base64 payload corruption
file-backed stdin path escape
environment variable injection
secret environment key injection
encoding divergence across platforms
```

RunSeal SHOULD prefer argv-array execution. Shell execution MUST be explicit when supported, and shell mode MUST NOT be inferred from command strings.

### Audit escape

An audit escape occurs when security-relevant behavior is not represented in `AuditEvent` records or when public audit data contains backend-private or sensitive material.

Conformance suites SHOULD include adversarial cases for:

```text
secret metadata keys
authorization header leakage
proxy credential leakage
audit path traversal
audit lookup by path
missing deny events
missing fail-closed events
event ordering drift
audit/event policy_hash mismatch
```

Audit output MUST be complete enough to explain security-relevant allow, deny, fail-closed, setup-required, timeout, cancellation, and backend-unavailable outcomes. Audit output MUST remain public-safe.

## Severity levels

RunSeal defines the following escape severity levels:

| Level | Name | Meaning | Backend promotion impact |
| --- | --- | --- | --- |
| S0 | no escape | The adversarial case is denied, fails closed, or completes without boundary violation. | Acceptable. |
| S1 | blocked attempt | A possible escape attempt was blocked and recorded. | Acceptable if audit and event records are correct. |
| S2 | partial boundary violation | A boundary was partially crossed, or a side effect occurred outside policy, but impact was contained. | Feature claim MUST NOT be promoted. |
| S3 | confirmed escape | The execution gained access or produced side effects outside policy, or enforcement silently downgraded. | Backend capability MUST be treated as unsupported until fixed and retested. |
| S4 | audit-opaque confirmed escape | A confirmed escape occurred and audit/events failed to expose the security-relevant behavior. | Release-blocking for any affected backend claim. |

Only S0 and S1 results are acceptable for a backend capability to be promoted.

## Conformance requirements

A backend capability MAY be claimed as supported only when all applicable adversarial cases for that capability produce S0 or S1 results.

A backend capability MAY be claimed as experimental only when:

1. The backend reports the capability as experimental.
2. Known unsupported edges fail closed.
3. Adversarial results are documented.
4. The public protocol, policy, audit, and event shapes remain stable.

A backend capability MUST remain unsupported when any required boundary cannot be enforced, observed, or audited.

A backend MUST NOT silently substitute local unrestricted execution for a sandboxed policy. `danger-full-access` is explicit local execution and is not a sandboxed fallback.

## Adversarial conformance result shape

Future conformance tooling SHOULD report each adversarial case with at least:

```text
case_id
backend_name
backend_status
platform
capability_under_test
sandbox_level
network_mode
expected_oracle
observed_result
severity
policy_hash_present
policy_epoch_present
audit_present
events_present
public_safe_output
```

Conformance reports MUST NOT include local private paths, private account names, raw OS rule names, secrets, credentials, or backend-private handles.

## Backend promotion gates

A backend feature may move from unsupported to experimental or supported only when:

1. The backend reports the capability honestly.
2. Runtime probes show required platform primitives are available when probes are relevant.
3. Implementation exists for the feature.
4. Functional conformance tests pass.
5. Adversarial conformance tests pass with only S0 or S1 results.
6. Unsupported edge cases fail closed.
7. Audit and event output remains complete and public-safe.
8. The public `PlatformSandboxPlan` output remains platform-neutral.
9. Windows reference behavior remains unchanged unless a separate RFC changes it.

## Public/private boundary

Escape definitions and conformance reports may describe behavior in platform-neutral language:

```text
filesystem read denied
filesystem write denied
network direct egress denied
managed proxy required
process cleanup enforced
runtime root rejected
policy epoch transition denied
setup required
backend unavailable
```

They MUST NOT expose backend-private implementation details in public protocol, audit, events, README examples, fixtures, or conformance output.

## Relationship to existing RFCs

- RFC-0003 defines `SandboxPolicy` normalization and `policy_hash`; this RFC adds adversarial policy-bypass cases.
- RFC-0004 defines `AuditEvent`; this RFC adds audit escape requirements.
- RFC-0006 defines execution protocol and event streaming; this RFC requires adversarial outcomes to preserve protocol shape.
- RFC-0007 defines the platform threat model and fail-closed posture; this RFC adds escape categories and severity levels.
- RFC-0008 defines implementation phases and conformance tests; this RFC expands conformance into adversarial categories.
- RFC-0010 defines the RFC/implementation boundary and backend-private redaction; this RFC applies that boundary to escape reporting.
- RFC-0012 defines Windows single identity and policy epoch behavior; this RFC adds epoch replay and drift escape cases.
- RFC-0014 defines portable backend onboarding; this RFC adds adversarial promotion gates for macOS and Linux.

## Acceptance criteria

This RFC is accepted when:

1. Escape categories are used as the top-level taxonomy for adversarial conformance tests.
2. Backend capability promotion requires S0 or S1 results for applicable adversarial cases.
3. S2, S3, and S4 results block affected backend capability claims.
4. Public reports remain platform-neutral and redact backend-private details.
5. README and README.zh-CN link to this RFC.

## Future work

Follow-up RFCs or implementation plans may define:

1. A concrete adversarial conformance harness format.
2. Standard case IDs and fixture layout.
3. Cross-backend benchmark reporting.
4. Fuzzing or property-test integration.
5. Community backend compliance badges.
6. Release gates for publishing backend support claims.
