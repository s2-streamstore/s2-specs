# SDK conformance vectors

Cases for the [SDK spec](../spec.md)'s rules whose behaviour is data. Every SDK runs every case through its s2-specs submodule, and a case it fails counts as its rule not yet met (Conformance).

Each file holds `{"cases": [...]}`. Every case names the `rule` it tests, and most have a `name`.

Values are encoded so that every language reads them exactly:

- Bytes are hex: `body_hex`, `bytes_hex`, `payload_hex`, and `hex` in a list of `bytes`.
- An integer that may exceed 2⁵³ is a decimal string, and so is every value in `numbers.json`, which also uses `NaN`, `Infinity`, `-Infinity` and decimal fractions.
- A long run of bytes is `{"zeros": n}` in a list of `bytes`, or `body_len`, a body of that many zero bytes. A long expected payload is its `payload_len` and `payload_sha256`.

A runner skips a case its language cannot express, such as `NaN` for a Go `uint64`, and reports the skip.

| File | Rules | A case gives | And expects |
|---|---|---|---|
| `endpoints.json` | C1, C3 | `endpoints`, with `account` and `basin`, each absent when not given; or `env`, the variables `from_env()` reads. And a `basin` name. | `account_origin`, the `basin_origin` for that basin, and whether the `s2-basin` header is sent; or `error` |
| `names.json` | C11 | a `basin` name | `valid` or `ValidationError` |
| `keys.json` | C6 | a key, as `text` or `bytes_hex` | the `s2-encryption-key` `header`, or `error` |
| `numbers.json` | A2d, D7 | a `field` and its `value`, or `millis` for a duration | `valid`, `ValidationError`, or `valid_if_representable`: valid where the language's type holds the value (D2), and `ValidationError` elsewhere |
| `records.json` | A2, D6 | a `record`; a `batch` of `count` records of `body_len` bytes each; or a batch's `fencing_token` | `metered_bytes`, or `valid` or `ValidationError` |
| `errors.json` | C12, E1, E5 | the `call`, `append` (with side effects) or `read`, and the response's `status` and `body` | the error's `type`, `status`, `code` (`null` for none), `message`, `is_retryable` and `has_no_side_effects`, and a typed error's value; or `if_representable` and `otherwise`, as D2 decides |
| `s2s-frames.json` | C4, T9 | the `bytes` of a session's response body | the `messages` the frame decoder yields, each with its payload and `reconnect_advised`, then how the body ends: `clean`, a `ServerError` from a terminal frame, `ConnectionLostError` or `ProtocolError` |

In `numbers.json`, `valid` means the SDK takes the value without `ValidationError`; the server may still reject it. Its fields are `record.timestamp`, `append.match_seq_num`, `read.start.seq_num`, `read.start.timestamp`, `read.stop.count`, `read.stop.bytes`, `read.stop.until`, `trim.seq_num`, `read.stop.wait`, `stream_config.retention_age`, `stream_config.delete_on_empty.min_age`, `batching.linger`, `batching.max_batch_records` and `batching.max_batch_bytes`.

In `errors.json`, `message` is the `ServerError`'s message as E1 defines it, before any context E6 adds, and is the decimal status when the body is empty.

In `s2s-frames.json`, a clean end is the frame decoder's: T9 decides what it means for the session.
