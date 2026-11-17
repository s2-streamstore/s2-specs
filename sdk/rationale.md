# SDK spec rationale

Why [the SDK spec](spec.md) says what it does, by rule id. The spec is the requirement; this file is not. It records the reasons so that a later revision can change a rule knowing what it was for. When a revision changes or removes a rule, update its entry here.

Every reason below argues from the spec's four standards: correctness, elegance, idiom and efficiency.

## Reading the rules

- **Stable ids.** Each SDK tags its tests with rule ids. Renumbering would silently break those tags in four repositories, so a removed rule's id is retired, never reused.
- **Canonical names.** When four SDKs name one thing four ways (`inflight` and `unacked`; `RangeNotSatisfiableError` and `ReadUnwritten`), docs, support and tests across SDKs all get harder. Casing and naming idiom stay each language's own, as the Languages table shows.
- **Every rule is a requirement.** Sentences of advice to callers left it unclear whether an SDK had to do anything. Advice belongs in each SDK's docs.

## Failure model

The spec's core is one chain of ideas: attempt, definite, final error, wrapped, outcome, failure. Their definitions are spread across Terms, E, T and A, and are each right where they stand, but a reader had to assemble the model. The section tells it once and cites the rules, so it can never disagree with them.

## C. Client

- **C1.** A client that reads the environment on its own behaves differently from one machine to the next, and can pick up a token the caller did not mean to use. `from_env()` keeps the convenience and makes it visible.
- **C7.** Closing the client is the one abort every language has. Python and Rust have no other way to abort a read session.

## D. Data

- **D1.** `AppendRecord` and `SequencedRecord` are the names three of the four SDKs already use for each.
- **D2.** The API's timestamps are integer milliseconds, and a client may choose them, so they need not be wall-clock times. A date type implies a meaning, and conversions, that the API does not have.
- **D4.** `wait` and the other stop options are the API's, with the API's defaults. The SDK leaves an option the caller did not set to the API rather than choosing its own default, so a read behaves the same through every SDK and through the API directly.
- **D5.** `ignore_command_records` was removed. Filtering in the SDK hid where a read got to: a unary read whose records were all command records returned nothing, and the caller could not tell where to resume. Putting that back needed a `next_seq_num` the server does not send, with awkward cases of its own. `is_command_record()` makes filtering one line for the caller, and a read then always shows where it got to.

## E. Errors

- **E1.** The `412` and `416` errors are the server errors callers handle most, so each gets a type. They are named for what happened, not for the HTTP status: `ReadUnwrittenError` rather than `RangeNotSatisfiableError`, to match `SeqNumMismatchError` and `FencingTokenMismatchError`.
- **E2.** Only a `ServerError` carries a status, and SDK errors are told apart by type. An SDK error with a made-up status (TypeScript's `499`) or code (Go's `VALIDATION`) can be mistaken for one from the server.
- **E3.** A retried append can fail definitely on its last attempt after an earlier attempt took effect. Reporting the bare definite error would tell the caller "not appended" when that is unknown. The wrapper does not match as its inner error's type, so code that catches `SeqNumMismatchError` to mean "nothing was appended" is not misled. Go keeps the inner error out of the `Unwrap` chain because `errors.As` follows it. The type was `AppendIndefiniteFailureError`; the new name is shorter, and says that the append's outcome is indefinite.
- **E4.** The closed error is its own type, so a caller can tell "this request was dropped because the session closed" from the error that closed it, which `failure` holds.
- **E5.** `is_retryable` has one meaning: the failure may be temporary. Whether repeating is safe is `has_no_side_effects`. An earlier table mixed the two, so a wrapped append error said "not retryable" while an unwrapped indefinite `503` said "retryable", though both were as unsafe to repeat. A code at an unexpected status, or no code at all (as from a load balancer), falls back to the status, since the code cannot be trusted. "Definite" and `has_no_side_effects` name one idea twice: the spec needs a short word, and callers a descriptive one; renaming the predicate would cost every SDK a change for no gain in meaning.

## T. Retries, timeouts and cancelling

- **T2a.** Ensure, reconfigure, delete and revoke each describe an end state, so repeating one cannot do more than the first did.
- **T2d.** Issuing an access token twice makes two tokens. Without a request token the SDK cannot tell a retry from a new request, so it retries only when the first attempt certainly did nothing.
- **T4a.** A heartbeat shows the connection works. A session waiting at the tail sees only heartbeats; if they did not restore the budget, unrelated failures spread over hours of waiting would add up and end the session.
- **T4b.** Some readers, such as a consumer that follows a stream for its whole life, should never give up on a temporary failure. The policy is `unbounded` rather than `indefinite`, which everywhere else in the spec means "may have taken effect".
- **T5.** A planned handover is not a failure, so it uses no budget and does not back off. The append handover follows `s2s.md`, which has the server acknowledge everything it accepted before the session moves.
- **T6c.** An ack is the server's answer to a batch, so the ack timeout and `request_timeout` are one idea, "how long the server takes to answer", and one setting. Python already worked this way.

## A. Appending

- **A2.** Each limit is one the server enforces, except a Producer's `max_batch_bytes`, which a larger record could never fit in. Checking first gives the caller a `ValidationError` before anything is sent, and never closes a session over one bad record.
- **A3b.** A Producer creates and owns its session. When the caller supplied the session, two parties could close it, and the spec needed a rule for a caller closing the Producer's session behind its back.
- **A6.** One table decides every batch's outcome, so every SDK reports the same thing for the same failure. The example after A9 is the check. A Producer's ticket gives the record's own `seq_num` because that, not the batch's, is what a caller of `submit()` wants.
- **A6c.** `SessionClosedError` for batches behind a definite rejection is true only because the server appends nothing after such a rejection. TypeScript's fetch transport has no server-side session, so it gives the guarantee itself, by sending one batch at a time.
- **A8c.** An idle append session's abort loses nothing, so it is a clean close. A read session's abort always fails iteration (R6d), because a clean finish would look like a stop condition to the code iterating it.

## R. Reading

- **R1.** `read()` returns the server's batch unchanged, so the next read always starts one past its last record, or where it started if it returned none.
- **R3a.** A batch view delivers what the caller can act on. Heartbeats change state; delivering them as empty batches would expose the transport.
- **R3b.** Read-ahead already exists in the transport's flow control, measured in bytes and tuned per transport. One decoded batch keeps decoding off the caller's path; a deeper queue of batches, of up to 1 MiB each, would repeat what flow control does, with a less predictable bound on memory.
- **R4.** `next_seq_num` follows what was delivered, not fetched, so a checkpoint never skips a record the caller did not see. `last_observed_tail` and `is_caught_up` follow what the server reported, since that is all they can know.

## M. Managing basins, streams and access tokens

- **M2.** `created`, `updated` and `unchanged` are parallel; the `config_` prefix on two of the three added nothing.
- **M4.** Deleting makes sure a resource is gone, so "already gone" is success. `ignore_not_found` was removed: the SDK cannot tell a retry's "not found" from a resource that never existed, so the error it guarded appeared only when no retry happened. A caller who needs to know a resource existed can check first.
- **M6, M8.** A page is returned as the server sent it, so it is never short and `has_more` stays exact. `list_all_*` is the convenience, and leaves out resources being deleted, as all four SDKs already did.

## L. Logging

- **L2.** A library cannot know what deserves an application's attention. Failures are returned to the caller; logging them as well would report each one twice.

## Server guarantees

Several rules are correct only if the server behaves a certain way. Listing those guarantees in one place makes them a contract the server can test, and shows which are not yet written down.
