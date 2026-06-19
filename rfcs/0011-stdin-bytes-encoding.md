# RFC-0011: stdin bytes and file input

- Status: Accepted for MVP

## Summary

RFC-0006 defines `stdin.mode` values but leaves the concrete payload shape for `bytes` underspecified. This RFC fixes the MVP wire format for byte stdin and defines file-backed stdin for larger payloads that should not be embedded in the JSON-RPC request.

## Decision

`ExecutionRequest.stdin` with `mode: "bytes"` MUST use base64 payload encoding:

```json
{
  "stdin": {
    "mode": "bytes",
    "data": "base64:aGVsbG8K",
    "encoding": "base64"
  }
}
```

Rules:

- `stdin.mode` MUST be `"bytes"`.
- `stdin.encoding` MUST be `"base64"`.
- `stdin.data` MUST be a string prefixed with `base64:`.
- The bytes after the prefix MUST be valid RFC 4648 base64 using the standard alphabet with padding.
- Empty stdin bytes are represented as `data: "base64:"`.
- The decoded bytes MUST be written exactly to the child process stdin, then stdin MUST be closed.
- Implementations MUST enforce a bounded decoded byte limit and reject oversized payloads before execution.
- Implementations MUST reject unknown fields inside the `stdin` object.

For larger stdin, clients SHOULD write the payload to a file under the execution `cwd` and pass `mode: "file"`:

```json
{
  "stdin": {
    "mode": "file",
    "path": ".runseal/input/stdin.bin"
  }
}
```

Rules:

- `stdin.mode` MUST be `"file"`.
- `stdin.path` MUST be a string path to an existing regular file.
- Relative paths MUST be resolved under `ExecutionRequest.cwd`.
- Absolute paths MAY be accepted only when their canonical target is under `ExecutionRequest.cwd`.
- Implementations MUST reject paths whose canonical target is outside `ExecutionRequest.cwd`.
- Implementations MUST enforce a bounded file byte limit and reject oversized files before execution.
- Implementations MUST read the file and write those bytes exactly to child process stdin, then stdin MUST be closed.
- Implementations MUST reject unknown fields inside the `stdin` object.

`stdin.mode: "empty"` remains the default and MUST close stdin without data.

`stdin.mode: "inherit"` and `stdin.mode: "stream"` remain reserved protocol modes. Implementations that cannot safely support them MUST reject them with `INVALID_REQUEST` rather than silently falling back to host stdin.

## Audit boundary

Raw stdin can contain secrets. Audit events MUST NOT include raw stdin bytes, decoded text, the base64 payload, or the stdin file path.

Audit events MAY include non-sensitive stdin metadata:

```json
{
  "stdin": {
    "mode": "bytes",
    "byte_count": 6
  }
}
```

## Rationale

Using the same `base64:` data convention as execution stream events keeps the v1 protocol binary-safe without adding transport-specific framing. Requiring an explicit `encoding` field makes request validation strict and leaves room for future encodings only through a new RFC.

File-backed stdin avoids JSON-RPC and command-line payload size limits while keeping host file reads bounded by the execution workspace. It is a local transport optimization, not a permission bypass.

## Conformance

Conformance tests SHOULD verify:

1. `stdin.bytes` delivers decoded bytes exactly to the child process.
2. Invalid base64 payloads are rejected before execution.
3. Unknown stdin fields are rejected before execution.
4. `stdin.file` delivers file bytes exactly to the child process.
5. `stdin.file` rejects paths outside `ExecutionRequest.cwd`.
6. Audit JSONL does not contain raw stdin, the encoded payload, or the stdin file path.
