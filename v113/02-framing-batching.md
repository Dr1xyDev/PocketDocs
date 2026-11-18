# 02 — Game packet framing and the Batch wrapper

Every MCPE game packet (the `0x01`–`0x5d` IDs listed in [04-packet-ids.md](04-packet-ids.md)) is transmitted in one common envelope. In protocol 113 both directions use the **Batch** wrapper exclusively.

## The 0xFE identifier

The first byte of the RakNet encapsulated payload (after RakNet's encapsulation header, see [01-transport-raknet.md](01-transport-raknet.md)) is the game-layer packet ID.

* In protocol 113 this byte is **always `0xFE` (254, Batch)**, from both the client and the server. The vanilla client wraps *every* game packet it sends — including `LoginPacket` — inside a Batch, so a 1.1.x-compatible endpoint can treat the game stream as exclusively `0xFE`-framed.

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

## Compression details

* The zlib stream is standard `zlib` (raw deflate wrapped in a zlib header with an Adler-32 trailer). Decoders must accept any valid zlib stream; in particular some implementations **store** small batches uncompressed (compression level 0) to save CPU.
* Chunks are the largest traffic source: one `FullChunkDataPacket` per column is batched on its own.

## Encryption interplay

Protocol 113 supports an AES-256-CFB8 encrypted channel (see [05-login-and-session.md](05-login-and-session.md#encryption-handshake)). When encryption is enabled:

```
0xFE | AES-256-CFB8( zlib stream )
```

Both sides encrypt only the batch body: the `0xFE` byte is prepended *after* encryption when sending, and stripped *before* decryption when receiving. The `0xFE` byte itself is **never** encrypted.

Checksum trailer: 1.1-era server implementations append an **8-byte checksum** to every encrypted payload they send — the first 8 bytes of `sha256(counter_LE ‖ payload ‖ key)` (counter = a little-endian 64-bit integer, incremented per encrypted payload) — and do **not** verify or strip a checksum on receive. Vanilla 1.1.x clients send encrypted batches **without** a checksum; a client-compatible endpoint should therefore send without one and be prepared for a trailing 8-byte checksum on payloads it receives.

## Where each direction puts things

* **C2S**: the vanilla 1.1.x client wraps *every* game packet in a Batch — including `LoginPacket` itself. There are no "unbatched" client packets in protocol 113.
* **S2C**: servers likewise deliver every game packet inside a Batch. Batches are never nested (a Batch is never a sub-packet of another Batch).

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
