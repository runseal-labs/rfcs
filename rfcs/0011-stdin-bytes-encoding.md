# RFC-0011: stdin bytes encoding

- Status: Accepted for MVP

## Summary

RFC-0006 defines `stdin.mode` values but leaves the concrete payload shape for `bytes` underspecified. This RFC fixes the MVP wire format for byte stdin so clients and implementations can interoperate without guessing.

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

`stdin.mode: "empty"` remains the default and MUST close stdin without data.

`stdin.mode: "inherit"` and `stdin.mode: "stream"` remain reserved protocol modes. Implementations that cannot safely support them MUST reject them with `INVALID_REQUEST` rather than silently falling back to host stdin.

## Audit boundary

Raw stdin can contain secrets. Audit events MUST NOT include raw stdin bytes, decoded text, or the base64 payload.

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

## Conformance

Conformance tests SHOULD verify:

1. `stdin.bytes` delivers decoded bytes exactly to the child process.
2. Invalid base64 payloads are rejected before execution.
3. Unknown stdin fields are rejected before execution.
4. Audit JSONL does not contain raw stdin or the encoded payload.
