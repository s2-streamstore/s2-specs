# S2 SDK spec

**Revision 1.**

What every S2 SDK (Python, TypeScript, Go, Rust) does. It describes where the SDKs are headed, not where each one is today.

Every rule, and every choice the spec leaves to an SDK, is held to four standards: correctness, elegance, idiom and efficiency. Where a rule's reason is not obvious, [rationale.md](rationale.md) gives it, by rule id.

**Reading the rules.**

- Every rule is a requirement. Where an SDK has a choice, the rule says it may.
- Each chapter has a letter and numbered rules, cited by id, such as `A6b`. Ids are stable: a later revision never renumbers a rule, and never reuses the id of one it removes. Add the revision, as in `r1 A6b`, when the wording matters.
- Names are canonical. Every SDK uses the spec's words, in its own casing and naming idiom (for error types, see [Languages](#errors)), and may add a unit suffix where the language has no duration type. `name()` is a call, `Name` is a type, and any other `name` is an accessor, option, setting, value or wire name.

| Chapter | Covers |
|---|---|
| C | Client |
| D | Data: records and read inputs |
| E | Errors |
| T | Retries, timeouts and cancelling |
| A | Appending: `append()`, `AppendSession`, `Producer`, `Batcher` |
| R | Reading: `read()`, `check_tail()`, `ReadSession` |
| M | Managing basins, streams and access tokens |
| F | Forward compatibility |
| L | Logging |

Terms and the error model come first. Settings and constants, Server guarantees, Languages, Conformance and Revisions follow the chapters.

## Terms

- **Unary call, session.** A unary call is one request and one response. A session is a long-running append or read on one stream, which the SDK keeps going across attempts.
- **Connection.** A TCP connection, with TLS for https, that requests travel over. The client owns the connections and shares them among requests (C8).
- **Attempt.** One try at an operation: one request and its response. A session's attempt is one request, on which it sends its batches, including any an earlier attempt sent that have no outcome yet.
- **Sent.** A request or batch is sent in an attempt once any of its bytes may have left the SDK during it. An attempt sends nothing after it ends (T10), so whether it sent something is settled when its error arrives.
- **Ack.** The server's confirmation that a batch of appended records is durable.
- **Ticket.** What `submit()` returns once it has accepted a record or batch. Awaiting it gives that record's or batch's outcome (A6).
- **Position.** A sequence number and a timestamp.
- **Definite.** An error is definite when it certainly had no side effect: on a unary call, its attempt had none; on an append session, no batch that was unacked when it arrived had one. Its `has_no_side_effects` (E5) then says so. These are definite:
  - any error from an operation with no side effects, such as a read;
  - a `412`, typed or not (E5e);
  - a server error whose code the error-code table, [`error-codes.json`](../s2/v1/error-codes.json), lists with no side effect (E5a);
  - an error that arrives before the request, or any unacked batch, was sent, or that the server reports as unprocessed (a `ConnectError`, `REFUSED_STREAM`, an HTTP/2 stream above a GOAWAY's `last_stream_id`), unless it is an abort;
  - `SessionClosedError` (E4).

  Every other error is **indefinite**: it may have had an effect. That includes codes the SDK does not recognize, timeouts, failed SDK checks, and every `AbortedError` on a call with side effects.
- **Final error.** The error an operation or a session stopped on: the last attempt's error, which was not retried.
- **Closing, closed.** A session closes once, for one of the reasons A8 or R7 lists. An append session is closing from the moment `close()` is called until it closes. A closing or closed session takes on no new work; what it already holds finishes as those rules say.
- **Failure.** What a session reports as the reason it closed: its final error, wrapped as A9 says, or `AbortedError`. A session that closed cleanly has none.
- **Tail.** The position the server reports as the end of the stream: the sequence number the next record will get, and the timestamp of the last one. The server reports it with records near the end of the stream, and on its own as a **heartbeat** while a read session waits there.
- **Delivered.** A record is delivered when iteration hands it to the caller. Records the SDK has fetched and is still holding are not delivered.

## Error model

How the SDK handles an error, told once. This section adds no rule; each step cites the rule that governs it.

An operation makes one or more attempts. An attempt that fails has an error, which is definite if it certainly had no side effect, and indefinite otherwise (Terms). The SDK repeats an attempt only when its error may be temporary, repeating it is safe, and the budget allows (T1). The error it stops on is the final error. A planned handover from an attempt that made progress is not a retry, and uses no budget (T5).

A unary call returns its result, or fails with its final error. A definite final error speaks only for the last attempt: if an earlier attempt of an append may have taken effect, the append fails with `IndefiniteAppendError` holding it (E3).

A session retries the same way, across attempts. An append session gives every batch one outcome, its ack or an error, which the batch's records share (A6). Either kind of session closes once, and reports a failure if it ended on an error (A9, R8).

Every error that reaches the caller says what is safe to do next: `has_no_side_effects` is true when trying again cannot repeat an effect, and `is_retryable` is true when trying again could succeed (E5).

## C. Client

1. **Construction.** A client is built from an access token and optional settings, and never reads the environment on its own. `from_env()` builds one from `S2_ACCESS_TOKEN`, `S2_ACCOUNT_ENDPOINT` and `S2_BASIN_ENDPOINT`, and fails with `ValidationError` if `S2_ACCESS_TOKEN` is unset, or if only one of the endpoint variables is set.
2. **Settings.** Every setting has the name and default that Settings lists, in every SDK. A setting outside its range (D7) fails with `ValidationError` where it is given.
3. **Endpoints.** The `endpoints` setting is an account endpoint and a basin endpoint, given together. An endpoint is a host, optionally preceded by `http://` or `https://` and followed by a port and a `/`. Without a scheme it is https, and both endpoints have the same scheme. The host is a DNS name or an IP address, an IPv6 one in brackets. A basin endpoint may start with `{basin}.`, which the SDK replaces with the basin's name; its host must then be a DNS name. A basin endpoint without `{basin}.` serves every basin, so the SDK names the basin in the `s2-basin` header, and only then. Anything else, such as one endpoint without the other, a path, a query or user info, fails with `ValidationError` when the client is built.
4. **Compression.** `compression` is `none`, `gzip` or `zstd`. With `gzip` or `zstd`, the SDK compresses request bodies with it and asks for responses in it alone, on unary calls and sessions alike; a session frame is compressed only when its payload is at least 1 KiB. With `none`, it compresses nothing and asks for nothing, though a transport may still ask on its own. Whatever the setting, the SDK decodes any response or session frame compressed with `gzip` or `zstd`, error responses included. A session frame longer than [`s2s.md`](../s2/v1/s2s.md) allows, or that decompresses to more, fails with `ProtocolError`.
5. **Transports.** An SDK may carry calls over more than one protocol, choosing which itself. Every rule in this spec holds on each transport; transports may differ only in speed.
6. **Encryption key and stream config.** An encryption key is set on the stream handle, and sent as `s2-encryption-key` with every append and read. A key is bytes, given as bytes or as standard base64 text, ignoring surrounding whitespace. Text that is not base64 fails with `ValidationError` when the key is made. The server checks the key's length, which depends on the stream's cipher. A stream config, used to create the stream on first use, is sent as `s2-stream-config`; a session sends it with every request.
7. **Concurrency.** A client, the basin and stream handles made from it, and the sessions opened from those are safe to use from concurrent tasks. The one exception is iterating a read session, which has one consumer (R4c).
8. **Closing.** The client owns the connections; basin and stream handles only refer to it. Every client has `close()`. Closing it aborts every session opened from it (A8c, R7d), and fails every unary call in flight with `AbortedError`, since it may already have taken effect. It ends every connection, including one still being set up, without waiting for the server, and then returns; every call to it gives the same result. After that, a call on the client, or on a handle made from it, fails at once with `ClientClosedError`; a call on a session it aborted follows A4c and R7. A client dropped without `close()` stays alive while any session opened from it is open.
9. **User agent.** `s2-sdk-<language>/<version>`, where the language is `python`, `typescript`, `go` or `rust`.
10. **Inputs.** The SDK reads what a caller gives it when the call takes it, and never modifies it. A client takes its settings when built; a call or session takes its options, config and records when made, opened or accepted. Changing them afterwards changes nothing, on retries and new requests included (see [Languages](#numbers-and-byte-inputs)).
11. **Names.** A basin name that breaks the API's rule for one (`BasinNameStr` in [`openapi.json`](../s2/v1/openapi.json)) fails with `ValidationError` wherever the SDK takes it. Other names are left to the server. The SDK percent-encodes any name it puts in a URL path.
12. **Redirects.** The SDK never follows a redirect: a `3xx` response is a `ServerError` with its status (E1).

## D. Data

1. **Records.** An `AppendRecord` has a body, headers and an optional timestamp; the server decides the stored timestamp from it, as the stream's timestamping config says. A `SequencedRecord`, as read, has a body, headers, its `seq_num` and its `timestamp`. A body is bytes. Headers are an ordered list of name and value pairs, both bytes. An SDK may add string conveniences on top.
2. **Numbers.** A sequence number is an integer. A record's timestamp, and so a position's, is an integer number of milliseconds, by convention since the Unix epoch, and is never exposed or accepted as a date type. A time the API gives as a date-time, such as `created_at`, is exposed as the language's date type. The SDK gives every number the server sends exactly. If the language's type cannot hold one (see [Languages](#numbers-and-byte-inputs)), the response or session message fails with `ProtocolError`; a `412` or `416` whose value it cannot hold is a plain `ServerError` (E1).
3. **Read start.** At most one of `seq_num`, `timestamp` or `tail_offset`, in a type that cannot hold two, and `clamp`. Their meaning is the API's. A read with no start sends none, and the API begins at the tail.
4. **Read stop.** Any of: `count` and `bytes` limits; `until`, a timestamp to stop before; and `wait`, how long to wait for new records. Their meaning and defaults are the API's, and the SDK leaves any the caller did not set to the API. On a session, `wait` bounds the time between records.
5. **Command records.** `fence(token)` and `trim(seq_num)` build them, and `is_command_record()` recognizes one that was read; all three are public. Reads return command records like any other.
6. **Metered bytes.** A record's metered bytes are 8, plus 2 per header, plus the length of every header name and value, plus the length of the body. `metered_bytes()` is public, and every limit on the size of a record or batch (A2, A4, A5, D4) is in metered bytes.
7. **Checking numbers.** A number a caller gives is checked when the SDK takes it (C10). One that is not a whole number in its range fails with `ValidationError`: NaN, an infinity, a fraction, a negative, or one above its maximum. A sequence number, timestamp, count or byte limit ranges from 0 to 2⁶⁴ − 1, or to the end of the language's type if it ends sooner (see [Languages](#numbers-and-byte-inputs)). A duration the API counts in whole seconds, such as `wait`, a retention age or delete-on-empty's `min_age`, must be a whole number of seconds. The SDK never rounds, truncates, wraps or clamps a number it is given.

## E. Errors

1. **Server errors.** Every response with a non-2xx status, a session's terminal frame included, fails with a `ServerError` with that status, whatever its body. It exposes the code and message when the body is the API's error object, exactly as returned. Otherwise it has no code, and its message is the body's text, or the status when the body is empty or cannot be read. Status and code are the contract; the message is for people and is never parsed. Two kinds have types of their own, named for what happened rather than for their status, when the body holds the value the type carries; otherwise the error is a plain `ServerError`. Each is also a `ServerError`, in each language's way (see [Languages](#errors)):
   - **a.** A `412` whose body holds `seq_num_mismatch` is a `SeqNumMismatchError` with `assignable_seq_num`, the sequence number the stream would assign next; one whose body holds `fencing_token_mismatch` is a `FencingTokenMismatchError` with `current_fencing_token`, the stream's fencing token.
   - **b.** A `416` whose body holds `tail` is a `ReadUnwrittenError` with it.
2. **SDK errors.** Only a `ServerError` carries a status or a code. The SDK's own errors are told apart by type, and the SDK picks the type from the operation that failed, never from the text of an error's message:

   | Error | When | Definite, on a call with side effects | Retryable |
   |---|---|---|---|
   | `ValidationError` | input fails validation (A2, C1, C2, C3, C6, C11, D7) | yes | no |
   | `ConnectError` | a connection cannot be made, or `connection_timeout` passes (T6a) | yes | yes |
   | `ConnectionLostError` | a request or its connection breaks, or a response ends early (T9) | only if nothing unacked was sent, or the server reports it unprocessed | yes |
   | `TimeoutError` | `request_timeout` passes, for a response or an ack (T6b, T6c), or a read session's heartbeat timeout does (T6d) | no | yes |
   | `ProtocolError` | a successful response or a session message cannot be decoded, or fails an SDK check | no | no |
   | `ClientClosedError` | a call is made on a closed client (C8) | yes | no |
   | `SessionClosedError` | the session closed or started closing first (E4) | yes | no |
   | `AbortedError` | a session is aborted (A8c, R7d), a call is cancelled (T7), or a unary call is in flight when the client closes (C8) | no | no |
   | `IndefiniteAppendError` | an append's definite final error follows an attempt that may have taken effect (E3) | no | as its inner error |

   On a call without side effects, such as a read, every error is definite (Terms).
3. **Wrapping.** A definite final error describes only the last attempt. If an earlier attempt of the same append may have taken effect, the append fails instead with `IndefiniteAppendError` holding the final error, and that is indefinite. This applies to `append()` and to every batch in a session (A6b).
   - **a.** A wrapper is not the error inside it. `IndefiniteAppendError` and `SessionClosedError`, the two wrappers, do not copy their inner error's status or code, and do not match as its type.
   - **b.** The inner error is reachable through a typed accessor: `final_attempt_error` on `IndefiniteAppendError`, and `failure` on `SessionClosedError`. Where matching by type does not follow causes (Python, TypeScript, Rust), the inner error is also the wrapper's cause; in Go, where `errors.As` follows `Unwrap`, it is not (see [Languages](#errors)).
4. **`SessionClosedError`.** The session closed, or started closing, without doing what was asked, and never will: an append session will not append the record or batch, and a read session will not deliver another record or catch up. It is definite. Its message includes the failure's message. `failure` is the session's failure, or empty if it had none when the call failed.
5. **Classification.** Every error has two public predicates. `has_no_side_effects` is true when the error is definite (Terms). `is_retryable` is true when the error's cause may be temporary, so the same request might succeed if made again; whether making it again is safe is for `has_no_side_effects` and T2 to say.
   - **a.** For a `ServerError`, both come from the error-code table, [`error-codes.json`](../s2/v1/error-codes.json). Each SDK's copy of it is complete, and is tested against it through the SDK's s2-specs submodule.
   - **b.** A code counts only when it arrives with the HTTP status the table lists for it. Otherwise the error is treated as having no code.
   - **c.** A `ServerError` with no code, or with a code the SDK does not recognize, is retryable by status alone: `408`, `429`, `500`, `502`, `503`, `504`.
   - **d.** For an SDK error, both come from the table in E2.
   - **e.** A `412` is definite and not retryable, whatever its body: the server checks an append's conditions before appending anything.
6. **Messages.** Every error's message names the call it came from, the basin and stream that call was for, if any, and how many attempts were made, if more than one.
7. **Unobserved errors.** An error the caller never awaits, such as a ticket's or that of `caught_up()`, reports nothing: no unhandled-rejection or unretrieved-exception warning, and no log above debug.
8. **Base type.** Every error the SDK raises, a `ServerError` included, is an `S2Error`, in each language's way (see [Languages](#errors)), so a caller can catch them all with one type.

## T. Retries, timeouts and cancelling

1. **What is retried.** Apart from a planned handover (T5), an attempt is retried when its error is retryable (E5), repeating it is safe (T2) and the budget allows (T3). Otherwise that error is the final error: a unary call fails with it, and a session closes (A8b, R7c). The final error is the last attempt's own error, wrapped only as E3 says; no error reports that retries ran out.
2. **Safe to repeat.**
   - **a.** Reads, and calls that set a state (ensure, reconfigure, delete, revoke), are always safe to repeat.
   - **b.** Creating a basin or a stream carries a request token in `s2-request-token`, made once per call and reused on every attempt, so it is always safe to repeat too.
   - **c.** An append is safe to repeat as `append_retry_policy` says: under `all`, always; under `no_side_effects`, only after a definite error.
   - **d.** Any other call that is not idempotent, such as issuing an access token, is safe to repeat only after a definite error.
3. **Budget and backoff.** The budget is `max_attempts`, counting the first. The delay before retry *n*, counting from 0, is `base + random(0, base)`, where `base = min(min_base_delay × 2ⁿ, max_base_delay)`. On a unary call, a `retry-after-ms` response header raises the delay to at least that value.
4. **Sessions.**
   - **a. Restoring the budget.** Each ack on an append session, and each batch or heartbeat on a read session, restores the whole budget: the attempt in progress counts as the first again, and the next retry is retry 0 (T3).
   - **b. Read retry policy.** A read session's `retry_policy` is `budgeted` or `unbounded`. An `unbounded` session ignores `max_attempts`: it retries every retryable error, with the backoff of T3.
5. **Handover.** When the server plans to go away, a session moves to a new request on a different connection, without using its budget and without backoff, if the old request restored the budget at least once (T4a). Otherwise the move counts as a retry (T1), so a run of requests that end at once backs off and gives up as T3 says. On reconnect advice or `server_draining`, the SDK opens no new request on the connection the server is leaving, for any session or call; it lets requests already on it finish, then ends it.
   - **a. Reconnect advice.** A read session moves at once. An append session stops sending on the advised request, and sends on a new one once the old one has acked everything sent on it ([s2s](../s2/v1/s2s.md)). A session acts on advice at most once every 60 s.
   - **b. Draining.** On `server_draining`, a session moves to a new request as T5 says. An append session resends every batch the old request did not ack: none of them took effect.
   - **c. Unary calls.** On a unary call, `server_draining` is an ordinary retryable error.
6. **Timeouts.** Each applies within one attempt, not to a call as a whole, and fails the attempt: `connection_timeout` with `ConnectError`, the others with `TimeoutError`.
   - **a. Connection.** `connection_timeout` bounds establishing a connection, TLS included.
   - **b. Request.** `request_timeout` bounds how long the server takes to answer: on a unary call, from sending the request to receiving the whole response; on a session, until the response begins. A unary read with `wait` gets `request_timeout` plus `wait`.
   - **c. Ack.** On an append session, an ack is the answer to a batch, so `request_timeout` bounds it too. Only the oldest unacked batch is timed, from when it became the oldest or its last byte was sent, whichever is later.
   - **d. Heartbeat.** On a read session, a request times out when the SDK has waited 20 s for the server's next message, heartbeats included, and none came. Time the SDK spends not reading, because the caller has not taken what it holds (R4b), does not count.
7. **Cancelling.** Cancelling a unary call stops it at once, including while it waits to retry, and it is not retried; a cancelled `append()` may have taken effect. Cancelling a wait on a session, such as in `submit()`, `flush()`, on a ticket, in `caught_up()` or for the next record, only stops that wait: no outcome changes, the session carries on, and a later wait gives what the cancelled one would have (A4c, R7b). Cancelling an abort's trigger aborts instead (A8c). How a call or a wait is cancelled, and what the caller then sees, depends on the language (see [Languages](#cancelling-a-call)).
8. **Attempt header.** Every request carries `s2-attempt`, one more than the number of retries used from the budget, as T3 and T4a count them: `1` on a first try, and `2` on retry 0.
9. **How a response ends.** A response ends cleanly when it finishes at a message boundary without a terminal frame. Each way an attempt's response can end has one result, from the first row that fits:

   | The response | Append session | Read session | Unary call |
   |---|---|---|---|
   | has a non-2xx status or a terminal frame | `ServerError` (E1) | `ServerError` (E1) | `ServerError` (E1) |
   | ends cleanly after reconnect advice the session acts on | handover (T5a) | handover (T5a) | — |
   | ends cleanly with nothing unacked | no error, and no budget used (A3) | — | — |
   | ends cleanly with batches unacked | `ConnectionLostError` | — | — |
   | ends cleanly when a stop condition could have been met | — | iteration finishes (R7a) | — |
   | ends cleanly when none could | — | `ConnectionLostError` | — |
   | is cut off inside a message or body, or its connection breaks | `ConnectionLostError` | `ConnectionLostError` | `ConnectionLostError` |
   | holds a whole message or body that does not decode | `ProtocolError` | `ProtocolError` | `ProtocolError` |

10. **Ending an attempt.** An attempt ends when its error arrives, when a handover leaves it (T5), or when its session closes or aborts. Ending it resets its request unless its response has already ended. After that it sends nothing, nothing it receives counts, and every timer and wait it started stops. What it received before it ended takes effect first, in the order received: an ack settles its batch (A6), and a fetched record is delivered (R7c). An attempt has one error, the first to arrive; a later one comes from an attempt that has ended, and is ignored.

**Example.** An append session with the default `max_attempts` of 3. Each row that makes a request gives the `s2-attempt` it sends.

| What happens | `s2-attempt` |
|---|---|
| A batch is submitted, and the session makes its first request | 1 |
| The batch is acked: the budget is restored, and this request counts as attempt 1 | — |
| The connection is lost; the session makes a new request after retry 0's delay | 2 |
| That request breaks too, before any ack; the session makes a new one after retry 1's delay | 3 |
| An ack arrives: the budget is restored, and this request counts as attempt 1 | — |
| The server advises a reconnect; the move uses no budget | 1 |
| That request ends at once with `server_draining`, before any ack; the move counts as retry 0 | 2 |
| That request breaks before any ack; retry 1 | 3 |
| That request breaks before any ack: three attempts have failed, so the session fails with `ConnectionLostError` | — |

## A. Appending

`append()`, and two ways to append continuously: `AppendSession`, which takes batches, and `Producer`, which takes records, batches them, and owns an `AppendSession` (A3b). Rules A3 to A9 cover both, called sessions in this chapter; where they differ, the rule names one.

1. **Unary append.** `append()` sends one batch and returns its ack, retrying as `append_retry_policy` says (T2c). An ack holds three positions: `start`, the first record appended; `end`, one past the last record appended; and `tail`, the stream's tail, which is past `end` when others appended at the same time. A session's acks hold the same three.
2. **Validation.** `append()` and `submit()` check a record or batch before anything else: before sending, and before waiting for capacity. An invalid one fails with `ValidationError`; nothing is accepted, and the session stays open. These are invalid:
   - **a.** a record over 1 MiB or, on a Producer, over `max_batch_bytes`;
   - **b.** a batch that is empty, or over 1000 records or 1 MiB;
   - **c.** a fencing token over 36 bytes, on a batch, in `fence()`, or in a Producer's options, which are checked when it opens;
   - **d.** a record's timestamp, or a batch's `match_seq_num`, that fails D7.
3. **Opening.** A session is opened from a stream handle, with its options. Opening sends nothing: the session makes its first request when it has a batch to send. A response that ends cleanly while nothing is unacked uses no budget and is not an error (T9); the session makes a new request when it next has a batch to send. An error on any attempt, the first included, is handled as T1 says, and reaches the caller through tickets and the session's failure.
   - **a.** An `AppendSession`'s options are `max_unacked_bytes` and `max_unacked_batches`.
   - **b.** A `Producer`'s options are batching (`linger`, `max_batch_records`, `max_batch_bytes`), an optional `fencing_token`, an optional starting `match_seq_num`, and `max_unacked_bytes`. It opens the `AppendSession` it owns, and closing or aborting the Producer closes or aborts that session.
4. **Acceptance.** `submit()` waits until there is capacity for the record or batch, then accepts it and returns a ticket. Where an SDK splits it in two (see [Languages](#accepting-a-record-or-batch)), a rule about `submit()` applies to the pair.
   - **a.** Capacity is `max_unacked_bytes` and, on an `AppendSession`, `max_unacked_batches`. An accepted record counts against it until its batch has an outcome, including while a Producer is still batching it. Capacity is granted first come, first served.
   - **b.** Records are ordered by acceptance.
   - **c.** `submit()` never half-accepts. A cancelled `submit()` accepted nothing. One still waiting when the session starts closing or closes, or made after, fails at once with `SessionClosedError`; `flush()` and `close()` never do.
5. **Batching.** A Producer batches the records it accepts, and a `Batcher` batches records for `append()` or an `AppendSession`, both by these rules. A batch is complete when it is full, or when `linger` has passed since its first record arrived, whichever comes first. A batch never exceeds `max_batch_records` or `max_batch_bytes`. The fencing token goes on every batch. When a starting `match_seq_num` is given, every batch carries one: that value on the first, then the previous batch's plus its record count.
   - **a. Producer.** A record arrives when it is accepted (A4), and a complete batch is sent at once.
   - **b. `Batcher`.** It takes a sequence of records and gives a sequence of batches (see [Languages](#batcher)). Its options are those a Producer has for batching, plus `fencing_token` and a starting `match_seq_num` (A3b). It reads no record while a complete batch waits to be taken. When its input ends, it gives any partial batch at once, then ends. When its input fails, or gives a record over 1 MiB or `max_batch_bytes`, it ends at once with that error or with `ValidationError`, and drops its partial batch. When its consumer stops taking batches, it stops and drops its partial batch. It drops a record in no other case.
6. **Outcomes.** Each batch has one outcome, shared by its records: a batch is atomic, so each record gets its batch's outcome. On success, an `AppendSession`'s ticket gives the batch's ack, and a Producer's gives a `RecordAck`: the record's own `seq_num`, and `batch`, its batch's ack. An outcome never changes; where a ticket can be awaited more than once, every await gives the same outcome.
   - **a.** If the final error came from the server, it belongs to the oldest unacked batch, provided that batch was sent in the final attempt: that batch was **rejected**. On an abort, no batch is rejected, and `AbortedError` takes the place of the final error. Each batch gets the first row that fits:

     | Batch | Outcome |
     |---|---|
     | Acked | the ack |
     | Rejected | the final error |
     | Behind a batch rejected with a definite error | `SessionClosedError` |
     | Never sent in any attempt, or still being batched by a Producer | `SessionClosedError` |
     | Any other | the final error |

   - **b.** A definite outcome is then wrapped if an earlier attempt that sent the batch may have taken effect (E3).
   - **c.** `SessionClosedError` behind a definite rejection relies on the server: once it rejects a batch with a definite error, no later batch in the session becomes durable. Where the server sees no session, the SDK gives that guarantee itself (C5).
   - **d.** A Producer never overrides an outcome its session reported.
7. **Flushing.** `flush()`, on a Producer only, sends any partial batch at once, then returns once every record accepted before it is acked. Otherwise it fails with the failure. A failure that affects only later records does not fail it, and neither does the Producer being closed. A `submit()` that returned before `flush()` began is covered; concurrent ones may fall on either side.
8. **Closing.** A session closes for one of three reasons.
   - **a. The caller.** `close()` makes the session closing: it stops accepting, sends any partial batch at once, and waits until everything accepted has an outcome, and for nothing else. The session then closes, ending its request (T10), and `close()` returns, or fails with the failure. Every call to `close()` that completes gives the same result.
   - **b. A failure.** The final error closes the session at once, closing or not, and every batch gets its outcome (A6).
   - **c. An abort.** Closing the client (C8), or a trigger that depends on the language (see [Languages](#ending-a-session)), closes the session at once, closing or not, and every batch gets its outcome (A6). The session's failure is an indefinite `AbortedError`, unless no batch was awaiting an outcome: then nothing was lost, and the close is clean. `AbortedError` carries what triggered the abort as its cause, in languages where errors have causes. Cancelling any other call only stops that call's wait (T7).
9. **Failure.** A session's failure is its final error, wrapped if it is definite and any batch's outcome was wrapped (A6b), or `AbortedError` (A8c). A Producer's failure is that of the session it owns, or `AbortedError`.

**Example.** A session accepted batches A, B and C, in that order. A and B were sent; C was not. In each row, the last error is final: it was not retried. "closed" is `SessionClosedError`, `412` is either error of E1a, "aborted" is `AbortedError`, and `Indefinite(x)` is `IndefiniteAppendError` holding `x`.

| What happened | A | B | C | `close()` |
|---|---|---|---|---|
| Server rejects A with `412` | `412` | closed | closed | `412` |
| Server fails A with `storage` (indefinite) | `storage` | `storage` | closed | `storage` |
| Connection lost | `ConnectionLostError` | `ConnectionLostError` | closed | `ConnectionLostError` |
| Connection lost and retried; A and B resent; server rejects A with `412` | `Indefinite(412)` | `Indefinite(closed)` | closed | `Indefinite(412)` |
| Client closed while A and B await acks | aborted | aborted | closed | aborted |

## R. Reading

`read()`, `check_tail()` and `ReadSession`. Rules R3 to R9 cover the session.

1. **Unary read.** `read()` returns one batch as the server sent it: its records, and the tail if the server reported one. A batch has no records only when a stop condition was met before any record, such as `wait` running out at the tail; it does not mean the end of the stream. With `wait` zero, a read that starts at the tail gets `416` from the API instead (E1b).
2. **Tail.** `check_tail()` returns the tail.
3. **Opening.** Opening a read session makes its first request at once, without waiting for a response. An error on any attempt, the first included, is handled as T1 says, and reaches the caller through iteration (R7c).
4. **Iteration.** Iterating a session yields records, one at a time, in sequence-number order, each delivered once.
   - **a.** Batches are how the server ships records, and carry no meaning. An SDK may also offer a batch view; it delivers each batch whole, never an empty one, and every rule here applies to it per batch. Heartbeats change state (R5), not what is delivered.
   - **b.** The session decodes at most one batch ahead of the one being delivered. Any further read-ahead is the transport's flow control.
   - **c.** Iteration has one consumer. The accessors in R5 are safe to call from anywhere, at any time, and never block.
5. **State.** Three accessors. They are updated before a record is handed over, so code that receives it already sees its effect. After iteration ends, they keep their last values, except `is_caught_up` (R5c).
   - **a. `next_seq_num`.** One past the last record delivered, or the start before any is. A `seq_num` start that is not clamped is known at once; any other start is resolved by the first response, to its first record's sequence number or else its tail (R9b), and until then `next_seq_num` is unknown. When delivery reaches a heartbeat, `next_seq_num` moves up to its tail: the server had nothing to send before it. Records removed by trimming or retention are absent, so it can jump. A new session started at `next_seq_num` delivers what this one would have delivered next.
   - **b. `last_observed_tail`.** The latest tail the server has reported, whether or not the session has delivered up to it. It never moves backward, and carries across attempts. `last_observed_tail.seq_num - next_seq_num` is a lower bound on how far behind the caller is.
   - **c. `is_caught_up`.** True while the session's request is open and the last thing the server reported on it is a tail equal to `next_seq_num`: everything up to a reported tail has been delivered, and nothing newer is known. A response that ends on a stop condition counts as open until iteration ends, so its last record can still catch the session up. Once iteration ends, it is false.
6. **Catching up.** `caught_up()` returns a future that resolves once the session is caught up: at once if it is now, or else the next time it is.
   - **a.** It resolves with the tail the session caught up to, and its value never changes.
   - **b.** It does not read: the caller keeps iterating while it is pending.
   - **c.** It stays pending across attempts.
   - **d.** If the session closes before catching up, it fails at once, with the failure, or with `SessionClosedError` when there is none.
7. **Closing.** A read session closes for one of four reasons.
   - **a. A stop condition.** `count`, `bytes` or `until` was reached, or `wait` ran out, as the server reports by ending its response cleanly (T9). Iteration finishes with no error.
   - **b. The caller.** The caller closes the session (see [Languages](#ending-a-session)). A wait for the next record that is pending then, or made after, fails with `SessionClosedError` (E4): only a stop condition ends iteration cleanly. In TypeScript, a read after `cancel()` ends as the Streams standard says. Cancelling a wait for the next record only stops that wait: nothing is lost, and the session stays open.
   - **c. A failure.** Every record already fetched is delivered first; then iteration fails with the failure, once, and then finishes.
   - **d. An abort.** Something outside the code that iterates the session aborts it: closing the client (C8), or the session's own abort where the language has one (see [Languages](#ending-a-session)). Records fetched but not delivered are dropped, and iteration fails with `AbortedError` (A8c), even when nothing was fetched: finishing cleanly would look like a stop condition.
8. **Failure.** A read session's failure is its final error, or `AbortedError` (R7d). A read has no side effects, so nothing is wrapped.
9. **New requests.** A session's later requests, after a retry or a handover (T1, T4, T5), are invisible to iteration.
   - **a.** Each resumes at the first record the session has not fetched, so nothing is repeated and nothing is missed.
   - **b.** A start given by timestamp or tail offset is resolved once. After the first response the session resumes by sequence number, never by re-evaluating the original start.
   - **c.** `count` and `bytes` cover the whole session, not each request, and a new request asks only for what is left of them. It asks for `wait` less the time since the session last fetched a record, or, before it has fetched any, since it first learned it was at the tail, rounded up to whole seconds. Heartbeats do not restart this clock. Only the server ends a session for `wait`.
   - **d.** Once the start is resolved, a new request made with no `wait` left that the server answers with `416` has reached the tail: the session closes on a stop condition (R7a), not a failure.

**Example.** A stream holds records 0–4, so its tail is 5. A session starts at 0.

| Step | `next_seq_num` | `last_observed_tail` | `is_caught_up` |
|---|---|---|---|
| Opened, nothing fetched | 0 | unknown | false |
| Records 0–4 fetched with tail 5; record 0 delivered | 1 | 5 | false |
| Records 1–4 delivered | 5 | 5 | true |
| Heartbeat with tail 5 | 5 | 5 | true |
| Another writer appends record 5; it is fetched with tail 6 | 5 | 6 | false |
| Record 5 delivered | 6 | 6 | true |
| The request breaks | 6 | 6 | false |
| New request; heartbeat with tail 6 | 6 | 6 | true |

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
2. **Ensure.** Ensuring a basin or stream creates it if it is missing, and otherwise makes its config the given one: a field the caller leaves out takes its default, as on create, unlike reconfigure (M3). It returns the resource and what its last attempt did, from the `s2-provision-result` header: `created`, `updated` or `noop`. A response without one of these values is `created` if its status is `201`, and `updated` otherwise. After a retry, a resource this call created may be reported as `updated` or `noop`.
3. **Reconfigure.** Reconfiguring changes only the fields it is given, and returns the config as it now stands. Each field has three states: left unchanged, which is the default; set to a value; or reset to the server's default.
4. **Delete.** Deleting a resource makes sure it is gone. On any attempt, it succeeds when the resource is already being deleted, or when the server reports the resource itself not found: `basin_not_found` for `delete_basin()`, `stream_not_found` for `delete_stream()`, and `access_token_not_found` for `revoke_access_token()`. Any other `404`, including one with no code, fails as usual. Revoking an access token is how it is deleted. Deleting a basin or stream starts its deletion, and until that finishes the resource is being deleted (M8).
5. **Issue.** Issuing an access token returns it, and the server reveals it only in this response. The SDK does not keep it or log it (L3), and retries only definite errors (T2d).
6. **List a page.** One call, which returns one page as the server sent it: the items, and `has_more`, which says whether more remain. It takes `prefix`, `start_after` and `limit`.
7. **List all.** Iterates every match lazily, fetching the next page only when the caller reaches it. It takes `prefix` and `start_after`. Each page is a unary call with its own retries, and a page that fails ends the iteration with that error.
8. **Resources being deleted.** A page from M6 includes basins and streams that are being deleted, each marked as such. Iteration under M7 leaves them out, unless `include_deleted` is set.

## F. Forward compatibility

A released SDK keeps working against a newer server.

1. **Parsing.** A response never fails to parse because of a field or value this SDK version does not know.
2. **Open enums.** Every enum the server sends is open: a value this SDK version does not know is exposed by its wire name, in the language's idiom (an `Unknown` variant, or an open string type). A type declared closed that can hold an unlisted value at run time, such as a TypeScript string union, does not meet this rule. Ensure's result is the exception: M2 says how it reads a value it does not know.
3. **Closed enums.** Every enum only the SDK sends is closed.
4. **Error codes.** An error code the SDK does not know follows E5c.

## L. Logging

1. **Off by default.** The SDK logs through the language's usual facility, and emits nothing unless the application turns it on.
2. **Debug or finer.** The SDK logs only at debug level or finer, even for retries, reconnects and errors. An error reaches the application as an error, not as a log line.
3. **Secrets.** Record bodies, access tokens and encryption keys never appear in a log line or an error's message. Tokens and keys are also masked when a client, a handle or their settings are printed.

## Settings and constants

Every SDK has these settings, with these names and defaults (C2).

| Setting | Set on | Default |
|---|---|---|
| `endpoints` | client | `a.s2.dev` and `{basin}.b.s2.dev` |
| `connection_timeout` | client | 3 s |
| `request_timeout` | client | 5 s |
| `max_attempts` | client | 3 (minimum 1) |
| `min_base_delay`, `max_base_delay` | client | 100 ms, 1 s |
| `append_retry_policy` | client | `all` |
| `compression` | client | `none` |
| `retry_policy` | `ReadSession` | `budgeted` |
| `max_unacked_bytes` | `AppendSession`, `Producer` | 5 MiB (minimum 1 MiB) |
| `max_unacked_batches` | `AppendSession` | none |
| `linger` | `Producer`, `Batcher` | 5 ms |
| `max_batch_records`, `max_batch_bytes` | `Producer`, `Batcher` | 1000 records, 1 MiB (also the maximum) |

Constants are fixed, and stated in the rule they belong to. This table only gathers them.

| Constant | Value | Rule |
|---|---|---|
| Largest record | 1 MiB | A2a |
| Largest batch | 1000 records, 1 MiB | A2b |
| Longest fencing token | 36 bytes | A2c |
| Smallest session frame that is compressed | 1 KiB | C4 |
| Shortest gap between acting on reconnect advice | 60 s | T5a |
| Heartbeat timeout | 20 s | T6d |
| Batches a read session decodes ahead of delivery | 1 | R4b |

## Server guarantees

Some rules are correct only because the server behaves a certain way. These are those guarantees, the rules that rely on them, and where each is documented, or else seen.

| Guarantee | Relied on by | Source |
|---|---|---|
| A batch becomes durable whole, or not at all. | A6 | [Appends](https://s2.dev/docs/concepts/appends) |
| A session's batches become durable in the order they were sent, and once one fails, no later batch does. | A6a, A6c | [Appends](https://s2.dev/docs/concepts/appends) |
| An append session over the append rate limit is slowed, not failed. | A4 | [Limits](https://s2.dev/docs/platform/limits) |
| A `412` means the append condition did not hold, and nothing was appended. | Terms, E5e | [Concurrency control](https://s2.dev/docs/concepts/concurrency-control) |
| Every session frame the server sends is within the limits `s2s.md` states, and any valid batch fits within them. | A2, C4, T9 | [`s2s.md`](../s2/v1/s2s.md) |
| An append session's response ends cleanly only once everything it received is acked. A read session's ends cleanly only on a stop condition or after reconnect advice. | T9, R7a | [`s2s.md`](../s2/v1/s2s.md) |
| An append session that stops sending after reconnect advice gets an ack for everything it sent, then a clean end. | T5a | [`s2s.md`](../s2/v1/s2s.md) |
| A session ended with `server_draining` sends every ack first; whatever it did not ack was never processed. | T5b | [`s2s.md`](../s2/v1/s2s.md) |
| A read session waiting at the tail gets a heartbeat at least every 15 s. | T6d | [`s2s.md`](../s2/v1/s2s.md) |
| A heartbeat carries the tail, and is sent only when the server has nothing left to send before it. | R5a, R5c, R6 | [`s2s.md`](../s2/v1/s2s.md) |
| With `wait` zero, a read that starts at the tail returns `416`. | R1, R9d | [Read](https://s2.dev/docs/api/records/read) |
| A read with no start begins at the tail. | D3 | s2-lite; not yet documented |
| An error the error-code table lists with no side effect had none. | Terms, E5a | [`error-codes.json`](../s2/v1/error-codes.json) |
| A unary call refused for its rate carries `retry-after-ms`. | T3 | the service; not yet documented |
| A create retried with the same request token does not conflict with its earlier attempt. | T2b, M1 | [`openapi.json`](../s2/v1/openapi.json) |
| Ensuring a basin or stream that exists makes its config the given one, with omitted fields at their defaults. | M2 | s2-lite; not yet documented |
| Ensure reports what it did in the `s2-provision-result` header: `created`, `updated` or `noop`. | M2 | s2-lite and the service; not yet documented |
| Deleting a basin or stream that is already being deleted succeeds. | M4 | s2-lite and the service; not yet documented |
| Revoking an access token that does not exist returns `404` with `access_token_not_found`. | M4 | the service; not yet documented |

## Languages

Where a rule depends on the language, this is what it means in each. Every table has a row per language, so a column compares them. An entry is tested under the rule it cites.

### Numbers and byte inputs

| | A sequence number, timestamp, count or byte limit (D2, D7) | A mutable input the SDK takes (C10) |
|---|---|---|
| Python | `int` | a `bytearray` is copied when taken |
| TypeScript | `number`, so its range ends at 2⁵³ − 1 (`Number.MAX_SAFE_INTEGER`) | a `Uint8Array` is copied when taken |
| Go | `uint64` | a `[]byte`, and anything a config or option holds by reference, is copied when taken |
| Rust | `u64` | none: an input is moved in or borrowed for the call |

### Accepting a record or batch

Go splits `submit()` (A4) in two, and Rust may.

| | Accept (A4) | Validate (A2) |
|---|---|---|
| Python | `submit()` | in `submit()` |
| TypeScript | `submit()` | in `submit()` |
| Go | `Submit()`, then `Wait()` on what it returns, which waits for capacity and gives the ticket | in `Submit()` |
| Rust | `submit()`; or `reserve()`, which waits for capacity for a given size, then the permit's `submit()`, which accepts at once and gives the ticket | in `submit()`; when split, `reserve()` checks the size it reserves, and the permit's `submit()` checks the rest, releasing the reservation if it fails. A batch larger than its reservation is invalid. |

### Batcher

A `Batcher` (A5b) is a stream transform, in each language's idiom. Go's is a function, named for what it does, as `slices.Chunk` is.

| | Shape (A5b) |
|---|---|
| Python | `Batcher(records, ...)`, an async iterable of batches over an async iterable of records |
| TypeScript | `Batcher`, a `TransformStream` from records to batches |
| Go | `Batch(ctx, records, opts)`: an `iter.Seq2[*AppendInput, error]` over a `<-chan AppendRecord` that the caller closes when its input ends. Leaving the loop stops it; cancelling `ctx` ends it with the context's error. |
| Rust | `Batcher`, a `Stream` of batches over a `Stream` of records |

### Ending a session

| | Close an append session or Producer (A8a) | Abort one (A8c) | Close a read session (R7b) | Abort one, besides closing the client (R7d) |
|---|---|---|---|---|
| Python | `close()`, or leave `async with` normally or on an `Exception` | cancel `close()`, or leave `async with` on any other `BaseException` (a cancellation, `KeyboardInterrupt`, `GeneratorExit`) | `close()`, or leave `async with` | none |
| TypeScript | `close()`, or `await using` | the abort signal given when opening it | `cancel()` it (it is a `ReadableStream`), or `await using` | the abort signal given when opening it |
| Go | `Close()` | cancel the context given when opening it | `Close()` | cancel the context given when opening it |
| Rust | `close()` | drop it, including an unfinished `close()` | drop it | none |

### Cancelling a call

A unary call, or a wait on a session (T7).

| | How (T7) | What the caller sees (T7) |
|---|---|---|
| Python | cancel the task | the task's cancellation |
| TypeScript | the call's abort signal | `AbortedError` |
| Go | cancel the call's context | `AbortedError` |
| Rust | drop the future | nothing: the future is gone |

### Errors

| | An error the spec names `XError` (E2) | The base type (E8) | A typed server error (E1) | A wrapper's cause, besides its accessor (E3b) | The cause `AbortedError` carries (A8c) |
|---|---|---|---|---|---|
| Python | class `XError`; `TimeoutError` also extends the builtin `TimeoutError` | class `S2Error`, which every error class extends | a subclass of `ServerError` | `__cause__` | the exception |
| TypeScript | class `XError`, whose `name` is `"XError"` | class `S2Error`, which every error class extends | a subclass of `ServerError` | `cause` | `signal.reason` |
| Go | type `*XError`, matched with `errors.As` | interface `S2Error`, which every error type implements | a type that `errors.As` also matches as `*ServerError` | none: the inner error stays out of the `Unwrap` chain | the context error, in the `Unwrap` chain, so `errors.Is(err, context.Canceled)` holds |
| Rust | variant `X` of the call's error enum | `S2Error`, which every call's error enum converts into | a variant of the call's error enum that carries the `ServerError` | `source()` | none |

### TypeScript stream adapters

An `AppendSession` or `Producer` also offers `writable`, which takes what `submit()` takes, and `readable`, which yields acks. A write is a `submit()` (A4), made when the stream hands it on, in the order written. A write queued before `close()` or `flush()` began is covered by it, like a `submit()` that returned before it (A7, A8a): they wait for the queue to drain into `submit()` first. A write still queued when the session fails or is aborted fails with the session's failure. Closing `writable`, or cancelling `readable` (as leaving a `for await` over it early does), is `close()` (A8a): the session stops accepting, and what it already accepted still gets its outcome. Aborting `writable` aborts the session (A8c). On failure, `readable` errors with the failure.

### TypeScript over `fetch`

Where TypeScript carries calls over `fetch` (C5), the runtime chooses the connections, so T5's move to a different connection holds only as far as the runtime honours the server's GOAWAY. A browser hides a redirect's status, so there a redirect fails with `ProtocolError` rather than a `ServerError` (C12).

## Conformance

An SDK conforms to a revision when it follows every rule in it. Each SDK's README names the revision it targets. To make conformance checkable:

- Every testable rule has a test in each SDK, tagged with its id, such as `A6b`.
- A testable rule without a tagged test counts as not yet met.
- Where a rule's cases are data, they live in [`vectors/`](vectors/), and every SDK runs every case through its s2-specs submodule. A case it fails counts as the rule not yet met.
- When a revision changes a rule, the Revisions section names it, so each SDK knows which tests to revisit.

## Revisions

- **1.** Initial version.
