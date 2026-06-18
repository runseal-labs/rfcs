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
