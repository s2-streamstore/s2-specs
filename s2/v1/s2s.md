#### Setup

* `Content-Type: s2s/proto` signals that a session is being requested.
* `Accept-Encoding` signals which compression algorithms are supported (service supports `zstd` and `gzip`). `Content-Encoding` is not sent as message-level compression is used.
* `200 OK` response establishes a session.

#### Message framing

_All integers use big-endian byte order. Messages smaller than 1KiB should not be compressed._

**Length prefix** (3 bytes): Total message length (flag + body)

**Flag byte** (1 byte): `[T][CC][RRRRR]`
- `T` (bit 7): Terminal flag (`1` = stream ends after this message)
- `CC` (bits 6-5): Compression (`00`=`none`, `01`=`zstd`, `10`=`gzip`)
- `RRRRR` (bits 4-0): Reserved

**Body** (variable):
- Regular message is a Protobuf
- Terminal message contains a 2-byte status code, followed by JSON error information (corresponding to unary response behavior)

#### Data flow

**Append** sessions are a bi-directional stream of [`AppendInput`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendInput) messages from *Client → Server*, and [`AppendAck`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.AppendAck) messages from *Server → Client*.

**Read** sessions are a uni-directional stream of [`ReadBatch`](https://buf.build/streamstore/s2/docs/main:s2.v1#s2.v1.ReadBatch) messages from *Server → Client*. When waiting for new records, an empty batch is sent as a heartbeat at least every 15 seconds.
