# 08 — Inventory, containers and items

Window management, item movement, crafting and item drops. Types: [03-data-types.md](03-data-types.md); the Slot structure defined there is used by almost every packet here. Field glosses on fields the reference never reads or writes (e.g. DropItem's type byte, ContainerSetSlot's hotbar fields) describe vanilla-era semantics that the reference does not exercise.

## Window / container IDs

Standard container types (`types/InventoryNetworkIds.php`):

| Value | Constant |
|------:|----------|
| -1 | `INVENTORY` (player inventory, used with special window IDs) |
| 0 | `CONTAINER` (chest) |
| 1 | `WORKBENCH` |
| 2 | `FURNACE` |
| 3 | `ENCHANTMENT` |
| 4 | `BREWING_STAND` |
| 5 | `ANVIL` |
| 6 | `DISPENSER` |
| 7 | `DROPPER` |
| 8 | `HOPPER` |
| 9 | `CAULDRON` |
| 10 | `MINECART_CHEST` |
| 11 | `MINECART_HOPPER` |
| 12 | `HORSE` |
| 13 | `BEACON` |
| 14 | `STRUCTURE_EDITOR` |
| 15 | `TRADING` |
| 16 | `COMMAND_BLOCK` |
| 17 | `JUKEBOX` |

Special window IDs used in place of a real container id when addressing player inventory areas (`ContainerSetContentPacket`):

| Value | Meaning |
|------:|---------|
| 0 | `SPECIAL_INVENTORY` (main inventory, 36 slots) |
| 0x77 | `SPECIAL_OFFHAND` |
| 0x78 | `SPECIAL_ARMOR` (4 slots: head, chest, legs, feet) |
| 0x79 | `SPECIAL_CREATIVE` |
| 0x7a | `SPECIAL_HOTBAR` |
| 0x7b | `SPECIAL_FIXED_INVENTORY` |

## Equipment

**MobEquipmentPacket (`0x1f`) — C2S** (server re-broadcasts to viewers)

| Field | Type | Notes |
|-------|------|-------|
| Entity ID | `VarInt` | |
| Item | Slot | held item |
| Inventory slot | `Byte` | slot the item came from (0–8 for hotbar) |
| Selected slot (hotbar) | `Byte` | hotbar index 0–8 |
| Window ID | `Byte` | usually 0 |

**MobArmorEquipmentPacket (`0x20`) — Both**

| Field | Type |
|-------|------|
| Entity ID | `VarInt` |
| Head slot | Slot |
| Chest slot | Slot |
| Legs slot | Slot |
| Feet slot | Slot |

**HurtArmorPacket (`0x26`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Health | `VarInt` | remaining armor durability |

## Interaction

**InteractPacket (`0x21`) — C2S**

| Field | Type | Notes |
|-------|------|-------|
| Action | `Byte` | 1 right-click, 2 left-click (attack/damage), 3 leave vehicle, 4 mouse-over (hover), 6 open inventory |
| Target entity ID | `VarInt` | entity acted upon; action 6 carries the entity whose inventory is opened (e.g. a horse) |

The reference exercises action 6 only for horse targets: its `InteractPacketHandler` opens the horse's inventory window and consumes the packet; every other action/target combination falls through to the main switch, where only left-click (2) does anything.

**UseItemPacket (`0x23`) — C2S**

Right-click / interact-with-block packet. Placing blocks, using items on blocks, eating starts, throwing projectiles (with an empty block target):

| Field | Type | Notes |
|-------|------|-------|
| Block position | BlockPosition | block being interacted with; for air actions (face −1) the fields carry the **aim direction vector** (each component × 32768) |
| Block ID | `UnsignedVarInt` | ID of the block at the target |
| Face | `VarInt` (signed) | 0–5 face clicked, **−1** for air actions (not 255) |
| Click cursor position | 3 × `LFloat` | offset of the click on the face |
| Player position | 3 × `LFloat` | client player position |
| Slot | `VarInt` | held hotbar slot |
| Item | Slot | the held item |

**DropItemPacket (`0x2e`) — C2S**

| Field | Type | Notes |
|-------|------|-------|
| Type | `Byte` | 0 drop whole stack, 1 drop one item |
| Item | Slot | |

**ItemFrameDropItemPacket (`0x47`) — C2S**

| Field | Type |
|-------|------|
| Frame position | BlockPosition |

## Containers

**ContainerOpenPacket (`0x30`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Window ID | `Byte` | server-assigned, increments per open window |
| Container type | `Byte` | table above |
| Block position | BlockPosition | |
| Entity ID | `VarInt` | for entity-attached containers (horse, minecart); `-1` otherwise |

**ContainerClosePacket (`0x31`) — Both** — client requests closing a window; server confirms.

| Field | Type |
|-------|------|
| Window ID | `Byte` |

**ContainerSetSlotPacket (`0x32`) — Both**

The core of 1.1.x inventory logic: the client reports the result of moving items by sending the resulting slot contents.

| Field | Type | Notes |
|-------|------|-------|
| Window ID | `Byte` | 0 = player inventory |
| Slot | `VarInt` | |
| Hotbar slot | `VarInt` | target hotbar index when slot 0–8 |
| Item | Slot | new contents |
| Selected slot | `Byte` | currently selected hotbar index |

**ContainerSetDataPacket (`0x33`) — S2C**

| Field | Type |
|-------|------|
| Window ID | `Byte` |
| Property | `VarInt` (0 furnace ticks, 1 fuel remaining …) |
| Value | `VarInt` |

**ContainerSetContentPacket (`0x34`) — S2C**

Full window replacement (also used at login for inventory/armor/hotbar/creative lists):

| Field | Type | Notes |
|-------|------|-------|
| Window ID | `UnsignedVarInt` | window ID or special ID |
| Target entity ID | `VarInt` | |
| Slot count | `UnsignedVarInt` | |
| Slots | count × Slot | |
| Hotbar count | `UnsignedVarInt` | always present |
| Hotbar entries | count × `VarInt` | hotbar slot → inventory slot index (only when window ID = 0) |

**ReplaceItemInSlotPacket (`0x48`) — S2C** — the body is a single Slot (item replacement announcement; rarely used in 1.1).

## Crafting

**CraftingDataPacket (`0x35`) — S2C**

| Field | Type |
|-------|------|
| Recipe count | `UnsignedVarInt` |
| Recipes | repeated entry below |
| Clean recipes | `Bool` |

Each entry starts with `VarInt` type, then type-specific body:

| Type | Name | Body |
|-----:|------|------|
| 0 | `SHAPELESS` | `UnsignedVarInt` ingredient count, ingredients (count × Slot), `UnsignedVarInt` result count, results (count × Slot), UUID |
| 1 | `SHAPED` | `VarInt` width, `VarInt` height, width×height Slots, `UnsignedVarInt` result count, results (count × Slot), UUID |
| 2 | `FURNACE` | `VarInt` input ID, Slot result |
| 3 | `FURNACE_DATA` | `VarInt` input ID, `VarInt` input damage, Slot result |
| 4 | `MULTI` | UUID |
| 5 | `SHULKER_BOX` | same body as SHAPELESS |

Recipe UUIDs are transmitted via the UUID type (two LE longs, see [03](03-data-types.md#uuid)).

**CraftingEventPacket (`0x36`) — C2S**

Client reports it performed a crafting-grid recipe:

| Field | Type | Notes |
|-------|------|-------|
| Window ID | `Byte` | |
| Recipe type | `VarInt` | |
| Recipe ID | UUID | |
| Input size | `UnsignedVarInt` | |
| Input items | count × Slot | the reference's decode reads at most 128 entries |
| Output size | `UnsignedVarInt` | |
| Output items | count × Slot | the reference's decode reads at most 128 entries |

**InventoryActionPacket (`0x2f`) — Both (legacy)**

Compatibility packet from the pre-1.0 era:

| Field | Type | Notes |
|-------|------|-------|
| Action | `UnsignedVarInt` | 0 give item, 2 enchant item |
| Item | Slot | |
| Enchantment ID | `VarInt` | |
| Enchantment level | `VarInt` | |

Stock 1.1.x clients do not drive inventory through this packet; they use `ContainerSetSlotPacket`. It is documented for completeness of the ID registry.
