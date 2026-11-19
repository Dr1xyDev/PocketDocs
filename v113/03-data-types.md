# 03 — Wire data types

All packet bodies are streams of the primitives below. Names follow the reference implementation (`BinaryStream` / `Binary`). **Unless stated otherwise, everything is little-endian.** Note that *RakNet-layer* messages (see [01-transport-raknet.md](01-transport-raknet.md)) use big-endian fields, but the game layer does not.

## Integer primitives

| Type | Size | Encoding | Signed |
|------|-----:|----------|:------:|
| `Byte` | 1 | raw byte | unsigned (0–255) |
| `Bool` | 1 | `0x00` false, anything else true | – |
| `Short` | 2 | big-endian | **unsigned** in the reference (`getShort()`); the signed variant is `SignedShort` |
| `LShort` | 2 | little-endian | **signed by default** in the reference (`getLShort()`; `getLShort(false)` for unsigned) |
| `Triad` / `LTriad` | 3 | `Triad` big-endian, `LTriad` little-endian | unsigned — RakNet uses the LE one |
| `Int` | 4 | **big-endian** | signed |
| `LInt` | 4 | little-endian | signed |
| `LLong` | 8 | little-endian | signed |
| `LFloat` | 4 | little-endian IEEE-754 single precision | signed |
| `VarInt` | 1–5 | LEB128 + zigzag | signed |
| `UnsignedVarInt` | 1–5 | LEB128 | unsigned |
| `VarLong` | 1–10 | LEB128 + zigzag | signed |
| `UnsignedVarLong` | 1–10 | LEB128 | unsigned |

## VarInt / VarLong (LEB128 + zigzag)

Bytes are LEB128: 7 data bits per byte, MSB set on every byte except the last. `UnsignedVarInt` stops after 5 bytes; `VarLong` variants after 10.

* `UnsignedVarInt` = the raw LEB128 value.
* `VarInt` = the raw LEB128 value **zigzag-decoded**: `decoded = (raw >> 1) ^ -(raw & 1)`. Encode: `raw = (n << 1) ^ (n >> 31)`.
* `VarLong` uses the same zigzag on 64 bits; `UnsignedVarLong` is plain.

Example: value `2` → VarInt raw `0x04`; value `-1` → raw `0x01`; value `-2` → raw `0x03`; value `300` → VarInt raw `0xD8 0x04` (zigzag 300 = 600; `0xAC 0x02` is the **UnsignedVarInt** encoding of 300).

Entity IDs ("EntityUniqueID"/"EntityRuntimeID") changed width across MCPE history; in protocol 113 **all** entity IDs are `VarInt` — with one caveated exception: the EventPacket (`0x41`) runtime-ID field, whose accessor is missing from the reference source so its width is unverifiable (see [09](09-player-input-and-ui.md#events)).

## String

| Field | Type |
|-------|------|
| Length | `UnsignedVarInt` (bytes) |
| Data | UTF-8 bytes, no terminator |

## UUID

16 bytes, transmitted as **two little-endian 64-bit halves, most significant half first**. Each half is written as two `LInt`s. With the canonical UUID string split into four 32-bit segments `seg0-seg1-seg2-seg3`, the wire order is:

```
LInt seg1, LInt seg0, LInt seg3, LInt seg2
```

i.e. bytes = `LE64( (seg0 << 32) | seg1 ) ++ LE64( (seg2 << 32) | seg3 )`.

## Float vectors and positions

**Vector3f** — 3 × `LFloat` (x, y, z). Implementations of the era round to 4 decimal places when reading; treat as ordinary floats on the wire.

**BlockPosition (getBlockCoords / putBlockCoords)**:

| Field | Type | Notes |
|-------|------|-------|
| X | `VarInt` (zigzag) | |
| Y | `UnsignedVarInt` | may be read as signed by newer versions; 1.1 uses unsigned |
| Z | `VarInt` (zigzag) | |

**SignedBlockPosition** — 3 × `VarInt` (x, y, z) — used by `ExplodePacket` records.

**ByteRotation** — one `Byte`; degrees = `value × (360 / 256)`. Write with **truncation**: `(int)(degrees × 256 / 360)` (the reference's `putByteRotation` casts to int — e.g. 50° → 35, not 36). Used for pitch/yaw/head-yaw in `MoveEntityPacket`.

## Slot (ItemStack) — item encoding

The single most reused structure in the protocol. All items in inventories, entities, drops, recipes, and entity metadata type 5 use it:

| Field | Type | Notes |
|-------|------|-------|
| Item ID | `VarInt` (zigzag) | `0` = empty item; **an `id ≤ 0` value ends the slot** (a negative zigzag value terminates too — no further fields) |
| Aux value | `VarInt` (zigzag) | packed: `(damage & 0x7FFF) << 8 \| (count & 0xFF)` — low byte is the stack **count**, upper 15 bits the item **damage/meta**; `0x7FFF` damage means "any damage" |
| NBT length | `LShort` | byte length of the NBT blob |
| NBT data | *nbtLen* bytes | little-endian NBT, **plain** flavor (see [NBT](#nbt)), only if `nbtLen > 0` |
| CanPlaceOn count | `VarInt` | number of block identifiers following |
| CanPlaceOn entries | count × String | |
| CanDestroy count | `VarInt` | number of block identifiers following |
| CanDestroy entries | count × String | |

Example: a stack of 64 unenchanted oak planks (ID 5) → `0A 80 01 00 00 00 00` (ID 5 zigzag → `0x0A`; auxValue = 64 zigzag → `0x80 0x01`; nbtLen = 0 → `00 00`; placeOn = 0 → `00`; destroy = 0 → `00`). An empty slot is exactly one byte `00`.

## Entity metadata

Serialized by `getEntityMetadata()` / `putEntityMetadata()` on the base `DataPacket`:

| Field | Type |
|-------|------|
| Entry count | `UnsignedVarInt` |
| Entries | repeated per table below |

Each entry:

| Field | Type |
|-------|------|
| Key | `UnsignedVarInt` |
| Type | `UnsignedVarInt` |
| Value | depends on type |

| Type ID | Type ID name | Value encoding |
|--------:|--------------|----------------|
| 0 | `DATA_TYPE_BYTE` | `Byte` |
| 1 | `DATA_TYPE_SHORT` | `LShort` (signed!) |
| 2 | `DATA_TYPE_INT` | `VarInt` |
| 3 | `DATA_TYPE_FLOAT` | `LFloat` |
| 4 | `DATA_TYPE_STRING` | String |
| 5 | `DATA_TYPE_SLOT` | Slot |
| 6 | `DATA_TYPE_POS` | `VarInt` x, `VarInt` y (signed), `VarInt` z |
| 7 | `DATA_TYPE_LONG` | `VarLong` |
| 8 | `DATA_TYPE_VECTOR3F` | 3 × `LFloat` |

The keys (property IDs) and the flag bitmask semantics (key 0 = `DATA_FLAGS`, key 91 = `DATA_FLAGS2`) are documented in [07-entities.md](07-entities.md#entity-metadata-keys-and-flags).

## Game rules

Used by `StartGamePacket` and `GameRulesChangedPacket`:

| Field | Type |
|-------|------|
| Rule count | `UnsignedVarInt` |
| Per rule: name | String |
| Per rule: type | `UnsignedVarInt` — `1` = Bool, `2` = UnsignedVarInt, `3` = LFloat |
| Per rule: value | Bool / UnsignedVarInt / LFloat by type |

Rules used by 1.1 servers include `naturalregeneration` (Bool) and `experimentalgameplay` (Bool).

## NBT

All NBT on the 113 wire is **little-endian**, in two flavors:

* **Network flavor** — block entities (`BlockEntityDataPacket`), tile spawn compounds (appended back-to-back to chunk data by `FullChunkDataPacket`), trade offers (`UpdateTradePacket`) and equip contents (`UpdateEquipPacket`, which sends the remainder of the packet as one compound): tag type `Byte`, **name length `UnsignedVarInt`**, name UTF-8, payload — with `IntTag` as `VarInt`, `LongTag` as `VarLong`, **string values `UnsignedVarInt`-prefixed** (plain LEB128 — a reader must *not* zigzag-decode them) and list counts `VarInt`-prefixed (zigzag). `BlockEntityDataPacket` sends the compound raw (no length prefix).
* **Plain little-endian flavor** — item NBT inside Slots only: name length `LShort` (the Slot's `LShort` length prefix, see the Slot table above), `IntTag`/`LongTag` as fixed-width `LInt`/`LLong`.

Compounds in both flavors end with a `TAG_End` (`0x00`) byte. Tag IDs `0x00`–`0x0b` exist in protocol 113 (no `LongArray`).
