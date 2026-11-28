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

**Append** sessions are a bi-directional stream of [`AppendInput`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendInput) messages from *Client → Server*, and [`AppendAck`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendAck) messages from *Server → Client*. The response ends cleanly only once every received input is acknowledged; otherwise it ends with a terminal message. Before any terminal error guaranteed to have no side effects (such as `412`, `server_draining`, `hot_server`, or per-batch validation rejections), the server acknowledges every earlier durable input and guarantees that no remaining unacknowledged input becomes durable, so clients can safely resubmit any unacknowledged inputs.

**Read** sessions are a uni-directional stream of [`ReadBatch`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.ReadBatch) messages from *Server → Client*. A heartbeat is an empty batch carrying the tail, sent only when the server has nothing earlier to send: at least every 15 seconds while waiting for new records, or before ending cleanly at the tail (guaranteed other than when exhausting `count` or `bytes`, unless the final record batch already caught up to the tail). The response ends cleanly only when the read's limits, `until`, or `wait` end it.

#### Reconnect advice

A server that is about to terminate sets the reconnect-advised flag on regular messages:

* **Read** sessions should re-establish on a new connection.
* **Append** sessions should stop sending inputs and half-close the request stream; the server acknowledges all accepted inputs and ends cleanly. A new session can be established concurrently and used once in-flight acknowledgements arrive.

Sessions still attached when the server drains end with a terminal `503` (`server_draining`). Under severe load shedding, the server may also end a session with a terminal `502` (`hot_server`) and close the connection; clients should retire the connection and reconnect on a new one with backoff.
