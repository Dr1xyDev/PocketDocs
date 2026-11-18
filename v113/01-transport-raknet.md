# 01 — Transport layer: RakNet over UDP

MCPE 1.1.x speaks its game protocol on top of a modified RakNet, carried over plain **UDP**. The default server port is **19132** (the client defaults to it; a server may listen anywhere, and the port is entered manually or discovered via LAN broadcast). RakNet provides reliability, ordering, fragmentation and session management; every MCPE "game packet" (see [02-framing-batching.md](02-framing-batching.md)) travels as the payload of exactly one RakNet encapsulated packet.

Reference for the byte formats below: `raklib/protocol/*` in Aquamarine (`stable`).

## Protocol constants

| Constant | Value |
|----------|-------|
| RakNet protocol version (sent in `Open Connection Request 1`) | **8** |
| Offline message *magic* (16 bytes) | `00 ff ff 00 fe fe fe fe fd fd fd fd 12 34 56 78` |
| Default MTU used in practice | 1492 (clients probe downward from here) |
| Maximum ordering channels | 32 |
| ACK message ID | `0xc0` |
| NACK message ID | `0xa0` |

## Offline (sessionless) messages

Offline messages are identified by a single message ID byte followed by the 16-byte *magic*. They are used before a session exists, and for ping/discovery at any time.

### Unconnected Ping (`0x01`) — C2S

Used by the server list and LAN discovery. Payload:

| Field | Type |
|-------|------|
| Send ping time | Long (8 bytes, BE in RakLib) |
| Magic | 16 bytes |
| Client GUID | Long (8 bytes, BE) |

### Unconnected Pong (`0x1c`) — S2C

| Field | Type |
|-------|------|
| Send ping time (echoed) | Long, BE |
| Server GUID | Long, BE |
| Magic | 16 bytes |
| Advertisement string | Short BE length + ASCII bytes |

The advertisement string is what the server-list entry displays. It is a semicolon-separated string:

```
MCPE;<MOTD>;<protocol number>;<network version>;<players online>;<max players>;<server GUID>;<software name>;<default gamemode name>;
```

For a protocol-113 server the protocol field is `113` and the version field is typically `1.1.7` (an empty version field is tolerated by the client).

### Open Connection Request 1 (`0x05`) — C2S

| Field | Type |
|-------|------|
| Magic | 16 bytes |
| RakNet protocol version | Byte (`8` for MCPE) |
| MTU probe padding | zero bytes, sized so the datagram equals the MTU being probed |

The client sends several probes at decreasing sizes (starting around 1492, falling back to 576 etc.); the largest that fits is chosen. If the server does not support the protocol version it answers with **Incompatible Protocol Version (`0x19`)**:

| Field | Type |
|-------|------|
| Server RakNet protocol version | Byte |
| Magic | 16 bytes |
| Server GUID | Long, BE |

### Open Connection Reply 1 (`0x06`) — S2C

| Field | Type |
|-------|------|
| Magic | 16 bytes |
| Server GUID | Long, BE |
| Has security | Byte (`0` for vanilla MCPE) |
| MTU size | Short, BE |

### Open Connection Request 2 (`0x07`) — C2S

| Field | Type |
|-------|------|
| Magic | 16 bytes |
| Server address | Address structure (see below) |
| MTU size | Short, BE |
| Client GUID | Long, BE |

**Address structure** (little-endian fields inside RakNet messages):

| Field | Type | Notes |
|-------|------|-------|
| IP version | Byte | 4 = IPv4, 6 = IPv6 |
| (IPv4 only) family | Byte | AF_INET = 2; servers write 2 |
| Port | Short, BE | |
| IPv4 address | 4 bytes | one byte per octet |
| (IPv6) family / port / flow info / address | — | AF_INET6 = 23, port BE, Int flow info BE, 16-byte address |

### Open Connection Reply 2 (`0x08`) — S2C

| Field | Type |
|-------|------|
| Magic | 16 bytes |
| Server GUID | Long, BE |
| Client address | Address structure |
| MTU size | Short, BE |
| Encryption enabled | Byte (`0`) |

### Connection Request (`0x09`) — C2S (inside a session)

| Field | Type |
|-------|------|
| Client GUID | Long, BE |
| Send ping time | Long, BE |
| Use security | Byte (`0`) |

### Connection Request Accepted (`0x10`) — S2C

| Field | Type |
|-------|------|
| Client address | Address structure |
| Unknown short | Short, BE (always 0) |
| System addresses | 20 × Address structure |
| Send ping time | Long, BE |
| Send pong time | Long, BE |

### New Incoming Connection (`0x13`) — C2S (response to `0x10`)

| Field | Type |
|-------|------|
| Server address | Address structure |
| System addresses | 20 × Address structure |
| Send ping time | Long, BE |
| Send pong time | Long, BE |

### Connected Ping (`0x00`) / Connected Pong (`0x03`)

Ping: one Long (BE) timestamp. Pong: the echoed timestamp followed by the responder's timestamp. Both are sent periodically inside an established session.

### Disconnection Notification (`0x15`)

Single byte, no payload. Graceful close from either side.

## Datagrams

Once a session exists, data is carried in datagrams. The datagram header is one byte:

| Bit | Meaning |
|-----|---------|
| `0x80` | Valid (always set) |
| `0x40` | ACK |
| `0x20` | NACK |
| `0x10` | Packet pair (unofficial; ignored by receivers) |
| `0x08` | Continuous send (unofficial; ignored) |
| `0x04` | Needs B and A/S (unofficial; ignored) |

Data datagrams (`0x80 | optional flags`) are followed by:

| Field | Type |
|-------|------|
| Sequence number | 3-byte little-endian triad |

Then a series of **encapsulated packets** until the datagram ends.

## Encapsulated packets

Each encapsulated packet carries exactly one higher-layer message (for MCPE: one `0xFE`-framed game packet or one RakNet control message):

| Field | Type | Present when |
|-------|------|--------------|
| Flags byte | Byte | always — reliability `<< 5`, split flag `0x10` |
| Payload length | Short, BE — **in bits** (`byteCount << 3`) | always; reader computes `ceil(bits / 8)` |
| Message index | 3-byte LE triad | reliability ≥ 2 (reliable) |
| Sequence index | 3-byte LE triad | reliability 1 or 4 (sequenced) |
| Order index | 3-byte LE triad | reliability 3 or 4 (ordered/sequenced) |
| Order channel | Byte | reliability 3 or 4 |
| Split count | Int, BE | split flag set |
| Split ID | Short, BE | split flag set |
| Split index | Int, BE | split flag set |
| Payload | length bytes | always |

### Reliability levels

| Value | Name | Reliable | Ordered |
|------:|------|:--------:|:-------:|
| 0 | Unreliable | – | – |
| 1 | Unreliable Sequenced | – | sequenced |
| 2 | Reliable | ✓ | – |
| 3 | Reliable Ordered | ✓ | ✓ |
| 4 | Reliable Sequenced | ✓ | sequenced |
| 5 | Unreliable with ACK receipt | – | – |
| 6 | Reliable with ACK receipt | ✓ | – |
| 7 | Reliable Ordered with ACK receipt | ✓ | ✓ |

MCPE game packets are always sent with **Reliable Ordered on order channel 0**, in both directions — vanilla 1.1.x clients use reliable-ordered channel 0 for session/login traffic and for gameplay alike.

### Split packets

Large payloads (a full Batch with chunks can exceed the MTU) are split into many encapsulated packets sharing one 16-bit split ID; the receiver reassembles by split index/count. This is the only fragmentation mechanism — there is no game-level fragmentation.

## ACK / NACK

Both start with the header byte (`0xc0` / `0xa0`) and share one body format:

| Field | Type |
|-------|------|
| Record count | Short, BE |
| Records | repeated: type byte + triad(s) |

Each record is: `0x00` = range (start triad + end triad, inclusive) or `0x01` = single (one triad). ACK acknowledges received datagram sequence numbers; NACK requests retransmission of missing ones.

## Session lifecycle summary

```
Client                                  Server
  |--- Unconnected Ping (0x01) ---------->|   (discovery)
  |<-- Unconnected Pong (0x1c) -----------|
  |--- Open Connection Request 1 (0x05) ->|
  |<-- Open Connection Reply 1 (0x06) ----|
  |--- Open Connection Request 2 (0x07) ->|
  |<-- Open Connection Reply 2 (0x08) ----|
  |--- Connection Request (0x09) -------->|     (reliable-ordered)
  |<-- Connection Request Accepted (0x10)-|
  |--- New Incoming Connection (0x13) --->|
  |--- Batch(Login) ---------------------->|   (game protocol from here on, see 05-login-and-session.md)
```

After this the connection carries only `0xFE`-framed game traffic (see [02-framing-batching.md](02-framing-batching.md)) and Connected Ping/Pong keepalives. Either side may end the session with Disconnection Notification (`0x15`).
