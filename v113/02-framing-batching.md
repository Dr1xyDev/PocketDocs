# 02 — Game packet framing and the Batch wrapper

Every MCPE game packet (the `0x01`–`0x5d` IDs listed in [04-packet-ids.md](04-packet-ids.md)) is transmitted in one common envelope. In protocol 113 both directions use the **Batch** wrapper exclusively.

## The 0xFE identifier

The first byte of the RakNet encapsulated payload (after RakNet's encapsulation header, see [01-transport-raknet.md](01-transport-raknet.md)) is the game-layer packet ID.

* For the vanilla client and server in 1.1.x this byte is **always `0xFE` (254, Batch)**.
* The reference implementation enforces this hard: any incoming encapsulated payload whose first byte is not `0xFE` throws `Unexpected non-FE packet` and the address is blocked (`RakLibInterface::MCPE_RAKNET_PACKET_ID = "\xfe"`).

So on the wire, one "game packet" is:

```
0xFE | zlib stream
```

## Batch format

| Field | Type | Notes |
|-------|------|-------|
| Packet ID | Byte | `0xFE` |
| Compressed payload | remaining bytes | zlib stream (RFC 1950, i.e. with zlib header/Adler32 trailer) |

After zlib inflation, the payload is a sequence of **sub-packets**:

| Per sub-packet | Type |
|----------------|------|
| Length | UnsignedVarInt (byte count) |
| Raw packet bytes | *length* bytes, starting with the game packet ID byte (`0x01`–`0x5d`) |

Sub-packets decode back-to-back until the inflated stream ends. There is no count field, no per-sub-packet compression and no padding.

Decompressed sub-packet bytes are then parsed as: `[packet ID byte][packet body …]`, with the ID byte consumed before the body fields.

### Limits enforced by the reference server

These are server-side anti-abuse limits, not protocol requirements, but useful when generating traffic for a 1.1.x server:

| Limit | Value |
|-------|-------|
| Compressed payload length | ≤ 524 288 bytes (512 KiB) |
| Decompressed payload length | ≤ 2 MiB |
| Sub-packets per batch | ≤ 250 |
| Single sub-packet length | ≤ 1 MiB |

## Compression details

* The zlib stream is standard `zlib` (raw deflate wrapped in a zlib header). Reference servers decode with PHP `zlib_decode`, which also accepts raw deflate; they compress with libdeflate at level 5 (`compressionLevel = -1/0` means **store**, used when a batch is smaller than the compression threshold).
* Chunks are the largest traffic source: one `FullChunkDataPacket` per column is batched alone, and the batch is stored uncompressed when the payload is under the server's threshold (`LevelProviderThreadHandler::prepareChunkPacket()`).

## Encryption interplay

Protocol 113 supports an AES-256-CFB8 encrypted channel (see [05-login-and-session.md](05-login-and-session.md#encryption-handshake)). When encryption is enabled:

```
0xFE | AES-256-CFB8( zlib stream )
```

The reference implementation strips the `0xFE` byte, decrypts the remainder, and re-attaches `0xFE`. Symmetrically on send: `0xFE` is prepended after encryption of the batch body. The `0xFE` byte itself is **never** encrypted.

Historical note on the checksum trailer: the reference `EncryptionContext::encrypt()` appends an 8-byte `sha256(counter_LE || payload || key)` prefix-truncated checksum after the payload (behaviour of 1.6+ clients), while `decrypt()` does **not** verify or strip a checksum — vanilla 1.1.x clients do not append one. When writing a server for 1.1.x, do not expect checksums on incoming batches; when acting as a client against a 1.1.x server, do not append them.

## Where each direction puts things

* **C2S**: the vanilla 1.1.x client wraps *every* game packet in a Batch — including `LoginPacket` itself. There are no "unbatched" client packets in protocol 113.
* **S2C**: the reference server wraps every game packet in a Batch as well (`putPacket()` re-routes any non-Batch packet through `batchPackets()`); `BatchPacket.canBeBatched()` is `false` so batches are never nested.

Worked example (hex, uncompressed for clarity):

```
FE                                  batch
  78 9C ...                         zlib header + deflated body
  inflated:
    08  (UnsignedVarInt)            length = 8 bytes
    09 01 06 44 72 31 78 79         sub-packet 0x09 TextPacket (chat, source "Dr1xy")
    03  (UnsignedVarInt)            length = 3 bytes
    21 04 00                        sub-packet 0x21 InteractPacket (action 4, target 0)
```
