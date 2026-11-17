# S2 SDK spec

**Revision 1.**

What every S2 SDK (Python, TypeScript, Go, Rust) does. Each chapter has a letter and numbered rules. A new revision may renumber them, so cite a rule with its revision: `r1 A4b`.

Two principles:

- **Every error describes the call or record it is attached to.**
- **A session's state describes what the caller has been given, not what the SDK has fetched.**

Names in this spec are canonical: every SDK uses the same words, in its own casing, and may add a unit suffix where the language has no duration type. Where an SDK splits a call in two (Go's `Submit` then `Wait`, Rust's `reserve` then `submit`), the rule applies to the pair.

| Chapter | Covers |
|---|---|
| C | Client: construction, defaults, compression, transports |
| E | Errors |
| T | Retries and timeouts |
| U | Unary calls: `append`, `read`, `check_tail` |
| A | Appending: `AppendSession` and `Producer` |
| R | Reading: `ReadSession` |
| D | Records and inputs |
| M | Managing resources |
| L | Listing |
| F | Forward compatibility |
| G | Logging |

## Terms

- **Unary call, session.** A unary call is one request and one response. A session is a long-running append or read on one stream, which the SDK keeps going across connections.
- **Ack.** The server's confirmation that a batch of appended records is durable.
- **Attempt.** One try at sending a request or batch, on one connection. An operation may take several attempts.
- **Sent.** A request or batch is sent once any of its bytes may have left the SDK. If the SDK cannot tell, it counts as sent.
- **Definite.** An error is definite if the attempt it reports certainly had no side effect. These are definite:
  - a server error whose code the [error-code table](https://s2.dev/docs/api/error-codes) lists with no side effect, arriving with that code's HTTP status;
  - a `412` append condition failure;
  - a failure before the request or batch was sent, or one the server reports as unprocessed (connect failure, `REFUSED_STREAM`, a stream above an HTTP/2 GOAWAY's `last_stream_id`);
  - the closed error (E4).

  Every other error is **indefinite**: the attempt may have taken effect. That includes codes the SDK does not recognize, ack timeouts, failed SDK checks and aborts.
- **Final error.** The error an operation or a session stopped on: the one that was not retried.
- **Wrapped.** A definite final error describes only the last attempt. If an earlier attempt of the same append may have taken effect, the SDK reports `AppendIndefiniteFailureError` holding the final error, and that is indefinite. This applies to unary `append()` and to every batch in a session.
- **Failure.** The error a session ended with.
- **Tail.** The position the server reports as the end of the stream: the sequence number the next record will get, and the timestamp of the last one. The server reports it with records near the end of the stream, and on its own as a **heartbeat** while a read session waits there.
- **Delivered.** A record is delivered when iteration hands it to the caller. Records the SDK has fetched and is still holding are not delivered.
- **Skipped.** A command record (a fence or a trim) that the caller asked to ignore. It is never delivered, and it counts as progress everywhere a delivered record does. The session moves past a skipped record as soon as every record before it has been delivered, so skipped records never hold it up.

## C. Client

1. **Construction.** A client is built from an access token and optional settings. It never reads the environment on its own. A separate, explicit call builds one from `S2_ACCESS_TOKEN`, `S2_ACCOUNT_ENDPOINT` and `S2_BASIN_ENDPOINT`.
2. **Endpoints** default to `a.s2.dev` for the account and `{basin}.b.s2.dev` for basins, over https. A basin endpoint without the `{basin}` placeholder is direct, and the SDK then names the basin in the `s2-basin` header.
3. **Defaults** are the same in every SDK:

   | Setting | Name | Default |
   |---|---|---|
   | Connection timeout | `connection_timeout` | 3 s |
   | Request timeout | `request_timeout` | 5 s |
   | Retry attempts | `max_attempts` | 3, counting the first |
   | Retry base delay | `min_base_delay`, `max_base_delay` | 100 ms, 1 s |
   | Append retry policy | `append_retry_policy` | `all` |
   | Read session retry policy | `retry_policy` | `budgeted` |
   | Compression | `compression` | `none` |
   | Batching linger | `linger` | 5 ms |
   | Batch limits | `max_batch_records`, `max_batch_bytes` | 1000 records, 1 MiB |
   | Unacked bytes limit | `max_unacked_bytes` | 5 MiB (minimum 1 MiB) |
   | Unacked batches limit | `max_unacked_batches` | none |

4. **Compression** is `none`, `gzip` or `zstd`. When it is set, request bodies are compressed and compressed responses are accepted, on unary calls and sessions alike. A session frame is compressed only when its payload is at least 1 KiB.
5. **Transports.** The SDK chooses which protocol carries a call. Every rule in this spec holds on every transport it offers; transports may differ only in speed.
6. **Stream settings.** An encryption key is set on the stream handle and sent as `s2-encryption-key` with every append and read. A stream config, used to create the stream on first use, is sent as `s2-stream-config`; a session sends it on every connection.
7. **Sharing.** A client, and the basin and stream handles made from it, are safe to use from concurrent tasks. Create one client and reuse it.
8. **Closing.** The client owns the connections; basin and stream handles only refer to it. Where the language needs explicit cleanup, the client has `close()`. Closing a client or a stream handle aborts the sessions opened from it (A6, R4d). A call on a closed client fails at once with a named SDK error, which is definite and is not retried.
9. **User agent** is `s2-sdk-<language>/<version>`.

## E. Errors

1. **Server errors.** An error the server returned exposes its HTTP status, its code (when the body has one) and its message, exactly as returned. Status and code are the contract; the message is for people and is never parsed. A `412` carries the expected sequence number or fencing token, and a `416` carries the tail, as typed fields.
2. **SDK errors.** Only errors that came from the server carry a status. Errors the SDK raises (validation, timeouts, transport failures, the closed error, `Aborted`) are told apart by type.
3. **Wrappers.** `AppendIndefiniteFailureError` and the closed error are not the error inside them. They do not copy its status or code, and must not match as its type: in Go keep it out of the `Unwrap` chain. Python's `__cause__`, JavaScript's `cause` and Rust's `source()` are fine, because `except`, `instanceof` and `match` do not follow them. The inner error stays reachable through a typed accessor.
4. **The closed error** is a dedicated type in each SDK (`SessionClosedError`; an enum variant in Rust), not only an error code. It means "not appended, and never will be", so it is definite. That describes only the session's last attempt, so like any definite outcome it can be wrapped (A4b). Its message includes the failure's message, and the failure, if any, is reachable through a typed accessor.
5. **Classification.** Every error has two public predicates: `has_no_side_effects`, true when it is definite, and `is_retryable`.
   - **a.** For a server error, both come from the error-code table. Each SDK's copy of the table is complete and is tested against the published one.
   - **b.** A code counts only when it arrives with the HTTP status the table lists for it. With any other status, the error is indefinite and not retryable.
   - **c.** A code the SDK does not recognize is indefinite, and retryable by status alone: `408`, `429`, `500`, `502`, `503`, `504`.

## T. Retries and timeouts

1. **What is retried.** An attempt is retried when its error is retryable (E5), repeating it is safe (rule 2) and the budget allows (rule 3).
   - **a.** Retryable SDK errors are connect failures, lost or reset connections, and the timeouts in rule 5.
   - **b.** Validation errors, aborts, and responses the SDK cannot decode are never retried.
   - **c.** An authentication failure is not retried, unless the SDK was given a way to refresh its token.
2. **Safe to repeat.**
   - **a.** Reads, and operations that set a state (ensure, reconfigure, delete), retry any retryable error.
   - **b.** Creating a basin or a stream carries a request token in `s2-request-token`, made once per call and reused on every attempt, so it also retries any retryable error.
   - **c.** Appends follow the append retry policy. `all` retries any retryable error. `no_side_effects` retries only when no unacked batch may have taken effect: the error is definite, or nothing unacked was sent.
   - **d.** Any other operation that is not idempotent, such as issuing an access token, retries only definite errors.
3. **Budget and backoff.** `max_attempts` counts the first try. The delay before retry *n*, counting from 0, is `base + random(0, base)`, where `base = min(min_base_delay × 2ⁿ, max_base_delay)`. On a unary call, a `retry-after-ms` response header raises the delay to at least that value.
4. **Sessions.**
   - **a.** Progress restores the whole budget. Progress is an ack on an append session, and any batch from the server on a read session, heartbeats included.
   - **b.** A read session's retry policy is budgeted or indefinite. An indefinite session never gives up on a retryable error: once the budget is spent, it waits `max_base_delay`, with jitter, before each further try.
   - **c.** A planned handover uses no budget. On `server_draining`, or when the server advises a reconnect, a session reconnects at once, without waiting. It acts on advice at most once every 60 s. On a unary call, `server_draining` is an ordinary retryable error.
5. **Timeouts.** Each is a named SDK error (E2) and is retryable.
   - **a. Connection timeout:** establishing a connection, TLS included. It is definite.
   - **b. Request timeout:** a unary call, from sending the request to receiving the whole response. A unary read with `wait` gets the request timeout plus `wait`. For a session it bounds only the wait for the response to begin.
   - **c. Ack timeout:** on an append session, a batch not acked within the request timeout after being sent.
   - **d. Heartbeat timeout:** on a read session, a connection that delivers nothing for 20 s, not even a heartbeat.

## U. Unary calls

`append`, `read` and `check_tail` on a stream, and every call in chapters M and L.

1. **One result or one error.** A unary call returns its result, or raises its final error once T allows no further retry. The request timeout (T5b) applies to each attempt.
2. **`append`** sends one batch and returns its ack.
   - **a.** The batch is validated first, as in A1. An invalid batch is not sent.
   - **b.** An ack holds three positions: `start`, the first record appended; `end`, one past the last record appended; and `tail`, the stream's tail, which is past `end` when others appended at the same time.
   - **c.** Retries follow the append retry policy (T2c). A definite final error is wrapped when an earlier attempt may have taken effect (see Terms).
3. **`read`** returns one batch: its records, the tail when the read is near the end of the stream, and `next_seq_num`.
   - **a.** `next_seq_num` is where to read next: one past the last record the server returned, skipped records included. It is unknown only when the server returned no records and no tail.
   - **b.** A batch can be empty, for example when every record in it was skipped. An empty batch does not mean the end of the stream; `next_seq_num` and the tail say where the caller is.
   - **c.** With `wait`, a read at the tail waits up to that long for new records.
4. **Cancelling** a unary call (Go's context, TypeScript's signal, cancelling the task in Python, dropping the future in Rust) stops it at once, including while it waits to retry. It is not retried. In Go and TypeScript the call reports `Aborted` (A6b); in Python and Rust the caller's own cancellation ends it. A cancelled `append` may have taken effect.

## A. Appending

`AppendSession` and `Producer`.

1. **Validation.** `submit()` rejects an invalid record or batch before waiting for capacity. That includes one over the batch limits (1000 records, 1 MiB). Nothing is accepted and nothing closes.
2. **Acceptance.** `submit()` waits until there is capacity for the record or batch, and only then accepts it.
   - **a.** Capacity is granted first come, first served. An accepted record counts against it until its batch has an outcome, including while it is still being batched.
   - **b.** Records are ordered by acceptance, and batches never exceed their limits.
   - **c.** `submit()` never half-accepts. A cancelled `submit()` accepted nothing, and one still waiting when the session closes raises the closed error.
   - **d.** Every method is safe to call from concurrent tasks.
3. **One failure, then closed.** A session's failure is its final error, wrapped if it is definite and any batch's outcome was wrapped. A Producer's failure is its session's failure, or `Aborted` (rule 6). The session is closed from `close()`, an abort or the failure onward.
4. **One outcome per batch, shared by its records.** A batch is atomic, so each record gets its batch's outcome. An outcome never changes; where a ticket can be awaited more than once, every await gives the same result.
   - **a.** If the final error came from the server, it belongs to the oldest unacked batch, provided that batch was sent: that batch was **rejected**. Each batch gets the first row that fits:

     | Batch | Outcome |
     |---|---|
     | Acked | the ack |
     | Rejected | the final error |
     | Behind a batch rejected with a definite error | the closed error |
     | Never sent, in any attempt | the closed error |
     | Any other | the final error |

   - **b.** A definite outcome is then wrapped if an earlier attempt of that batch may have taken effect.
   - **c.** The closed error behind a definite rejection relies on the server: after it rejects a batch with a definite error, no later batch in the session becomes durable. A transport without a server-side session (such as TypeScript's fetch transport) must give the same guarantee itself: one batch in flight at a time, and nothing sent after a failure.
   - **d.** A Producer never overrides an outcome its session reported.
5. **Calls.**
   - **a.** `submit()` once closed, for any reason: the closed error. Only `submit()` and tickets ever produce it.
   - **b.** `flush()` (Producer only): sends any partial batch at once, then returns once every record accepted before it is acked. Otherwise it raises the failure. A failure that affects only later records does not fail it. Neither does the Producer being closed. A `submit()` that returned before `flush()` began is covered; concurrent ones may fall on either side.
   - **c.** `close()`: stops accepting, sends any partial batch, waits until everything accepted has an outcome, then returns nothing or raises the failure. Every call gives the same result.
6. **Abort** ends the session at once with a named, indefinite `Aborted` failure, and rules 3–5 then apply. With no batch awaiting an outcome, an abort is a clean close, with no failure. Cancelling any other call only stops that call's wait.
   - **a. What aborts:** Python, cancelling `close()` or leaving `async with` on a `BaseException` that is not an `Exception` (a cancellation, `KeyboardInterrupt`, `GeneratorExit`); Rust, dropping the session or Producer, which includes dropping an unfinished `close()`; Go, cancelling the constructor's context; TypeScript, the session's abort signal.
   - **b. The reason:** `Aborted` carries what triggered it as its cause, where the language has one: TypeScript's `signal.reason`, Go's context error, Python's exception. In Go it is in the `Unwrap` chain, so `errors.Is(err, context.Canceled)` holds.
   - **c. What closes:** leaving `async with` any other way, and TypeScript's `await using`, are `close()`.
   - **d. Stream adapters** (TypeScript `readable`/`writable`): a write is a `submit()`, and closing the `writable` is `close()`. Erroring or cancelling either one aborts; leaving a `for await` over `readable` early cancels it. On failure, `readable` errors with the failure.
   - **e. Ownership:** a Producer owns its session, so closing or aborting the Producer does the same to the session. A session belongs to at most one Producer and is used only through it; if the caller closes it anyway, the Producer aborts.
7. **Construction.** A session or a Producer is opened from a stream handle, with one set of options.
   - **a.** A Producer's options are batching (`linger`, `max_batch_records`, `max_batch_bytes`), an optional `fencing_token`, an optional starting `match_seq_num`, and `max_unacked_bytes`. It creates the session it owns.
   - **b.** The fencing token goes on every batch. `match_seq_num` goes on the first batch and rises by each batch's record count.
   - **c.** A batch is sent when it is full or when the linger expires, whichever comes first.

**Example.** A session accepted batches A, B and C, in that order. A and B were sent; C was not. "closed" is the closed error, and `Indefinite(x)` is `AppendIndefiniteFailureError` holding `x`.

| What happened | A | B | C | `close()` |
|---|---|---|---|---|
| Server rejects A with `412` | `412` | closed | closed | `412` |
| Server fails A with `storage` (indefinite) | `storage` | `storage` | closed | `storage` |
| Connection lost, not retried | connection error | connection error | closed | connection error |
| Connection lost; A and B resent; server rejects A with `412` | `Indefinite(412)` | `Indefinite(closed)` | closed | `Indefinite(412)` |

## R. Reading

`ReadSession`.

1. **Iteration yields records**, one at a time, in sequence-number order, each exactly once.
   - **a.** Batches are how the server ships records, and they carry no meaning. An SDK may also offer a batch view; it delivers a batch as a whole, and every rule here applies to it unchanged.
   - **b.** The session fetches ahead by a bounded amount and no further until the caller takes records.
   - **c.** Iteration has one consumer. The accessors in rule 2 are safe to call from anywhere, at any time, and never block.
2. **State.** Three accessors, each updated before the record that changed it is handed over.
   - **a. `next_seq_num`:** the sequence number of the next record the session will deliver or skip. Every record still on the stream between the start and it was delivered or skipped, and none at or after it was. Records already removed by trimming or retention are absent, so sequence numbers can jump. It is known at once for an absolute start that is not clamped; otherwise it is unknown until the first record is delivered or the first tail arrives. It is what a caller checkpoints, and where a new session should start.
   - **b. `last_observed_tail`:** the latest tail the server reported, whether or not the session has delivered up to it. It never moves backward and survives reconnects. `last_observed_tail.seq_num - next_seq_num` is a lower bound on how far behind the caller is.
   - **c. `is_caught_up`:** true when the last thing the server reported is a tail equal to `next_seq_num`: everything up to a reported tail has been delivered or skipped, and nothing newer is known. It turns false when a record beyond that tail is fetched, when records arrive without a tail, and when the connection is lost, until the server reports again.
3. **`caught_up()`** returns an awaitable for the current caught-up state, or the next one if the session is behind.
   - **a.** It resolves with the tail the session caught up to. Once resolved, its value never changes.
   - **b.** It does not read. The caller keeps iterating while it is pending.
   - **c.** It stays pending across reconnects.
   - **d.** If the session ends before catching up, it raises the failure, or the closed error (E4) when there is none.
4. **Ending.**
   - **a. Cleanly:** a stop condition was met (`count`, `bytes`, `until`, or `wait` ran out). Iteration finishes with no error.
   - **b. By the caller:** Python, `close()` or leaving `async with`; Go, `Close()`; TypeScript, cancelling the stream or `await using`; Rust, dropping the session. Iteration finishes with no error. Cancelling a wait for the next record only stops that wait: nothing is lost and the session stays open.
   - **c. With a failure:** every record already fetched is delivered first, then iteration raises the failure once.
   - **d. By an abort:** TypeScript, the session's abort signal; Go, cancelling the constructor's context. It stops a session from outside the code that iterates it. Records fetched but not delivered are dropped, and iteration raises `Aborted` (A6b). Python and Rust have no abort: the consumer ends the session itself (4b).
   - **e.** After the end, `next_seq_num` and `last_observed_tail` keep their last values. `is_caught_up` keeps its last value on a clean or caller end, and is false after a failure.
5. **Reconnects are invisible to iteration.** T4 decides whether one happens.
   - **a.** The session resumes at the first record it has not fetched, so nothing is repeated and nothing is missed.
   - **b.** A start given by timestamp or tail offset is resolved once. After the first response the session resumes by sequence number, never by re-evaluating the original start.
   - **c.** `count`, `bytes` and `wait` cover the whole session, not each connection. Skipped records count against the limits, as they do on the server, and a reconnect asks only for the `wait` not yet spent at the tail.
6. **Errors.**
   - **a.** The failure is the session's final error, or `Aborted` (4d). A read has no side effects, so nothing is ever wrapped.
   - **b.** Reading from beyond the tail is a typed error that carries the tail (E1).

**Example.** A stream holds records 0–4, so its tail is 5. The session starts at 0, and record 2 is a skipped command record.

| Step | `next_seq_num` | `last_observed_tail` | `is_caught_up` |
|---|---|---|---|
| Opened, nothing fetched | 0 | unknown | false |
| Records 0–4 fetched with tail 5; record 0 delivered | 1 | 5 | false |
| Record 1 delivered | 3 | 5 | false |
| Records 3 and 4 delivered | 5 | 5 | true |
| Heartbeat with tail 5 | 5 | 5 | true |
| Record 5 fetched with tail 6, not yet delivered | 5 | 6 | false |
| Connection lost, then restored; heartbeat with tail 6 after record 5 is delivered | 6 | 6 | true |

## D. Records and inputs

1. **Records.** A body is bytes. Headers are an ordered list of name and value pairs, both bytes. An SDK may add string conveniences on top.
2. **Numbers.** A sequence number is an integer. A timestamp is milliseconds since the Unix epoch, exposed as an integer and not as a date type. A position is a sequence number and a timestamp.
3. **Read start** is exactly one of `seq_num`, `timestamp` or `tail_offset`, expressed so that two cannot be given at once. With `clamp`, a start beyond the tail reads from the tail. Without it, that read fails (R6b).
4. **Read stop** combines any of: `count` and `bytes` limits; `until`, a timestamp to stop before; and `wait`, the seconds to wait at the tail for new records.
5. **Command records.** `fence(token)` and `trim(seq_num)` build them, and `is_command_record` recognizes one that was read. All three are public. `ignore_command_records` is applied by the SDK, which skips them.
6. **Metered bytes.** A record's metered bytes are 8, plus 2 per header, plus the length of every header name and value, plus the length of the body. The SDK has a public helper for this, `metered_bytes`, and every byte limit in this spec is in metered bytes.
7. **`check_tail`** returns the tail, as a position.

## M. Managing resources

Basins, streams and access tokens. Every call here is unary (U).

1. **Create** makes a new resource, and fails with a conflict if the name is taken. It carries a request token (T2b), so a retry does not conflict with its own earlier attempt.
2. **Ensure** creates the resource if it is missing, and otherwise applies the given config. It returns the resource and which of three things happened: created, updated, or unchanged.
3. **Reconfigure** changes only the fields it is given, and returns the config as it now stands. Each field has three states: left unchanged, which is the default; set to a value; or reset to the server's default.
4. **Delete** starts deletion. The resource is then listed as being deleted until it is gone (L3).
   - **a.** Deleting a resource that does not exist is an error, unless the caller passes `ignore_not_found`.
   - **b.** On a retry, after an attempt that may have taken effect, "not found" means the delete worked, and the call succeeds.
5. **Issuing an access token** returns the token once. The SDK does not keep it or log it (G4), and retries only definite errors (T2d).

## L. Listing

Basins, streams and access tokens.

1. **`list`** makes one call and returns one page: the items, and whether more remain. It takes `prefix`, `start_after` and `limit`.
2. **`list_all`** iterates every match lazily, fetching the next page only when the caller reaches it. It takes `prefix` and `start_after`. Each page is a unary call with its own retries, and a page that fails ends the iteration with that error.
3. **Deleted resources.** `list_all` leaves out basins and streams that are being deleted, unless `include_deleted` is set.

## F. Forward compatibility

A released SDK keeps working against a newer server.

1. **Parsing.** A response never fails to parse because of a field or value this SDK version does not know.
2. **Unknown values** in a field the server may extend are exposed by their wire name, in the language's idiom: an `Unknown` variant, or an open string type. The type must allow for it: a closed type that can hold an unlisted value at run time is a bug.
3. **Client-only enums**, which are only ever sent, stay closed.
4. **Unknown error codes** follow E5c.

## G. Logging

1. **Off by default.** The SDK logs through the language's usual facility and emits nothing unless the application turns it on.
2. **Levels.** Nothing on a healthy path is logged above debug. Requests, batches and records are debug or finer, and so are retries and reconnects.
3. **Errors are returned, not logged.** A failure goes to the caller. The SDK does not also log it above debug.
4. **Secrets.** Record bodies, access tokens and encryption keys are never logged. Tokens and keys are also masked when a client or its settings are printed.

## Conformance

An SDK conforms to a revision when it follows every rule in it. To make that checkable:

- Every testable rule has a test in each SDK, tagged with its rule id, such as `A4b`.
- A rule without a tagged test counts as not yet met.
- When a revision changes a rule, the Revisions section names it, so each SDK knows which tests to revisit.

## Revisions

- **1.** Initial version.
