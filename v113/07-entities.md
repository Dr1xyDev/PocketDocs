# 07 — Entity packets

Everything about entities: spawning them, moving them, animating them, and the metadata system that drives their client-side appearance. Types: [03-data-types.md](03-data-types.md). IDs: [04-packet-ids.md](04-packet-ids.md).

## Entity metadata keys and flags

Entity metadata is a key/type/value map (encoding in [03](03-data-types.md#entity-metadata)). Keys used in protocol 113 (reference `Entity.php`):

| Key | Constant | Value type | Meaning |
|----:|----------|------------|---------|
| 0 | `DATA_FLAGS` | Long (VarLong) | 64-bit flag bitmask |
| 1 | `DATA_HEALTH` | Int | minecart/boat health |
| 2 | `DATA_VARIANT` | Int | variant number |
| 3 | `DATA_COLOR` | Byte | color index (sheep, collar…) |
| 4 | `DATA_NAMETAG` | String | display name |
| 5 | `DATA_OWNER_EID` | Long | owner entity |
| 6 | `DATA_TARGET_EID` | Long | attack target |
| 7 | `DATA_AIR` | Short | air ticks |
| 8 | `DATA_POTION_COLOR` | Int | potion ARGB color |
| 9 | `DATA_POTION_AMBIENT` | Byte | ambient potion |
| 10 | `DATA_JUMP_DURATION` | Byte/Long | jump duration |
| 11 | `DATA_HURT_TIME` | Int | minecart/boat hurt time |
| 12 | `DATA_HURT_DIRECTION` | Int | hurt direction |
| 13/14 | `DATA_PADDLE_TIME_LEFT/RIGHT` | Float | boat paddling |
| 15 | `DATA_EXPERIENCE_VALUE` | Int | XP orb amount |
| 16/17/18 | `DATA_MINECART_DISPLAY_BLOCK/OFFSET/HAS_DISPLAY` | Int/Int/Byte | minecart display block |
| 19/20/21 | `DATA_SWELL` / `DATA_OLD_SWELL` / `DATA_SWELL_DIRECTION` | Byte | creeper fuse swell |
| 22 | `DATA_CHARGE_AMOUNT` | Byte | charge amount |
| 23/24 | `DATA_ENDERMAN_HELD_ITEM_ID/DAMAGE` | Short | enderman block |
| 25 | `DATA_ENTITY_AGE` | Short | age |
| 26 | `LEASH_HOLDER_EID` | Int/Byte | leash holder |
| 27 | `DATA_PLAYER_FLAGS` | Byte | player flags |
| 28 | `DATA_PLAYER_INDEX` | Int | player index |
| 29 | `DATA_BED_POSITION` | Pos | sleeping position |
| 30–32 | `DATA_FIREBALL_X/Y/Z_POWER` | Float | fireball power |
| 37 | `DATA_POTION_AUX_VALUE` | Short | potion aux |
| 38 | `DATA_LEAD_HOLDER_EID` | Long | lead holder |
| 39 | `DATA_SCALE` | Float | entity scale (default 1) |
| 40 | `DATA_INTERACTIVE_TAG` | String | button text ("Feed", "Ride"…) |
| 41 | `DATA_NPC_SKIN_ID` | String | NPC skin |
| 42 | `DATA_URL_TAG` | String | URL tag |
| 43 | `DATA_MAX_AIR` | Short | max air ticks |
| 44 | `DATA_MARK_VARIANT` | Int | mark variant |
| 45–47 | `DATA_CONTAINER_TYPE/BASE_SIZE/STRENGTH_MODIFIER` | Byte/Int/Int | minecart containers |
| 48 | `DATA_BLOCK_TARGET` | Pos | end crystal target |
| 49 | `DATA_WITHER_INVULNERABLE_TICKS` | Int | wither invuln |
| 50–52 | `DATA_WITHER_TARGET_1/2/3` | Long | wither heads' targets |
| 53 | `DATA_WITHER_DONE` | Short | wither ready |
| 54/55 | `DATA_BOUNDING_BOX_WIDTH/HEIGHT` | Float | hitbox size |
| 56 | `DATA_FUSE_LENGTH` | Int | TNT fuse |
| 57 | `DATA_RIDER_SEAT_POSITION` | Vector3f | seat offset |
| 58 | `DATA_RIDER_ROTATION_LOCKED` | Byte | rotation lock |
| 59/60 | `DATA_RIDER_MAX/MIN_ROTATION` | Float | rotation limits |
| 61–63 | `DATA_AREA_EFFECT_CLOUD_RADIUS/WAITING/PARTICLE_ID` | Float/Int/Int | area effect cloud |
| 65 | `DATA_SHULKER_ATTACH_FACE` | Byte | shulker face |
| 67 | `DATA_SHULKER_ATTACH_POS` | Pos | shulker position |
| 68 | `DATA_TRADING_PLAYER_EID` | Long | trading player |
| 70–73 | `DATA_COMMAND_BLOCK_ENABLED/COMMAND/LAST_OUTPUT/TRACK_OUTPUT` | Byte/String/String/Byte | command block entity |
| 74 | `DATA_CONTROLLING_RIDER_SEAT_INDEX` | Byte | controlling seat |
| 75/76 | `DATA_STRENGTH/MAX_STRENGTH` | Int | llama strength |
| 77 | `DATA_EVOKER_SPELL_COLOR` | Int | evoker spell |
| 78 | `DATA_ARMOR_STAND_POSE_INDEX` | Int | armor stand pose |
| 79 | `DATA_ENDER_CRYSTAL_TIME_OFFSET` | Int | end crystal offset |
| 91 | `DATA_FLAGS2` | Long | extended flag bitmask (bits ≥ 64) |

**Flag bits** (applied to key 0; bit index ≥ 64 goes to key 91):

| Bit | Flag | Bit | Flag |
|----:|------|----:|------|
| 0 | `ONFIRE` | 23 | `SITTING` |
| 1 | `SNEAKING` | 24 | `ANGRY` |
| 2 | `RIDING` | 25 | `INTERESTED` |
| 3 | `SPRINTING` | 26 | `CHARGED` |
| 4 | `ACTION` (swinging) | 27 | `TAMED` |
| 5 | `INVISIBLE` | 28 | `LEASHED` |
| 6 | `TEMPTED` | 29 | `SHEARED` |
| 7 | `INLOVE` | 30 | `GLIDING` |
| 8 | `SADDLED` | 31 | `ELDER` |
| 9 | `POWERED` | 32 | `MOVING` |
| 10 | `IGNITED` | 33 | `BREATHING` |
| 11 | `BABY` | 34 | `CHESTED` |
| 12 | `CONVERTING` | 35 | `STACKABLE` |
| 13 | `CRITICAL` | 36 | `SHOW_BOTTOM` |
| 14 | `CAN_SHOW_NAMETAG` | 37 | `REARING` |
| 15 | `ALWAYS_SHOW_NAMETAG` | 38 | `VIBRATING` |
| 16 | `IMMOBILE`/`NO_AI` | 39 | `IDLING` |
| 17 | `SILENT` | 40 | `EVOKER_SPELL` |
| 18 | `WALLCLIMBING` | 41 | `CHARGE_ATTACK` |
| 19 | `CAN_CLIMB` | 43 | `WASD_CONTROLLED` |
| 20 | `SWIMMER` | 44 | `CAN_POWER_JUMP` |
| 21 | `CAN_FLY` | 45 | `LINGER` |
| 22 | `RESTING` | | |

## Add/Remove packets

**AddPlayerPacket (`0x0c`) — S2C**

| Field | Type |
|-------|------|
| UUID | UUID (see [03](03-data-types.md#uuid)) |
| Username | String |
| Entity unique ID | `VarInt` |
| Entity runtime ID | `VarInt` |
| Position | 3 × `LFloat` |
| Velocity | 3 × `LFloat` |
| Pitch | `LFloat` |
| Head yaw | `LFloat` |
| Yaw | `LFloat` |
| Held item | Slot |
| Metadata | EntityMetadata |

**AddEntityPacket (`0x0d`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Entity unique ID | `VarInt` | |
| Entity runtime ID | `VarInt` | |
| Entity type | `UnsignedVarInt` | legacy numeric entity ID (see `EntityIds`, e.g. 10 chicken … 45 zombie) |
| Position | 3 × `LFloat` | |
| Velocity | 3 × `LFloat` | |
| Pitch | `LFloat` | |
| Yaw | `LFloat` | |
| Attribute count | `UnsignedVarInt` | |
| Attributes | count × {String name, `LFloat` min, `LFloat` value, `LFloat` max} | |
| Metadata | EntityMetadata | |
| Link count | `UnsignedVarInt` | |
| Links | count × {`VarInt` from, `VarInt` to, `Byte` type} | 0 remove, 1 rider, 2 passenger |

Known attribute names (reference `Attribute.php`): `minecraft:absorption`, `minecraft:player.saturation`, `minecraft:player.exhaustion`, `minecraft:knockback_resistance`, `minecraft:health`, `minecraft:movement`, `minecraft:follow_range`, `minecraft:player.hunger`, `minecraft:attack_damage`, `minecraft:player.level`, `minecraft:player.experience`, `minecraft:horse.jump_strength`.

**RemoveEntityPacket (`0x0e`) — S2C**

| Field | Type |
|-------|------|
| Entity ID | `VarInt` |

**AddItemEntityPacket (`0x0f`) — S2C**

| Field | Type |
|-------|------|
| Entity unique ID | `VarInt` |
| Entity runtime ID | `VarInt` |
| Item | Slot |
| Position | 3 × `LFloat` |
| Velocity | 3 × `LFloat` |
| Metadata | EntityMetadata |

**AddHangingEntityPacket (`0x10`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Entity unique ID | `VarInt` | |
| Entity runtime ID | `VarInt` | |
| Position | BlockPosition | |
| Unknown | `VarInt` | rotation direction |

**AddPaintingPacket (`0x17`) — S2C**

| Field | Type |
|-------|------|
| Entity unique ID | `VarInt` |
| Entity runtime ID | `VarInt` |
| Position | BlockPosition |
| Direction | `VarInt` (0 south, 1 west, 2 north, 3 east) |
| Motive | String (painting title) |

**AddItemPacket (`0x4b`) — S2C** — legacy "creative content" item announcement; reference body is just:

| Field | Type |
|-------|------|
| Item | Slot |

**TakeItemEntityPacket (`0x11`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Item entity ID | `VarInt` | the item being taken |
| Collector entity ID | `VarInt` | who takes it |

## Movement packets

**MoveEntityPacket (`0x12`) — Both** (S2C broadcast; the reference additionally accepts it C2S for entities the player drives)

| Field | Type |
|-------|------|
| Entity runtime ID | `VarInt` |
| Position | 3 × `LFloat` |
| Pitch | ByteRotation |
| Head yaw | ByteRotation |
| Yaw | ByteRotation |
| On ground | `Bool` |
| Teleported | `Bool` |

**MovePlayerPacket (`0x13`) — Both** (C2S is the player's own movement report; S2C teleports/corrections)

| Field | Type | Notes |
|-------|------|-------|
| Entity runtime ID | `VarInt` | |
| Position | 3 × `LFloat` | |
| Pitch | `LFloat` | |
| Yaw | `LFloat` | |
| Body yaw (head yaw) | `LFloat` | |
| Mode | `Byte` | 0 normal, 1 reset (position sync), 2 teleport, 3 pitch-only |
| On ground | `Bool` | |
| Riding entity ID | `VarInt` | vehicle when riding, else 0 |
| Teleport cause | `LInt` | only when mode = 2 |
| Teleport item | `LInt` | only when mode = 2 |

## Attribute / effect / event packets

**UpdateAttributesPacket (`0x1e`) — S2C**

| Field | Type |
|-------|------|
| Entity ID | `VarInt` |
| Entry count | `UnsignedVarInt` |
| Entries | count × {`LFloat` min, `LFloat` max, `LFloat` current, `LFloat` default, String name} |

**MobEffectPacket (`0x1d`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Entity ID | `VarInt` | |
| Event | `Byte` | 1 add, 2 modify, 3 remove |
| Effect ID | `VarInt` | vanilla effect ID (1 speed … 25 bad omen era) |
| Amplifier | `VarInt` | |
| Particles | `Bool` | |
| Duration | `VarInt` | ticks |

**EntityEventPacket (`0x1c`) — Both**

| Field | Type |
|-------|------|
| Entity ID | `VarInt` |
| Event | `Byte` |
| Data | `VarInt` |

Event constants: `1` JUMP, `2` HURT_ANIMATION, `3` DEATH_ANIMATION, `4` ARM_SWING, `6` TAME_FAIL, `7` TAME_SUCCESS, `8` SHAKE_WET, `9` USE_ITEM, `10` EAT_GRASS, `11–14` FISH_HOOK_* (bubble/position/hook/tease), `15` SQUID_INK_CLOUD, `16` AMBIENT_SOUND / ZOMBIE_VILLAGER_CURE, `18` RESPAWN, `19/20` IRON_GOLEM_OFFER/WITHDRAW_FLOWER, `21` LOVE_PARTICLES, `24` WITCH_SPELL_PARTICLES, `25` FIREWORK_PARTICLES, `27` SILVERFISH_SPAWN_ANIMATION, `29/30` WITCH_DRINK/THROW_POTION, `31` MINECART_TNT_PRIME_FUSE, `32` PRIME_CREEPER, `34` PLAYER_ADD_XP_LEVELS, `35` ELDER_GUARDIAN_CURSE, `36` AGENT_ARM_SWING, `37` ENDER_DRAGON_DEATH, `38` DUST_PARTICLES, `39` ARROW_SHAKE, `57` EATING_ITEM, `60` BABY_ANIMAL_FEED, `61` DEATH_SMOKE_CLOUD, `62` COMPLETE_TRADE, `63` REMOVE_LEASH (data 1 = cut), `65` CONSUME_TOTEM, `66` PLAYER_CHECK_TREASURE_HUNTER_ACHIEVEMENT, `67` ENTITY_SPAWN, `68` DRAGON_PUKE, `69` ITEM_ENTITY_MERGE.

**EntityFallPacket (`0x25`) — C2S**

| Field | Type |
|-------|------|
| Entity runtime ID | `VarInt` |
| Fall distance | `LFloat` |
| Unknown flag | `Bool` |

**HurtArmorPacket (`0x26`) — S2C** — documented in [08](08-inventory-and-items.md).

**RiderJumpPacket (`0x14`) — C2S**

| Field | Type | Notes |
|-------|------|-------|
| Jump intensity | `VarInt` | horse jump charge |

**SpawnExperienceOrbPacket (`0x42`) — S2C**

| Field | Type |
|-------|------|
| Position | 3 × `LFloat` |
| Amount | `VarInt` |

## Linking, listing, boss bars

**SetEntityLinkPacket (`0x29`) — S2C**

| Field | Type | Notes |
|-------|------|-------|
| Rider (from) | `VarInt` | |
| Vehicle (to) | `VarInt` | |
| Link type | `Byte` | 0 remove, 1 ride, 2 passenger |

**SetEntityDataPacket (`0x27`) — S2C**

| Field | Type |
|-------|------|
| Entity ID | `VarInt` |
| Metadata | EntityMetadata |

**SetEntityMotionPacket (`0x28`) — S2C**

| Field | Type |
|-------|------|
| Entity ID | `VarInt` |
| Motion | 3 × `LFloat` |

**PlayerListPacket (`0x3f`) — S2C**

| Field | Type |
|-------|------|
| Action | `Byte` (0 add, 1 remove) |
| Entry count | `UnsignedVarInt` |
| Entries | action-dependent |

Add entry: UUID, `VarInt` entity ID, String name, String skin ID, String skin data (base64).
Remove entry: UUID only.

**BossEventPacket (`0x4c`) — Both**

| Field | Type |
|-------|------|
| Entity ID (boss) | `VarInt` |
| Type | `UnsignedVarInt` |

Type-dependent body:

| Type | Name | Additional fields |
|-----:|------|-------------------|
| 0 | `SHOW` (S2C) | String title, `LFloat` health percent, `LShort` unknown, `UnsignedVarInt` color, `UnsignedVarInt` overlay |
| 1 | `REGISTER_PLAYER` (C2S) | `VarInt` player entity ID |
| 2 | `HIDE` (S2C) | – |
| 3 | `UNREGISTER_PLAYER` (C2S) | `VarInt` player entity ID |
| 4 | `HEALTH_PERCENT` (S2C) | `LFloat` health percent |
| 5 | `TITLE` (S2C) | String title |
| 6 | unknown | `LShort` unknown, `UnsignedVarInt` color, `UnsignedVarInt` overlay |
| 7 | `TEXTURE` (S2C) | `UnsignedVarInt` color, `UnsignedVarInt` overlay |

(The reference's decode/encode fall through from `SHOW` into types 6/7 — a `SHOW` event therefore carries all five trailing fields, as listed above.)
