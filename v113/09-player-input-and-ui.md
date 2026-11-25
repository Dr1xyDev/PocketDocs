# 09 — Player input, commands and UI packets

The client→server gameplay surface (the "client protocol" core) plus the server-side UI/feedback packets. Types: [03-data-types.md](03-data-types.md).

## World interaction

**PlayerActionPacket (`0x24`) — C2S**

| Field | Type | Notes |
|-------|------|-------|
| Entity ID | `VarInt` | self entity |
| Action | `VarInt` | table below |
| Block position | BlockPosition | relevant block (0,0,0 when unused) |
| Face | `VarInt` | face for block actions, 255 otherwise |

| Value | Action | Notes |
|------:|--------|-------|
| 0 | `START_BREAK` | begin mining |
| 1 | `ABORT_BREAK` | stop mining, no break |
| 2 | `STOP_BREAK` | revert-mining broadcast, same as `ABORT_BREAK` (actual block removal is the separate `0x15` RemoveBlock packet) |
| 5 | `RELEASE_ITEM` | release bow/food/projectile |
| 6 | `STOP_SLEEPING` | leave bed |
| 7 | `SPAWN_SAME_DIMENSION` | respawn ready (same dimension) |
| 8 | `JUMP` | jump |
| 9 | `START_SPRINT` | |
| 10 | `STOP_SPRINT` | |
| 11 | `START_SNEAK` | |
| 12 | `STOP_SNEAK` | |
| 13 | `SPAWN_OVERWORLD` | plain respawn, handled identically to `SPAWN_SAME_DIMENSION` (no dimension-change logic — only `SPAWN_NETHER` touches that path) |
| 14 | `SPAWN_NETHER` | respawn into nether (dimension change) |
| 15 | `START_GLIDE` | elytra glide start |
| 16 | `STOP_GLIDE` | elytra glide end |
| 17 | `BUILD_DENIED` | server denied placement (client echoes; unhandled by the reference — falls through to its `assert` default) |
| 18 | `CONTINUE_BREAK` | mining continues (the reference broadcasts a `LEVEL_EVENT` particle for it) |

**RemoveBlockPacket (`0x15`) — C2S**

| Field | Type |
|-------|------|
| Position | BlockPosition |

**BlockPickRequestPacket (`0x22`) — C2S** (not handled by the reference — no first-stage handler and no main-switch case, so it is decoded and dropped)

Middle-click pick block:

| Field | Type | Notes |
|-------|------|-------|
| Position | BlockPosition | |
| Unknown | `Byte` | "add to creative inventory" flag |

**PlayerInputPacket (`0x39`) — C2S**

Riding/boat input:

| Field | Type |
|-------|------|
| Motion X | `LFloat` (strafe input, -1..1) |
| Motion Y | `LFloat` (forward input, -1..1) |
| Jumping flag | `Bool` (conventional name; the reference reads both Bools as unknown) |
| Unknown flag | `Bool` |

The reference's handler for this packet is an empty stub — the input is decoded but never acted on.

**RiderJumpPacket (`0x14`) — C2S** — see [07](07-entities.md).

**AnimatePacket (`0x2c`) — Both**

| Field | Type | Notes |
|-------|------|-------|
| Action | `VarInt` | table below |
| Entity ID | `VarInt` | |
| Rowing time | `LFloat` | only for actions 128/129 |

Actions: `1` SWING_ARM, `3` STOP_SLEEP, `4` CRITICAL_HIT, `128` ROW_RIGHT, `129` ROW_LEFT.

On C2S the reference rejects a copy whose entity ID differs from the sender's own — the sender is kicked and their address blocked.

## Chat and commands

**TextPacket (`0x09`) — Both**

| Field | Type | Notes |
|-------|------|-------|
| Type | `Byte` | table below |
| Source | String | CHAT/WHISPER/POPUP only |
| Message | String | all types |
| Parameter count | `UnsignedVarInt` | TRANSLATION only |
| Parameters | count × String | TRANSLATION only |

| Value | Type | Notes |
|------:|------|-------|
| 0 | `RAW` | plain text |
| 1 | `CHAT` | player chat (source + message) |
| 2 | `TRANSLATION` | translation key + `%` parameters |
| 3 | `POPUP` | popup (source + message) |
| 4 | `TIP` | tip |
| 5 | `SYSTEM` | system message |
| 6 | `WHISPER` | private message |

On C2S the reference processes only type 1 (CHAT): a message over 350 bytes is not acted on and the sender's address is blocked for 10 seconds; lines starting with `./` are treated as `/` (era fast-command hack) and dispatched as commands.

**CommandStepPacket (`0x4f`) — C2S**

The 1.1 command packet — the client sends the raw command line it wants executed:

| Field | Type | Notes |
|-------|------|-------|
| Command | String | e.g. `time set night` (without the leading slash) |
| Overload | String | `""` |
| Unknown step | `UnsignedVarInt` | |
| Current step | `UnsignedVarInt` | |
| Done | `Bool` | |
| Client ID | `UnsignedVarLong` | |
| Input JSON | String | JSON argument map |
| Output JSON | String | `""` from clients |

**AvailableCommandsPacket (`0x4e`) — S2C**

| Field | Type |
|-------|------|
| Commands JSON | String |
| Unknown | String — always `""`; never set by the reference implementation |

In protocol 113 the command list is still a **JSON blob** (the binary command enumeration came in later versions): an object keyed by command name, each with a `versions` array containing `aliases`, `description`, `overloads` and `permission`. The full schema is documented in [11-available-commands-json.md](11-available-commands-json.md).

**CommandBlockUpdatePacket (`0x50`) — C2S**

| Field | Type | Notes |
|-------|------|-------|
| Is block | `Bool` | false = minecart command block |
| Block position | BlockPosition | when is block |
| Mode | `UnsignedVarInt` | conventional labels: 0 sequential/impulse, 1 auto/repeat, 2 loop/chain — values carry no meaning to the reference server |
| Redstone mode | `Bool` | conventional: false = always active |
| Conditional | `Bool` | when is block |
| Minecart entity ID | `VarInt` | when not block |
| Command | String | |
| Last output | String | |
| Name | String | |
| Track output | `Bool` | |

## Adventure settings

**AdventureSettingsPacket (`0x37`) — Both**

| Field | Type |
|-------|------|
| Flags | `UnsignedVarInt` |
| User permission | `UnsignedVarInt` (0 normal, 1 operator, 2 host, 3 automation, 4 admin) |

Flag bits:

| Bit | Flag |
|----:|------|
| 0 | world immutable |
| 1 | no PvP |
| 2 | no PvM |
| 3 | no MvP |
| 5 | auto jump |
| 6 | allow flight |
| 7 | no clip |
| 8 | world builder |
| 9 | is flying |
| 10 | muted |

(Bit 4 is unused.) Clients send this to announce their own state; the reference server routes announced **flying (bit 9)** and **no-clip (bit 7)** without permission through the same cheat-handling path, which fires the flight-toggle event (server state wins; the settings packet is re-sent only when that event is cancelled — automatically when flying is announced without allow-flight, or by a plugin). A dedicated no-clip kick branch exists in the source but is unreachable dead code — in practice nothing beyond the revert is enforced. Everything else is advisory.

## View radius

**RequestChunkRadiusPacket (`0x45`) — C2S**

| Field | Type |
|-------|------|
| Radius | `VarInt` (chunks) |

The reference acts only on radii 4–80: anything else is not acted on and the sender's address is blocked; Android/iOS clients (device OS 1/2) announcing a radius above 22 are additionally address-blocked (the radius is still applied).

**ChunkRadiusUpdatedPacket (`0x46`) — S2C** — the server's accepted radius.

| Field | Type |
|-------|------|
| Radius | `VarInt` |

## Maps

**MapInfoRequestPacket (`0x44`) — C2S**

| Field | Type |
|-------|------|
| Map ID | `VarInt` |

**ClientboundMapItemDataPacket (`0x43`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Map ID | `VarInt` | |
| Update type | `UnsignedVarInt` | bitmask: `0x02` texture, `0x04` decorations, `0x08` entity |
| Entity count + entity IDs | `UnsignedVarInt` + count × `VarInt` | only when `0x08` |
| Scale | `Byte` | when `0x02` or `0x04` |
| Decoration count | `UnsignedVarInt` | only when `0x04` |
| Decorations | count × {`VarInt` packed (rotation low nibble, image high nibble), `Byte` x offset, `Byte` y offset, String label, `LInt` color} | the reference reads the color as ARGB but writes it as ABGR on encode (red/blue swapped on round-trip — an era inconsistency); vanilla side unconfirmable |
| Width, Height, X offset, Y offset | 4 × `VarInt` | only when `0x02` |
| Colors | width × height × `UnsignedVarInt` | row-major ABGR int per pixel |

## Events

**EventPacket (`0x41`) — S2C** (informational eventing; the client does not act on it)

| Field | Type |
|-------|------|
| Entity runtime ID | `UnsignedVarLong` (likely) — the reference's accessor for this field (`putEntityRuntimeId`) is missing from its codebase, so the width is unverifiable; treat as a caveat |
| Event data | `VarInt` |
| Type | `Byte` (0 achievement awarded, 1 entity interact, 2 portal built, 3 portal used, 4 mob killed, 5 cauldron used, 6 player death, 7 boss killed, 8 agent command, 9 agent created) |

**SimpleEventPacket (`0x40`)** — defined in the ID registry; no payload documentation is available for protocol 113.

## Titles and sounds

**SetTitlePacket (`0x59`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Type | `VarInt` | 0 clear, 1 reset, 2 title, 3 subtitle, 4 action bar, 5 times |
| Text | String | |
| Fade in | `VarInt` | ticks |
| Duration | `VarInt` | ticks |
| Fade out | `VarInt` | ticks |

**PlaySoundPacket (`0x57`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Sound name | String | e.g. `random.click` |
| Position × 8 | BlockPosition | write `(int)(worldX * 8)`; decode divides by 8 |
| Volume | `LFloat` | |
| Pitch | `LFloat` | |

**StopSoundPacket (`0x58`) — S2C**

| Field | Type |
|-------|------|
| Sound name | String |
| Stop all | `Bool` |

## Trading and equipment screens

**UpdateTradePacket (`0x51`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Window ID | `Byte` | |
| Window type | `Byte` | vanilla hardcodes `15` (`TRADING`); the reference never assigns it either, so the class default `15` would go out — not `0` |
| Unknown 1 | `VarInt` | no documented meaning in 1.1; the reference's send path assigns `0` |
| Unknown 2 | `VarInt` | no documented meaning in 1.1; the reference's send path assigns `0` |
| Is willing | `Bool` | villager willingness |
| Trader entity ID | `VarInt` | |
| Player entity ID | `VarInt` | |
| Display name | String | trader name |
| Offers | raw bytes | little-endian **network-flavor** NBT compound with `Recipes` list (see [03](03-data-types.md#nbt)) |

The reference never actually sends this packet — its only `VillagerTradeInventory` instantiation is commented out in `entity/passive/Villager.php` — so the send-path values above are hypothetical.

**UpdateEquipPacket (`0x52`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Window ID | `Byte` | |
| Window type | `Byte` | |
| Unknown | `VarInt` | vanilla sends 0 |
| Entity unique ID | `VarInt` | |
| Equipment NBT | raw bytes | little-endian **network-flavor** NBT compound (rest of packet) |
