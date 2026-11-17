# S2 SDK spec

**Revision 1.**

What every S2 SDK (Python, TypeScript, Go, Rust) does. It describes where the SDKs are headed, not where each one is today.

Every rule, and every choice the spec leaves to an SDK, is held to four standards: correctness, elegance, idiom and efficiency. Why each rule says what it does is in [rationale.md](rationale.md), by rule id.

**Reading the rules.**

- Every rule is a requirement. Where an SDK has a choice, the rule says it may.
- Each chapter has a letter and numbered rules, cited by id, such as `A6b`. Ids are stable: a later revision never renumbers a rule, and never reuses the id of one it removes. Add the revision, as in `r1 A6b`, when the wording matters.
- Names are canonical. Every SDK uses the spec's words, in its own casing and naming idiom (see Languages), and may add a unit suffix where the language has no duration type. `name()` is a call, `Name` is a type, and any other `name` is an accessor, option, setting or value.
- Where an SDK splits a call in two (Go's `Submit()` then `Wait()`, Rust's `reserve()` then `submit()`), rules about the call apply to the pair.

| Chapter | Covers |
|---|---|
| C | Client |
| D | Data: records and read inputs |
| E | Errors |
| T | Retries, timeouts and cancelling |
| A | Appending: `append()`, `AppendSession`, `Producer` |
| R | Reading: `read()`, `check_tail()`, `ReadSession` |
| M | Managing basins, streams and access tokens |
| F | Forward compatibility |
| L | Logging |

Terms and the failure model come first. Settings and constants, Server guarantees, Languages, Conformance and Revisions follow the chapters.

## Terms

- **Unary call, session.** A unary call is one request and one response. A session is a long-running append or read on one stream, which the SDK keeps going across connections.
- **Attempt.** One try at an operation, on one connection. A unary call sends its request once per attempt. A session's attempt is one connection, on which it sends its batches, including any an earlier attempt sent that have no outcome yet.
- **Sent.** A request or batch is sent in an attempt once any of its bytes may have left the SDK during it.
- **Ack.** The server's confirmation that a batch of appended records is durable.
- **Ticket.** What `submit()` returns once it has accepted a record or batch. Awaiting it gives that record's or batch's outcome (A6).
- **Definite.** An error is definite when the attempt it reports certainly had no side effect, and its `has_no_side_effects` (E5) then says so. These are definite:
  - any error from an operation with no side effects, such as a read;
  - a server error whose code the [error-code table](https://s2.dev/docs/api/error-codes) lists with no side effect (E5);
  - a `SeqNumMismatchError` or `FencingTokenMismatchError` (E1a);
  - a failure before the request or batch was sent, or one the server reports as unprocessed (a connect failure, `REFUSED_STREAM`, a stream above an HTTP/2 GOAWAY's `last_stream_id`);
  - `SessionClosedError` (E4).

  Every other error is **indefinite**: the attempt may have taken effect. That includes codes the SDK does not recognize, ack timeouts, failed SDK checks and aborted appends.
- **Wrapped.** Held inside `IndefiniteAppendError`, because the error is definite but an earlier attempt may have taken effect (E3).
- **Final error.** The error an operation or a session stopped on: the last attempt's error, which was not retried.
- **Closed.** A session is open until it closes, once, for one of the reasons A8 or R6 lists. A closed session takes on no new work; what it already holds finishes as those rules say.
- **Failure.** What a session reports as the reason it closed: its final error, wrapped as A9 says, or `AbortedError`. A session that closed cleanly has none.
- **Tail.** The position the server reports as the end of the stream: the sequence number the next record will get, and the timestamp of the last one. The server reports it with records near the end of the stream, and on its own as a **heartbeat** while a read session waits there.
- **Delivered.** A record is delivered when iteration hands it to the caller. Records the SDK has fetched and is still holding are not delivered.

## Failure model

How the SDK handles a failure, told once. This section adds no rule; each step cites the rule that governs it.

An operation makes one or more attempts. An attempt that fails has an error, which is definite if the attempt certainly had no side effect, and indefinite otherwise (Terms). The SDK repeats an attempt only when its error may be temporary, repeating it is safe, and the budget allows (T1). The error it stops on is the final error.

A unary call returns its result, or fails with its final error. A definite final error speaks only for the last attempt: if an earlier attempt of an append may have taken effect, the append fails with `IndefiniteAppendError` holding it (E3).

A session retries the same way, across connections. An append session gives every batch one outcome, its ack or an error, which the batch's records share (A6). Either kind of session closes once, and reports a failure if it ended on an error (A9, R7).

Whatever error reaches the caller says what is safe to do next: `has_no_side_effects`, whether trying again could repeat an effect, and `is_retryable`, whether trying again could succeed (E5).

## C. Client

1. **Construction.** A client is built from an access token and optional settings, and never reads the environment on its own. `from_env()` builds one from `S2_ACCESS_TOKEN`, `S2_ACCOUNT_ENDPOINT` and `S2_BASIN_ENDPOINT`.
2. **Endpoints.** The account endpoint defaults to `a.s2.dev`, and the basin endpoint to `{basin}.b.s2.dev`, over https. A basin endpoint without the `{basin}` placeholder serves every basin, so the SDK names the basin in the `s2-basin` header.
3. **Compression.** `compression` is `none`, `gzip` or `zstd`. Unless it is `none`, request bodies are compressed and compressed responses are accepted, on unary calls and sessions alike. A session frame is compressed only when its payload is at least 1 KiB.
4. **Transports.** An SDK may carry calls over more than one protocol, choosing which itself. Every rule in this spec holds on each transport; transports may differ only in speed.
5. **Encryption key and stream config.** An encryption key is set on the stream handle, and sent as `s2-encryption-key` with every append and read. A stream config, used to create the stream on first use, is sent as `s2-stream-config`; a session sends it on every connection.
6. **Concurrency.** A client, the basin and stream handles made from it, and the sessions opened from those are safe to use from concurrent tasks. The one exception is iterating a read session, which has one consumer (R3c).
7. **Closing.** The client owns the connections; basin and stream handles only refer to it. Where the language needs explicit cleanup, the client has `close()`. Closing a client aborts every session opened from it (A8c, R6d). A call on a closed client fails at once with `ClientClosedError`.
8. **User agent.** `s2-sdk-<language>/<version>`, where the language is `python`, `typescript`, `go` or `rust`.

## D. Data

1. **Records.** An `AppendRecord` has a body, headers and an optional timestamp; the server decides the stored timestamp from it, as the stream's timestamping config says. A `SequencedRecord`, as read, has a body, headers, its `seq_num` and its `timestamp`. A body is bytes. Headers are an ordered list of name and value pairs, both bytes. An SDK may add string conveniences on top.
2. **Numbers.** A sequence number is an integer. A timestamp is milliseconds since the Unix epoch, exposed as an integer and not as a date type. A position is a sequence number and a timestamp.
3. **Read start.** Exactly one of `seq_num`, `timestamp` or `tail_offset`, in a type that cannot hold two. With `clamp`, a start beyond the tail reads from the tail. Without it, the read fails with `ReadUnwrittenError` (E1b).
4. **Read stop.** Any of: `count` and `bytes` limits; `until`, a timestamp to stop before; and `wait`, how long to wait for new records. Their meaning and defaults are the API's, and the SDK leaves any the caller did not set to the API. So `wait` defaults to zero when `count`, `bytes` or `until` bounds the read, and otherwise waits forever. A unary read is always bounded, by the API's own limits if not the caller's, and waits at most 60 s. On a session, `wait` bounds the time between records.
5. **Command records.** `fence(token)` and `trim(seq_num)` build them, and `is_command_record()` recognizes one that was read; all three are public. Reads return command records like any other.
6. **Metered bytes.** A record's metered bytes are 8, plus 2 per header, plus the length of every header name and value, plus the length of the body. `metered_bytes()` is public, and every byte limit in this spec is in metered bytes.

## E. Errors

1. **Server errors.** A `ServerError` exposes its HTTP status, its code (when the body has one) and its message, exactly as returned. Status and code are the contract; the message is for people and is never parsed. Two kinds have types of their own, named for what happened rather than for their status, and each is also a `ServerError`:
   - **a.** A `412` is a `SeqNumMismatchError` with `expected_seq_num`, or a `FencingTokenMismatchError` with `expected_fencing_token`.
   - **b.** A `416` is a `ReadUnwrittenError` with the stream's `tail`.
2. **SDK errors.** Only a `ServerError` carries a status. The SDK's own errors are told apart by type:

   | Error | When | Definite, on a call with side effects | Retryable |
   |---|---|---|---|
   | `ValidationError` | input fails validation (A2) | yes | no |
   | `ConnectError` | a connection cannot be made, or `connection_timeout` passes (T6a) | yes | yes |
   | `ConnectionLostError` | a connection breaks or is reset | only if nothing was sent, or the server reports the request unprocessed | yes |
   | `RequestTimeoutError` | `request_timeout` passes (T6b) | no | yes |
   | `AckTimeoutError` | a batch is not acked in time (T6c) | no | yes |
   | `HeartbeatTimeoutError` | a read session hears nothing (T6d) | — | yes |
   | `ProtocolError` | a response cannot be decoded, or fails an SDK check | no | no |
   | `ClientClosedError` | a call is made on a closed client (C7) | yes | no |
   | `SessionClosedError` | the session closed first (E4) | yes | no |
   | `AbortedError` | a session is aborted (A8c, R6d), or a unary call is cancelled (T7) | no | no |
   | `IndefiniteAppendError` | an append's definite final error follows an attempt that may have taken effect (E3) | no | as its inner error |

   On a call without side effects, such as a read, every error is definite (Terms).
3. **Wrapping.** A definite final error describes only the last attempt. If an earlier attempt of the same append may have taken effect, the append fails instead with `IndefiniteAppendError` holding the final error, and that is indefinite. This applies to `append()` and to every batch in a session (A6b).
   - **a.** A wrapper is not the error inside it. `IndefiniteAppendError` and `SessionClosedError` do not copy their inner error's status or code, and do not match as its type.
   - **b.** The inner error is reachable through a typed accessor: `final_attempt_error` on `IndefiniteAppendError`, and `failure` on `SessionClosedError`. Where the language's cause chain cannot make the wrapper match as the inner type, the inner error is also the wrapper's cause (see Languages).
4. **`SessionClosedError`.** The session closed without doing what was asked, and never will: an append session will not append the record or batch, and a read session will not catch up. It is definite. For an append it describes only the last attempt, so like any definite outcome it is wrapped if an earlier attempt may have taken effect (E3). Its message includes the failure's message, and `failure` is empty when the session closed without one.
5. **Classification.** Every error has two public predicates. `has_no_side_effects` is true when the error is definite. `is_retryable` is true when the failure may be temporary, so the same request might succeed if made again; whether making it again is safe is for `has_no_side_effects` and T2 to say.
   - **a.** For a `ServerError`, both come from the error-code table. Each SDK's copy of the table is complete and is tested against the published one.
   - **b.** A code counts only when it arrives with the HTTP status the table lists for it. Otherwise the error is treated as having no code.
   - **c.** A `ServerError` with no code, or with a code the SDK does not recognize, is indefinite, and retryable by status alone: `408`, `429`, `500`, `502`, `503`, `504`.
   - **d.** For an SDK error, both come from the table in E2.
6. **Messages.** Every error's message names the call it came from, and the basin and stream that call was for, if any.

## T. Retries, timeouts and cancelling

1. **What is retried.** An attempt is retried when its error is retryable (E5), repeating it is safe (T2) and the budget allows (T3). Otherwise that error is the final error: a unary call fails with it, and a session closes (A8b, R6c).
2. **Safe to repeat.**
   - **a.** Reads, and calls that set a state (ensure, reconfigure, delete, revoke), are always safe to repeat.
   - **b.** Creating a basin or a stream carries a request token in `s2-request-token`, made once per call and reused on every attempt, so it is always safe to repeat too.
   - **c.** An append is safe to repeat as `append_retry_policy` says. Under `all`, it always is. Under `no_side_effects`, it is safe only when no unacked batch may have taken effect: the error is definite, or nothing unacked was sent.
   - **d.** Any other call that is not idempotent, such as issuing an access token, is safe to repeat only after a definite error.
3. **Budget and backoff.** The budget is `max_attempts`, counting the first. The delay before retry *n*, counting from 0, is `base + random(0, base)`, where `base = min(min_base_delay × 2ⁿ, max_base_delay)`. On a unary call, a `retry-after-ms` response header raises the delay to at least that value.
4. **Sessions.**
   - **a. Restoring the budget.** Each ack on an append session, and each batch on a read session, heartbeats included, restores the whole budget.
   - **b. Read retry policy.** A read session's `retry_policy` is `budgeted` or `unbounded`. An `unbounded` session ignores `max_attempts`: it retries every retryable error, with the backoff of T3.
5. **Handover.** When the server plans to go away, a session moves to a new connection without using its budget and without backoff.
   - **a. Reconnect advice.** A read session reconnects at once. An append session stops sending on the advised connection, and sends on a new one once the old one has acked everything sent on it ([s2s](../s2/v1/s2s.md)). A session acts on advice at most once every 60 s.
   - **b. Draining.** On `server_draining`, a session reconnects at once. An append session resends every batch the old connection did not ack: none of them took effect.
   - **c. Unary calls.** On a unary call, `server_draining` is an ordinary retryable error.
6. **Timeouts.** A timeout fails the attempt with the error E2 lists for it. Each applies within one attempt, not to a call as a whole.
   - **a. Connection.** `connection_timeout` bounds establishing a connection, TLS included.
   - **b. Request.** `request_timeout` bounds how long the server takes to answer: on a unary call, from sending the request to receiving the whole response; on a session, until the response begins. A unary read with `wait` gets `request_timeout` plus `wait`.
   - **c. Ack.** On an append session, an ack is the answer to a batch, so `request_timeout` bounds it too: a batch not acked within `request_timeout` of being sent times out.
   - **d. Heartbeat.** On a read session, a connection that receives nothing for 20 s, not even a heartbeat, times out.
7. **Cancelling.** Cancelling a unary call stops it at once, including while it waits to retry, and it is not retried. How a call is cancelled, and what the caller then sees, depends on the language (see Languages). A cancelled `append()` may have taken effect.
8. **Attempt header.** Every request carries `s2-attempt`, the attempt's number as T3 counts it: `1` for the first try, and one more for each retry. On a session, the count starts again at `1` whenever the budget is restored (T4a), and a planned handover (T5) does not raise it.

## A. Appending

`append()`, and two ways to append continuously: `AppendSession`, which takes batches, and `Producer`, which takes records, batches them, and owns an `AppendSession` (A8e). Rules A3 to A9 cover both, called sessions in this chapter; where they differ, the rule names one.

1. **Unary append.** `append()` sends one batch and returns its ack, retrying as `append_retry_policy` says (T2c). An ack holds three positions: `start`, the first record appended; `end`, one past the last record appended; and `tail`, the stream's tail, which is past `end` when others appended at the same time. A session's acks hold the same three.
2. **Validation.** `append()` and `submit()` check a record or batch before anything else: before sending, and before waiting for capacity. An invalid one fails with `ValidationError`; nothing is accepted, and the session stays open. These are invalid:
   - **a.** a record over 1 MiB or, on a Producer, over `max_batch_bytes`;
   - **b.** a batch that is empty, or over 1000 records or 1 MiB;
   - **c.** a fencing token over 36 bytes.
3. **Opening.** A session is opened from a stream handle, with its options.
   - **a.** An `AppendSession`'s options are `max_unacked_bytes` and `max_unacked_batches`.
   - **b.** A `Producer`'s options are batching (`linger`, `max_batch_records`, `max_batch_bytes`), an optional `fencing_token`, an optional starting `match_seq_num`, and `max_unacked_bytes`. It opens the `AppendSession` it owns.
4. **Acceptance.** `submit()` waits until there is capacity for the record or batch, then accepts it and returns a ticket.
   - **a.** Capacity is `max_unacked_bytes` and, on an `AppendSession`, `max_unacked_batches`. An accepted record counts against it until its batch has an outcome, including while a Producer is still batching it. Capacity is granted first come, first served.
   - **b.** Records are ordered by acceptance.
   - **c.** `submit()` never half-accepts. A cancelled `submit()` accepted nothing. One still waiting when the session closes, or made after it has, fails with `SessionClosedError`; `flush()` and `close()` never do.
5. **Batching.** A Producer sends a batch when it is full, or when `linger` has passed since its first record was accepted, whichever comes first. A batch never exceeds `max_batch_records` or `max_batch_bytes`. The fencing token goes on every batch. `match_seq_num` goes on the first batch, and rises by each batch's record count.
6. **Outcomes.** Each batch has one outcome, shared by its records: a batch is atomic, so each record gets its batch's outcome. On success, an `AppendSession`'s ticket gives the batch's ack, and a Producer's gives an `IndexedAppendAck`: the record's own `seq_num`, and `batch`, its batch's ack. An outcome never changes; where a ticket can be awaited more than once, every await gives the same outcome.
   - **a.** If the final error came from the server, it belongs to the oldest unacked batch, provided that batch was sent: that batch was **rejected**. Each batch gets the first row that fits:

     | Batch | Outcome |
     |---|---|
     | Acked | the ack |
     | Rejected | the final error |
     | Behind a batch rejected with a definite error | `SessionClosedError` |
     | Never sent, in any attempt | `SessionClosedError` |
     | Any other | the final error |

   - **b.** A definite outcome is then wrapped if an earlier attempt that sent the batch may have taken effect (E3).
   - **c.** `SessionClosedError` behind a definite rejection relies on the server: once it rejects a batch with a definite error, no later batch in the session becomes durable. A transport without a server-side session (such as TypeScript's fetch transport) gives the same guarantee itself: one batch in flight at a time, and nothing sent after a failure.
   - **d.** A Producer never overrides an outcome its session reported.
7. **Flushing.** `flush()`, on a Producer only, sends any partial batch at once, then returns once every record accepted before it is acked. Otherwise it fails with the failure. A failure that affects only later records does not fail it, and neither does the Producer being closed. A `submit()` that returned before `flush()` began is covered; concurrent ones may fall on either side.
8. **Closing.** A session closes for one of three reasons.
   - **a. The caller.** `close()` stops accepting, sends any partial batch, and waits until everything accepted has an outcome. Then it returns, or fails with the failure. Every call to it gives the same result.
   - **b. A failure.** The final error closes the session at once, and every batch gets its outcome (A6).
   - **c. An abort.** Closing the client (C7), or a trigger that depends on the language (see Languages), closes the session at once with an indefinite `AbortedError` failure, and every batch gets its outcome (A6). With no batch awaiting an outcome, nothing was lost, so an abort is a clean close with no failure. Cancelling any other call only stops that call's wait.
   - **d. Cause.** `AbortedError` carries what triggered the abort as its cause, in languages where errors have causes.
   - **e. Ownership.** A Producer owns its session: closing or aborting the Producer closes or aborts the session.
9. **Failure.** A session's failure is its final error, wrapped if it is definite and any batch's outcome was wrapped (A6b), or `AbortedError` (A8c). A Producer's failure is that of the session it owns, or `AbortedError`.

**Example.** A session accepted batches A, B and C, in that order. A and B were sent; C was not. In each row, the last error is final: it was not retried. "closed" is `SessionClosedError`, `412` is either error of E1a, and `Indefinite(x)` is `IndefiniteAppendError` holding `x`.

| What happened | A | B | C | `close()` |
|---|---|---|---|---|
| Server rejects A with `412` | `412` | closed | closed | `412` |
| Server fails A with `storage` (indefinite) | `storage` | `storage` | closed | `storage` |
| Connection lost | `ConnectionLostError` | `ConnectionLostError` | closed | `ConnectionLostError` |
| Connection lost and retried; A and B resent; server rejects A with `412` | `Indefinite(412)` | `Indefinite(closed)` | closed | `Indefinite(412)` |

## R. Reading

`read()`, `check_tail()` and `ReadSession`. Rules R3 to R8 cover the session. Its `next_seq_num` describes what the caller has been given, not what the SDK has fetched; `last_observed_tail` and `is_caught_up` describe what the server has reported.

1. **Unary read.** `read()` returns one batch as the server sent it: its records, and the tail if the server reported one. A batch has no records only when a stop condition was met before any record, such as `wait` running out at the tail; it does not mean the end of the stream.
2. **Tail.** `check_tail()` returns the tail.
3. **Iteration.** Iterating a session yields records, one at a time, in sequence-number order, each delivered once.
   - **a.** Batches are how the server ships records, and carry no meaning. An SDK may also offer a batch view; it delivers each batch whole, never an empty one, and every rule here applies to it per batch. Heartbeats change state (R4), not what is delivered.
   - **b.** The session decodes at most one batch ahead of the one being delivered. Any further read-ahead is the transport's flow control.
   - **c.** Iteration has one consumer. The accessors in R4 are safe to call from anywhere, at any time, and never block.
4. **State.** Three accessors. A record's effect on them is visible before the record is delivered.
   - **a. `next_seq_num`.** One past the last record delivered, or the start before any is. A `seq_num` start that is not clamped is known at once; any other start is resolved by the first response, to its first record's sequence number or else its tail (R8b), and until then `next_seq_num` is unknown. Records removed by trimming or retention are absent, so it can jump. A new session started at `next_seq_num` delivers what this one would have delivered next.
   - **b. `last_observed_tail`.** The latest tail the server has reported, whether or not the session has delivered up to it. It never moves backward, and survives reconnects. `last_observed_tail.seq_num - next_seq_num` is a lower bound on how far behind the caller is.
   - **c. `is_caught_up`.** True when the last thing the server reported is a tail equal to `next_seq_num`: everything up to a reported tail has been delivered, and nothing newer is known. It turns false when a record beyond that tail is fetched, when records arrive without a tail, and when the connection is lost, until the server reports again.
5. **Catching up.** `caught_up()` returns a future that resolves once the session is caught up: at once if it is now, or else the next time it is.
   - **a.** It resolves with the tail the session caught up to, and its value never changes.
   - **b.** It does not read: the caller keeps iterating while it is pending.
   - **c.** It stays pending across reconnects.
   - **d.** If the session closes before catching up, it fails with the failure, or with `SessionClosedError` when there is none.
6. **Closing.** A read session closes for one of four reasons.
   - **a. A stop condition.** `count`, `bytes` or `until` was reached, or `wait` ran out. Iteration finishes with no error.
   - **b. The caller.** The consumer closes the session (see Languages). Iteration finishes with no error. Cancelling a wait for the next record only stops that wait: nothing is lost, and the session stays open.
   - **c. A failure.** Every record already fetched is delivered first; then iteration fails with the failure, once.
   - **d. An abort.** Something outside the code that iterates the session aborts it: closing the client (C7), or the session's own abort where the language has one (see Languages). Records fetched but not delivered are dropped, and iteration fails with `AbortedError` (A8d), even when nothing was fetched: finishing cleanly would look like a stop condition.
   - **e.** After closing, `next_seq_num` and `last_observed_tail` keep their last values. `is_caught_up` keeps its last value after a stop condition or the caller, and is false after a failure or an abort.
7. **Failure.** A read session's failure is its final error, or `AbortedError` (R6d). A read has no side effects, so nothing is wrapped.
8. **Reconnects.** They are invisible to iteration. T4 and T5 decide when one happens.
   - **a.** The session resumes at the first record it has not fetched, so nothing is repeated and nothing is missed.
   - **b.** A start given by timestamp or tail offset is resolved once. After the first response the session resumes by sequence number, never by re-evaluating the original start.
   - **c.** `count` and `bytes` cover the whole session, not each connection, and `wait` bounds the time between records across connections. A reconnect asks only for what is left of each.

**Example.** A stream holds records 0–4, so its tail is 5. A session starts at 0.

| Step | `next_seq_num` | `last_observed_tail` | `is_caught_up` |
|---|---|---|---|
| Opened, nothing fetched | 0 | unknown | false |
| Records 0–4 fetched with tail 5; record 0 delivered | 1 | 5 | false |
| Records 1–4 delivered | 5 | 5 | true |
| Heartbeat with tail 5 | 5 | 5 | true |
| Another writer appends record 5; it is fetched with tail 6 | 5 | 6 | false |
| Record 5 delivered | 6 | 6 | true |
| Connection lost | 6 | 6 | false |
| Reconnected; heartbeat with tail 6 | 6 | 6 | true |

## M. Managing basins, streams and access tokens

Every call here is unary, and named for its verb and resource:

| | Basins | Streams | Access tokens |
|---|---|---|---|
| Create (M1) | `create_basin()` | `create_stream()` | — |
| Ensure (M2) | `ensure_basin()` | `ensure_stream()` | — |
| Reconfigure (M3) | `reconfigure_basin()` | `reconfigure_stream()` | — |
| Delete (M4) | `delete_basin()` | `delete_stream()` | `revoke_access_token()` |
| Issue (M5) | — | — | `issue_access_token()` |
| List a page (M6) | `list_basins()` | `list_streams()` | `list_access_tokens()` |
| List all (M7) | `list_all_basins()` | `list_all_streams()` | `list_all_access_tokens()` |

An SDK that groups calls by resource may leave the resource out of the name (Go's `Basins.Ensure()`). A call this chapter does not cover, such as `get_basin_config()`, is an ordinary unary call, retried as T2 says.

1. **Create.** Creating a basin or stream makes a new one, and fails with a conflict if the name is taken. It carries a request token (T2b), so a retry does not conflict with its own earlier attempt.
2. **Ensure.** Ensuring a basin or stream creates it if it is missing, and otherwise applies the given config. It returns the resource and what happened: `created`, `updated` or `unchanged`.
3. **Reconfigure.** Reconfiguring changes only the fields it is given, and returns the config as it now stands. Each field has three states: left unchanged, which is the default; set to a value; or reset to the server's default.
4. **Delete.** Deleting a resource makes sure it is gone, so it succeeds when the resource does not exist or is already being deleted, on any attempt. Revoking an access token is how it is deleted. Deleting a basin or stream starts its deletion, and until that finishes the resource is being deleted (M8).
5. **Issue.** Issuing an access token returns it, and the server reveals it only in this response. The SDK does not keep it or log it (L3), and retries only definite errors (T2d).
6. **List a page.** One call, which returns one page as the server sent it: the items, and `has_more`, which says whether more remain. It takes `prefix`, `start_after` and `limit`.
7. **List all.** Iterates every match lazily, fetching the next page only when the caller reaches it. It takes `prefix` and `start_after`. Each page is a unary call with its own retries, and a page that fails ends the iteration with that error.
8. **Resources being deleted.** A page from M6 includes basins and streams that are being deleted, each marked as such. Iteration under M7 leaves them out, unless `include_deleted` is set.

## F. Forward compatibility

A released SDK keeps working against a newer server.

1. **Parsing.** A response never fails to parse because of a field or value this SDK version does not know.
2. **Open enums.** Every enum the server sends is open: a value this SDK version does not know is exposed by its wire name, in the language's idiom (an `Unknown` variant, or an open string type). A closed type that can hold an unlisted value at run time is a bug.
3. **Closed enums.** Every enum only the SDK sends is closed.
4. **Error codes.** An error code the SDK does not know follows E5c.

## L. Logging

1. **Off by default.** The SDK logs through the language's usual facility, and emits nothing unless the application turns it on.
2. **Debug or finer.** The SDK logs only at debug level or finer, even for retries, reconnects and failures. A failure reaches the application as an error, not as a log line.
3. **Secrets.** Record bodies, access tokens and encryption keys are never logged. Tokens and keys are also masked when a client, a handle or their settings are printed.

## Settings and constants

Settings have the same names and defaults in every SDK.

| Setting | Set on | Default |
|---|---|---|
| `connection_timeout` | client | 3 s |
| `request_timeout` | client | 5 s |
| `max_attempts` | client | 3 |
| `min_base_delay`, `max_base_delay` | client | 100 ms, 1 s |
| `append_retry_policy` | client | `all` |
| `compression` | client | `none` |
| `retry_policy` | read session | `budgeted` |
| `max_unacked_bytes` | `AppendSession`, `Producer` | 5 MiB (minimum 1 MiB) |
| `max_unacked_batches` | `AppendSession` | none |
| `linger` | `Producer` | 5 ms |
| `max_batch_records`, `max_batch_bytes` | `Producer` | 1000 records, 1 MiB (also the maximum) |

Constants are fixed, and stated in the rule they belong to. This table only gathers them.

| Constant | Value | Rule |
|---|---|---|
| Largest record | 1 MiB | A2a |
| Largest batch | 1000 records, 1 MiB | A2b |
| Longest fencing token | 36 bytes | A2c |
| Smallest session frame that is compressed | 1 KiB | C3 |
| Longest `wait` on a unary read | 60 s | D4 |
| Shortest gap between acting on reconnect advice | 60 s | T5a |
| Heartbeat timeout | 20 s | T6d |
| Batches a read session decodes ahead of delivery | 1 | R3b |

## Server guarantees

Some rules are correct only because the server behaves a certain way. These are those guarantees, the rules that rely on them, and where each is documented, or else seen.

| Guarantee | Relied on by | Source |
|---|---|---|
| A batch becomes durable whole, or not at all. | A6 | [Appends](https://s2.dev/docs/concepts/appends) |
| A session's batches become durable in the order they were sent, and once one fails, no later batch does. | A6a, A6c | [Appends](https://s2.dev/docs/concepts/appends) |
| An append session that stops sending after reconnect advice gets an ack for everything it sent, then a clean end. | T5a | [`s2s.md`](../s2/v1/s2s.md) |
| A session ended with `server_draining` sends every ack first; whatever it did not ack was never processed. | T5b | [`s2s.md`](../s2/v1/s2s.md) |
| A read session waiting at the tail gets a heartbeat at least every 15 s. | T6d | [`s2s.md`](../s2/v1/s2s.md) |
| An error the error-code table lists with no side effect had none, a `412` included. | Terms, E5a | [Error codes](https://s2.dev/docs/api/error-codes) |
| A create retried with the same request token does not conflict with its earlier attempt. | T2b, M1 | [`openapi.json`](../s2/v1/openapi.json) |
| Ensure reports what it did in the `s2-provision-result` header: `created`, `updated` or `noop`. | M2 | s2-lite and the service; not yet documented |
| Deleting a basin or stream that is already being deleted succeeds. | M4 | s2-lite and the service; not yet documented |
| Revoking an access token that does not exist returns `404`. | M4 | the service; not yet documented |

## Languages

Where a rule depends on the language, this is what it means in each.

| | Python | TypeScript | Go | Rust |
|---|---|---|---|---|
| An error the spec names `XError` (E2) | class `XError` | class `XError` | type `*XError`, matched with `errors.As` | variant `X` of the call's error enum |
| Abort an append session or Producer (A8c) | cancel `close()`, or leave `async with` on a `BaseException` that is not an `Exception` (a cancellation, `KeyboardInterrupt`, `GeneratorExit`) | the session's abort signal | cancel the constructor's context | drop it, including an unfinished `close()` |
| Close an append session or Producer (A8a) | `close()`, or leave `async with` any other way | `close()`, or `await using` | `Close()` | `close()` |
| Close a read session (R6b) | `close()`, or leave `async with` | `cancel()` it (it is a `ReadableStream`), or `await using` | `Close()` | drop it |
| Abort a read session, besides closing the client (R6d) | none | the session's abort signal | cancel the constructor's context | none |
| Cancel a unary call (T7) | cancel the task | the call's abort signal | cancel the call's context | drop the future |
| What a cancelled unary call reports (T7) | the task's cancellation | `AbortedError` | `AbortedError` | nothing: the future is gone |
| The cause `AbortedError` carries (A8d) | the exception | `signal.reason` | the context error, in the `Unwrap` chain, so `errors.Is(err, context.Canceled)` holds | none |
| A wrapper's cause, besides its typed accessor (E3b) | `__cause__` | `cause` | none: the inner error is kept out of the `Unwrap` chain | `source()` |

A wrapper's inner error can be its cause in Python, TypeScript and Rust because `except`, `instanceof` and `match` do not follow causes. Go's `errors.Is` and `errors.As` do follow `Unwrap`, so a Go wrapper does not unwrap to its inner error.

**TypeScript stream adapters.** An `AppendSession` or `Producer` also offers `writable`, which takes what `submit()` takes, and `readable`, which yields acks. A write is a `submit()`, and closing `writable` is `close()`. Aborting `writable`, or cancelling `readable` (as leaving a `for await` over it early does), aborts the session. On failure, `readable` errors with the failure.

## Conformance

An SDK conforms to a revision when it follows every rule in it. Each SDK's README names the revision it targets. To make conformance checkable:

- Every testable rule has a test in each SDK, tagged with its id, such as `A6b`.
- A rule without a tagged test counts as not yet met.
- When a revision changes a rule, the Revisions section names it, so each SDK knows which tests to revisit. Ids are stable, so a tag never needs renaming.

## Revisions

- **1.** Initial version.
