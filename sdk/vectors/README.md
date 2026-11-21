# SDK Conformance Vectors

Shared data-driven test cases for rules in the [SDK specification](../spec.md). Every SDK runs all vectors through its `s2-specs` submodule; failing any case means the corresponding rule is not satisfied (see [Conformance](../spec.md#conformance)).

Each file contains `{"cases": [...]}`. Every case specifies the `rule` ID it tests, and most include a descriptive `name`.

Values are encoded so every language can parse them losslessly:

- Byte sequences are hex-encoded (`body_hex`, `bytes_hex`, `payload_hex`, and `hex` items in a `bytes` array).
- Integers that may exceed `2⁵³`—and all values in `numbers.json`—are decimal strings. `numbers.json` also uses `"NaN"`, `"Infinity"`, `"-Infinity"`, and decimal fractions.
- Large runs of zero bytes are encoded as `{"zeros": n}` inside a `bytes` array, or via `body_len` (a body of `n` zero bytes). Large expected payloads specify `payload_len` and `payload_sha256` instead of raw hex.

A test runner skips any case its language's type system cannot express (such as `NaN` for a Go `uint64`) and reports the skip.

| File | Rules | Input | Expected output |
|---|---|---|---|
| `endpoints.json` | C1, C3 | `endpoints` (`account` and/or `basin`, each omitted when absent), `access_token`, or `env` (variables read by `from_env()`), plus a `basin` name | `account_origin`, `basin_origin` for that basin, and whether the `s2-basin` header is sent; or `error` |
| `names.json` | C9 | A `basin` name | `valid` or `ValidationError` |
| `keys.json` | C4 | An encryption key as `text` or `bytes_hex` | The `s2-encryption-key` `header` value, or `error` |
| `numbers.json` | A2d, C2, D5 | A `field` and its `value`, or `millis` for a duration | `valid`, `ValidationError`, or `valid_if_representable` (valid if the language's type represents the value accurately per D4, otherwise `ValidationError`) |
| `dates.json` | D4 | A wall-clock `field` (`access_token.expires_at` or `metrics.timestamp`), `epoch_secs` (signed whole seconds since Unix epoch), and optional `subsec_millis` (`0`–`999`, added to `epoch_secs` so `-1` with `999` is `1969-12-31T23:59:59.999Z`) | `valid`, `ValidationError`, or `{"unix_secs": "..."}` (the floored whole Unix second) |
| `records.json` | A2, D2, D3 | A `record`, a `batch` of `count` records with `body_len` bytes each, a batch `fencing_token`, a `fence` or `trim` command input, or an `is_command_record` record | `metered_bytes`, `valid`, `ValidationError`, encoded command `headers` and `body_hex` (or `if_representable`/`otherwise` per D4), or boolean `is_command_record` |
| `errors.json` | C8, E2, E4 | The `call` kind (`append` for a call with side effects, or `read`) plus the response `status` and `body` | Error `type`, `status`, `code` (`null` if none), `message`, `is_retryable`, `has_no_side_effects`, and any typed payload; or `if_representable` / `otherwise` branches per D4 |
| `s2s-frames.json` | C6, E2, F1, T9 | Raw `bytes` of a session response body | Decoded `messages` (each with payload and `reconnect_advised`), followed by stream termination: `clean`, `ServerError` (from a terminal frame, with `status`, `code`, and `message`), `ConnectionLostError`, or `ProtocolError` |

Notes:

- In `numbers.json`, `valid` means the SDK accepts the value without raising `ValidationError`; the server may still reject it. Covered fields: `record.timestamp`, `append.match_seq_num`, `read.start.seq_num`, `read.start.timestamp`, `read.stop.count`, `read.stop.bytes`, `read.stop.until`, `trim.seq_num`, `read.stop.wait`, `stream_config.retention_age`, `stream_config.delete_on_empty.min_age`, `client.connection_timeout`, `client.request_timeout`, `client.max_attempts`, `client.min_base_delay`, `client.max_base_delay`, `session.max_unacked_bytes`, `session.max_unacked_batches`, `batching.linger`, `batching.max_batch_records`, and `batching.max_batch_bytes`.
- In `errors.json` and `s2s-frames.json`, `message` is the raw `ServerError` message defined by E2 (before adding operation context per E7), or the decimal HTTP status when the body is empty or not valid UTF-8.
- In `s2s-frames.json`, `clean` refers to frame-decoder termination; T9 determines how the session interprets a clean response end.
