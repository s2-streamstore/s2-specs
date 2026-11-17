# SDK spec rationale

Why [the SDK spec](spec.md) says what it does, where the reason is not obvious, by rule id. The spec is the requirement; this file is not. It records the reasons so that a later revision can change a rule knowing what it was for. When a revision changes or removes a rule, update its entry here.

Every reason below argues from the spec's four standards: correctness, elegance, idiom and efficiency.

## Reading the rules

- **Stable ids.** Each SDK tags its tests with rule ids. Renumbering would silently break those tags in four repositories, so a removed rule's id is retired, never reused.
- **Canonical names.** When four SDKs name one thing four ways (`inflight` and `unacked`; `RangeNotSatisfiableError` and `ReadUnwritten`), docs, support and tests across SDKs all get harder. Casing and naming idiom stay each language's own; the Languages section shows how for error types.
- **Every rule is a requirement.** Sentences of advice to callers left it unclear whether an SDK had to do anything. Advice belongs in each SDK's docs.

## Terms and the error model

- **Definite.** On a session, an attempt is a connection that may already have acked batches. Judging an error against the whole connection would make every late error indefinite, so it is judged against the batches that were unacked when it arrived. That also lets T2c say simply "only after a definite error".
- **Closing.** Between `close()` and the session closing, a `submit()` waiting for capacity could otherwise wait behind a close it can never outlive. Naming the state lets A4c fail it at once, and lets A8 say what a failure or an abort does while it lasts.
- **Error model.** The spec's core is one chain of ideas: attempt, definite, final error, wrapped, outcome, failure. Their definitions are spread across Terms, E, T and A, and are each right where they stand, but a reader had to assemble the model. The section tells it once and cites the rules, so it can never disagree with them.

## C. Client

- **C1.** A client that reads the environment on its own behaves differently from one machine to the next, and can pick up a token the caller did not mean to use. `from_env()` keeps the convenience and makes it visible.
- **C2.** A setting out of range is a mistake in the caller's code, so it fails where it is made, not later on some call that uses it.
- **C8.** Every client has `close()`, so every language has one way to end everything a client started, and Python and Rust, which have no other way to abort a read session, have that one. A client dropped without `close()` stays alive while its sessions are open, so a session never fails because its owner let go of a handle. A unary call in flight when the client closes fails with `AbortedError`, as if cancelled, because it may already have taken effect; `ClientClosedError` is kept for calls made after, which certainly sent nothing.

## D. Data

- **D1.** `AppendRecord` and `SequencedRecord` are the names three of the four SDKs already use for each.
- **D2.** A record's timestamp is integer milliseconds, and a client may choose it, so it need not be a wall-clock time. A date type implies a meaning, and conversions, that the API does not have, and cannot hold every `u64` exactly; accepting one as input implies the same. Times the API gives as RFC 3339 date-times, such as `created_at`, are wall-clock times, so a date type fits them.
- **D3.** A start is optional, as in the API, which begins at the tail; D4's reasoning applies.
- **D4.** `wait` and the other stop options are the API's, with the API's defaults. The SDK leaves an option the caller did not set to the API rather than choosing its own default, so a read behaves the same through every SDK and through the API directly. The spec does not restate what the options mean, or when the API answers `416`, so it cannot drift from the API's docs; the server guarantees table lists the one case rules rely on.
- **D5.** `ignore_command_records` was removed. Filtering in the SDK hid where a read got to: a unary read whose records were all command records returned nothing, and the caller could not tell where to resume. Putting that back needed a `next_seq_num` the server does not send, with awkward cases of its own. `is_command_record()` makes filtering one line for the caller, and a read then always shows where it got to.

## E. Errors

- **E1.** The `412` and `416` errors are the server errors callers handle most, so each gets a type. They are named for what happened, not for the HTTP status: `ReadUnwrittenError` rather than `RangeNotSatisfiableError`, to match `SeqNumMismatchError` and `FencingTokenMismatchError`. The `412` fields hold the stream's values, so they are named for them, in the docs' words: `current_fencing_token` and `assignable_seq_num`. "Expected", as three SDKs had it, reads as the value the caller sent.
- **E2.** Only a `ServerError` carries a status, and SDK errors are told apart by type. An SDK error with a made-up status (TypeScript's `499`) or code (Go's `VALIDATION`) can be mistaken for one from the server. One `TimeoutError` covers responses, acks and heartbeats: the three are classified alike and handled alike, and the message says which fired. Python's also extends the builtin `TimeoutError`, so code that catches the builtin catches it too.
- **E3.** A retried append can fail definitely on its last attempt after an earlier attempt took effect. Reporting the bare definite error would tell the caller "not appended" when that is unknown. The wrapper does not match as its inner error's type, so code that catches `SeqNumMismatchError` to mean "nothing was appended" is not misled. The inner error can be the wrapper's cause in Python, TypeScript and Rust because `except`, `instanceof` and `match` do not follow causes. Go's `errors.Is` and `errors.As` do follow `Unwrap`, so a Go wrapper does not unwrap to its inner error. `AbortedError` is not a wrapper, so its cause can stay in Go's `Unwrap` chain. The type was `AppendIndefiniteFailureError`; the new name is shorter, and says that the append's outcome is indefinite.
- **E4.** The closed error is its own type, so a caller can tell "this request was dropped because the session closed" from the error that closed it, which `failure` holds.
- **E5.** `is_retryable` has one meaning: the cause may be temporary. Whether repeating is safe is `has_no_side_effects`. An earlier table mixed the two, so a wrapped append error said "not retryable" while an unwrapped indefinite `503` said "retryable", though both were as unsafe to repeat. A code at an unexpected status, or no code at all (as from a load balancer), falls back to the status, since the code cannot be trusted. A `412`'s body names the expected value rather than a code, so the error-code table cannot classify it, and Terms makes it definite directly. "Definite" and `has_no_side_effects` name one idea twice: the spec needs a short word, and callers a descriptive one; renaming the predicate would cost every SDK a change for no gain in meaning.
- **E7.** In Node, an unhandled rejection can end the process, and Python's asyncio logs an error for an exception nobody retrieved. A caller who ignores tickets and relies on `close()` should see neither.
- **E8.** A caller that handles every S2 error the same way, say by logging it and moving on, needs one type to catch. Each SDK had one, but not every error derived from it. Go spells the base as an interface, since its error types are separate structs.

## T. Retries, timeouts and cancelling

- **T2a.** Ensure, reconfigure, delete and revoke each describe an end state, so repeating one cannot do more than the first did.
- **T2d.** Issuing an access token takes an `id` that must be unique, so a retry cannot make a second token. But if the first attempt succeeded and its response was lost, a retry fails with a definite `409`, which says nothing happened, while the token was created and its secret, revealed only in that lost response, is gone. Appends avoid the same trap by wrapping (E3); issuing has no wrapper, so it retries only when the first attempt certainly did nothing. A request token, as creates have, would need the server to keep the secret to send it again.
- **T4a.** A heartbeat shows the connection works. A session waiting at the tail sees only heartbeats; if they did not restore the budget, unrelated failures spread over hours of waiting would add up and end the session.
- **T4b.** Some readers, such as a consumer that follows a stream for its whole life, should never give up on a temporary failure. The policy is `unbounded` rather than `indefinite`, which everywhere else in the spec means "may have taken effect".
- **T5.** A planned handover is not a failure, so it uses no budget and does not back off. The append handover follows `s2s.md`, which has the server acknowledge everything it accepted before the session moves. A handover from a connection that never made progress counts as a retry, so a run of connections that each end at once on a draining server backs off and gives up like any other run of errors, instead of looping.
- **T6c.** An ack is the server's answer to a batch, so the ack timeout and `request_timeout` are one idea, "how long the server takes to answer", and one setting. Python already worked this way. Only the oldest unacked batch is timed. The server slows a session over its rate limit by reading it more slowly, so batches the SDK has already written can queue in the transport for longer than `request_timeout`; timing each from when it was written would fail a session that is only being throttled. The oldest batch is next in line, so its ack is due soon whatever is queued behind it. Its timer starts no earlier than its last byte is written, so a large batch on a slow link is not timed while still being sent.
- **T6d.** The timer runs only while the SDK waits for the server. A caller who takes a long time over a record stops the SDK reading, so flow control holds the server back and nothing arrives; counting that time would fail a session for being slow, not for being broken. Go and Rust already work this way.
- **T8.** The attempt header shows the service how often SDKs need to retry, without any reporting from the SDK. It is sent on the first try too, so a missing header means a client that does not send it, not a first attempt. On a session it counts only errors since the session last made progress, so a reconnect after one error reads `2` however long the session has run, and a planned handover never looks like an error. Counting retries used, rather than connections, needs no case of its own for a handover, which uses none, or for a transport where each request is its own attempt.

## A. Appending

- **A2.** Each limit is one the server enforces, except a Producer's `max_batch_bytes`, which a larger record could never fit in. Checking first gives the caller a `ValidationError` before anything is sent, and never closes a session over one bad record.
- **A3.** Opening an append session does not connect, because it has nothing to send. An idle connection that drops then costs nothing, and the first error a caller sees comes through a ticket or `close()`, like every later one.
- **A3b.** A Producer creates and owns its session. When the caller supplied the session, two parties could close it, and the spec needed a rule for a caller closing the Producer's session behind its back.
- **A5b.** Every SDK had a standalone batcher, under its own name and shape: Go's `Batcher`, TypeScript's `BatchTransform`, Rust's `AppendRecordBatches`, and Python's `append_record_batches()` and `append_inputs()`. It serves a caller who drives `append()` or an `AppendSession` directly and still wants batching. Its batches carry the fencing token and `match_seq_num`, so they go straight to `append()` or `submit()`.
- **A6.** One table decides every batch's outcome, so every SDK reports the same thing for the same error. The example after A9 is the check. A Producer's ticket gives the record's own `seq_num` because that, not the batch's, is what a caller of `submit()` wants. Its type is `RecordAck`, not `IndexedAppendAck` as three SDKs had it: it has no index, and it pairs with the batch's `AppendAck`.
- **A6c.** `SessionClosedError` for batches behind a definite rejection is true only because the server appends nothing after such a rejection. TypeScript's fetch transport has no server-side session, so it gives the guarantee itself: one batch in flight at a time, and nothing sent after a definite rejection.
- **A8c.** An idle append session's abort loses nothing, so it is a clean close. A read session's abort always fails iteration (R7d), because a clean finish would look like a stop condition to the code iterating it.

## R. Reading

- **R1.** `read()` returns the server's batch unchanged, so the next read always starts one past its last record, or where it started if it returned none.
- **R3.** Opening a read session connects at once, so read-ahead starts; errors still arrive through iteration, so a caller handles them in one place.
- **R4a.** A batch view delivers what the caller can act on. Heartbeats change state; delivering them as empty batches would expose the transport.
- **R4b.** Read-ahead already exists in the transport's flow control, measured in bytes and tuned per transport. One decoded batch keeps decoding off the caller's path; a deeper queue of batches, of up to 1 MiB each, would repeat what flow control does, with a less predictable bound on memory.
- **R5.** `next_seq_num` follows what was delivered, not fetched, so a checkpoint never skips a record the caller did not see. A heartbeat reached by delivery moves it up to the tail, because the server had nothing before it; otherwise a session started among trimmed records would never catch up. `last_observed_tail` and `is_caught_up` follow what the server reported, since that is all they can know, and `is_caught_up` holds only while connected, since a lost connection may have missed a newer tail. A connection that ends on a stop condition missed nothing, so it counts until iteration ends; otherwise whether the last record caught the session up would depend on whether the SDK had read the end of the response yet. A session whose iteration has ended observes nothing more, so it is not caught up, however it ended.
- **R7b.** A reader that closes a session after leaving its own loop sees nothing more. A reader whose session was closed by other code would otherwise finish cleanly, which looks like a stop condition, so a wait for the next record after a close fails with `SessionClosedError`, which is what that error means. This turns on what happens, not on which task called `close()`, which Go cannot know. TypeScript keeps the Streams standard, where reads after `cancel()` end cleanly; its abort signal is the way to stop a reader with an error.
- **R9d.** With `wait` zero, the API answers a read at the tail with `416`. A bounded session that reconnects there has nothing left to read, so that is its stop condition, not a failure. The rule applies only to a reconnect made with no `wait` left: a `416` with `wait` remaining means something else, such as a stream deleted and created again, and must not end a session silently.

## M. Managing basins, streams and access tokens

- **M2.** `created`, `updated` and `noop` are the values the server sends in `s2-provision-result`, and the names Go and TypeScript already use. The result describes only the last attempt, which is why, after a retry, it can say `updated` or `noop` for a resource the call created.
- **M4.** Deleting makes sure a resource is gone, so "already gone" is success. `ignore_not_found` was removed: the SDK cannot tell a retry's "not found" from a resource that never existed, so the error it guarded appeared only when no retry happened. Only the resource's own not-found code counts: a `404` for a missing basin when deleting a stream, or one with no code from a proxy, means the request was wrong, not that the resource is gone. Delete returns nothing about whether the resource was there, for the same reason: after a retry, "not found" can follow the call's own deletion. A caller who needs to know a resource existed can check first.
- **M6, M8.** A page is returned as the server sent it, so it is never short and `has_more` stays exact. `list_all_*` is the convenience, and leaves out resources being deleted, as all four SDKs already did.

## L. Logging

- **L2.** A library cannot know what deserves an application's attention. Errors are returned to the caller; logging them as well would report each one twice.

## Settings and constants

- **`linger`.** The 5 ms default matches the server's limit of 200 batches a second per stream and connection: a Producer whose batches are not full sends at most that many.
- **`max_attempts`.** It counts the first try, so it is at least 1.

## Languages

- **TypeScript stream adapters.** Leaving a `for await` over acks early is how a caller stops watching acks, not a request to throw away appends, so cancelling `readable` closes the session gracefully, as the Producer already did. Aborting `writable` stays the way to abort.
- **Python `async with`.** Leaving the block on an ordinary exception closes gracefully, so a bug in the body does not lose appends that were already accepted; the exception propagates once they have outcomes. Cancellation, `KeyboardInterrupt` and `GeneratorExit` abort, because the program is stopping.

## Server guarantees

Several rules are correct only if the server behaves a certain way. Listing those guarantees in one place, each with its source, makes them easy to check when the server changes.
