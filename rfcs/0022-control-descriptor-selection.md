# RFC-0022: Control descriptor selection

- Status: Draft
- Created: 2026-10-23
- Project: RunSeal
- Amends: RFC-0021

## Request

For pipe executions, `io.control` MUST contain `mode: "pipe"` and an integer `child_fd` from 3 through 255. The backend MUST expose the separate duplex byte channel at exactly that descriptor in the child. Descriptors 0 through 2 remain standard streams. Omitted, fractional, out-of-range, or unsupported requests MUST fail before launch. No arbitrary host handle inheritance is permitted.

The CLI control descriptor remains 3. Descriptor selection changes transport only and MUST NOT change sandbox policy or allow an execution outside its managed process range. PTY plus control remains unsupported.

## Lifecycle

Control input, output, half-close, backpressure and cleanup follow RFC-0021. Control output MUST remain separate from stdout and stderr. Cancellation acknowledgement does not prove cleanup; clients MUST await terminal `cleanup_complete` before treating the managed range as empty.

Terminal interrupt and resize acknowledgements only confirm acceptance. They MUST NOT be interpreted as foreground group identity or proof of native signal delivery. Clients without positive process observations MUST report unknown identity and activity.

## Conformance

Windows conformance MUST exercise descriptors 3 and 7 in unrestricted and sandboxed pipe execution, independent stdin/control traffic, control input half-close, invalid descriptor rejection, cancellation followed by fresh admission, and preservation of standard output. Tests MUST use the actual native descriptor, not an environment-only simulation.
