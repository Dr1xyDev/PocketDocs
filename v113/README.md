# Minecraft: Pocket Edition 1.1.x — Protocol 113 Documentation

This directory documents the **client protocol of Minecraft: Pocket Edition 1.1.0 through 1.1.7**, network protocol **113**.

Everything here is reconstructed byte-by-byte from working protocol implementations of the era — primarily [Aquamarine-Team/Aquamarine](https://github.com/Aquamarine-Team/Aquamarine) (branch `stable`, a GenisysPro/PocketMine-MP lineage server that implements protocol 113 exactly), cross-checked against [GenisysPro/GenisysPro](https://github.com/GenisysPro/GenisysPro). Both repositories carry an **identical** packet ID table for protocol 113 (93 IDs, `0x01`–`0x5d`, verified by diff). Historical version numbers are taken from the Minecraft Wiki *Protocol version* list (mirrored on minecraft.fandom.com).

**Scope:** this documentation covers only the **wire protocol that speaks with the client** — transport, framing, packet layouts and the client's side of every exchange. Server software internals (packet dispatch, anti-abuse limits, permission logic, implementation quirks) are intentionally out of scope; where a byte format was taken from server software, only the wire-visible part is documented.

## Version ↔ protocol mapping

| Protocol | Game version | Notes |
|---------:|--------------|-------|
| 84 | 0.15.10 | pre-1.1 context |
| 90 | 0.16.0 build 5 | pre-1.1 context |
| 92 | 1.0.0.1 | 1.0 release era |
| 100 | 1.0.2 | 1.0 release era |
| 105 | 1.0.6.0 (alpha build) | 1.0 era |
| 106 | 1.0.6 | 1.0 era |
| 107 | 1.0.9 | 1.0 era |
| **110** | 1.1.0.5 (beta build) | accepted by 1.1.x-era server software |
| **111** | 1.1.0.8 (beta build) | accepted by 1.1.x-era server software |
| **112** | 1.1.0.9 (beta build) | accepted by 1.1.x-era server software |
| **113** | **1.1.0 – 1.1.7 releases** | **this documentation** |

Protocol-113 implementations of the era declare `CURRENT_PROTOCOL = 113` with `MINECRAFT_VERSION = ["v1.1.0", "v1.1.1", "v1.1.2", "v1.1.3", "v1.1.4", "v1.1.5", "v1.1.7"]` and `MINECRAFT_VERSION_NETWORK = "1.1.7"` — i.e. every 1.1.x release spoke the same protocol 113; only the 1.1.0 pre-release builds had their own numbers (110/111/112). A client announcing protocol 113 identifies itself as a 1.1.x client.

## File index

| File | Contents |
|------|----------|
| [01-transport-raknet.md](01-transport-raknet.md) | UDP transport: RakNet offline handshake, datagrams, encapsulation, reliability, ordering channels, server discovery |
| [02-framing-batching.md](02-framing-batching.md) | The `0xFE` Batch packet, zlib compression, sub-packet framing, encryption interplay, size limits |
| [03-data-types.md](03-data-types.md) | Every wire-level data type: primitives, VarInt/VarLong (LEB128 + zigzag), String, UUID, Vector3f, positions, Slot/ItemStack, entity metadata, game rules, wire NBT flavors |
| [04-packet-ids.md](04-packet-ids.md) | Complete packet ID registry for protocol 113 (`0x01`–`0x5d` + `0xFE`) with directions and category routing |
| [05-login-and-session.md](05-login-and-session.md) | Login packet, auth chain JWTs, client-data JWT claims, encryption handshake, PlayStatus, Disconnect, the six resource-pack packets, Transfer, full session flow |
| [06-spawn-and-world.md](06-spawn-and-world.md) | StartGame and every spawn/world packet, chunk serialization format (sub-chunks, heightmap, biomes, extra data, tile NBT), block updates, LevelEvent/LevelSoundEvent constants |
| [07-entities.md](07-entities.md) | Add/Remove/Move entity packets, entity metadata key and flag tables, attributes, effects, links, PlayerList, BossEvent, EntityEvent constants |
| [08-inventory-and-items.md](08-inventory-and-items.md) | Container window IDs, equipment, container open/close/set packets, UseItem, crafting data/event, drop-item, item frames, legacy InventoryAction |
| [09-player-input-and-ui.md](09-player-input-and-ui.md) | Client→server gameplay: PlayerAction actions, RemoveBlock, Animate, PlayerInput, AdventureSettings flags, chat/Text, commands, maps, titles, sounds, trade/equip |
| [10-entity-ids.md](10-entity-ids.md) | Legacy numeric entity type table (`AddEntityPacket` type field, `0x0a`–`0x69`) — the complete set for protocol 113 |
| [11-available-commands-json.md](11-available-commands-json.md) | The `AvailableCommandsPacket` JSON schema: command entries, versions, overloads, parameters |

## Conventions

- **C2S** = client → server, **S2C** = server → client. "The client" is the Minecraft PE 1.1.x game client; this documentation is written from the client protocol's perspective, so client-side encode/decode obligations are called out per packet.
- Byte layouts are given in transmission order. Unless a field explicitly says *big-endian*, all multi-byte scalars are **little-endian**.
- `VarInt` (zigzag) / `UnsignedVarInt` (plain LEB128) etc. follow the naming of the reference implementation; see [03-data-types.md](03-data-types.md) for exact encodings.
- Packet names use the reference implementation class names (`LoginPacket`, `StartGamePacket`, …), which match the names used by PocketMine/GenisysPro lineage servers.
- Wire tables use: *offsets relative to the packet body*, *types as defined in 03-data-types.md*.
- "Reference implementation" = Aquamarine `stable` (commit `47766f0`). It is cited only as the provenance of byte formats; its internal behaviour is not documented here.

## Sources

- `Aquamarine-Team/Aquamarine` — `stable` branch (commit `47766f0`): packet classes, wire helpers, entity metadata/ID constants, encryption scheme, RakNet formats.
- `GenisysPro/GenisysPro` — `master`: protocol ID table (identical to Aquamarine's, 93 entries).
- Minecraft Wiki, *Protocol version* (minecraft.wiki / minecraft.fandom.com mirror) for the historical protocol↔version table.

**Accuracy audit:** every byte layout, constant and enumeration in this directory was cross-checked against the sources above by a six-way verification pass (RakNet, data types, login, world/entities, inventory/UI, entity IDs/command JSON), followed by a second-round audit that re-verified every correction and swept hedged claims (2026-11-18), then dedicated full-text audits of the transport, framing, data-type, packet-registry, login, world, input/UI, entity and command documents plus row-level audits of the inventory/entity-ID constant tables, and regression sweeps of the final state (latest 2026-11-22); client-side behaviours that the sources cannot confirm are marked as such in the text.
