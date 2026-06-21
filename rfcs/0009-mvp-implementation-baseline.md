# RFC-0009: MVP implementation baseline

- Status: Accepted for MVP
- Created: 2026-06-15
- Project: RunSeal

## Summary

This RFC records the PRD-readiness review for the RunSeal RFC set. The goal is to make the repository implementation-ready for AI coding agents without relying on chat-only context or private implementation provenance.

Conclusion: the RFC set is PRD-ready for a Windows-first MVP with Windows as the reference backend and enterprise security baseline. macOS is an experimental backend contribution track, and Linux remains a future/community backend behind the same protocol. The remaining unresolved items are intentionally marked as post-MVP extension points and do not block starting implementation.

## External evidence reviewed

The current direction is supported by public sources and comparable open-source implementations:

- OpenAI Codex docs: sandbox and approval are separate controls; local commands use OS-enforced sandboxing; default local posture limits writes to the workspace and keeps network off unless configured; network proxy constrains enabled network access.
  - https://developers.openai.com/codex/concepts/sandboxing
  - https://developers.openai.com/codex/agent-approvals-security
  - https://developers.openai.com/codex/permissions
- Claude Code sandboxing references: macOS Seatbelt and Linux bubblewrap patterns validate the OS-native local sandbox direction; permission prompts and sandbox enforcement are separate layers.
  - https://code.claude.com/docs/en/sandboxing
  - https://www.claudecodecamp.com/p/claude-code-sandboxing-how-sandbox-works-and-what-it-doesn-t-protect
- Gemini CLI sandbox docs: validate macOS Seatbelt profiles, container-based alternatives, network/proxy modes, and explicit mount/permission configuration as common agent-sandbox UX patterns.
  - https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/sandbox.md
- Arapuca: cross-platform Rust process sandbox using Linux Landlock/seccomp/cgroups/netns, macOS Seatbelt, and Windows AppContainer/Job Objects/restricted tokens, with fail-closed posture.
  - https://github.com/sergio-correia/arapuca
- Landstrip: coding-agent sandbox using Landlock, Seatbelt, and LPAC AppContainer; accepts a policy subset and explicitly reports unsupported Windows network policies.
  - https://github.com/jarkkojs/landstrip
- Heel: native OS-level sandbox for LLM-generated code with macOS Seatbelt, Linux Landlock/seccomp, planned Windows AppContainer, network policies, and credential/home protection controls.
  - https://github.com/lexoliu/heel

These sources support RunSeal's public positioning: a lightweight OS-native local sandbox layer with a stable protocol, policy/audit model, a Windows reference backend for the MVP, and macOS/Linux backends that can be contributed and promoted through the same abstraction and conformance suite.

## Accepted MVP boundaries

The following boundaries are accepted and implementation-ready:

1. **Product scope**: RunSeal is a local OS-native execution sandbox layer for AI agents, not a hosted sandbox service, Docker replacement, VM platform, or cloud multi-tenant control plane.
2. **Platform priority**: Windows is the first-class MVP reference backend and initial enterprise security baseline. macOS is experimental, and Linux remains a future/community backend behind the same abstraction.
3. **Public policy model**: filesystem sandbox level and network mode are separate dimensions.
4. **Sandbox levels**: `read-only`, `workspace-contained`, `workspace-write`, and `danger-full-access` are the MVP filesystem levels.
5. **Network modes**: `disabled` and `proxy` are the MVP network modes. Unmanaged direct networking is outside the MVP policy surface.
6. **Fail-closed posture**: unsupported or partially enforceable policies must return structured errors; the implementation must not silently fall back to unrestricted execution.
7. **Windows backend target**: use restricted local execution identity plus OS policy controls such as ACLs, restricted tokens, Job Objects/AppContainer where available, and network restrictions/proxy-only egress.
8. **Windows identity and epoch model**: Windows MVP uses one private sandbox identity, one active effective policy cohort, immutable per-execution `policy_hash`/`policy_epoch`, same-policy concurrency only, and fail-closed mixed-policy transitions.
9. **macOS backend target**: use `/usr/bin/sandbox-exec` with generated Seatbelt profiles, safe dynamic path injection, canonical/raw path modeling, and proxy-only egress when network is enabled, but treat macOS as experimental until conformance evidence proves each supported capability.
10. **Linux posture**: do not implement Linux in MVP; return unsupported for sandboxed execution unless the request explicitly uses `danger-full-access` local execution.
11. **Proxy posture**: MVP starts with a managed HTTP proxy guard and proxy lifecycle audit events. Domain/CIDR/method/path rules and body inspection are post-MVP extensions.
12. **Secrets posture**: real credentials stay outside sandbox process environment. Credential injection belongs at the proxy boundary or future trusted host extensions.
13. **Protocol shape**: JSON-RPC 2.0 over stdio is the MVP transport; Unix socket/named pipe/HTTP bridge can follow without changing method semantics.
14. **Execution object**: the public model is `Execution`; the CLI verb is `runseal exec`; raw process spawning remains an implementation detail.
15. **Audit model**: JSONL audit events are required for execution lifecycle, policy decisions, denials, backend setup failures, proxy lifecycle, and final results.
16. **Approval boundary**: approval UI/orchestration remains a host-application concern in v1. RunSeal returns `APPROVAL_REQUIRED` where needed but does not own interactive approval flow in MVP.
17. **MCP boundary**: MCP is an adapter over the stable protocol, not a core v1 transport.
18. **Public terminology**: public docs and code must use RunSeal terminology only and must not include private product names, internal repository names, private issue/MR IDs, customer names, or chat artifacts.

## Implementation-ready work packages

The first coding pass can start without further product clarification:

1. Create the Rust workspace/crate skeleton described in RFC-0008.
2. Implement the policy model and canonical JSON validation for RFC-0003.
3. Implement `runseal exec`, `runseal capabilities`, and `runseal explain-policy` CLI shells.
4. Implement the backend trait and capability reporting.
5. Implement explicit local execution for `danger-full-access` as the non-sandbox baseline.
6. Implement Windows backend scaffolding with fail-closed capability checks and the declared acceptance tests.
7. Add macOS/Linux backend skeletons or contribution guides that report unsupported capabilities fail-closed.
8. Implement JSONL audit writer and event schema alignment with RFC-0004.
9. Implement JSON-RPC stdio methods from RFC-0006.
10. Add conformance tests that distinguish supported, unsupported, experimental, denied, failed, and skipped states.

## Acceptance criteria for the RFC set

The RFC set reaches PRD-ready state when all of the following are true:

- README and README.zh-CN describe the same public product boundary and link to all RFCs.
- RFC-0001 defines the OS-native abstraction and closes upstream reuse/backend exposure questions for MVP.
- RFC-0002 defines `disabled` and `proxy` as MVP network modes, with direct/domain routing/body inspection deferred.
- RFC-0003 defines stable policy shape, canonical JSON, MVP profiles, and strict effective-policy hashing.
- RFC-0004 defines event envelopes and MVP audit retention/output-body decisions.
- RFC-0005 defines synthetic home, cache posture, and explicit real-home handling.
- RFC-0006 defines JSON-RPC stdio MVP, error model, event streaming, and closed protocol decisions.
- RFC-0007 defines threat model, platform matrix, Windows reference backend expectations, macOS experimental posture, Linux future/community posture, and fail-closed requirements.
- RFC-0008 defines implementation phases, work packages, conformance tests, and technical-preview acceptance criteria.
- RFC-0010 defines the RFC/implementation repository boundary, Windows reference extraction rules, redaction requirements, and conformance gates.
- RFC-0011 defines the binary-safe `stdin.bytes` request encoding and audit boundary.
- RFC-0012 defines the Windows single-identity model, global policy epoch invariants, and freeze-gate requirements.
- RFC-0013 records the future optional local service-mode direction and explicitly keeps the MVP CLI/direct-execution path intact.
- RFC-0014 records the future portable backend onboarding path for macOS and Linux, keeping Windows as the MVP reference baseline while requiring capability-driven, conformance-gated promotion for other platforms.
- RFC-0015 defines escape categories, severity levels, and adversarial conformance gates for backend capability promotion.
- RFC-0016 defines the adversarial conformance case manifest, oracle model, result schema, and CI/release gate tiers.
- No RFC or README contains private/internal product references or transient chat artifacts.

## Remaining non-blockers

The following are intentionally not required before MVP coding begins:

- Full enterprise route composition and organization-wide policy inheritance.
- Response body inspection/redaction plugins.
- macOS enterprise-baseline backend implementation.
- Linux backend implementation.
- VM/microVM/container daemon backends.
- Interactive approval UI.
- MCP server implementation.
- Complete automatic discovery of every package-manager cache and language runtime.
- Long-lived explicit session APIs beyond implicit execution sessions. RFC-0013 defines the follow-up service-mode direction, but service mode is not required before the Windows-first MVP implementation begins.
- macOS and Linux capability-probe or enforcement implementations. RFC-0014 defines the follow-up portable backend onboarding direction, but non-Windows backend promotion remains gated on conformance evidence.

## Review decision

The RFC set is accepted as the Windows-first MVP PRD baseline for implementation. RFC-0013, RFC-0014, RFC-0015, and RFC-0016 are included in the repository planning set as post-MVP/future-direction RFCs and do not block the Windows-first MVP implementation. Future changes should be filed as follow-up RFCs or amendments when they expand scope beyond the accepted MVP boundary, introduce service-mode public behavior, add adversarial conformance requirements, define conformance harness behavior, or promote a non-Windows backend based on conformance evidence.
