# 06 — Spawn, world and chunk packets

Packets that describe the world itself: spawning the player, block and level state, chunks, dimension changes and global events. Types are defined in [03-data-types.md](03-data-types.md); IDs in [04-packet-ids.md](04-packet-ids.md).

## StartGamePacket (`0x0b`) — S2C

The single most important spawn packet; sent once, before chunks:

| # | Field | Type | Notes |
|--:|-------|------|-------|
| 1 | Entity unique ID | `VarInt` (entity ID, zigzag) | the player's entity |
| 2 | Entity runtime ID | `VarInt` | same value as the unique ID |
| 3 | Player gamemode | `VarInt` | 0 survival, 1 creative, 2 adventure; spectator is sent as creative (`getClientFriendlyGamemode`) |
| 4–6 | Player position | 3 × `LFloat` | x, y (feet + eye offset), z |
| 7 | Pitch | `LFloat` | |
| 8 | Yaw | `LFloat` | |
| 9 | World seed | `VarInt` | always `-1` (hardcoded by the reference implementation) |
| 10 | Dimension | `VarInt` | 0 overworld, 1 nether, 2 end |
| 11 | World generator | `VarInt` | 0 old/infinite-legacy, 1 infinite, 2 flat |
| 12 | World gamemode | `VarInt` | default gamemode of the world |
| 13 | Difficulty | `VarInt` | 0 peaceful … 3 hard |
| 14–16 | World spawn position | BlockPosition | `VarInt` x, `UnsignedVarInt` y, `VarInt` z |
| 17 | Achievements disabled | `Bool` | |
| 18 | Day-cycle stop time | `VarInt` | `-1` = normal cycle, else frozen time |
| 19 | Education mode | `Bool` | |
| 20 | Rain level | `LFloat` | |
| 21 | Lightning level | `LFloat` | |
| 22 | Commands enabled | `Bool` | |
| 23 | Texture packs required | `Bool` | |
| 24 | Game rules | GameRules | see [03](03-data-types.md#game-rules) |
| 25 | Level ID | String | the level's numeric ID as a string on the reference implementation (e.g. `"1"`); some implementations send `""` |
| 26 | World name | String | shown above the hotbar |
| 27 | Premium world template ID | String | always `""` in the reference implementation |

## Player state packets

**SetHealthPacket (`0x2a`) — S2C**

| Field | Type |
|-------|------|
| Health | `VarInt` |

**SetSpawnPositionPacket (`0x2b`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Spawn type | `VarInt` | 0 = player spawn, 1 = world spawn |
| Position | BlockPosition | |
| Spawn forced | `Bool` | |

**RespawnPacket (`0x2d`) — S2C**

| Field | Type |
|-------|------|
| Position | 3 × `LFloat` |

**SetTimePacket (`0x0a`) — S2C**

| Field | Type |
|-------|------|
| Time | `VarInt` |

**SetDifficultyPacket (`0x3c`) — S2C**

| Field | Type |
|-------|------|
| Difficulty | `UnsignedVarInt` (0–3) |

**SetCommandsEnabledPacket (`0x3b`) — S2C**

| Field | Type |
|-------|------|
| Enabled | `Bool` |

**SetPlayerGameTypePacket (`0x3e`) — Both** (server changes gamemode; client echoes the change)

| Field | Type |
|-------|------|
| Gamemode | `VarInt` (0–3) |

**ChangeDimensionPacket (`0x3d`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Dimension | `VarInt` | 0/1/2 |
| Position | 3 × `LFloat` | player position in the new dimension (×8/÷8 scaled between overworld and nether) |
| Respawn | `Bool` | |

The client answers the dimension change with `0x24 PlayerAction` actions 13 (`SPAWN_OVERWORLD`), 14 (`SPAWN_NETHER`) or 7 (`SPAWN_SAME_DIMENSION`) — in the reference, 7 and 13 route to the respawn path, while 14's dimension-confirmation branch is dead code (its `awaitingDimensionChange` flag is only ever set by a commented-out block).

**ShowCreditsPacket (`0x4d`) — S2C**

| Field | Type |
|-------|------|
| Entity ID | `VarInt` |
| Type | `VarInt` (0 = start credits, 1 = end credits, skip to respawn) |

**GameRulesChangedPacket (`0x49`) — S2C** — body = GameRules structure.

**CameraPacket (`0x4a`) — S2C**

| Field | Type |
|-------|------|
| Camera entity ID | `VarInt` |
| Player entity ID | `VarInt` |

## Chunk and block packets

### FullChunkDataPacket (`0x3a`) — S2C

| Field | Type |
|-------|------|
| Chunk X | `VarInt` |
| Chunk Z | `VarInt` |
| Chunk data | String (`UnsignedVarInt` length + bytes) |

One packet per column. The payload ("chunk data") layout, exactly as serialized for the client:

| Section | Size | Contents |
|---------|-----:|----------|
| Sub-chunk count | 1 byte | number of following sub-chunk sections |
| Sub-chunk sections | count × 10 241 bytes | see below (1 + 4096 + 2048×3) |
| Heightmap | 256 × `LShort` | one column height per (x,z), little-endian |
| Biome IDs | 256 bytes | one byte per (x,z) column |
| Border block count | 1 byte | always `0x00` (entries would crash the stock client) |
| Extra data | VarInt count + entries | entry = `VarInt` key, `LShort` value |
| Tile NBT | variable | serialized little-endian NBT compounds of the chunk's tiles, back-to-back |

Each **sub-chunk section** is:

| Field | Size | Contents |
|-------|-----:|----------|
| Storage version | 1 byte | `0x00` (legacy, non-paletted) |
| Block IDs | 4096 bytes | one byte per block, **x-major order**: index = `x×256 + z×16 + y` |
| Block metadata | 2048 bytes | 4-bit nibbles in the same x-major order; of each byte's pair, the even **y** goes in the low nibble |
| Sky light | 2048 bytes | nibbles, same order |
| Block light | 2048 bytes | nibbles, same order |

### UpdateBlockPacket (`0x16`) — S2C

| Field | Type | Notes |
|-------|------|-------|
| Position | BlockPosition | |
| Block ID | `UnsignedVarInt` | |
| Meta + flags | `UnsignedVarInt` | packed `meta & 0x0F`, flags in the upper 4 bits |

Flag bits: `0b0001` neighbors, `0b0010` network, `0b0100` no-graphic, `0b1000` priority. The client applies the block visually when the network flag (`0b0010`) is set. The reference implementation sends `0x30 | meta` (neighbors + network) or `0xB0 | meta` (neighbors + network + priority); vanilla clients commonly send `0x20 | meta` (network flag only).

### BlockEventPacket (`0x1b`) — S2C

| Field | Type | Notes |
|-------|------|-------|
| Position | BlockPosition | |
| Case 1 | `VarInt` | event type (e.g. chest open/close state) |
| Case 2 | `VarInt` | event parameter |

### BlockEntityDataPacket (`0x38`) — Both

Server→client: full replacement of a tile's NBT. Client→server: sign text and similar edits.

| Field | Type |
|-------|------|
| Position | BlockPosition |
| NBT | raw little-endian compound (rest of packet) |

### ExplodePacket (`0x18`) — S2C

| Field | Type |
|-------|------|
| Origin | 3 × `LFloat` |
| Radius × 32 | `VarInt` (write `(int)(radius * 32)`) |
| Destroyed block count | `UnsignedVarInt` |
| Destroyed blocks | count × SignedBlockPosition (3 × `VarInt`) |

## LevelEventPacket (`0x1a`) — S2C

| Field | Type |
|-------|------|
| Event ID | `VarInt` |
| Position | 3 × `LFloat` |
| Data | `VarInt` |

Event ID constants:

| Value | Constant | Meaning |
|------:|----------|---------|
| 1000 | `EVENT_SOUND_CLICK` | click |
| 1001 | `EVENT_SOUND_CLICK_FAIL` | failed click |
| 1002 | `EVENT_SOUND_SHOOT` | bow shot |
| 1003 | `EVENT_SOUND_DOOR` | door |
| 1004 | `EVENT_SOUND_FIZZ` | fizz |
| 1005 | `EVENT_SOUND_IGNITE` | TNT ignite |
| 1007 | `EVENT_SOUND_GHAST` | ghast moan |
| 1008 | `EVENT_SOUND_GHAST_SHOOT` | fireball |
| 1009 | `EVENT_SOUND_BLAZE_SHOOT` | blaze shot |
| 1010 | `EVENT_SOUND_DOOR_BUMP` | door bump |
| 1012 | `EVENT_SOUND_DOOR_CRASH` | door crash |
| 1018 | `EVENT_SOUND_ENDERMAN_TELEPORT` | enderman teleport |
| 1020–1022 | `EVENT_SOUND_ANVIL_BREAK/USE/FALL` | anvil sounds |
| 1030 | `EVENT_SOUND_POP` | item pop |
| 1032 | `EVENT_SOUND_PORTAL` | portal |
| 1040–1044 | `EVENT_SOUND_ITEMFRAME_*` | item frame add/remove/place/rotate |
| 1050 | `EVENT_SOUND_CAMERA` | camera |
| 1051 | `EVENT_SOUND_ORB` | XP orb pickup |
| 1052 | `EVENT_SOUND_TOTEM` | totem (the reference broadcasts it on totem use) |
| 2000 | `EVENT_PARTICLE_SHOOT` | shoot particles |
| 2001 | `EVENT_PARTICLE_DESTROY` | block break particles |
| 2002 | `EVENT_PARTICLE_SPLASH` | splash potion |
| 2003 | `EVENT_PARTICLE_EYE_DESPAWN` | eye despawn |
| 2004 | `EVENT_PARTICLE_SPAWN` | generic particle spawn |
| 2006 | `EVENT_GUARDIAN_CURSE` | elder guardian curse |
| 2008 | `EVENT_PARTICLE_BLOCK_FORCE_FIELD` | force field particles |
| 2009 | `EVENT_PARTICLE_PROJECTILE_HIT` | projectile hit |
| 2013 | `EVENT_PARTICLE_ENDERMAN_TELEPORT` | enderman teleport particles |
| 2014 | `EVENT_PARTICLE_PUNCH_BLOCK` | block punch particles |
| 3001 | `EVENT_START_RAIN` | start rain |
| 3002 | `EVENT_START_THUNDER` | start thunder |
| 3003 | `EVENT_STOP_RAIN` | stop rain |
| 3004 | `EVENT_STOP_THUNDER` | stop thunder |
| 3005 | `EVENT_PAUSE_GAME` | pause (data 1 = pause, 0 = resume) |
| 3500 | `EVENT_REDSTONE_TRIGGER` | redstone trigger |
| 3501–3508 | `EVENT_CAULDRON_*` | cauldron explode/dye/clean/fill/take |
| 3600/3601 | `EVENT_BLOCK_START_BREAK` / `EVENT_BLOCK_STOP_BREAK` | mining progress |
| 4000 | `EVENT_SET_DATA` | legacy block data change |
| 9800 | `EVENT_PLAYERS_SLEEPING` | all players sleeping |

`0x4000` (`EVENT_ADD_PARTICLE_MASK`) is a bit-OR mask added to particle event IDs, not an event of its own.

## LevelSoundEventPacket (`0x19`) — Both

Client→server copies are relayed by the server to nearby viewers.

| Field | Type | Notes |
|-------|------|-------|
| Sound | `Byte` | enum below |
| Position | 3 × `LFloat` | |
| Extra data | `VarInt` | block/item id context, `-1` default |
| Pitch | `VarInt` | `1` default |
| Baby sound | `Bool` | |
| Relative volume disabled | `Bool` | |

Sound enum: `0` ITEM_USE_ON, `1` HIT, `2` STEP, `3` JUMP, `4` BREAK, `5` PLACE, `6` HEAVY_STEP, `7` GALLOP, `8` FALL, `9` AMBIENT, `10` AMBIENT_BABY, `11` AMBIENT_IN_WATER, `12` BREATHE, `13` DEATH, `14` DEATH_IN_WATER, `15` DEATH_TO_ZOMBIE, `16` HURT, `17` HURT_IN_WATER, `18` MAD, `19` BOOST, `20` BOW, `21` SQUISH_BIG, `22` SQUISH_SMALL, `23` FALL_BIG, `24` FALL_SMALL, `25` SPLASH, `26` FIZZ, `27` FLAP, `28` SWIM, `29` DRINK, `30` EAT, `31` TAKEOFF, `32` SHAKE, `33` PLOP, `34` LAND, `35` SADDLE, `36` ARMOR, `37` ADD_CHEST, `38` THROW, `39` ATTACK, `40` ATTACK_NODAMAGE, `41` WARN, `42` SHEAR, `43` MILK, `44` THUNDER, `45` EXPLODE, `46` FIRE, `47` IGNITE, `48` FUSE, `49` STARE, `50` SPAWN, `51` SHOOT, `52` BREAK_BLOCK, `53` REMEDY, `54` UNFECT, `55` LEVELUP, `56` BOW_HIT, `57` BULLET_HIT, `58` EXTINGUISH_FIRE, `59` ITEM_FIZZ, `60` CHEST_OPEN, `61` CHEST_CLOSED, `62` SHULKERBOX_OPEN, `63` SHULKERBOX_CLOSED, `64` POWER_ON, `65` POWER_OFF, `66` ATTACH, `67` DETACH, `68` DENY, `69` TRIPOD, `70` POP, `71` DROP_SLOT, `72` NOTE, `73` THORNS, `74` PISTON_IN, `75` PISTON_OUT, `76` PORTAL, `77` WATER, `78` LAVA_POP, `79` LAVA, `80` BURP, `81` BUCKET_FILL_WATER, `82` BUCKET_FILL_LAVA, `83` BUCKET_EMPTY_WATER, `84` BUCKET_EMPTY_LAVA, `85` GUARDIAN_FLOP, `86` ELDERGUARDIAN_CURSE, `87` MOB_WARNING, `88` MOB_WARNING_BABY, `89` TELEPORT, `90` SHULKER_OPEN, `91` SHULKER_CLOSE, `92` HAGGLE, `93` HAGGLE_YES, `94` HAGGLE_NO, `95` HAGGLE_IDLE, `96` CHORUSGROW, `97` CHORUSDEATH, `98` GLASS, `99` CAST_SPELL, `100` PREPARE_ATTACK, `101` PREPARE_SUMMON, `102` PREPARE_WOLOLO, `103` FANG, `104` CHARGE, `105` CAMERA_TAKE_PICTURE, `106` DEFAULT, `107` UNDEFINED.
