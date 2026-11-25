# S2 SDK Specification

**Revision 1 (Draft).**

This specification defines the common behavior of all S2 SDKs (Python, TypeScript, Go, and Rust). It describes the target behavior across all four SDKs, not the current state of any individual implementation.

Rules in this specification—and choices left to individual SDKs—are guided by four principles: correctness, elegance, idiom, and efficiency. Non-obvious rules are explained by rule ID in [rationale.md](rationale.md).

## Reading this specification

- Every rule is a requirement. Where an SDK has a choice, the rule explicitly uses *may*.
- Each chapter has a letter prefix and numbered rules referenced by ID, such as `A6b`. While this revision is a draft, rule IDs may still change; once Revision 1 is finalized, rule IDs are permanently stable across future revisions (rules are never renumbered, and removed IDs are never reused). Include the revision (for example, `r1 A6b`) when the exact wording matters.
- Names are canonical. Every SDK uses the specification's terminology in its own casing and naming convention (see [Languages](#errors) for error types), and may add unit suffixes where a duration is represented as a number (for example, where the language lacks a duration type, or for whole-second API fields represented as integer seconds). In this document, `name()` is a function or method, `Name` is a type, and any other `name` is an accessor, option, setting, value, or wire name.

| Chapter | Covers |
|---|---|
| C | Client |
| D | Data: records, numbers, and timestamps |
| E | Errors |
| T | Retries, timeouts, and canceling |
| A | Appending: `append()`, `AppendSession`, `Producer`, `Batcher` |
| R | Reading: `read()`, `check_tail()`, `ReadSession` |
| M | Managing basins, streams, access tokens, and network policies |
| F | Forward compatibility |
| L | Logging |

[Terms](#terms) and [Error model](#error-model) introduce the core concepts before the chapters. [Settings and constants](#settings-and-constants), [Server guarantees](#server-guarantees), [Languages](#languages), [Conformance](#conformance), and [Revisions](#revisions) follow them.

## Terms

- **Unary call, session.** A unary call is a single request and response. A session is a long-running append or read stream on a single S2 stream, which the SDK maintains across attempts.
- **Connection.** A TCP connection (with TLS for HTTPS) over which requests travel. The client owns connections and pools them across requests (C12, C13).
- **Attempt.** One try at an operation: a single request and its response. For a session, an attempt is one streaming request on which the SDK sends batches, including any unacked batches sent on earlier attempts.
- **Sent.** A request or batch is considered sent in an attempt once any of its bytes may have left the SDK during that attempt. Because an attempt sends nothing after it ends (T10), whether it sent a request or batch is settled by the time its error arrives.
- **Ack.** The server's confirmation that an appended batch of records is durable.
- **Ticket.** The handle returned by `submit()` once a record or batch is accepted. Awaiting a ticket resolves to that record's or batch's outcome (A6).
- **Position.** A sequence number and a timestamp.
- **Definite.** An error is definite when the failed attempt certainly had no side effect—meaning no records were appended and no caller-requested mutation took effect (basin-level `create_stream_on_read` is an idempotent auto-creation convenience rather than a data mutation): on a unary call, the attempt had no effect; on an append session, none of the batches that were unacked when the error arrived took effect. For a definite error, `has_no_side_effects` (E4) is true. These errors are definite:
  - any error from an operation without side effects (such as a read) or from `submit()`, which atomically accepts nothing on failure (A4c);
  - any `412` response, whether typed or plain (E4e);
  - any server error whose code is listed with no side effects in [`error-codes.json`](../s2/v1/error-codes.json) (E4a);
  - `ValidationError`, `ClientClosedError`, `ConnectError`, and any `ConnectionLostError` that occurs before the request (or any unacked batch) was sent or that the server reports as unprocessed (`REFUSED_STREAM`, or—when the transport exposes that guarantee—an HTTP/2 stream above a `GOAWAY` `last_stream_id`);
  - `SessionClosedError` (E6).

  Every other error is **indefinite**: it may have had a side effect. Indefinite errors include unrecognized error codes, timeouts, failed SDK checks, and every `AbortedError` on an operation with side effects other than `submit()`.
- **Final error.** The error that ends an operation or session: the last attempt's error, which was not retried.
- **Closing, closed.** A session closes once, for one of the reasons listed in A8 or R9. An append session is **closing** from the moment `close()` is called until it finishes closing. A closing or closed session accepts no new work; work already accepted finishes according to A8 or R9.
- **Failure.** The error a session reports as the reason it closed: its final error (wrapped per A9) or `AbortedError`. A cleanly closed session has no failure.
- **Tail.** The end-of-stream position reported by the server: the sequence number the next record will receive, and the timestamp of the last record. The server reports the tail alongside records near the end of the stream, and on its own as a **heartbeat** while a read session waits at the tail.
- **Fetched, delivered.** A record is **fetched** once iteration decodes its batch from the server (R6b), and **delivered** once iteration hands it to the caller. Records fetched in the current batch but not yet handed to the caller are buffered, not yet delivered.

## Error model

This section summarizes how the SDK handles errors across attempts, unary calls, and sessions. It is informative and adds no rules; each step cites its governing rule.

An operation makes one or more attempts. A failed attempt produces an error that is definite if it certainly had no side effects, and indefinite otherwise (Terms). The SDK retries an attempt only when its error is retryable, repeating it is safe, and retry budget remains (T1). The error the operation stops on is its final error. A planned handover from an attempt that made progress is not a retry and consumes no budget (T6).

A unary call returns its result or fails with its final error. Because a definite final error guarantees no side effects only for the last attempt, an append whose earlier attempt may have taken effect fails instead with `IndefiniteAppendError` wrapping that final error (E5).

Sessions retry in the same way across attempts. An append session assigns every batch a single outcome—an ack or an error—which all records in the batch share (A6). Both append and read sessions close once and report a failure if they ended on an error (A9, R10).

Every error surfaced to the caller exposes two predicates indicating what is safe to do next: `has_no_side_effects` is true when the failed attempt certainly had no side effects, and `is_retryable` is true when retrying may succeed (E4).

## C. Client

1. **Construction.** A client is constructed from an access token and optional settings, and never reads environment variables implicitly. Requests authenticate with `Authorization: Bearer <token>`. An empty token, or one containing characters invalid in an HTTP header value (such as newlines or NUL bytes), fails with `ValidationError` when the client is constructed. `from_env()` constructs a client from `S2_ACCESS_TOKEN`, `S2_ACCOUNT_ENDPOINT`, and `S2_BASIN_ENDPOINT`, treating empty endpoint variables as unset and failing with `ValidationError` if `S2_ACCESS_TOKEN` is unset or empty, or if only one of the endpoint variables is set.
2. **Settings.** Every setting has the name, default, and valid range listed in [Settings and constants](#settings-and-constants). A setting outside its valid range (D5) fails with `ValidationError` where it is supplied; cross-field constraints (such as `min_base_delay ≤ max_base_delay`) are checked when the client is constructed.
3. **Endpoints.** The `endpoints` setting pairs an account endpoint and a basin endpoint, which must be provided together. Each endpoint consists of a host, optionally preceded by `http://` or `https://` and optionally followed by a port (`1`–`65535`) and a trailing `/`. Without a scheme, the default is `https`, and both endpoints must use the same scheme. The host must be a DNS name or an IP address (with IPv6 addresses enclosed in brackets). A basin endpoint's host may begin with `{basin}.`, which the SDK replaces with the basin name; when it does, the host must be a DNS name. A basin endpoint without `{basin}.` serves all basins, so the SDK sends the basin name in the `s2-basin` header (and only then). Any other form—such as providing only one endpoint, or including a path, query, fragment, or user info—fails with `ValidationError` when the client is constructed.
4. **Encryption key and stream config.** An encryption key is configured on the stream handle and sent in the `s2-encryption-key` header on every append and read (not `check_tail()`). A key represents non-empty raw bytes and may be supplied as bytes or as standard base64 text (ignoring surrounding whitespace). An empty key or non-base64 text fails with `ValidationError` when the key is constructed. Key length validation is otherwise left to the server, since the required length depends on the stream's cipher. A `stream_config` for create-on-first-use is passed per call (`append()`, `read()`) or when opening a session (`AppendSession`, `Producer`, `ReadSession`)—not on `check_tail()`. When provided, it is encoded as compact JSON in the `s2-stream-config` header (omitted when unset), and sessions send it on every request.
5. **Transports.** An SDK may support multiple transports and choose between them automatically. Every rule in this specification holds on every transport; transports may differ only in performance.
6. **Compression.** `compression` is `none`, `gzip`, or `zstd`. With `gzip` or `zstd`, the SDK compresses non-empty request bodies with that algorithm and advertises only that algorithm for responses, on both unary calls and sessions; a session frame is compressed only when its payload is at least 1 KiB. With `none`, the SDK compresses nothing and advertises no encoding, though an underlying transport may still advertise one on its own. Regardless of this setting, the SDK decompresses any response or session frame compressed with `gzip` or `zstd`, including error responses. A session frame whose wire length or decompressed size exceeds the limits in [`s2s.md`](../s2/v1/s2s.md) fails with `ProtocolError`.
7. **User agent.** Requests send `s2-sdk-<language>/<version>`, where `<language>` is `python`, `typescript`, `go`, or `rust`.
8. **Redirects.** The SDK never follows HTTP redirects. A `3xx` response fails with a `ServerError` carrying its HTTP status (E2).
9. **Names.** Any basin name that violates `BasinNameStr` in [`openapi.json`](../s2/v1/openapi.json) fails with `ValidationError` wherever the SDK accepts a basin name. Other resource names are not required to be validated by the SDK, though an SDK may expose typed wrappers that validate against the documented grammar in [`openapi.json`](../s2/v1/openapi.json) (failing with `ProtocolError` if a response violates that grammar, as in D4). Every name interpolated into a URL path is percent-encoded.
10. **Inputs.** The SDK reads caller inputs at the time they are passed and never mutates them: settings when the client is constructed, and options, configs, and records when a call is made, a session is opened, or a record/batch is accepted. Subsequent mutations by the caller have no effect, including on retries and reconnected requests (see [Languages](#numbers-and-byte-inputs)).
11. **Concurrency.** Clients, basin handles, stream handles, and sessions are safe to use concurrently across tasks or threads. The sole exception is iterating a read session, which is single-consumer (R6c).
12. **Closing.** The client owns its connections; basin and stream handles merely reference the client. Every client provides `close()`. Calling `close()` aborts all sessions opened from the client (A8c, R9d) and fails all in-flight unary calls with `AbortedError`, since they may already have taken effect. It terminates all connections—including connections still being established—without waiting for the server, then returns. Repeated calls to `close()` are idempotent and return the same result. After `close()`, any call on the client or its handles fails immediately with `ClientClosedError` (for lazy `list_all_*` iterators, on their next page fetch, M7); calls on sessions it aborted follow A4c and R9. Dropping a client without calling `close()` keeps it alive while any session opened from it remains open.
13. **Connection pooling.** Each client owns its HTTP/2 connection pool(s) (C12), pooling connections per origin (scheme, host, and port) and reusing live connections across requests. An SDK may share one pool across unary calls and sessions, or keep separate pools for them; every SDK-managed HTTP/2 pool obeys the rules below. If an SDK allows deriving client, basin, or stream handles with overridden per-handle settings (such as retry configuration), derived handles share the parent client's connection pool(s) and `close()` lifecycle (C12). Retiring a connection (T6) or closing a dead connection removes only that connection from its pool, leaving healthy sibling connections to the same origin in use.
    - **a. Concurrent stream limit.** By default, each pooled HTTP/2 connection carries up to 50 concurrent requests—clamped to the server's advertised `SETTINGS_MAX_CONCURRENT_STREAMS` when lower and exposed by the transport—reusing connections with spare capacity before opening another. An SDK may expose a transport setting to tune this cap up to `SETTINGS_MAX_CONCURRENT_STREAMS` (100 on S2). When every live connection to an origin is at capacity or retiring (T6), a new request opens an additional connection rather than waiting for an active stream to finish or failing.
    - **b. Keep-alive.** While a pooled HTTP/2 connection has active requests, the SDK sends an HTTP/2 `PING` after at most 20 s without receiving a frame. If no `PING` acknowledgment arrives within 10 s, the SDK closes the connection and removes it from the pool, handling active requests as a broken connection (T9).
    - **c. Idle reaping.** A pooled connection with no active requests is closed and removed from the pool once it has been idle for 90 s. Retired connections still close as soon as their active requests finish (T6), without waiting for the idle timeout.
    - **d. Flow-control isolation.** Paused or slow read sessions exerting backpressure (R6b) must not exhaust connection-level receive flow control and stall other concurrent requests or sessions sharing the connection—for example, by sizing the connection receive window to at least the connection's concurrent stream limit multiplied by its per-stream receive window, or by isolating each read session onto its own connection.

## D. Data

1. **Records.** An `AppendRecord` consists of a body, headers, and an optional timestamp; the server determines the stored timestamp according to the stream's timestamping config. A `SequencedRecord` returned by reads consists of a body, headers, `seq_num`, and `timestamp`. A body is bytes, and headers are an ordered list of `(name, value)` byte pairs. SDKs may provide string convenience APIs on top. Transports that carry records over JSON use `s2-format: base64` whenever needed to preserve arbitrary byte sequences losslessly.
2. **Command records.** `fence(token, [timestamp])` and `trim(seq_num, [timestamp])` construct command records, and `is_command_record()` identifies command records returned by reads. All three are public. `fence()` sets a single header `("", "fence")` and a body containing the UTF-8 bytes of `token` (an empty string clears the fencing token). `trim()` sets a single header `("", "trim")` and an 8-byte big-endian unsigned integer body encoding `seq_num`. Both accept an optional `timestamp` (validated per D5). `is_command_record()` returns true if and only if a record has exactly one header and that header's name is empty (`b""`). Reads return command records like any other record.
3. **Metered bytes.** A record's metered size in bytes is `8 + 2 × len(headers) + sum(len(name) + len(value)) + len(body)`. `metered_bytes()` is public, and every record or batch size limit (A2, A4, A5, R2) is measured in metered bytes.
4. **Numbers and timestamps.** Sequence numbers are integers. Record timestamps—and therefore position timestamps—are integer milliseconds (by convention since the Unix epoch) and are never accepted or exposed as a date type. Wall-clock timestamps in the API use the language's native date type: both RFC 3339 date-times (such as `created_at` and `expires_at`) and metrics timestamps (`start`, `end`, and datapoint or record timestamps, which the API represents as Unix epoch seconds). Languages with union types (Python, TypeScript) may also accept integer Unix epoch seconds as input for metrics `start` and `end`. For all wall-clock date inputs, invalid dates (such as `Invalid Date`), timezone-naive Python `datetime` values, and RFC 3339 inputs outside years `0000`–`9999`—or outside the language's native date type's representable range if narrower (such as Python's `datetime.MINYEAR == 1` or Rust's `jiff::Timestamp::MAX`)—fail with `ValidationError`. For metrics timestamp inputs, date inputs have sub-second components floored to whole Unix seconds, and any date or integer value outside `0` to `2³² − 1` fails with `ValidationError`. The SDK must preserve every number returned by the server exactly. If a numeric or wall-clock timestamp value exceeds what the language's type can represent accurately (see [Languages](#numbers-and-byte-inputs)), the response or session message fails with `ProtocolError`; for a `412` or `416` response whose value cannot be represented, the SDK falls back to a plain `ServerError` (E2).
5. **Numeric validation.** Caller-supplied numbers and durations are validated when accepted (C10). Any non-finite value (`NaN`, infinities), negative value, or value outside its valid range fails with `ValidationError`, as does any fractional value where an integer or whole number of seconds is required. Sequence numbers, timestamps, counts, and byte limits are integers from `0` to `2⁶⁴ − 1`, or to the maximum exact integer of the language's type if lower (see [Languages](#numbers-and-byte-inputs)). Durations that the API counts in whole seconds must be whole numbers of seconds within their API range: `0` to `2³² − 1` seconds for `wait`, and `0` to `2⁶⁴ − 1` seconds (or the language's maximum exact integer if lower) for retention age and delete-on-empty `min_age`. Client and batching durations (`connection_timeout`, `request_timeout`, `min_base_delay`, `max_base_delay`, `linger`) may be fractional where the language represents durations as floating-point seconds, subject to their valid ranges in [Settings and constants](#settings-and-constants). Apart from flooring sub-second precision on wall-clock date inputs for whole-second metrics parameters (D4), the SDK never rounds, truncates, wraps, or clamps a caller-supplied number.

## E. Errors

1. **Base type.** Every error raised by the SDK, including `ServerError`, implements or extends a single base type `S2Error` (see [Languages](#errors)), allowing callers to catch all SDK errors with one type.
2. **Server errors.** Every non-2xx HTTP response—including a session's terminal frame or SSE `error` event—fails with a `ServerError` carrying that status code, regardless of its body; a terminal frame or SSE `error` event carrying a status outside `300`–`599` fails with `ProtocolError`. When the body matches the API's error JSON object, the `ServerError` exposes `code` and `message` verbatim. Otherwise it has no code, and `message` is the body text—or the decimal HTTP status if the body is empty, not valid UTF-8, or unreadable. Status and code are the programmatic contract; `message` is for humans and must never be parsed. Two response kinds have dedicated error types named for the condition rather than the HTTP status, used whenever the body contains the expected payload (otherwise a plain `ServerError` is returned). Each dedicated type is also a `ServerError` in each language's idiom (see [Languages](#errors)):
   - **a.** A `412` whose body contains `seq_num_mismatch` produces a `SeqNumMismatchError` exposing `assignable_seq_num` (the sequence number the stream will assign next); one whose body contains `fencing_token_mismatch` produces a `FencingTokenMismatchError` exposing `current_fencing_token` (the stream's current fencing token).
   - **b.** A `416` whose body contains `tail` produces a `ReadUnwrittenError` exposing `tail`.
3. **SDK errors.** Only `ServerError` carries an HTTP status or error code. SDK-originated errors are distinguished solely by type, chosen based on the operation that failed—never by inspecting error message text:

   | Error | When raised | Definite (on a call with side effects) | Retryable |
   |---|---|---|---|
   | `ValidationError` | Input fails validation (A2, A5, C1, C2, C3, C4, C9, D4, D5) | yes | no |
   | `ConnectError` | A connection cannot be established, or `connection_timeout` elapses (T7a) | yes | yes |
   | `ConnectionLostError` | A request or connection breaks, or a response ends prematurely (T9) | only if nothing unacked was sent, or the server reports it unprocessed | yes |
   | `TimeoutError` | `request_timeout` elapses for a response, upload progress, or ack (T7b, T7c), or a read session's heartbeat timeout elapses (T7d) | no | yes |
   | `ProtocolError` | A response or session message cannot be decoded, is malformed (such as a terminal frame with an invalid status, E2), or violates an SDK check | no | no |
   | `ClientClosedError` | A call is made on a closed client (C12) | yes | no |
   | `SessionClosedError` | The session closed or began closing before handling the call (E6) | yes | no |
   | `AbortedError` | A session is aborted (A8c, R9d), a call is canceled (T8), or a unary call is in flight when the client closes (C12) | no | no |
   | `IndefiniteAppendError` | A definite final error on an append follows an earlier attempt that may have taken effect (E5) | no | same as its inner error |

   On an operation without side effects (such as a read) or on `submit()` (which atomically accepts nothing on failure, A4c—including Go's `Submit()` + `Wait()` pair, A4), every error is definite (Terms).
4. **Classification.** Every error exposes two public predicates: `has_no_side_effects`, which is true when the error is definite (Terms), and `is_retryable`, which is true when the underlying cause may be transient so repeating the request might succeed (whether repeating is safe is governed by `has_no_side_effects` and T2).
   - **a.** For `ServerError`, both predicates come from [`error-codes.json`](../s2/v1/error-codes.json). Each SDK maintains a complete copy verified against that file through its `s2-specs` submodule.
   - **b.** A server error code applies only when it arrives with the HTTP status listed for it in `error-codes.json`. Otherwise the error is classified as having no code.
   - **c.** A `ServerError` with no code or an unrecognized code is retryable based on status alone: `408`, `429`, `500`, `502`, `503`, and `504`.
   - **d.** For SDK errors, both predicates come from the table in E3.
   - **e.** Every `412` response is definite and non-retryable, regardless of its body, because the server validates append conditions before mutating the stream.
5. **Wrapping.** A definite final error guarantees no side effects only for the last attempt. If an earlier attempt of the same append may have taken effect, the append fails instead with `IndefiniteAppendError` wrapping that final error, which is indefinite. This applies to unary `append()` and to every batch in an append session (A6b).
   - **a.** Wrapper errors (`IndefiniteAppendError` and `SessionClosedError`) are distinct from their inner error: they do not copy its status or code, and they do not match as its type.
   - **b.** The inner error is exposed through a typed accessor: `final_attempt_error` on `IndefiniteAppendError`, and `failure` on `SessionClosedError`. In languages where type matching does not traverse error causes (Python, TypeScript, Rust), the inner error is also set as the wrapper's cause; in Go, where `errors.As` traverses `Unwrap`, it is not (see [Languages](#errors)).
6. **`SessionClosedError`.** Indicates that a session closed or started closing without performing the requested operation, and never will: an append session will not append the record or batch, and a read session will not deliver another record or catch up. It is definite. Its message includes the session failure's message, if any. Its `failure` accessor returns the session's failure, or is empty if the session had none when the call failed.
7. **Messages.** Every error message identifies the operation that failed, the basin and stream names (when applicable), and the number of attempts made if more than one.
8. **Unobserved errors.** Errors that the caller never awaits—such as an unawaited ticket or `caught_up()` future—must not produce unhandled-rejection or unretrieved-exception warnings, and must not be logged above debug level.

## T. Retries, timeouts, and canceling

1. **Retry policy.** Aside from planned handovers (T6), an attempt is retried if and only if its error is retryable (E4), repeating it is safe (T2), and retry budget remains (T3). Otherwise that error becomes the final error: a unary call fails with it, and a session closes (A8b, R9c). The final error is the last attempt's own error, wrapped only as required by E5; the SDK never replaces it with a "retries exhausted" error.
2. **Safety of retrying.**
   - **a.** Reads and state-setting operations (ensure, reconfigure, delete, revoke, `set_default_location()`) are always safe to repeat.
   - **b.** Creating a basin or stream sends a request token in `s2-request-token`, generated once per call and reused across all attempts, so it is always safe to repeat.
   - **c.** Appends are retried according to `append_retry_policy`: under `all`, always; under `no_side_effects`, only after a definite error or when the failed attempt sent no unacked batch (or, for unary `append()`, before the request was sent).
   - **d.** Any other non-idempotent operation without a request token, such as issuing an access token or creating a network policy, is safe to repeat only after a definite error.
3. **Budget and backoff.** The retry budget is `max_attempts`, including the initial attempt. The delay before retry *n* (counting from 0) is `base + random(0, base)`, where `base = min(min_base_delay × 2ⁿ, max_base_delay)`. On a unary call, a valid non-negative integer `retry-after-ms` response header raises the delay to at least that value (and is not capped by `max_base_delay`); a malformed header value is ignored.
4. **Attempt header.** Every request sends `s2-attempt`, set to one more than the number of retries consumed from the current budget (as counted by T3 and T5a): `1` on a first try, `2` on retry 0, and so on.
5. **Sessions.**
   - **a. Restoring the budget.** Each ack on an append session, and each non-empty batch or heartbeat on a read session, resets the retry budget: the active attempt counts as attempt 1 again, and the next retry is retry 0 (T3).
   - **b. Read retry policy.** A read session's `retry_policy` is `budgeted` or `unbounded`. An `unbounded` session ignores `max_attempts` and retries every retryable error using the backoff in T3.
6. **Connection retirement and handover.** When the server signals a planned disconnect, a session moves to a new request on a different connection without consuming retry budget or backing off, provided the previous request reset the budget at least once (T5a). Otherwise the move counts as a retry (T1), so a sequence of requests that fail immediately backs off and gives up under T3. An append session with nothing unacked and nothing queued to send does not open a new request until the next batch is ready (A3); a read session does not open a new request until iteration next pulls from the transport (R5, R6b). Upon receiving (or, on a read session, decoding) reconnect advice, `server_draining`, or `hot_server`, the SDK immediately retires that connection—opening no new requests on it (for any session or call), letting requests already on it finish, and then closing it.
   - **a. Reconnect advice.** A read session ends the advised attempt immediately (T10) and moves to a new request on a different connection per T6 (immediately if already waiting for a record, such as after a heartbeat). Any tail reported on the advised message still takes effect through delivery of that message's records—resolving any pending `caught_up()` if `next_seq_num` reaches that tail—before the session transitions out of the open state for the handover, so `is_caught_up` is false once that final record or heartbeat is returned (R7c, R8). An append session stops starting new batches on the advised request, finishes any partially written input frame, half-closes its request stream, and hands over to a new request as soon as everything sent on the old request has been acknowledged—without needing to wait for the old response's trailing end-of-stream ([`s2s.md`](../s2/v1/s2s.md)). A session performs an advised handover at most once every 60 s; during the cooldown, the connection is still retired for new requests (T6), while the session ignores the advisory and continues using its active request normally.
   - **b. Draining.** On `server_draining`, a session moves to a new request as described in T6. An append session resends every batch the old request did not acknowledge, since none of them took effect.
   - **c. Unary calls and `hot_server`.** On a unary call, `server_draining`—and on both unary calls and sessions, `hot_server`—retires the connection as T6 describes, but is otherwise handled as an ordinary retryable error (consuming retry budget and backing off, unless an append session has nothing unacked or queued to send, T9).
7. **Timeouts.** Each timeout applies within a single attempt—not across the overall call—and fails that attempt: `connection_timeout` fails with `ConnectError`, and the others fail with `TimeoutError`.
   - **a. Connection.** `connection_timeout` bounds establishing a connection, including TLS.
   - **b. Request.** `request_timeout` bounds how long the server takes to respond: on a unary call, from sending the request until the entire response is received; on a session, until a `200` response begins, or until the entire response is received if non-2xx. A unary read with `wait` has a timeout of `request_timeout + wait`.
   - **c. Ack.** On an append session, `request_timeout` also bounds upload stalls and ack latency. Only the oldest unacked batch is timed: while its bytes are still being sent, the attempt times out if `request_timeout` elapses with no upload progress since the established attempt first attempted to write it (including waiting for initial HTTP/2 send flow-control credit—distinct from `Sent`, which governs retry safety), it became the oldest unacked batch, or its last upload progress, whichever is latest; once it is the oldest unacked batch and its last byte has been sent, the attempt times out if `request_timeout` elapses without an ack, starting from whichever of those two events occurred later.
   - **d. Heartbeat.** On a read session, an active request times out if 20 s elapse after waiting on the transport begins without receiving a server message (including heartbeats) or forward response byte progress. Because reading is on-demand (R6b), the timer is not armed while the caller is consuming records from the current batch or parked between deliveries before waiting on the transport. Once armed by a pending transport read, canceling a wait (T8) neither disarms nor restarts the timer—only a decoded server message or forward response byte progress resets it—and any messages or response bytes already buffered by the transport are always read before checking whether the deadline has expired.
8. **Canceling.** Canceling a unary call stops it immediately—including while waiting to retry—and does not retry it; a canceled `append()` may have taken effect. Canceling a wait on a session (in `submit()`, `flush()`, `close()` except where canceling `close()` is the language's abort trigger, awaiting a ticket, `caught_up()`, or reading the next record) cancels only that wait: no outcome changes, the session continues, and a subsequent wait produces what the canceled wait would have (A4c, A8a, R9b). Canceling a session's abort trigger aborts the session instead (A8c). How cancellation is triggered and surfaced depends on the language (see [Languages](#canceling-a-call)).
9. **How a response ends.** A response ends cleanly when it finishes at a message boundary without a terminal frame (on `s2s`) or with a `[DONE]` event (on SSE). When an attempt's response ends, the SDK applies the first matching row:

   | Response ending | Append session | Read session | Unary call |
   |---|---|---|---|
   | Non-2xx status, well-formed terminal frame, or SSE `error` event | if nothing is unacked or queued to send and the error is retryable: retire connection if `server_draining`/`hot_server` (T6), no error, no budget used (A3); otherwise `ServerError` (E2) | clean stop if required by R11d; otherwise `ServerError` (E2) | `ServerError` (E2) |
   | Clean end after reconnect advice the session acted on, with all batches sent on that attempt acked | handover (T6a) | — | — |
   | Clean end with nothing unacked or queued to send | no error, no budget used (A3) | — | — |
   | Clean end with unacked or queued batches | `ConnectionLostError` | — | — |
   | Clean end with any stop option (`count`, `bytes`, `until`, or `wait`) set | — | continue with refreshed `wait` if required by R11c; otherwise iteration finishes (R9a) | — |
   | Clean end with no stop option set | — | `ConnectionLostError` | — |
   | Truncated inside a message or body, SSE EOF without `[DONE]`, or connection breaks | no error, no budget used (A3) if nothing unacked or queued to send; otherwise `ConnectionLostError` | `ConnectionLostError` | `ConnectionLostError` |
   | Complete message or body that fails to decode | `ProtocolError` | `ProtocolError` | `ProtocolError` |

10. **Ending an attempt.** An attempt ends when its error arrives, when a handover leaves it (T6), or when its session closes or aborts. Ending an attempt resets its underlying request unless the response has already ended—including after a clean append close (A8a) or handover (T6a), even if the request stream was half-closed first—so the session never waits on a trailing response end-of-stream or leaves a pooled stream slot held open (C13). After an attempt ends, it sends nothing further, anything it receives is ignored, and all timers and waits it started are stopped. Events received (or, on a read session, decoded) before it ended take effect first, in order: acks settle their batches (A6), and fetched records are delivered (R9a, R9c). An attempt has at most one error—the first to arrive; any later error is ignored.

**Example.** An append session using the default `max_attempts` of 3, showing the `s2-attempt` header sent on each request:

| Event | `s2-attempt` |
|---|---|
| A batch is submitted; the session opens its first request | 1 |
| The batch is acked; the budget resets, and this request counts as attempt 1 | — |
| Connection is lost; the session opens a new request after retry 0's delay | 2 |
| That request breaks before any ack; the session opens a new request after retry 1's delay | 3 |
| An ack arrives; the budget resets, and this request counts as attempt 1 | — |
| Server advises a reconnect; the handover uses no budget | 1 |
| That request ends immediately with `server_draining` before any ack; the move counts as retry 0 | 2 |
| That request breaks before any ack; retry 1 | 3 |
| That request breaks before any ack; all 3 attempts have failed, so the session fails with `ConnectionLostError` | — |

## A. Appending

This chapter covers unary `append()` and two continuous append abstractions: `AppendSession`, which accepts batches, and `Producer`, which accepts individual records, batches them, and owns an `AppendSession` (A3b). Rules A3 through A10 apply to both `AppendSession` and `Producer` (referred to together as sessions in this chapter), except where a rule names one specifically.

1. **Unary append.** `append()` sends one batch and returns its ack, retrying according to `append_retry_policy` (T2c). An ack contains three positions: `start` (the first appended record), `end` (one past the last appended record), and `tail` (the stream's tail, which may be past `end` if concurrent appends occurred). Session acks contain the same three positions. An ack missing any of these positions, whose `end.seq_num < start.seq_num` or `end.seq_num − start.seq_num` does not match the batch's record count, or (on a session) that arrives with no sent unacked batch or overlaps an earlier ack (`start.seq_num < previous end.seq_num`) fails with `ProtocolError`.
2. **Validation.** `append()` and `submit()` validate records and batches before doing anything else—before sending and before waiting for capacity. Invalid input fails with `ValidationError`, accepts nothing, and leaves the session open. An input is invalid if it contains:
   - **a.** a record exceeding 1 MiB, or exceeding `max_batch_bytes` on a `Producer`;
   - **b.** an empty batch, a batch with more than 1000 records, or a batch exceeding 1 MiB;
   - **c.** a fencing token exceeding 36 bytes (on a batch, in `fence()`, or in a `Producer`'s or `Batcher`'s options when opened);
   - **d.** a record timestamp or a batch, `Producer`, or `Batcher` `match_seq_num` that violates D5.
3. **Opening.** A session is opened from a stream handle with its options. Opening sends no request; the session opens its first request when it has a batch to send. A response that ends cleanly, breaks, or ends with a retryable non-2xx status or terminal error while nothing is unacked or queued to send is not an error and uses no budget (T9); the session opens a new request when the next batch is ready. An error on any attempt—including the first—is handled under T1 and surfaces through tickets and the session's failure.
   - **a.** `AppendSession` options are `max_unacked_bytes`, `max_unacked_batches`, and an optional `stream_config` (C4).
   - **b.** `Producer` options are batching (`linger`, `max_batch_records`, `max_batch_bytes`), an optional `fencing_token`, an optional starting `match_seq_num`, `max_unacked_bytes`, and an optional `stream_config` (C4). The `Producer` opens and owns its `AppendSession`; closing or aborting the `Producer` closes or aborts that session.
4. **Acceptance.** `submit()` waits until capacity is available for the record or batch, accepts it, and returns a ticket. Where an SDK splits acceptance into two steps (see [Languages](#accepting-a-record-or-batch)), rules about `submit()` apply to the pair.
   - **a.** Capacity is bounded by `max_unacked_bytes` and, on `AppendSession`, `max_unacked_batches`. An accepted record counts against capacity until its batch has an outcome, including while a `Producer` is still batching it. Capacity is granted FIFO (first come, first served).
   - **b.** Records are ordered by acceptance.
   - **c.** `submit()` is atomic and never partially accepts. A canceled `submit()` accepts nothing. Any `submit()` waiting for capacity when the session begins closing or closes—or called afterward—fails immediately with `SessionClosedError`. `flush()` and `close()` never fail with `SessionClosedError`.
5. **Batching.** Both `Producer` and `Batcher` group records into batches under these rules. A batch is completed when it becomes full or when `linger` elapses after its first record arrives, whichever comes first. A batch never exceeds `max_batch_records` or `max_batch_bytes`. If `fencing_token` is set, every batch carries it. If a starting `match_seq_num` is set, the first batch carries that value, and each subsequent batch carries the previous batch's `match_seq_num` plus its record count; if accepting a record would cause its batch's `end` sequence number (`match_seq_num` plus the count of records including this one) to exceed the representable integer range (D5), a `Producer` rejects `submit()` with `ValidationError` without accepting the record, and a `Batcher` ends with `ValidationError`.
   - **a. `Producer`.** A record arrives when accepted (A4), and each completed batch is submitted immediately. If a `submit()` waits for capacity while a non-empty partial batch is buffered and no other batch is unacked (A4a), the `Producer` completes and submits that partial batch immediately rather than waiting for `linger`.
   - **b. `Batcher`.** Transforms a sequence of records into a sequence of batches (see [Languages](#batcher)). It accepts the same batching options as `Producer`, plus `fencing_token` and a starting `match_seq_num` (A3b). It does not read another input record while a completed batch is waiting to be consumed. When its input ends cleanly, it emits any partial batch immediately and ends. If its input fails, or yields a record exceeding 1 MiB or `max_batch_bytes` or carrying an invalid timestamp (D5), it terminates immediately with that error (or `ValidationError`) and drops its partial batch. If a record would overflow `match_seq_num` (A5), it emits any valid partial batch whose `end` reached the maximum and then terminates with `ValidationError`. If the consumer stops pulling batches, it stops and drops its partial batch. It never drops records in any other case.
6. **Outcomes.** Each batch has a single outcome shared by all its records, since batches are atomic. On success, an `AppendSession` ticket resolves to the batch's ack, and a `Producer` ticket resolves to a `RecordAck` containing the record's own `seq_num` and `batch` (its batch's ack). Once settled, an outcome never changes; where a ticket can be awaited multiple times, every await returns the same outcome.
   - **a.** If the final error came from the server, it applies to the oldest unacked batch, provided that batch was sent in the final attempt: that batch is **rejected**. On an abort, no batch is rejected, and `AbortedError` takes the place of the final error. Each batch receives the outcome from the first matching row:

     | Batch state | Outcome |
     |---|---|
     | Acked | the ack |
     | Rejected | the final error |
     | Behind a batch rejected with a definite error | `SessionClosedError` |
     | Never sent in any attempt, or still being batched by a `Producer` | `SessionClosedError` |
     | Any other | the final error |

   - **b.** Any definite error outcome is then wrapped in `IndefiniteAppendError` if an earlier attempt that sent the batch may have taken effect (E5).
   - **c.** Returning `SessionClosedError` for batches behind a definite rejection relies on the server guaranteeing that no later batch in the session becomes durable after a definite rejection. On transports without a server-side session, the SDK enforces this guarantee itself (C5).
   - **d.** A `Producer` never overrides an outcome reported by its `AppendSession`.
7. **Flushing.** `flush()` (on `Producer` only) takes a single cut point in the acceptance order (A4b): every record accepted before that cut is covered by `flush()` and any partial batch containing them is submitted immediately, while no record accepted after that cut is co-batched with covered records. `flush()` returns once every covered record has been acknowledged; otherwise it fails with the session's failure. A failure affecting only later records does not fail `flush()`, nor does closing the `Producer`. Any `submit()` whose acceptance completed before `flush()` began (A4) is covered; concurrent `submit()` calls may fall on either side (consistently for both coverage and batching). Calling `flush()` after the `Producer` has closed succeeds if all accepted records were acknowledged, and otherwise fails with the session's failure (A4c).
8. **Closing.** A session closes for one of three reasons:
   - **a. Caller-initiated close (`close()`).** Calling `close()` puts the session into the closing state: it stops accepting new work, immediately submits any partial batch, and waits until all accepted work has an outcome (and for nothing else). The session then closes, ends its active request (T10), and `close()` returns—or fails with the session's failure. Every call to `close()` that completes returns the same result.
   - **b. Terminal failure.** A final error closes the session immediately, whether or not `close()` was already called, and settles every batch per A6.
   - **c. Abort.** Closing the client (C12), or a language-specific abort trigger (see [Languages](#ending-a-session)), closes the session immediately, whether or not `close()` was already called, and settles every batch per A6; once the session has already closed, aborting has no effect. The session's failure is an indefinite `AbortedError`, unless no accepted record or batch was awaiting an outcome—in which case nothing was lost and the close is clean. `AbortedError` carries the abort trigger as its cause in languages where errors have causes. Canceling any other call stops only that call's wait (T8).
9. **Failure.** A session's failure is its final error—wrapped in `IndefiniteAppendError` if the final error is definite and any batch's outcome was wrapped (A6b)—or `AbortedError` (A8c). A `Producer`'s failure is that of its owned session, or `AbortedError`.
10. **State.** Sessions expose non-blocking progress accessors that are safe to call concurrently from anywhere, at any time (C11). Counters are updated before the corresponding ticket, `flush()`, or `close()` settles so callers observing a settled outcome immediately see its effect:
    - **a. `unacked_bytes`.** Metered bytes currently held against `max_unacked_bytes` capacity (A4a), including records a `Producer` is still batching and (where acceptance is split) bytes reserved by outstanding permits. Returns to `0` once all permits and accepted work have settled (acked, failed, or aborted).
    - **b. `unacked_batches` (`AppendSession`) and `unacked_records` (`Producer`).** Number of accepted batches or records currently awaiting an outcome. Incremented when accepted and decremented whenever the batch settles (acked, failed, or aborted), returning to `0` once all accepted work has settled.
    - **c. `acked_records` (`Producer`).** Cumulative number of records durably acknowledged by the `Producer`. Monotonically non-decreasing, never incremented on failure or abort, and retained after the `Producer` closes.

**Example.** A session accepts batches A, B, and C in order; A and B were sent, and C was not. In each row, the last error is final (not retried). Here, "closed" is `SessionClosedError`, `412` is either error from E2a, "aborted" is `AbortedError`, and `Indefinite(x)` is `IndefiniteAppendError` wrapping `x`:

| Scenario | A | B | C | `close()` |
|---|---|---|---|---|
| Server rejects A with `412` | `412` | closed | closed | `412` |
| Server fails A with `storage` (indefinite) | `storage` | `storage` | closed | `storage` |
| Connection lost | `ConnectionLostError` | `ConnectionLostError` | closed | `ConnectionLostError` |
| Connection lost and retried; A and B resent; server rejects A with `412` | `Indefinite(412)` | `Indefinite(closed)` | closed | `Indefinite(412)` |
| Client closed while A and B await acks | aborted | aborted | closed | aborted |

## R. Reading

This chapter covers `read()`, `check_tail()`, and `ReadSession`. Rules R1 and R2 define read options shared by `read()` and `ReadSession`; rules R5 through R11 apply to `ReadSession`.

1. **Read start.** At most one of `seq_num`, `timestamp`, or `tail_offset`—represented in a type that cannot hold more than one—together with `clamp`. Their semantics are defined by the API. Omitting a start sends no start parameter, causing the API to begin at the tail.
2. **Read stop.** Any combination of `count` and `bytes` limits, an exclusive `until` timestamp, and `wait` (how long to wait for new records). Their semantics and defaults are defined by the API, and the SDK leaves omitted options unset for the server. On a session, `wait` bounds the idle time between records.
3. **Unary read.** `read()` returns a single batch exactly as sent by the server: its records, and the tail if reported. A batch is empty only when a stop condition was met before any record—such as `wait` expiring at the tail—and does not indicate end-of-stream. With `wait` set to 0, a read starting at the tail returns `416` from the API instead (E2b).
4. **Tail.** `check_tail()` returns the stream's tail position.
5. **Opening.** Opening a read session sends no request; the session initiates its first request when iteration first pulls (mirroring A3). An error on any attempt—including the first—is handled under T1 and surfaces through iteration (R9c).
6. **Iteration.** Iterating a read session yields records one at a time, in sequence-number order, delivering each record once. A decoded batch containing a record whose `seq_num` is less than or equal to a previously decoded record's `seq_num`, less than the session's resolved cursor (`next_seq_num`, R7a) once known, or whose successor (`seq_num + 1`) is not representable in the language's sequence-number domain (D4), is rejected atomically: none of its records are delivered, it does not update session state (R7), and the session fails with `ProtocolError` (R9c).
   - **a.** Server batching is a transport framing detail with no semantic meaning (other than atomic rejection of an invalid batch, R6). An SDK may also provide a batch view; it delivers each batch whole, never delivers an empty batch, and follows every rule in this chapter per batch. Heartbeats update session state (R7) without yielding records or batches.
   - **b.** Reading is on-demand with no session-level read-ahead (0 batches decoded ahead) and no background driver task: opening requests, reading and decoding a `200` response body, processing heartbeats, and starting retries or reconnects occur only while iteration is actively requesting the next item. Once iteration starts an attempt, establishing that attempt (connection, request headers, T7a/T7b deadlines, and reading a non-2xx error body) may continue while iteration is not waiting so canceling a wait does not abort the attempt (T8). Byte-level network read-ahead is handled by the transport's flow control (C13d), which must return receive credit only as iteration consumes bytes, and incremental frame decoding must preserve partial-frame state across canceled waits (T8); where a transport read cannot be canceled independently of its underlying stream (see [Languages](#canceling-a-call)), a single read already in flight when a wait is canceled may finish into the session's byte buffer without decoding.
   - **c.** Iteration is single-consumer. The state accessors in R7 are safe to call concurrently from anywhere, at any time, and never block (see [Languages](#read-session-state)).
7. **State.** A read session exposes three accessors, updated before each record is handed to the caller so the caller immediately sees the record's effect. Because reading is on-demand (R6b), state reflects messages and disconnects decoded as of the last pull (not live stream lag or liveness while parked; use `check_tail()` for a fresh tail); closing or aborting the session (R9b, R9d) still clears `is_caught_up` immediately even when not iterating. After iteration ends, they retain their last values, except `is_caught_up` (R7c).
   - **a. `next_seq_num`.** One past the last delivered record's sequence number, or the starting position before any record has been delivered. An unclamped `seq_num` start is known immediately; any other start is resolved by the first valid decoded record batch or heartbeat—to its first record's sequence number, or else its tail (R11b)—and is unknown until then. (A batch containing a record whose successor `seq_num + 1` is not representable under D4 is rejected atomically per R6, preserving `next_seq_num`.) When iteration decodes a heartbeat (or receives a stop-condition `416` carrying a tail, R11d), `next_seq_num` advances to at least that tail's sequence number, since the server had no earlier records to send. Because records removed by trimming or retention are absent, `next_seq_num` can jump forward. Opening a new session at `next_seq_num` delivers what this session would have delivered next.
   - **b. `last_observed_tail`.** The latest tail reported by a valid decoded server message (or stop-condition `416`, R11d), regardless of whether the session has delivered up to it. It never moves backward and persists across attempts. `max(0, last_observed_tail.seq_num - next_seq_num)` is a lower bound on how far behind the caller is (since later batches omit `tail` when the reader falls behind the tail).
   - **c. `is_caught_up`.** True while the session is open with an active request and the most recent message decoded on that request (or a stop-condition `416` carrying a tail, R11d) reported a tail equal to `next_seq_num` (everything up to a reported tail has been delivered, and nothing newer has been observed). A response that ends on a stop condition still counts as open until iteration finishes returning end-of-stream, so its final record (or stop-condition `416` tail) can still catch the session up and keep `is_caught_up` true until iteration ends. On a message carrying reconnect advice that ends the attempt under T6a, the message's tail still takes effect through delivery of that message's records—resolving any pending `caught_up()` if `next_seq_num` reaches that tail—so `is_caught_up` is false once that final record or heartbeat is returned. Once the session closes (other than a caught-up stop condition before iteration finishes returning end-of-stream, R9) or iteration ends, `is_caught_up` is false.
8. **Catching up.** `caught_up()` returns a future that resolves once the session is caught up: immediately if it is currently caught up, or the next time it becomes caught up.
   - **a.** It resolves to the tail position the session caught up to, and its value never changes.
   - **b.** It does not drive reading; the caller must continue iterating the session while it is pending, including on an empty stream (R6b).
   - **c.** It remains pending across retries and handovers.
   - **d.** If the session closes before catching up, it fails immediately with the session's failure, or with `SessionClosedError` if the session closed cleanly—including when closed or aborted (R9b, R9d) while not being iterated.
9. **Closing.** A read session closes once, for one of four reasons. Under R9a or R9c, the close reason locks in once all already-fetched records have been delivered (immediately if none are buffered); until then, R9b or R9d closes the session immediately instead, even when not being iterated. Once locked in, a later close or abort does not change the close reason or failure: under R9c—or under R9a when not caught up—the session closes immediately and fails pending `caught_up()` calls per R8d; under R9a when caught up, `is_caught_up` stays true (and `caught_up()` resolves to that tail) until iteration finishes returning end-of-stream (R7c) or a later close or abort ends observation, at which point `is_caught_up` becomes false and subsequent `caught_up()` calls fail with `SessionClosedError` (while iteration still finishes cleanly):
   - **a. Stop condition.** `count`, `bytes`, or `until` was reached, or `wait` expired, signaled by the server ending the response cleanly (T9, except as continued by R11c), by remaining `count` or `bytes` reaching `0` (R11c), or by a stop-condition `416` (R11d). All already-fetched records are delivered first; iteration then finishes without error.
   - **b. Caller-initiated close.** The caller closes the session (see [Languages](#ending-a-session)). Any fetched but undelivered records are dropped, and any wait for the next record that is pending then, or started afterward, fails with `SessionClosedError` (E6): only a stop condition ends iteration cleanly. (In TypeScript, pending and subsequent reads on a canceled `ReadableStream` resolve as done per the Streams standard.) Canceling a wait for the next record stops only that wait: no records are lost and the session stays open.
   - **c. Terminal failure.** All already-fetched records are delivered first; iteration then fails once with the session's failure and terminates (see [Languages](#ending-a-session)).
   - **d. Abort.** Code outside the iteration loop aborts the session—by closing the client (C12), or via the session's own abort trigger where the language provides one (see [Languages](#ending-a-session)). Any fetched but undelivered records are discarded so they are never delivered, and pending or subsequent iteration fails once with a definite `AbortedError` (Terms, E3), even when nothing was fetched, since finishing cleanly would look like a stop condition.
10. **Failure.** A read session's failure is its final error or `AbortedError` (R9d). Reads have no side effects, so nothing is wrapped.
11. **Resuming across attempts.** Subsequent requests after a retry or handover (T1, T5, T6) are transparent to iteration.
    - **a.** Each new request resumes at the first unfetched record, so no record is duplicated or skipped.
    - **b.** A start specified by `timestamp`, `tail_offset`, or an omitted start—as well as `clamp`—is resolved once. Until the first decoded record batch or heartbeat resolves an absolute position, retries resend the original start and `clamp` unchanged; afterward, subsequent attempts resume by `seq_num` without `clamp` and never re-evaluate the original start.
    - **c.** `count` and `bytes` apply across the entire session, not per request, so each new request asks only for the remaining budget; if remaining `count` or `bytes` reaches `0` after fetching records, the stop condition is met and the session closes cleanly (R9a) rather than opening another request. If `wait` was set, a new request sets it to `max(0, wait - elapsed)` rounded up to whole seconds, where `elapsed` is the time since the session last fetched a record—or, if none has been fetched yet, since it first learned it was at the tail (and `0` if neither has happened yet). Heartbeats do not restart this clock. Only the server ends a session for `wait`; however, if a resumed request sent a reduced `wait` (`<` configured `wait`), fetched at least one record, has not exhausted `count` or `bytes`, and ends cleanly while the recomputed remaining `wait` is still `> 0`, the session continues on a new request with the refreshed `wait` without consuming retry budget or backing off (T9).
    - **d.** Once the start has been resolved, if a new request made with no `wait` remaining—either `wait` is `0` (explicitly or after R11c reaches `0`), or `wait` was omitted with `count`, `bytes`, or `until` set (where the API defaults `wait` to `0`)—receives `416`, the SDK checks any wire `tail.seq_num` against the request's `seq_num` before converting the typed error under D4/E2b: if `tail.seq_num` is less than the request's `seq_num`, the session fails with `ReadUnwrittenError` (E2b) or plain `ServerError` when D4 prevents constructing `ReadUnwrittenError`; otherwise the session has reached the tail and closes on a stop condition (R9a) rather than failing. When such a stop-condition `416` carries a representable `tail` (`tail.seq_num >=` the request's `seq_num`), it updates `last_observed_tail` (R7b) and, once previously fetched records have been delivered, advances `next_seq_num` to at least `tail.seq_num` and resolves any pending `caught_up()` to `tail` if `tail.seq_num == next_seq_num` before iteration ends (R7, R8).

**Example.** A stream holds records 0–4 (tail is 5). A read session starts at sequence number 0:

| Step | `next_seq_num` | `last_observed_tail` | `is_caught_up` |
|---|---|---|---|
| Opened; nothing fetched yet | 0 | unknown | false |
| Records 0–4 fetched with tail 5; record 0 delivered | 1 | 5 | false |
| Records 1–4 delivered | 5 | 5 | true |
| Heartbeat received with tail 5 | 5 | 5 | true |
| Another writer appends records 5–6; batch fetched with tail 7 and record 5 delivered | 6 | 7 | false |
| Record 6 delivered | 7 | 7 | true |
| Active request breaks | 7 | 7 | false |
| Reconnected; heartbeat received with tail 7 | 7 | 7 | true |

## M. Managing basins, streams, access tokens, and network policies

All operations in this chapter are unary calls named by verb and resource:

| Operation | Basins | Streams | Access tokens | Network policies |
|---|---|---|---|---|
| Create (M1) | `create_basin()` | `create_stream()` | — | `create_network_policy()` |
| Ensure (M2) | `ensure_basin()` | `ensure_stream()` | — | — |
| Reconfigure (M3) | `reconfigure_basin()` | `reconfigure_stream()` | — | — |
| Delete (M4) | `delete_basin()` | `delete_stream()` | `revoke_access_token()` | `delete_network_policy()` |
| Issue (M5) | — | — | `issue_access_token()` | — |
| List a page (M6) | `list_basins()` | `list_streams()` | `list_access_tokens()` | `list_network_policies()` |
| List all (M7) | `list_all_basins()` | `list_all_streams()` | `list_all_access_tokens()` | `list_all_network_policies()` |

An SDK that groups methods by resource may omit the resource name from the method (for example, Go's `Basins.Ensure()`). Any API call not explicitly covered here, such as `get_basin_config()`, `get_network_policy()`, or `set_default_location()`, is an ordinary unary call retried according to T2.

1. **Create.** Creates a new basin, stream, or network policy, failing with a conflict error if the name or slug is already taken. `create_basin()` and `create_stream()` send a request token (T2b) so a retry does not conflict with an earlier attempt from the same call. `create_network_policy()` carries no request token and retries only definite errors (T2d).
2. **Ensure.** Creates a basin or stream if it does not exist, or updates its config to match the provided config—omitted fields take their defaults (as on create), unlike reconfigure (M3). Returns the resource together with the outcome of the last attempt, read from the `s2-provision-result` header: `created`, `updated`, or `noop`. If the header is missing or has an unrecognized value, a `201` response is interpreted as `created` and any other 2xx status as `updated`. After a retry, a resource created by this call may be reported as `updated` or `noop`.
3. **Reconfigure.** Modifies only the specified fields and returns the resulting config. Each field supports three states: unchanged (the default), set to a new value, or reset to the server's default.
4. **Delete.** Ensures a resource is gone. On any attempt, deletion succeeds if the resource is already being deleted, or if the server returns that resource's own not-found code: `basin_not_found` for `delete_basin()`, `stream_not_found` for `delete_stream()`, `access_token_not_found` for `revoke_access_token()`, and `network_policy_not_found` for `delete_network_policy()`. Any other error—including `409 network_policy_in_use` or a `404` with another or missing code—fails as usual. Revoking an access token is its delete operation. Deleting a basin or stream starts asynchronous deletion; until it completes, the resource is marked as being deleted (M8).
5. **Issue.** Issues an access token and returns it; the server reveals the secret only in this response. The SDK never retains or logs it (L3), and retries only definite errors (T2d).
6. **List a page.** Makes a single request returning one page exactly as sent by the server: the items and `has_more` indicating whether more pages remain. Accepts `prefix`, `start_after`, and `limit`.
7. **List all.** Lazily iterates over all matching resources, fetching the next page only when needed. Accepts `prefix` and `start_after`. Each page fetch is an independently retried unary call, and a failed page terminates iteration with that error. Because creating the iterator is lazy, closing the client (C12) before the first iteration or between pages causes the next page fetch to fail with `ClientClosedError`, while an in-flight page fetch fails with `AbortedError`. When advancing to the next page, `start_after` is set to the key of the last item on the unfiltered server page (`name` for basins and streams, `id` for access tokens, `slug` for network policies), even if all items on that page were filtered out by M8. If the server returns an empty page with `has_more` set, iteration ends rather than looping.
8. **Resources being deleted.** Pages from M6 include basins and streams that are currently being deleted, each marked accordingly. Iteration under M7 excludes them unless `include_deleted` is set.

## F. Forward compatibility

A released SDK continues working against newer server versions.

1. **Parsing.** A response never fails to parse because it contains an unknown field or an unknown enum or union value.
2. **Open enums and tagged unions.** Every enum and externally tagged union (`oneOf`, such as `Metric`, `RetentionPolicy`, `PrivateNet`, and `ResourceSet`) returned by the server is open: an unrecognized value or variant is exposed by its wire name or tag in the language's idiom (such as an `Unknown` variant/type, an open string type, or a dedicated unknown-tag field on a Go struct). A closed type that can merely hold an unlisted value at runtime—such as a TypeScript string or object union without an unknown branch, or a Go struct that silently unmarshals an unknown tag as all-`nil`—does not satisfy this rule. Ensure's result is the sole exception; M2 defines how unknown values are interpreted.
3. **Closed enums.** Every enum sent only by the SDK is closed.
4. **Error codes.** Unrecognized server error codes follow E4c.

## L. Logging

1. **Off by default.** The SDK logs through the language's standard logging facility and emits nothing unless enabled by the application.
2. **Debug or finer.** The SDK logs only at debug level or finer, including for retries, reconnects, and errors. Errors are delivered to the caller as errors, not as log messages.
3. **Secrets.** Record bodies, access tokens, and encryption keys never appear in log output or SDK-generated error messages (distinct from verbatim server or proxy error bodies surfaced per E2). Tokens and keys are also redacted when a client, handle, or settings object is formatted or printed.

## Settings and constants

Every SDK exposes these settings with these canonical names, defaults, and valid ranges (C2):

| Setting | Configured on | Default | Valid range |
|---|---|---|---|
| `endpoints` | client | `a.s2.dev` and `{basin}.b.s2.dev` | valid endpoint pair (C3) |
| `connection_timeout` | client | 3 s | `> 0` |
| `request_timeout` | client | 5 s | `> 0` |
| `max_attempts` | client | 3 | `≥ 1` |
| `min_base_delay`, `max_base_delay` | client | 100 ms, 1 s | `≥ 0`, with `min_base_delay ≤ max_base_delay` |
| `append_retry_policy` | client | `all` | `all`, `no_side_effects` |
| `compression` | client | `none` | `none`, `gzip`, `zstd` |
| `retry_policy` | `ReadSession` | `budgeted` | `budgeted`, `unbounded` |
| `max_unacked_bytes` | `AppendSession`, `Producer` | 5 MiB | `≥ 1 MiB` |
| `max_unacked_batches` | `AppendSession` | none | `≥ 1` when set |
| `linger` | `Producer`, `Batcher` | 5 ms | `≥ 0` |
| `max_batch_records` | `Producer`, `Batcher` | 1000 records | `1` to `1000` |
| `max_batch_bytes` | `Producer`, `Batcher` | 1 MiB | `8` bytes to `1 MiB` |

The following constants are fixed and defined by the cited rules:

| Constant | Value | Rule |
|---|---|---|
| Maximum record size | 1 MiB | A2a |
| Maximum batch size | 1000 records, 1 MiB | A2b |
| Maximum fencing token length | 36 bytes | A2c |
| Minimum session frame payload compressed | 1 KiB | C6 |
| Default concurrent requests per HTTP/2 connection | 50 | C13a |
| HTTP/2 keep-alive ping interval | 20 s | C13b |
| HTTP/2 keep-alive ping timeout | 10 s | C13b |
| Idle pooled connection timeout | 90 s | C13c |
| Minimum interval between acting on reconnect advice | 60 s | T6a |
| Read session heartbeat timeout | 20 s | T7d |

## Server guarantees

Several SDK rules rely on specific server behavior. The table below lists each guarantee, the rules that depend on it, and where it is documented.

| Guarantee | Relied on by | Source |
|---|---|---|
| A batch becomes durable atomically: whole or not at all. | A6 | [Appends](https://s2.dev/docs/concepts/appends) |
| Within a session, batches become durable in the order sent, and once one fails, no subsequent batch becomes durable. | A6a, A6c | [Appends](https://s2.dev/docs/concepts/appends) |
| An append session exceeding the append rate limit is throttled rather than failed. | A4, T7c | [Limits](https://s2.dev/docs/platform/limits) |
| A `412` response means an append condition was not met and nothing was appended. | Terms, E4e | [Concurrency control](https://s2.dev/docs/concepts/concurrency-control) |
| Each HTTP/2 connection supports up to 100 concurrent streams (`SETTINGS_MAX_CONCURRENT_STREAMS = 100`). | C13a | [Limits](https://s2.dev/docs/platform/limits) |
| Every session frame sent by the server stays within the size limits in `s2s.md`, and any valid batch fits within them. | A2, C6, T9 | [`s2s.md`](../s2/v1/s2s.md) |
| An append session's response ends cleanly only after every received input is acked. A read session's response ends cleanly (or emits `[DONE]` on SSE) only on a stop condition. | T9, R9a | [`s2s.md`](../s2/v1/s2s.md), [Read](https://s2.dev/docs/api/records/read) |
| An append session that stops sending after reconnect advice receives an ack for every batch it sent, followed by a clean end. | T6a | [`s2s.md`](../s2/v1/s2s.md) |
| Before emitting any terminal error classified with `side_effects: "none"` in `error-codes.json` (including `server_draining`, `hot_server`, `412`, and mid-session input rejections), the server emits acks for every earlier accepted input that became durable and guarantees that no remaining unacked input can become durable. | A6a, A6c, T2c, T6b, T6c | [`s2s.md`](../s2/v1/s2s.md), [`error-codes.json`](../s2/v1/error-codes.json) |
| A `502` response or terminal frame with code `hot_server` indicates server overload and connection closure without side effects. | T6, T6c, T9 | [`s2s.md`](../s2/v1/s2s.md), [`error-codes.json`](../s2/v1/error-codes.json) |
| A read session waiting at the tail receives a heartbeat at least every 15 s. | T7d | [`s2s.md`](../s2/v1/s2s.md) |
| A heartbeat carries the tail and is sent only when the server has no earlier records left to send. | R7a, R7c, R8 | [`s2s.md`](../s2/v1/s2s.md) |
| With `wait` of 0, a read starting at the tail returns `416`. | R3, R11d | [Read](https://s2.dev/docs/api/records/read) |
| A read with no start parameter begins at the tail. | R1 | [`openapi.json`](../s2/v1/openapi.json) |
| Errors listed with no side effects in `error-codes.json` had none. | Terms, E4a | [`error-codes.json`](../s2/v1/error-codes.json) |
| A unary append rejected for rate limiting includes `retry-after-ms`. | T3 | [`openapi.json`](../s2/v1/openapi.json) |
| Retrying a create request with the same `s2-request-token` does not conflict with the earlier attempt. | T2b, M1 | [`openapi.json`](../s2/v1/openapi.json) |
| Ensuring an existing basin or stream sets its config to the provided one, with omitted fields reset to their defaults. | M2 | [`openapi.json`](../s2/v1/openapi.json) |
| Ensure reports its outcome in the `s2-provision-result` header: `created`, `updated`, or `noop`. | M2 | [`openapi.json`](../s2/v1/openapi.json) |
| Deleting a basin or stream that is already being deleted succeeds. | M4 | [`openapi.json`](../s2/v1/openapi.json) |
| Deleting a non-existent basin or stream returns `404` with code `basin_not_found` or `stream_not_found`. | M4 | [`error-codes.json`](../s2/v1/error-codes.json) |
| Revoking a non-existent access token returns `404` with code `access_token_not_found`. | M4 | [`openapi.json`](../s2/v1/openapi.json), [`error-codes.json`](../s2/v1/error-codes.json) |
| Deleting a non-existent network policy returns `404` with code `network_policy_not_found`. | M4 | [`error-codes.json`](../s2/v1/error-codes.json) |

## Languages

This section specifies how language-dependent rules map onto Python, TypeScript, Go, and Rust. Each entry is tested under the rule ID it cites.

### Numbers and byte inputs

| Language | Sequence number, timestamp, count, or byte limit (D4, D5) | Mutable caller inputs (C10) |
|---|---|---|
| Python | `int` | `bytearray` inputs are copied on acceptance |
| TypeScript | `number`, so the exact integer range ends at `2⁵³ − 1` (`Number.MAX_SAFE_INTEGER`) | `Uint8Array` inputs are copied on acceptance |
| Go | `uint64` | `[]byte` and any reference-held config or option fields are copied when passed (for split `Submit()`/`Wait()`, snapshotted in `Submit()` before validation so `Wait()` uses that snapshot) |
| Rust | `u64` | None: inputs are moved in or borrowed only for the duration of the call |

Whole-second API duration fields (`wait`, retention age, and delete-on-empty `min_age`, D5) may be represented either as a duration type (validated at runtime for whole-second precision and range) or as an integer count of seconds with a unit suffix where appropriate (such as Rust's `wait_secs: u32`, `RetentionPolicy::Age(u64)`, and `min_age_secs: u64`, which enforce those constraints statically).

### Accepting a record or batch

Go splits `submit()` (A4) into two steps, and Rust may as well.

| Language | Acceptance (A4) | Validation (A2) |
|---|---|---|
| Python | `submit()` | In `submit()` |
| TypeScript | `submit()` | In `submit()` |
| Go | `Submit()` snapshots the record or batch and any reference-held options (C10), validates the snapshot, and binds it to the returned future; `Wait()` waits for capacity for that snapshot and returns the ticket | In `Submit()` |
| Rust | `submit()`; or `reserve()`, which waits for capacity of a given size, followed by the permit's `submit()`, which accepts immediately and returns the ticket | In `submit()`; when split, `reserve()` validates the reserved size, and the permit's `submit()` validates the rest, releasing the reservation on failure. A batch larger than its reservation is invalid. |

### Batcher

`Batcher` (A5b) is a stream transform in each language's idiom. Go exposes it as a function named for the operation, like `slices.Chunk`.

| Language | Shape (A5b) |
|---|---|
| Python | `Batcher(records, ...)`: an async iterable of batches over an async iterable of records |
| TypeScript | `Batcher`: a `TransformStream` from records to batches |
| Go | `Batch(ctx, records, opts)`: an `iter.Seq2[*AppendInput, error]` over a `<-chan AppendRecord` closed by the caller when input ends. Breaking out of the loop stops it; canceling `ctx` terminates it with `*AbortedError` wrapping `ctx.Err()` (E1, T8). |
| Rust | `Batcher`: a `Stream` of batches over a `Stream` of records |

### Ending a session

| Language | Close append session / `Producer` (A8a) | Abort append session / `Producer` (A8c) | Close read session (R9b) | Abort read session, other than client `close()` (R9d) |
|---|---|---|---|---|
| Python | `close()`, or exiting `async with` normally or on an `Exception` | Cancel `close()`, or exit `async with` on any non-`Exception` `BaseException` (`CancelledError`, `KeyboardInterrupt`, `GeneratorExit`) | `close()`, or exit `async with` | None |
| TypeScript | `close()`, `writable.close()`, `readable.cancel()`, or `await using` | The `AbortSignal` passed when opening the session, or `writable.abort()` | `cancel()` (it is a `ReadableStream`), or `await using` | The `AbortSignal` passed when opening the session |
| Go | `Close()` | Cancel the `context.Context` passed when opening the session | `Close()` | Cancel the `context.Context` passed when opening the session |
| Rust | `close()` | `abort()`, or drop it | Drop it | None |

In TypeScript, a `ReadSession` is a `ReadableStream` with `highWaterMark: 0` that enqueues at most one record (or batch) per `pull()` (R6b, R7) and has no per-read `AbortSignal` (T8); reads on a canceled stream resolve as done (R9b), and an errored stream rejects subsequent direct `reader.read()` calls with the same error while `for await` exits on the first rejection (R9c, R9d). Because `ReadableStream` has no dequeue hook, R7 state updates when `pull()` enqueues the record for the pending read (so detaching a reader via `reader.releaseLock()` while a `pull()` is already in flight and then canceling without reading again leaves that enqueued record counted in `next_seq_num`).

### Read session state

In Rust, iterating a `ReadSession` requires `&mut self`, so `ReadSession` also exposes a cloneable, thread-safe state handle (`state()`) providing the R7 accessors (`next_seq_num`, `last_observed_tail`, `is_caught_up`) and `caught_up()` concurrently while another task iterates the session (R6c); `state()` and `caught_up()` still observe client close (`S2::close()`, C12/R9d) and session drop (R9b) immediately even when no task is polling the session.

### Canceling a call

Applies to unary calls and waits on a session (T8).

| Language | Trigger (T8) | Effect seen by caller (T8) |
|---|---|---|
| Python | Cancel the task | Task cancellation (`CancelledError`) |
| TypeScript | The call's `AbortSignal` | `AbortedError` |
| Go | Cancel the call's `context.Context` | `AbortedError` |
| Rust | Drop the future | None (the future is dropped) |

In Go, `net/http`'s `Response.Body.Read` cannot be canceled independently without closing the response body: a Go `ReadSession` may offer no per-read cancellation (a blocked read is interrupted only by ending the session via R9d or R9b), or—if per-read cancellation is offered—allow a single in-flight transport read to complete into the session's byte buffer without decoding (R6b). In either case, Go must check buffered data before checking the T7d heartbeat deadline rather than relying on an unprioritized `select`.

### Errors

| Language | Error named `XError` (E3) | Base type (E1) | Typed server error (E2) | Wrapper cause, in addition to accessor (E5b) | Cause carried by `AbortedError` (A8c, R9d, T8; none for `client.close()`, C12) |
|---|---|---|---|---|---|
| Python | Class `XError`; `TimeoutError` also subclasses the built-in `TimeoutError` | Class `S2Error`, subclassed by every error | Subclass of `ServerError` | `__cause__` | The triggering exception |
| TypeScript | Class `XError`, with `name === "XError"` | Class `S2Error`, subclassed by every error | Subclass of `ServerError` | `cause` | `signal.reason`, or the reason passed to `writable.abort()` |
| Go | Struct pointer `*XError`, matched with `errors.As` | Interface `S2Error`, implemented by every error | Type that `errors.As` also matches as `*ServerError` | None (kept out of the `Unwrap` chain) | The context error, in the `Unwrap` chain so `errors.Is(err, context.Canceled)` holds |
| Rust | Variant `X` of the operation's error enum | `S2Error`, which every operation's error enum converts into | Variant of the operation's error enum carrying `ServerError` | `source()` | None |

In Rust, the E4 predicates are exposed on `S2Error` and the operation error enums; standalone single-kind errors (such as `ValidationError` and `ClientClosedError`) convert into them rather than repeating constant predicates.

### TypeScript stream adapters

`AppendSession` and `Producer` also expose `writable` (accepting the same input as `submit()`) and `readable` (yielding acks). Each write performs a `submit()` (A4) in order when the stream hands it off. A write that fails validation (A2) rejects with `ValidationError` and errors `writable` per the Streams standard, while leaving the underlying session open (already-accepted work and direct `submit()` calls are unaffected). When `writable` is used, writes already handed to `writable` before `close()` or `flush()` begins drain into `submit()` before the session enters `closing` (A8a) or takes the flush cut point (A7), so queued stream writes are covered like completed `submit()` calls; once the session enters `closing`, any further direct `submit()` or stream write reaching the SDK sink fails with `SessionClosedError` (A4c). A write reaching the SDK sink when the session is closing, closed, failed, or aborted fails with `SessionClosedError` (A4c); rejections produced by the native `WritableStream` before sink dispatch (such as `writer.write()` after `writer.close()`, or writes still queued in `writable` when `writable.abort()` is called) retain their native Web Streams error. Closing `writable` or canceling `readable` (for example, breaking out of a `for await` loop over acks) performs a graceful `close()` (A8a): the session stops accepting new work, while already-accepted work still settles. Aborting `writable` aborts the session immediately—even while a write is waiting on `submit()`—and resolves once teardown completes (A8c), carrying any abort reason as `AbortedError`'s `cause`. On failure, `readable` errors with the session's failure.

### TypeScript over `fetch`

When TypeScript sends requests over `fetch` (C5)—including unary calls when sessions use `s2s`—the runtime manages connection pooling, keep-alive, and idle lifetimes, so C13 and moving to a different connection on handover (T6) hold only to the extent the runtime supports them. Because browsers hide redirect status codes, a redirect in a browser environment fails with `ProtocolError` rather than `ServerError` (C8).

## Conformance

An SDK conforms to a revision when it satisfies every rule in that revision. Each SDK's README declares the revision it targets. To keep conformance verifiable:

- Every testable rule has a test in each SDK tagged with its rule ID (such as `A6b`). When a rule has lettered sub-rules, each sub-rule is tagged individually; if the parent rule also states requirements beyond its sub-rules (such as `E2` or `T6`), the parent ID is tagged as well.
- Any testable rule lacking a tagged test counts as unmet.
- Rules whose test cases are data-driven are defined in [`vectors/`](vectors/), and every SDK runs all vectors through its `s2-specs` submodule. Failing any vector counts as its rule being unmet.
- Once a revision is finalized, any later revision lists its changed rule IDs in [Revisions](#revisions) so every SDK knows which tests to revisit.

## Revisions

- **1 (Draft).** Initial revision, in draft until validated across SDK implementations.
