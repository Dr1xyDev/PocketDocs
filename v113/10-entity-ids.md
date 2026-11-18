# 10 — Legacy entity type IDs

`AddEntityPacket` (`0x0d`, see [07-entities.md](07-entities.md)) identifies the entity with a single **legacy numeric ID** in its `type` field (`UnsignedVarInt`). These are the Mojang-era Pocket Edition numeric IDs; they are also what older plugins and level formats use. The table below is the complete set for protocol 113.

| ID | Entity | ID | Entity |
|---:|--------|---:|--------|
| 10 | Chicken | 51 | NPC |
| 11 | Cow | 52 | Wither |
| 12 | Pig | 53 | Ender Dragon |
| 13 | Sheep | 54 | Shulker |
| 14 | Wolf | 55 | Endermite |
| 15 | Villager | 56 | Learn-to-Code Mascot |
| 16 | Mooshroom | 57 | Vindicator |
| 17 | Squid | 58–60 | *unused* |
| 18 | Rabbit | 61 | Armor Stand |
| 19 | Bat | 62 | Tripod Camera |
| 20 | Iron Golem | 63 | Player |
| 21 | Snow Golem | 64 | Item (dropped item) |
| 22 | Ocelot | 65 | TNT |
| 23 | Horse | 66 | Falling Block |
| 24 | Donkey | 67 | Moving Block |
| 25 | Mule | 68 | XP Bottle |
| 26 | Skeleton Horse | 69 | XP Orb |
| 27 | Zombie Horse | 70 | Eye of Ender Signal |
| 28 | Polar Bear | 71 | Ender Crystal |
| 29 | Llama | 72 | Fireworks Rocket |
| 30 | Parrot | 73–75 | *unused* |
| 31 | *unused* | 76 | Shulker Bullet |
| 32 | Zombie | 77 | Fishing Hook |
| 33 | Creeper | 78 | Chalkboard |
| 34 | Skeleton | 79 | Dragon Fireball |
| 35 | Spider | 80 | Arrow |
| 36 | Zombie Pigman | 81 | Snowball |
| 37 | Slime | 82 | Egg |
| 38 | Enderman | 83 | Painting |
| 39 | Silverfish | 84 | Minecart |
| 40 | Cave Spider | 85 | Large Fireball (ghast fireball) |
| 41 | Ghast | 86 | Splash Potion |
| 42 | Magma Cube | 87 | Ender Pearl |
| 43 | Blaze | 88 | Leash Knot |
| 44 | Zombie Villager | 89 | Wither Skull |
| 45 | Witch | 90 | Boat |
| 46 | Stray | 91 | Wither Skull Dangerous (blue) |
| 47 | Husk | 92 | *unused* |
| 48 | Wither Skeleton | 93 | Lightning Bolt |
| 49 | Guardian | 94 | Small Fireball (blaze fireball) |
| 50 | Elder Guardian | 95 | Area Effect Cloud |

| ID | Entity | ID | Entity |
|---:|--------|---:|--------|
| 96 | Hopper Minecart | 101 | Lingering Potion |
| 97 | TNT Minecart | 102 | Llama Spit |
| 98 | Chest Minecart | 103 | Evocation Fang |
| 99 | *unused* | 104 | Evoker (Evocation Illager) |
| 100 | Command Block Minecart | 105 | Vex |

## Notes

* **Players are never sent through `AddEntityPacket`** — they use `AddPlayerPacket` (`0x0c`). The `63` (Player) type exists in the ID space but is not sent for remote players in protocol 113.
* Dropped items are spawned with `AddItemEntityPacket` (`0x0f`) rather than `AddEntityPacket` with type 64; type 64 is the same entity in the legacy ID space.
* IDs 31, 58–60, 73–75, 92 and 99 are not assigned in protocol 113.
* The `type` value is the *only* entity identity in `AddEntityPacket` — there are no string identifiers on the wire in protocol 113 (string-based identifiers arrived in later protocol versions).
