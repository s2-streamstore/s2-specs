#### Setup

* `Content-Type: s2s/proto` signals that a session is being requested.
* `Accept-Encoding` signals which compression algorithms are supported (service supports `zstd` and `gzip`). `Content-Encoding` is not sent as message-level compression is used.
* `200 OK` response establishes a session.

#### Message framing

_All integers use big-endian byte order. Messages smaller than 1KiB should not be compressed._

**Length prefix** (3 bytes): Total message length (flag + body), at least 1 and at most 2.5 MiB

**Flag byte** (1 byte): `[T][CC][A][RRRR]`
- `T` (bit 7): Terminal flag (`1` = stream ends after this message)
- `CC` (bits 6-5): Compression (`00`=`none`, `01`=`zstd`, `10`=`gzip`)
- `A` (bit 4): Reconnect advised (`1` = server suggests reconnecting, only on messages from *Server → Client*)
- `RRRR` (bits 3-0): Reserved (ignored on receive)

**Body** (variable):
- Regular message is a Protobuf
- Terminal message contains a 2-byte non-2xx HTTP status code (`300`–`599`), followed by JSON error information (corresponding to unary response behavior); any other status code is a protocol error

A message's body decompresses to at most 2.5 MiB − 1 bytes, enough for any valid batch. Either side treats a message past these limits as a protocol error. A terminal message is never compressed and never carries reconnect advice (its non-terminal flag bits are ignored on receive).

#### Data flow

**Append** sessions are a bi-directional stream of [`AppendInput`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendInput) messages from *Client → Server*, and [`AppendAck`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendAck) messages from *Server → Client*. The response ends cleanly only once every input the server received is acknowledged; otherwise it ends with a terminal message. Before emitting any terminal error classified with `side_effects: "none"` in [`error-codes.json`](error-codes.json) (including `server_draining`, `hot_server`, `412`, and mid-session input rejections), the server emits acknowledgements for every earlier accepted input that became durable and guarantees that no remaining unacknowledged input can become durable.

**Read** sessions are a uni-directional stream of [`ReadBatch`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.ReadBatch) messages from *Server → Client*. A heartbeat is an empty batch carrying the tail, sent only when the server has nothing earlier to send; while waiting for new records, one is sent at least every 15 seconds. The response ends cleanly only when the read's limits, `until`, or `wait` end it. When a read session with no `wait` (either `wait` is `0`, or `wait` is omitted with `count`, `bytes`, or `until` set) reaches the tail with no earlier records to send—including starting at the tail, across a trimmed gap up to the tail, or after clamping to the tail—the server establishes `200 OK`, emits a heartbeat carrying the tail, and ends the response cleanly rather than returning `416`; `416` is returned only when the requested starting position is beyond the tail without `clamp`.

#### Reconnect advice

A server that is about to terminate sets the reconnect-advised flag on regular messages.

* **Read** sessions should be re-established with a fresh request. Read sessions still attached when the server drains are ended by the server with a terminal `503` with error code `server_draining`.
* **Append** sessions should stop sending inputs and half-close the request stream. The server acknowledges all accepted inputs and then ends the session cleanly. A new session can be established concurrently, and should start being used once all acknowledgements for in-flight appends have been received.
* Append sessions still attached when the server drains are ended by the server: it stops reading inputs, acknowledges accepted inputs, and sends a terminal `503` with error code `server_draining`. Acknowledgements always precede the terminal message, so an input is either acknowledged or was never processed, and unresolved inputs can be safely resubmitted on a new session.
* Under severe load shedding (such as when the frontend cannot reserve initial session memory), the server may also end a session with a terminal `502` and error code `hot_server` and close the connection; as with other `side_effects: "none"` terminal errors, no unacknowledged input became durable, and clients should retire the connection and reconnect on a new connection with backoff.
