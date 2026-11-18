# 04 — Packet ID registry (protocol 113)

Protocol 113 defines a **contiguous block of 93 game packet IDs, `0x01`–`0x5d`**, plus the Batch wrapper `0xFE`. The table below is the registry from the reference implementation's `ProtocolInfo` (Aquamarine `stable`); it is byte-for-byte identical to GenisysPro's. Directions: **C2S** = client → server, **S2C** = server → client.

| ID | Hex | Packet | Direction | Documented in |
|---:|-----|--------|:---------:|---------------|
| 1 | `0x01` | LoginPacket | C2S | [05](05-login-and-session.md) |
| 2 | `0x02` | PlayStatusPacket | S2C | [05](05-login-and-session.md) |
| 3 | `0x03` | ServerToClientHandshakePacket | S2C | [05](05-login-and-session.md) |
| 4 | `0x04` | ClientToServerHandshakePacket | C2S | [05](05-login-and-session.md) |
| 5 | `0x05` | DisconnectPacket | S2C | [05](05-login-and-session.md) |
| 6 | `0x06` | ResourcePacksInfoPacket | S2C | [05](05-login-and-session.md) |
| 7 | `0x07` | ResourcePackStackPacket | S2C | [05](05-login-and-session.md) |
| 8 | `0x08` | ResourcePackClientResponsePacket | C2S | [05](05-login-and-session.md) |
| 9 | `0x09` | TextPacket | Both | [09](09-player-input-and-ui.md) |
| 10 | `0x0a` | SetTimePacket | S2C | [06](06-spawn-and-world.md) |
| 11 | `0x0b` | StartGamePacket | S2C | [06](06-spawn-and-world.md) |
| 12 | `0x0c` | AddPlayerPacket | S2C | [07](07-entities.md) |
| 13 | `0x0d` | AddEntityPacket | S2C | [07](07-entities.md) |
| 14 | `0x0e` | RemoveEntityPacket | S2C | [07](07-entities.md) |
| 15 | `0x0f` | AddItemEntityPacket | S2C | [07](07-entities.md) |
| 16 | `0x10` | AddHangingEntityPacket | S2C | [07](07-entities.md) |
| 17 | `0x11` | TakeItemEntityPacket | S2C | [07](07-entities.md) |
| 18 | `0x12` | MoveEntityPacket | Both (clients driving a vehicle report the vehicle's movement) | [07](07-entities.md) |
| 19 | `0x13` | MovePlayerPacket | Both | [07](07-entities.md) |
| 20 | `0x14` | RiderJumpPacket | C2S | [07](07-entities.md) |
| 21 | `0x15` | RemoveBlockPacket | C2S | [09](09-player-input-and-ui.md) |
| 22 | `0x16` | UpdateBlockPacket | S2C | [06](06-spawn-and-world.md) |
| 23 | `0x17` | AddPaintingPacket | S2C | [07](07-entities.md) |
| 24 | `0x18` | ExplodePacket | S2C | [06](06-spawn-and-world.md) |
| 25 | `0x19` | LevelSoundEventPacket | Both (client→server is broadcast to viewers) | [06](06-spawn-and-world.md) |
| 26 | `0x1a` | LevelEventPacket | S2C | [06](06-spawn-and-world.md) |
| 27 | `0x1b` | BlockEventPacket | S2C | [06](06-spawn-and-world.md) |
| 28 | `0x1c` | EntityEventPacket | Both | [07](07-entities.md) |
| 29 | `0x1d` | MobEffectPacket | S2C | [07](07-entities.md) |
| 30 | `0x1e` | UpdateAttributesPacket | S2C | [07](07-entities.md) |
| 31 | `0x1f` | MobEquipmentPacket | C2S (servers also broadcast S2C copies) | [08](08-inventory-and-items.md) |
| 32 | `0x20` | MobArmorEquipmentPacket | Both | [08](08-inventory-and-items.md) |
| 33 | `0x21` | InteractPacket | C2S | [08](08-inventory-and-items.md) |
| 34 | `0x22` | BlockPickRequestPacket | C2S | [09](09-player-input-and-ui.md) |
| 35 | `0x23` | UseItemPacket | C2S | [08](08-inventory-and-items.md) |
| 36 | `0x24` | PlayerActionPacket | C2S | [09](09-player-input-and-ui.md) |
| 37 | `0x25` | EntityFallPacket | C2S | [07](07-entities.md) |
| 38 | `0x26` | HurtArmorPacket | S2C | [08](08-inventory-and-items.md) |
| 39 | `0x27` | SetEntityDataPacket | S2C | [07](07-entities.md) |
| 40 | `0x28` | SetEntityMotionPacket | S2C | [07](07-entities.md) |
| 41 | `0x29` | SetEntityLinkPacket | S2C | [07](07-entities.md) |
| 42 | `0x2a` | SetHealthPacket | S2C | [06](06-spawn-and-world.md) |
| 43 | `0x2b` | SetSpawnPositionPacket | S2C | [06](06-spawn-and-world.md) |
| 44 | `0x2c` | AnimatePacket | Both | [09](09-player-input-and-ui.md) |
| 45 | `0x2d` | RespawnPacket | S2C | [06](06-spawn-and-world.md) |
| 46 | `0x2e` | DropItemPacket | C2S | [08](08-inventory-and-items.md) |
| 47 | `0x2f` | InventoryActionPacket | Both (legacy/compat; see notes) | [08](08-inventory-and-items.md) |
| 48 | `0x30` | ContainerOpenPacket | S2C | [08](08-inventory-and-items.md) |
| 49 | `0x31` | ContainerClosePacket | Both | [08](08-inventory-and-items.md) |
| 50 | `0x32` | ContainerSetSlotPacket | Both | [08](08-inventory-and-items.md) |
| 51 | `0x33` | ContainerSetDataPacket | S2C | [08](08-inventory-and-items.md) |
| 52 | `0x34` | ContainerSetContentPacket | S2C | [08](08-inventory-and-items.md) |
| 53 | `0x35` | CraftingDataPacket | S2C | [08](08-inventory-and-items.md) |
| 54 | `0x36` | CraftingEventPacket | C2S | [08](08-inventory-and-items.md) |
| 55 | `0x37` | AdventureSettingsPacket | Both | [09](09-player-input-and-ui.md) |
| 56 | `0x38` | BlockEntityDataPacket | Both | [06](06-spawn-and-world.md) |
| 57 | `0x39` | PlayerInputPacket | C2S | [09](09-player-input-and-ui.md) |
| 58 | `0x3a` | FullChunkDataPacket | S2C | [06](06-spawn-and-world.md) |
| 59 | `0x3b` | SetCommandsEnabledPacket | S2C | [06](06-spawn-and-world.md) |
| 60 | `0x3c` | SetDifficultyPacket | S2C | [06](06-spawn-and-world.md) |
| 61 | `0x3d` | ChangeDimensionPacket | S2C | [06](06-spawn-and-world.md) |
| 62 | `0x3e` | SetPlayerGameTypePacket | Both | [09](09-player-input-and-ui.md) |
| 63 | `0x3f` | PlayerListPacket | S2C | [07](07-entities.md) |
| 64 | `0x40` | SimpleEventPacket | reserved — defined in the registry, **no wire payload documented for 1.1** | — |
| 65 | `0x41` | EventPacket | S2C | [09](09-player-input-and-ui.md) |
| 66 | `0x42` | SpawnExperienceOrbPacket | S2C | [07](07-entities.md) |
| 67 | `0x43` | ClientboundMapItemDataPacket | S2C | [09](09-player-input-and-ui.md) |
| 68 | `0x44` | MapInfoRequestPacket | C2S | [09](09-player-input-and-ui.md) |
| 69 | `0x45` | RequestChunkRadiusPacket | C2S | [09](09-player-input-and-ui.md) |
| 70 | `0x46` | ChunkRadiusUpdatedPacket | S2C | [09](09-player-input-and-ui.md) |
| 71 | `0x47` | ItemFrameDropItemPacket | C2S | [08](08-inventory-and-items.md) |
| 72 | `0x48` | ReplaceItemInSlotPacket | S2C | [08](08-inventory-and-items.md) |
| 73 | `0x49` | GameRulesChangedPacket | S2C | [06](06-spawn-and-world.md) |
| 74 | `0x4a` | CameraPacket | S2C | [06](06-spawn-and-world.md) |
| 75 | `0x4b` | AddItemPacket | S2C | [07](07-entities.md) |
| 76 | `0x4c` | BossEventPacket | Both | [07](07-entities.md) |
| 77 | `0x4d` | ShowCreditsPacket | S2C | [06](06-spawn-and-world.md) |
| 78 | `0x4e` | AvailableCommandsPacket | S2C | [09](09-player-input-and-ui.md) |
| 79 | `0x4f` | CommandStepPacket | C2S | [09](09-player-input-and-ui.md) |
| 80 | `0x50` | CommandBlockUpdatePacket | C2S | [09](09-player-input-and-ui.md) |
| 81 | `0x51` | UpdateTradePacket | S2C | [09](09-player-input-and-ui.md) |
| 82 | `0x52` | UpdateEquipPacket | S2C | [09](09-player-input-and-ui.md) |
| 83 | `0x53` | ResourcePackDataInfoPacket | S2C | [05](05-login-and-session.md) |
| 84 | `0x54` | ResourcePackChunkDataPacket | S2C | [05](05-login-and-session.md) |
| 85 | `0x55` | ResourcePackChunkRequestPacket | C2S | [05](05-login-and-session.md) |
| 86 | `0x56` | TransferPacket | S2C | [05](05-login-and-session.md) |
| 87 | `0x57` | PlaySoundPacket | S2C | [09](09-player-input-and-ui.md) |
| 88 | `0x58` | StopSoundPacket | S2C | [09](09-player-input-and-ui.md) |
| 89 | `0x59` | SetTitlePacket | S2C | [09](09-player-input-and-ui.md) |
| 90 | `0x5a` | AddBehaviorTreePacket | reserved — defined in the registry, **no wire payload documented for 1.1** | — |
| 91 | `0x5b` | StructureBlockUpdatePacket | reserved — defined in the registry, **no wire payload documented for 1.1** | — |
| 92 | `0x5c` | ShowStoreOfferPacket | reserved — defined in the registry, **no wire payload documented for 1.1** | — |
| 93 | `0x5d` | PurchaseReceiptPacket | reserved — defined in the registry, **no wire payload documented for 1.1** | — |
| 254 | `0xFE` | BatchPacket | Both (wrapper) | [02](02-framing-batching.md) |

## Notes on directions

* Packets that originate from the vanilla client are the session/login set (`0x01` Login, `0x04` ClientToServerHandshake, `0x08` ResourcePackClientResponse, `0x55` ResourcePackChunkRequest) plus gameplay reports: `0x13` MovePlayer, `0x12` MoveEntity (driving a vehicle), `0x19` LevelSoundEvent, `0x1c` EntityEvent, `0x1f` MobEquipment, `0x20` MobArmorEquipment, `0x21` Interact, `0x23` UseItem, `0x24` PlayerAction, `0x15` RemoveBlock, `0x22` BlockPickRequest, `0x25` EntityFall, `0x14` RiderJump, `0x2c` Animate, `0x39` PlayerInput, `0x2e` DropItem, `0x31` ContainerClose, `0x32` ContainerSetSlot, `0x36` CraftingEvent, `0x38` BlockEntityData, `0x37` AdventureSettings, `0x09` Text, `0x4f` CommandStep, `0x44` MapInfoRequest, `0x45` RequestChunkRadius, `0x47` ItemFrameDropItem. Everything else in the registry is server→client.
* Both directions are documented for every packet regardless, since proxies and third-party clients need both.
* `InventoryActionPacket` (`0x2f`) with actions *give item* (0) / *enchant item* (2) is a compatibility leftover from the pre-1.0 era; stock 1.1.x clients drive inventory changes through `ContainerSetSlotPacket` instead (see [08](08-inventory-and-items.md)).
