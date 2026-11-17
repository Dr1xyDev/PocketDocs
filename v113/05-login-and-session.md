# 05 — Login, session lifecycle and resource packs

This file covers everything from the established UDP session (end of [01-transport-raknet.md](01-transport-raknet.md)) to the player seeing the world, plus disconnection and transfer. All game packets here travel inside `0xFE` batches (see [02-framing-batching.md](02-framing-batching.md)).

## Session flow overview

The reference server's sequence (`Player.php`, `processLogin()` / `completeLoginSequence()` / `doFirstSpawn()`):

```
Client                                    Server
  |--- 0x01 Login ------------------------>|  (Xbox chain verified; optional encryption)
  |<-- 0x03 ServerToClientHandshake -------|  (only if encryption enabled)
  |--- 0x04 ClientToServerHandshake ------>|
  |<-- 0x02 PlayStatus(LOGIN_SUCCESS) -----|
  |<-- 0x06 ResourcePacksInfo ------------ |
  |--- 0x08 ResourcePackClientResponse --->|  (SEND_PACKS if packs need downloading)
  |<-- 0x53 ResourcePackDataInfo × N ------|
  |--- 0x55 ResourcePackChunkRequest × M ->|
  |<-- 0x54 ResourcePackChunkData × M -----|
  |--- 0x08 ResourcePackClientResponse --->|  (HAVE_ALL_PACKS)
  |<-- 0x07 ResourcePackStack -------------|
  |--- 0x08 ResourcePackClientResponse --->|  (COMPLETED)
  |<-- 0x0b StartGame ---------------------|
  |--- 0x45 RequestChunkRadius ----------->|
  |<-- 0x46 ChunkRadiusUpdated ------------|
  |<-- 0x3a FullChunkData × N -------------|  (batched)
  |<-- spawn packets (SetTime, UpdateAttributes, PlayerList, entities…) |
  |<-- 0x02 PlayStatus(PLAYER_SPAWN) ------|
  |--- 0x13 MovePlayer / 0x24 PlayerAction |  (normal gameplay begins)
```

The client may send `0x45 RequestChunkRadius` at any point after StartGame; the server answers with `0x46 ChunkRadiusUpdated`.

## LoginPacket (`0x01`) — C2S

| Offset | Field | Type | Notes |
|-------:|-------|------|-------|
| 0 | Protocol version | `Int` (**big-endian**) | `113` for 1.1.x releases; servers answer `0x02 PlayStatus(LOGIN_FAILED_CLIENT)` if lower than theirs, `LOGIN_FAILED_SERVER` if higher |
| 4 | Game edition | `Byte` | `0` = Pocket edition |
| 5 | Connection request | String (`UnsignedVarInt` length + bytes) | a self-contained sub-buffer, decoded below |

The *connection request* buffer contains the auth payload:

| Offset | Field | Type | Notes |
|-------:|-------|------|-------|
| 0 | Chain JSON length | `LInt` (little-endian!) | |
| 4 | Chain JSON | bytes | JSON object `{"chain": ["<JWT>", "<JWT>", …]}` |
| +0 | Client data JWT length | `LInt` | |
| +4 | Client data JWT | bytes | single JWT, **unsigned** (no signature) |

### The auth chain

Each string in `chain` is a JWT (`base64url(header).base64url(payload).base64url(signature)`), signed **ES384** (ECDSA P-384, 96-byte raw signature) and chained: each link's header contains `x5u` (the public key that must match the previous payload's `identityPublicKey`); the last link is signed by Mojang's public key. The reference server verifies this chain in `VerifyLoginTask` and marks the player Xbox-authenticated when it holds — servers in offline mode skip the check.

Claims read by the reference from the chain (link containing `extraData`):

| Claim | Meaning |
|-------|---------|
| `extraData.displayName` | Xbox Live gamertag → used as the player name |
| `extraData.identity` | Xbox identity UUID (string) |
| `extraData.XUID` | Xbox User ID string |
| `extraData.titleId` | title ID (not read by the reference) |
| `identityPublicKey` | public key for the *next* chain link (base64 DER) |
| `exp`, `nbf` | standard JWT expiry/not-before timestamps |

### The client-data JWT payload

Claims read by the reference implementation (all optional in practice):

| Claim | Type | Meaning |
|-------|------|---------|
| `ClientRandomId` | Long | random client ID (session identifier shown in server logs) |
| `ServerAddress` | String | `host:port` the client is connecting to |
| `SkinId` | String | skin identifier string |
| `SkinData` | String | base64 of raw skin image bytes (64×64 RGBA = 16384 bytes, or 64×32 = 8192 for legacy) |
| `SkinImageWidth` / `SkinImageHeight` | Int | skin dimensions (present in 1.1 builds) |
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

**DeviceOS values** (from the reference `types/OS.php`): `-1` unknown, `1` Android, `2` iOS, `3` macOS/OSX, `4` Amazon FireOS, `5` Gear VR, `6` HoloLens, `7` Windows 10 (UWP), `8` Windows 32-bit, `9` Dedicated, `10` tvOS, `11` PlayStation, `12` Nintendo, `13` Xbox, `14` Windows Phone. (Values ≥ 9 are not produced by 1.1.x clients; they are listed for completeness.)

There are **no further fields after the client-data JWT** in protocol 113 — username, UUID and skin all come from the JWTs.

## Encryption handshake

`EncryptionContext` in the reference implements the 1.1-era scheme:

1. Server generates an ephemeral **ECDH key pair on secp384r1** and a random 16-byte salt/token.
2. Server sends **`0x03 ServerToClientHandshake`** — sent *unencrypted*, immediately before enabling its cipher.
3. Client derives: shared secret = ECDH(client's `identityPublicKey` private counterpart, server public key); encryption key = **SHA-256(salt ‖ shared-secret)** (salt first, per `EncryptionUtils::generateKey()`).
4. Client replies **`0x04 ClientToServerHandshake`** and enables its cipher; the server, upon receiving `0x04`, continues login processing (`processLogin()`).

Both `0x03` and `0x04` are sent/received *inside* `0xFE` batches; the first encrypted bytes are the *next* batch's payload (after the `0xFE` byte).

**ServerToClientHandshakePacket (`0x03`) — S2C**

| Field | Type |
|-------|------|
| Public key | String — base64 DER of the server's ephemeral public key |
| Server token | String — base64 of the 16-byte salt used in key derivation |

**ClientToServerHandshakePacket (`0x04`) — C2S**

No payload.

**Cipher**: AES-256-CFB8, IV = first 16 bytes of the encryption key (reference `EncryptionContext::cfb8()`). Vanilla 1.1.x clients do **not** append a checksum after each encrypted payload; see the note in [02-framing-batching.md](02-framing-batching.md).

Encryption is optional: reference servers fall back to plain login when encryption is disabled, and offline-mode servers may skip Xbox verification entirely.

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
| Must accept | `Bool` | forces the client to accept or disconnect |
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
| Must accept | `Bool` | servers typically send `false` (forcing true makes the client drop its own local packs) |
| Behavior stack count | `UnsignedVarInt` | |
| Behavior stack entries | String pack ID, String version — per entry | |
| Resource stack count | `UnsignedVarInt` | |
| Resource stack entries | String pack ID, String version — per entry | |

### Download sub-protocol (`0x53` / `0x55` / `0x54`)

**ResourcePackDataInfoPacket (`0x53`) — S2C**

| Field | Type |
|-------|------|
| Pack ID | String |
| Max chunk size | `LInt` (reference uses 128 KiB per chunk) |
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
* On modernized reference servers the login chain may be skipped entirely for offline play, in which case the client-data JWT claims become the only identity source (`ClientRandomId`, `SkinData`, `DeviceOS`…).
