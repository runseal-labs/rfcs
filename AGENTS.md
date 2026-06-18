# Agent Rules

This repository is the public RFC and planning repository for RunSeal.

## Repository Boundary

- This repo owns public contracts: protocol, policy schema, threat model, platform status, implementation plan, conformance requirements, and acceptance criteria.
- Concrete runtime code belongs in `runseal-labs/runseal`, not here.
- If an implementation detail changes public behavior, update the relevant RFC here before or alongside the implementation change.
- If an implementation detail does not change the public contract, keep it in the implementation repository.

## Public Terminology

Use RunSeal terminology only:

- `RunSeal`
- `Execution`
- `SandboxPolicy`
- `SandboxLevel`
- `NetworkPolicy`
- `BackendCapabilities`
- `SandboxBackend`
- `PlatformSandboxPlan`
- `AuditEvent`
- `runtime root`
- `synthetic home`
- `protected subpath`
- `managed proxy`

Do not include private product names, internal repository names, private issue or MR IDs, customer names, internal codenames, internal filesystem paths, screenshots, logs, or chat-only context.

## RFC Authoring Rules

- Keep RFCs durable and implementation-neutral unless the RFC explicitly defines backend behavior.
- Use normative language (`MUST`, `MUST NOT`, `SHOULD`, `MAY`) for stable protocol, policy, conformance, and fail-closed requirements.
- Keep Windows as the MVP reference backend and enterprise security baseline unless a later RFC changes that status.
- Keep macOS experimental and Linux future/community until conformance evidence justifies promotion.
- Public APIs must not expose backend-private controls such as ACLs, SIDs, token attributes, firewall rule names, WFP callouts, Seatbelt fragments, sandbox-exec flags, or namespace details.
- Rollback/checkpoint behavior is not part of the MVP security boundary unless a future RFC explicitly adds it.

## Index And Status Hygiene

- When adding an RFC, update `README.md`, `README.zh-CN.md`, and any baseline/status RFC that enumerates accepted RFCs.
- Number RFCs monotonically and never renumber existing RFC files.
- Preserve accepted RFCs as historical records; use amendments or new RFCs for material changes.
- Keep English and Chinese README positioning aligned.

## Validation

Before committing:

- Run `rg -n -i "tailos|tyclaw|myprojects|ferstar|private issue|private MR" README.md README.zh-CN.md rfcs` and ensure matches are only generic redaction guidance or public source URLs.
- Run `git diff --check`.
- Review links for new RFC files and README index entries.
