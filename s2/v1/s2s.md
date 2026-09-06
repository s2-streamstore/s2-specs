#### Setup

* `Content-Type: s2s/proto` signals that a session is being requested.
* `Accept-Encoding` signals which compression algorithms are supported (service supports `zstd` and `gzip`). `Content-Encoding` is not sent as message-level compression is used.
* `200 OK` response establishes a session.

#### Message framing

_All integers use big-endian byte order. Messages smaller than 1KiB should not be compressed._

**Length prefix** (3 bytes): Total message length (flag + body)

**Flag byte** (1 byte): `[T][CC][A][RRRR]`
- `T` (bit 7): Terminal flag (`1` = stream ends after this message)
- `CC` (bits 6-5): Compression (`00`=`none`, `01`=`zstd`, `10`=`gzip`)
- `A` (bit 4): Reconnect advised (`1` = server suggests reconnecting, only on messages from *Server → Client*)
- `RRRR` (bits 3-0): Reserved

**Body** (variable):
- Regular message is a Protobuf
- Terminal message contains a 2-byte status code, followed by JSON error information (corresponding to unary response behavior)

#### Data flow

**Append** sessions are a bi-directional stream of [`AppendInput`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendInput) messages from *Client → Server*, and [`AppendAck`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendAck) messages from *Server → Client*.

**Read** sessions are a uni-directional stream of [`ReadBatch`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.ReadBatch) messages from *Server → Client*. When waiting for new records, an empty batch is sent as a heartbeat at least every 15 seconds.

#### Reconnect advice

A server that is about to terminate sets the reconnect-advised flag on regular messages.

* **Read** sessions should be re-established with a fresh request.
* **Append** sessions should stop sending inputs and half-close the request stream. The server acknowledges all accepted inputs and then ends the session cleanly. A new session can be established concurrently, and should start being used once all acknowledgements for in-flight appends have been received.
* Append sessions still attached when the server drains are ended by the server: it stops reading inputs, acknowledges accepted inputs, and sends a terminal `503` with error code `server_draining`. Acknowledgements always precede the terminal message, so an input is either acknowledged or was never processed, and unresolved inputs can be safely resubmitted on a new session.
