# RFC-0012: Windows single identity and global policy epoch model

- Status: Accepted for MVP
- Created: 2026-06-20
- Project: RunSeal

## Summary

Windows MVP uses one private sandbox identity and one active effective policy cohort. This RFC codifies design decisions already present in RFC-0006, RFC-0007, and RFC-0010, and adds normative requirements for policy epoch transitions. It prevents future re-introduction of dual-user (offline/online) identity schemas and keeps the public API free of backend-private details.

## Motivation

Earlier prototypes of RunSeal explored a dual-user sandbox identity model with separate `offline_user` and `online_user` identities. Experience showed that this approach:

- Multiplied setup complexity (two identities, two profile states, two sets of ACLs).
- Created ambiguous partial-setup states that were hard to detect or repair.
- Leaked identity details (SID counts, account names) into the public API surface.
- Did not improve security — network mode, filesystem level, and runtime roots are policy decisions, not identity properties.

The MVP adopts a single-identity model where identity is a setup concern and policy is the enforcement concern.

## Design decisions

- **No dual offline/online sandbox users.** RunSeal uses exactly one private sandbox identity for all sandboxed commands.
- **Identity is not policy.** Network mode, filesystem level, runtime roots, and proxy configuration are policy/enforcement state, not separate users or identity profiles.
- **Policy changes are epoch transitions.** Per RFC-0006 §Agent app global policy model, changing the global sandbox policy requires a policy epoch transition. A transition must occur only when no active sandboxed executions exist, or it must fail closed.
- **Mixed policy concurrency under the same sandbox identity is outside the MVP.** Concurrent executions are supported only when they share the same active policy hash and policy epoch.
- **Same-policy concurrency requires refcounted or generation-bound global enforcement state.** Global enforcement resources (network guards, proxy listeners, filesystem ACLs) must be tied to the active policy epoch, not to individual executions.
- **Runtime roots, synthetic home, and temporary directories remain per-execution** even when policy enforcement is shared across executions.
- **Cleanup is per-execution (job/process tree), not per sandbox user.** No mass kill of all processes under the sandbox identity.
- **Stale dual-user setup artifacts from earlier prototypes MUST be detected** during backend initialization and MUST cause a structured fail-closed error. Manual repair or reinstall is required.
- **Public API exposes no backend-private identity details.** No SID, ACL, WFP rule names, token attributes, Job Object handles, or helper identity details appear in the public protocol, policy schema, or audit event shapes.
- **Effective policy hash incorporates enforcement state.** Per RFC-0003 §Effective policy hash, the `policy_hash` represents the full normalized enforcement state, not only the user-supplied policy JSON.

## Design invariants

The following invariants apply to all Windows MVP backends:

1. A single sandbox identity has at most one active effective policy cohort at any time.
2. All concurrent sandbox executions share the active `policy_hash` and `policy_epoch`.
3. A policy transition MUST NOT modify SID-scoped or global enforcement state while old-epoch executions are still active.
4. Global enforcement state MUST be refcounted or generation-bound.
5. Execution cleanup is per-execution (job/process tree), not per sandbox user (no mass kill).
6. Runtime roots, synthetic home, and temporary directories remain per-execution even when policy enforcement is shared.

## Execution binding semantics

Before starting a sandboxed process, the Windows backend MUST:

1. Normalize the requested policy into the effective enforcement state.
2. Compute the effective `policy_hash`.
3. Compare the requested `policy_hash` with the `policy_hash` bound to the active `policy_epoch` for the sandbox identity.
4. If no sandboxed executions are active and the hash differs, create or activate a new `policy_epoch`.
5. If sandboxed executions are active, require the requested hash to match the active epoch's `policy_hash`.
6. Create an `Execution` and bind it to the active `policy_hash` and `policy_epoch`.
7. Increment the active execution count for that epoch before process start.

If the requested `policy_hash` differs from the active epoch while sandboxed executions are active, the backend MUST reject the request fail-closed with `POLICY_TRANSITION_BUSY` or an equivalent structured error. It MUST NOT spawn the process and MUST NOT mutate global enforcement state.

For a started execution, `policy_hash` and `policy_epoch` are immutable. Every execution-scoped audit event, JSON-RPC event notification, and final `ExecutionResult` MUST echo the same values.

## Runtime isolation semantics

Shared policy enforcement does not imply shared runtime state. Every sandboxed execution MUST receive its own runtime root, synthetic home/profile roots, temporary directory, environment block, process tree, and cleanup scope.

The Windows backend MUST prevent one execution from reading or writing another execution's runtime root unless an explicit future policy extension allows that behavior. Cleanup MUST terminate only the process tree associated with the target execution and MUST NOT mass-kill by sandbox identity.

## Network semantics

`network.disabled` MUST block direct egress at the OS enforcement boundary. `network.proxy` MUST allow only the managed proxy path and MUST block direct socket bypass. Unmanaged direct networking remains outside the MVP policy surface.

## Freeze gate requirements

The implementation repository MUST include automated semantic tests, runnable on Windows locally and later in CI, for at least:

- Format and static checks for Rust code.
- Protocol and audit event shape, including `runseal_version`, `policy_hash`, and `policy_epoch`.
- Execution immutability for `policy_hash` and `policy_epoch`.
- Same-policy concurrency sharing one epoch.
- Mixed-policy execution rejection while active executions exist.
- Policy transition gating and post-drain epoch activation.
- Per-execution runtime root, synthetic home, temp, and environment isolation.
- Execution-scoped process cleanup without affecting unrelated executions.
- `network.disabled` blocking direct egress.
- `network.proxy` blocking direct socket bypass and requiring the managed proxy path.
- Legacy dual-user setup artifact detection with structured fail-closed behavior.
- Unsupported capability requests failing closed instead of falling back to unrestricted execution.

RunSeal MUST NOT claim the Windows backend has reached the semantic freeze gate until these tests pass on the Windows reference backend.

## Cross-references

- **RFC-0003 §Effective policy hash:** hash composition requirements.
- **RFC-0006 §Agent app global policy model:** epoch transition rules and invariants.
- **RFC-0007 §Windows backend model:** single identity requirement and public API hygiene.
- **RFC-0010 §Windows backend identity boundary:** additional public API hygiene rules.

## Backward compatibility

Old dual-user setup artifacts from earlier prototype installations are not supported. The backend must detect them during initialization and fail closed with a structured error. Users must repair or reinstall RunSeal to use the single-identity model.

## Out of scope

- Multi-identity or multi-tenant sandbox models.
- Per-execution policy identity switching.
- Runtime migration of enforcement state across policy epochs.
