# RFC-0022: Control descriptors and terminal observations

- Status: Draft
- Created: 2026-10-23
- Project: RunSeal
- Amends: RFC-0021

## Request

For pipe executions, `io.control` MUST contain `mode: "pipe"` or `mode: "socket"`, plus an integer `child_fd` from 3 through 255. The backend MUST expose the separate duplex byte channel at exactly that descriptor in the child. Descriptors 0 through 2 remain standard streams. Omitted, fractional, out-of-range, or unsupported requests MUST fail before launch. No arbitrary host handle inheritance is permitted.

The CLI uses socket mode at descriptor 3. Descriptor selection changes transport only and MUST NOT change sandbox policy or allow an execution outside its managed process range. PTY plus control remains unsupported.

## Lifecycle

Socket mode retains RFC-0021 independent half-close. Pipe mode exposes a native asynchronous byte pipe; closing input seals further host writes but MUST NOT claim peer EOF, since Windows duplex pipes cannot independently close their write direction. Pipe clients MUST use message framing or an application completion message; output remains readable until execution cleanup. Both modes retain bounded backpressure and owned cleanup. Control output MUST remain separate from stdout and stderr. Cancellation acknowledgement does not prove cleanup; clients MUST await terminal `cleanup_complete` before treating the managed range as empty.

`execution.started` MUST include the actual launched `process_id`, including when a helper owns the launch. An admission receipt is not a started-process identity.

`signalExecution` returns acceptance and a positive per-execution `sequence`. On Windows, after the native terminal input writer successfully writes the interrupt byte, the backend MUST emit `execution.signal_delivered` with the same sequence, `signal: "interrupt"`, and `target_process_id`. Queuing alone MUST NOT emit delivery. Failed writes or termination before delivery MUST leave the signal unconfirmed. On Windows this confirms a Ctrl-C write to the owned console, identified by its root process; it does not claim a POSIX foreground group or application handling. Resize receipts still confirm acceptance only. Shell activity without positive observations remains unknown.

On Windows, pipe mode MUST expose a genuine named byte pipe compatible with inherited asynchronous pipe APIs. The connected endpoint MUST be verified before inheritance. Socket mode uses a separate local socket endpoint suitable for CRT byte I/O and independent half-close. No transport grants general network access or changes sandbox policy.

## Conformance

Windows conformance MUST exercise descriptors 3 and 7 in unrestricted and sandboxed pipe execution, independent stdin/control traffic, control input half-close, invalid descriptor rejection, cancellation followed by fresh admission, and preservation of standard output. Tests MUST use the actual native descriptor, not an environment-only simulation.

Conformance MUST verify real child PID equality, acceptance preceding native interrupt delivery, no false delivery on write failure, and native asynchronous pipe I/O with independent stdin and framed completion. Socket mode MUST retain independent half-close.

## macOS interactive execution amendment

macOS MAY implement PTY and control-channel execution under the existing policy boundary. Enabling these capabilities MUST NOT bypass filesystem, process, environment, or network enforcement. Linux capability claims are unaffected. A platform MUST remain fail-closed for combinations it cannot implement; PTY plus control remains unsupported.

The macOS control endpoint MUST be an inherited, native duplex byte channel at the requested descriptor. Both modes MUST preserve separation from stdin/stdout/stderr and bounded backpressure. Socket mode MUST independently half-close host input while retaining readable child output. Pipe mode MUST preserve the framed-completion contract above: sealing host writes MUST NOT promise peer EOF. Descriptor remapping MUST preserve the spawn-error channel, handle descriptor collisions, close unrelated inherited endpoints, and avoid granting general network access.

A macOS PTY MUST provide a real controlling terminal, initial dimensions, resize notification, and terminal input/output. `execution.started.process_id` identifies the launched root process, not necessarily the foreground job. For POSIX native interrupt delivery, `execution.signal_delivered` MUST include a positive `target_process_group_id` identifying the foreground process group that actually received the signal. The backend MUST obtain this identity from the terminal and confirm successful native delivery before emitting the event with the accepted sequence. It MUST NOT substitute the root process ID for an unobserved foreground group. `target_process_id` retains its process-identity meaning and MUST NOT be used to disguise a process-group observation. Windows retains its existing console input-write acknowledgement. Delivery does not prove application handling. An unavailable foreground observation MUST remain unknown; a client MUST NOT cache an old group as a current observation.

Cancellation, timeout, host disconnection, and root-process exit MUST clean up the complete owned execution range, including foreground and background jobs in distinct process groups. Signalling only the original root group is insufficient. The backend MUST join I/O and confirm quiescence before `cleanup_complete: true`; failed or incomplete observation, termination, or joining MUST report cleanup failure rather than a successful terminal result. Native input and output must not indefinitely block cancellation or leak threads. Sandbox policy MUST prevent escape from the owned range or the backend MUST track and terminate such descendants reliably.

If a backend uses a kernel-managed process cohort to establish this range, the cohort MUST be isolated to one execution before untrusted code starts. Membership MUST survive fork, exec, session changes, and reparenting; untrusted processes MUST NOT be able to escape it. A process-table snapshot alone is insufficient proof of emptiness. The backend MUST close trusted launch/restart admission and obtain kernel evidence that no live cohort members remain, including execution helpers, before confirming cleanup. Process signalling MUST bind to a stable process identity rather than a PID that can be reused between observation and signalling. A surviving trusted cleanup owner MUST handle host disconnection without relying on the disconnected host to request cleanup. Failure to initialize, observe, authenticate, seal, terminate, or verify this boundary MUST fail closed; it MUST NOT silently select an execution path with weaker cleanup guarantees. Internal control endpoints MUST authenticate their peer and enforce bounded I/O. Concurrent executions MUST NOT share a cohort whose cleanup could terminate another execution.

Durable recovery records MUST be protected from untrusted execution, including execution with broad filesystem access within the restricted policy. Recovery MUST verify boot identity, complete owner birth identity and exclusive ownership before acting. A positive drained-cohort proof MUST be durably recorded before unregistering its kernel owner. An absent cohort identifier alone MUST NOT substitute for that proof. Unknown ownership, corrupt admitted records, or failed observation MUST block recovery and new admission rather than silently releasing the boundary. Concurrent recovery MUST preserve live executions.

macOS conformance MUST exercise restricted and explicit unrestricted execution, descriptors 3 and 7, large and fragmented bidirectional control traffic, independent stdin/control, both close modes, and stalled-peer cancellation. PTY checks MUST cover real TTY identity, input echo, resize, raw ETX input, foreground-job interruption while preserving the shell, and cancellation with background jobs. Both transports MUST pass fresh admission after cancellation and host-disconnection cleanup. Tests MUST assert policy enforcement, cleanup proof, and absence of surviving owned processes; a passing pipe-execution test is not PTY/control conformance evidence. Capability promotion requires passing these native checks; changing a status field alone is not implementation.
