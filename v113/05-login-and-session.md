# 05 — Login, session lifecycle and resource packs

This file covers everything from the established UDP session (end of [01-transport-raknet.md](01-transport-raknet.md)) to the player seeing the world, plus disconnection and transfer. All game packets here travel inside `0xFE` batches (see [02-framing-batching.md](02-framing-batching.md)).

## Session flow overview

The canonical protocol-113 session sequence:

```
Client                                    Server
  |--- 0x01 Login ------------------------>|
  |<-- 0x02 PlayStatus(LOGIN_SUCCESS) -----|  (sent immediately, before chain verification)
  |<-- 0x03 ServerToClientHandshake -------|  (only if encryption enabled)
  |--- 0x04 ClientToServerHandshake ------>|
  |<-- 0x02 PlayStatus(LOGIN_SUCCESS) -----|  (sent again once the login is accepted)
  |<-- 0x06 ResourcePacksInfo ------------ |
  |--- 0x08 ResourcePackClientResponse --->|  (SEND_PACKS if packs need downloading)
  |<-- 0x53 ResourcePackDataInfo × N ------|
  |--- 0x55 ResourcePackChunkRequest × M ->|
  |<-- 0x54 ResourcePackChunkData × M -----|
  |--- 0x08 ResourcePackClientResponse --->|  (HAVE_ALL_PACKS)
  |<-- 0x07 ResourcePackStack -------------|
  |--- 0x08 ResourcePackClientResponse --->|  (COMPLETED)
  |<-- 0x0b StartGame ---------------------|
  |<-- 0x0a SetTime / 0x1e UpdateAttributes / 0x3f PlayerList  (right after StartGame)
  |--- 0x45 RequestChunkRadius ----------->|
  |<-- 0x46 ChunkRadiusUpdated ------------|
  |<-- 0x3a FullChunkData × N -------------|  (batched)
  |<-- 0x02 PlayStatus(PLAYER_SPAWN) ------|
  |<-- entity/item spawn packets ----------|  (with and after the spawn chunks)
  |--- 0x13 MovePlayer / 0x24 PlayerAction |  (normal gameplay begins)
```

The client may send `0x45 RequestChunkRadius` at any point after StartGame; the server answers with `0x46 ChunkRadiusUpdated`.

## LoginPacket (`0x01`) — C2S

| Offset | Field | Type | Notes |
|-------:|-------|------|-------|
| 0 | Protocol version | `Int` (**big-endian**) | `113` for 1.1.x releases — the reference also accepts 110-112; only protocols outside that list get `0x02 PlayStatus(LOGIN_FAILED_CLIENT)` if lower than 113, `LOGIN_FAILED_SERVER` otherwise, then a close with **no DisconnectPacket** (notify off) |
| 4 | Game edition | `Byte` | `0` = Pocket edition |
| 5 | Connection request | String (`UnsignedVarInt` length + bytes) | a self-contained sub-buffer, decoded below |

The *connection request* buffer contains the auth payload:

| Offset | Field | Type | Notes |
|-------:|-------|------|-------|
| 0 | Chain JSON length | `LInt` (little-endian!) | |
| 4 | Chain JSON | bytes | JSON object `{"chain": ["<JWT>", "<JWT>", …]}` |
| +0 | Client data JWT length | `LInt` | |
| +4 | Client data JWT | bytes | single JWT **signed ES384 with the key carried at the end of the chain** |

### The auth chain

Each string in `chain` is a JWT (`base64url(header).base64url(payload).base64url(signature)`), signed **ES384** (ECDSA P-384, 96-byte raw signature) and chained: each link's header contains `x5u` (the public key that must match the previous payload's `identityPublicKey`); the first link is self-signed. A chain is treated as **Xbox-authenticated when any link is signed by Mojang's root public key**. Servers verify every signature (chain *and* client-data JWT) and reject invalid ones; "offline mode" only relaxes the requirement that the chain be Mojang-signed — it does not skip signature verification.

Claims present in the chain (the link containing `extraData`):

| Claim | Meaning |
|-------|---------|
| `extraData.displayName` | Xbox Live gamertag → used as the player name |
| `extraData.identity` | Xbox identity UUID (string) |
| `extraData.XUID` | Xbox User ID string |
| `extraData.titleId` | title ID (optional) |
| `identityPublicKey` | public key for the *next* chain link (base64 DER) |
| `exp`, `nbf` | standard JWT expiry/not-before timestamps |

### The client-data JWT payload

Claims the client puts in the client-data JWT (all optional in practice):

| Claim | Type | Meaning |
|-------|------|---------|
| `ClientRandomId` | Long | random per-launch client ID |
| `ServerAddress` | String | `host:port` the client is connecting to |
| `SkinId` | String | skin identifier string |
| `SkinData` | String | base64 of raw skin image bytes (64×64 RGBA = 16384 bytes, or 64×32 = 8192 for legacy) |
| `SkinImageWidth` / `SkinImageHeight` | Int | skin dimensions (present in 1.1 builds) — **not read by the reference implementation at all** |
| `LanguageCode` | String | e.g. `en_US` |
| `DeviceModel` | String | device model string |
| `DeviceOS` | Int | platform — see table below |
| `CurrentInputMode` | Int | input mode: 1 mouse+keyboard, 2 touch, 3 gamepad, 4 motion controller |
| `DefaultInputMode` | Int | same enumeration |
| `UIProfile` | Int | 0 = classic UI, 1 = pocket UI |
| `GuiScale` | Int | selected GUI scale |
| `TenantId` | Int | education/tenant ID (0 for consumer clients) |
| `GameVersion` | String | e.g. `1.1.7` |
| `ADRole` | Int | education-edition role |

**DeviceOS values** (as encoded by 1.1.x-era clients): `-1` unknown, `1` Android, `2` iOS, `3` macOS/OSX, `4` Amazon FireOS, `5` Gear VR, `6` HoloLens, `7` Windows 10 (UWP), `8` Windows 32-bit, `9` Dedicated, `10` tvOS, `11` PlayStation, `12` Nintendo, `13` Xbox, `14` Windows Phone. (Values ≥ 9 are not produced by 1.1.x clients; they are listed for completeness.)

> Note on claim interpretation: the reference implementation parses 14 client-data claims but only *acts* on `ClientRandomId` (client-ID ban checks), `SkinId`/`SkinData` (skin handling), `DeviceOS` and `DeviceModel` (platform gating). `ServerAddress`, `LanguageCode`, the input-mode values and `GuiScale` are parsed and stored but never consumed by it. `SkinImageWidth`/`SkinImageHeight` are never read at all, and `UIProfile`, `TenantId`, `GameVersion`, `ADRole` are parsed into the packet object only. The chain-token `XUID`/`titleId` keys are likewise never read by the reference (client-side conventions).

There are **no further fields after the client-data JWT** in protocol 113 — username, UUID and skin all come from the JWTs.

## Encryption handshake

The 1.1-era scheme:

1. Server generates an **ECDH key pair on secp384r1** — created once per server process and reused for every session (not per-connection ephemeral) — plus a random 16-byte salt/token.
2. Server sends **`0x03 ServerToClientHandshake`** — sent *unencrypted*, immediately before enabling its cipher.
3. Client derives: shared secret = ECDH(client's `identityPublicKey` private counterpart, server public key); encryption key = **SHA-256(salt ‖ shared-secret)** (salt first).
4. Client replies **`0x04 ClientToServerHandshake`** and enables its cipher; upon receiving `0x04` — only if the server previously sent `0x03` and is awaiting the handshake (an unsolicited `0x04` is silently ignored) — the server continues the login sequence.

Both `0x03` and `0x04` are sent/received *inside* `0xFE` batches; the first encrypted bytes are the *next* batch's payload (after the `0xFE` byte).

**ServerToClientHandshakePacket (`0x03`) — S2C**

| Field | Type |
|-------|------|
| Public key | String — base64 DER of the server's ECDH public key (public half of the key pair created once per server process, see step 1 above) |
| Server token | String — the **raw 16-byte salt** (varint length + raw bytes, *not* base64) used in key derivation |

**ClientToServerHandshakePacket (`0x04`) — C2S**

No payload.

**Cipher**: AES-256-CFB8, IV = first 16 bytes of the encryption key. Vanilla 1.1.x clients do **not** append a checksum after each encrypted payload; see the note in [02-framing-batching.md](02-framing-batching.md).

Encryption is optional — a server may skip the handshake entirely and continue in plaintext.

## PlayStatusPacket (`0x02`) — S2C

| Field | Type |
|-------|------|
| Status | `Int` (**big-endian**) |

| Status | Meaning |
|-------:|---------|
| 0 | `LOGIN_SUCCESS` — client proceeds through resource packs / waits for StartGame |
| 1 | `LOGIN_FAILED_CLIENT` — client outdated ("outdated client" screen) |
| 2 | `LOGIN_FAILED_SERVER` — server outdated ("outdated server" screen) |
| 3 | `PLAYER_SPAWN` — sent once the world is ready; client transitions to in-game |
| 4 | `LOGIN_FAILED_INVALID_TENANT` — education edition mismatch |
| 5 | `LOGIN_FAILED_VANILLA_EDU` — vanilla client joining an Edu server |
| 6 | `LOGIN_FAILED_EDU_VANILLA` — Edu client joining a vanilla server |

## DisconnectPacket (`0x05`) — S2C

| Field | Type | Present when |
|-------|------|--------------|
| Hide disconnection screen | `Bool` | always |
| Message | String | only if not hidden |

`hideDisconnectionScreen = true` closes the connection without showing a reason screen.

## Resource pack exchange

### ResourcePacksInfoPacket (`0x06`) — S2C

| Field | Type | Notes |
|-------|------|-------|
| Must accept | `Bool` | forces the client to accept or disconnect; reference sends `resourcePacksRequired()` (config `force_resources`, default `false`) |
| Behavior pack entry count | `LShort` | |
| Behavior pack entries | repeated below | |
| Resource pack entry count | `LShort` | |
| Resource pack entries | repeated below | |

Each entry:

| Field | Type |
|-------|------|
| Pack ID (UUID string) | String |
| Pack version | String (usually `x.y.z`) |
| Pack size | `LLong` (bytes) |
| Encryption key | String (unused in 1.1, sent empty) |

### ResourcePackClientResponsePacket (`0x08`) — C2S

| Field | Type |
|-------|------|
| Status | `Byte` |
| Pack count | `LShort` |
| Pack IDs | count × String |

| Status | Meaning |
|-------:|---------|
| 1 | `REFUSED` — client declines packs; server kicks |
| 2 | `SEND_PACKS` — client wants to download the listed packs |
| 3 | `HAVE_ALL_PACKS` — client has everything; server answers with `0x07` |
| 4 | `COMPLETED` — client accepted the stack; server proceeds to StartGame |

### ResourcePackStackPacket (`0x07`) — S2C

| Field | Type | Notes |
|-------|------|-------|
| Must accept | `Bool` | always `false` — hardcoded at the reference's send site regardless of config; forcing `true` makes the client drop its own local packs |
| Behavior stack count | `UnsignedVarInt` | |
| Behavior stack entries | String pack ID, String version — per entry | |
| Resource stack count | `UnsignedVarInt` | |
| Resource stack entries | String pack ID, String version — per entry | |

### Download sub-protocol (`0x53` / `0x55` / `0x54`)

**ResourcePackDataInfoPacket (`0x53`) — S2C**

| Field | Type |
|-------|------|
| Pack ID | String |
| Max chunk size | `LInt` (128 KiB is the typical chunk size) |
| Chunk count | `LInt` (`ceil(size / maxChunkSize)`) |
| Compressed pack size | `LLong` |
| SHA-256 hash | String (32 raw bytes) |

**ResourcePackChunkRequestPacket (`0x55`) — C2S**

| Field | Type |
|-------|------|
| Pack ID | String |
| Chunk index | `LInt` (0-based) |

**ResourcePackChunkDataPacket (`0x54`) — S2C**

| Field | Type |
|-------|------|
| Pack ID | String |
| Chunk index | `LInt` |
| Progress | `LLong` (byte offset of this chunk in the pack) |
| Data length | `LInt` |
| Data | *length* bytes |

## TransferPacket (`0x56`) — S2C

Tells the client to disconnect and connect to another server:

| Field | Type |
|-------|------|
| Address | String |
| Port | `LShort` (default 19132) |

## Post-login gameplay handshake notes

* After `0x0b StartGame` the client requests a view radius (`0x45`) and starts accepting `0x3a FullChunkData` batches. Servers must send `0x02 PlayStatus(PLAYER_SPAWN)` only once enough chunks are out; the client freezes movement until then.
* If encryption was negotiated, the entire post-login flow still runs over the encrypted batches — nothing changes at the game-packet layer.
* Servers verify the login chain even in offline mode; offline mode only relaxes the Mojang-signed requirement. Identity data (name, UUID, skin, device info) always comes from the JWT claims (`ClientRandomId`, `SkinData`, `DeviceOS`…).
