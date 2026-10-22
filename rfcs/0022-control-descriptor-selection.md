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

`signalExecution` returns acceptance and a positive per-execution `sequence`. After the native terminal input writer successfully writes the interrupt byte, the backend MUST emit `execution.signal_delivered` with the same sequence, `signal: "interrupt"`, and `target_process_id`. Queuing alone MUST NOT emit delivery. Failed writes or termination before delivery MUST leave the signal unconfirmed. On Windows this confirms a Ctrl-C write to the owned console, identified by its root process; it does not claim a POSIX foreground group or application handling. Resize receipts still confirm acceptance only. Shell activity without positive observations remains unknown.

On Windows, pipe mode MUST expose a genuine named byte pipe compatible with inherited asynchronous pipe APIs. The connected endpoint MUST be verified before inheritance. Socket mode uses a separate local socket endpoint suitable for CRT byte I/O and independent half-close. No transport grants general network access or changes sandbox policy.

## Conformance

Windows conformance MUST exercise descriptors 3 and 7 in unrestricted and sandboxed pipe execution, independent stdin/control traffic, control input half-close, invalid descriptor rejection, cancellation followed by fresh admission, and preservation of standard output. Tests MUST use the actual native descriptor, not an environment-only simulation.

Conformance MUST verify real child PID equality, acceptance preceding native interrupt delivery, no false delivery on write failure, and native asynchronous pipe I/O with independent stdin and framed completion. Socket mode MUST retain independent half-close.
