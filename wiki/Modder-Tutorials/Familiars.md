---
title: "Familiars"
order: 4
published: true
draft: false
---

# Familiars

Familiars are cosmetic NPCs that follow a player around. Each player can have at most one familiar out at a time,
summoning a different one replaces the current one, and using the same summon item again dismisses it. The framework
forces every familiar to be **invulnerable** (cannot be killed) and **intangible** (player attacks and movement pass
straight through the hitbox). Familiars despawn when their owner leaves the world or dies, and respawn automatically
when the owner enters another world (instance entry, world transition, reconnect) or revives after death.

Familiars are allowed everywhere by default. Server owners can block them per world through the `FamiliarsWorldBlacklist`
list in the plugin's `FrameworkConfig.json` (at `mods/MAJOR76_MajorDungeons/FrameworkConfig.json` on the server), or
from the in-game config page. Entries are world name patterns and support `*` wildcards, for example `*Dungeon_Of_Fear*`.
A blacklisted world blocks both summoning and the automatic respawn on entry.

## What You Author

A familiar needs three files, all sharing the familiar's `<id>` for the role and asset:

1. A **role** at `Server/NPC/Roles/<id>.json`, the visual model and motion controller (`Walk` or `Fly`).
2. A **familiar asset** at `Server/NPC/Familiars/<id>.json`, follow tuning (leash distance, hover height, teleport offset).
3. A **summon item** at `Server/Item/Items/<...>.json` with a `SpawnFamiliar` interaction. The item is not consumed when used.

The familiar asset's existence is what flags the role as a familiar.

## Step 1 - Create the NPC Role

Standard Hytale NPC role file. The role owns:

- The visual model (`Appearance`)
- The motion controller (`MotionControllerList`) and its physics caps (`MaxWalkSpeed` / `MaxHorizontalSpeed`, `Gravity`, `MaxFallSpeed`, `MaxClimbHeight`, `MaxTurnSpeed`, etc.)
- Animation transitions between idle and motion states

The role does **not** need an AI behavior tree. No `Player` sensor, no `Seek` BodyMotion, an empty `Instructions: []` block is the right setup, the framework drives the follow logic directly.

`Server/NPC/Roles/MyFamiliar.json`

```json
{
  "Type": "Generic",
  "StartState": "Idle",
  "DefaultSubState": "Default",
  "Appearance": "Wraith_Lantern",
  "Invulnerable": true,
  "MaxHealth": 50,
  "MotionControllerList": [
    {
      "Type": "Fly",
      "MaxHorizontalSpeed": 12,
      "MaxClimbSpeed": 8,
      "MaxSinkSpeed": 8,
      "MaxFallSpeed": 20,
      "MaxClimbAngle": 60,
      "MaxSinkAngle": 60,
      "Acceleration": 6,
      "Deceleration": 6,
      "Gravity": 1,
      "MaxTurnSpeed": 360,
      "MaxRollAngle": 30,
      "MaxRollSpeed": 180,
      "RollDamping": 0.9,
      "MinHeightOverGround": 0,
      "MaxHeightOverGround": 256,
      "DesiredAltitudeWeight": 0,
      "AutoLevel": true
    }
  ],
  "Instructions": [
    {
      "Instructions": [
        {
          "Sensor": {
            "Type": "State",
            "State": "Idle"
          },
          "Instructions": [
            {
              "Sensor": {
                "Type": "State",
                "State": ".Default"
              },
              "Instructions": []
            }
          ]
        }
      ]
    }
  ],
  "NameTranslationKey": "MyMod.npcRoles.MyFamiliar.name"
}
```

The empty `Instructions` scaffold defines the `Idle` and `Idle.Default` states referenced by `StartState` and `DefaultSubState` (the role validator requires every named state to be defined somewhere) with empty inner instruction lists so no AI behavior runs. The framework drives the follow each tick.

Familiars are forced **invulnerable** and **intangible** regardless of role-file declarations, attacks and player movement pass through them, they cannot be killed.

## Step 2 - Create the Familiar Asset

`Server/NPC/Familiars/MyFamiliar.json`

```json
{
  "MaxFollowDistance": 20,
  "TeleportOffset": {
    "X": 1.5,
    "Y": 0.5,
    "Z": 1.5
  },
  "HoverHeightOffset": 0.5
}
```

Or with per-axis control:

```json
{
  "MaxFollowDistance": {
    "Horizontal": 30,
    "Vertical": 10
  },
  "TeleportOffset": {
    "X": 1.5,
    "Y": 0.5,
    "Z": 1.5
  },
  "HoverHeightOffset": 0.5
}
```

| Field                 | Default           | Description                                                                                                                                                                                                                                                                                                                                   |
|-----------------------|-------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `MaxFollowDistance`   | `20`              | Leash trigger distance in blocks. Accepts either a single number (applied to both horizontal XZ and vertical Y axes) or an object `{ "Horizontal": ..., "Vertical": ... }` for per-axis control. `{ "X": ..., "Y": ... }` works as an alias. When either axis separation exceeds its limit the framework teleports the familiar to the owner. |
| `TeleportOffset`      | `(1.5, 0.5, 1.5)` | Offset relative to the owner where the teleport lands. The Y default of `0.5` puts the familiar at roughly head height (the player's transform sits at the feet).                                                                                                                                                                             |
| `HoverHeightOffset`   | `0`               | Vertical offset above the owner's feet the follow system aims for. Default `0` keeps a ground-walking pet at foot level so the field can be omitted entirely on Walk familiars. Set explicitly on flying familiars (e.g. `0.5` for shoulder/head height, `1.5` for overhead).                                                                 |
| `FollowRadius`        | `1.5`             | Horizontal dead-zone radius around the owner where the familiar stops chasing and switches to its idle animation. Prevents clipping/oscillation.                                                                                                                                                                                              |
| `FollowGain`          | `2.5`             | How aggressively the speed ramps up past the dead-zone. Higher = full speed sooner, lower = gradual acceleration.                                                                                                                                                                                                                             |
| `FollowSpeedRatio`    | `1.0`             | **Walk path only.** Upper bound on the `0..1` relative speed sent to the motion controller. `1.0` = the role's full max speed, `0.5` = half. Use for a "lazy" pet.                                                                                                                                                                            |
| `MaxHorizontalSpeed`  | `8`               | Hard cap in blocks/sec on the per-tick horizontal velocity for `Fly` familiars and for `Walk` familiars while the owner is flying/gliding. The role's own speed cap is still the absolute ceiling. |
| `MaxVerticalSpeed`    | `6`               | Hard cap in blocks/sec on the per-tick vertical velocity (same conditions as `MaxHorizontalSpeed`). |
| `GravityCompensation` | `1.5`             | Additive Y velocity applied while airborne to soften gravity drag so the pet keeps up with a flying owner. For `Fly` familiars, roughly match the role's `Gravity` to hover, set lower to drift down when idle. For `Walk` familiars while the owner flies or glides, a value around `5` lets the pet hover with the player and slowly settle when the player lands. |

The two-axis form lets you bias the leash toward whichever axis is more likely to overshoot. A flying familiar that
handles big drops can use `{ "Horizontal": 20, "Vertical": 50 }`, a fast ground pet that can't keep up across vertical
gaps can use `{ "Horizontal": 30, "Vertical": 8 }` so it teleports up sooner.

All fields are optional. An empty `{}` body is valid and uses the defaults above. Values reload live, editing a
familiar JSON and triggering an asset reload applies to every existing familiar within ~1 tick, no respawn required. (
Role-file values like `MotionControllerList` only apply to newly spawned familiars since the motion controller is
constructed once per entity.)

## Step 3 - Create the Summon Item

The item uses the `SpawnFamiliar` interaction in its `Secondary` (right-click) slot:

`Server/Item/Items/MyFamiliarItem.json`

```json
{
  "TranslationProperties": {
    "Name": "MyMod.items.MyFamiliarItem.name",
    "Description": "MyMod.items.MyFamiliarItem.description"
  },
  "Quality": "Epic",
  "PlayerAnimationsId": "Item",
  "Interactions": {
    "Secondary": {
      "Interactions": [
        {
          "Type": "SpawnFamiliar",
          "EntityId": "MyFamiliar",
          "SpawnParticleId": "Magic_Signature_Ready",
          "SpawnSoundId": "SFX_Cult_Magic_Cast",
          "DespawnParticleId": "Magic_Signature_Ready",
          "DespawnSoundId": "SFX_Cult_Magic_Cast",
          "Effects": {
            "ItemAnimationId": "Interact"
          }
        }
      ]
    }
  },
  "Model": "Items/Consumables/Food/Item.blockymodel",
  "Texture": "Items/Consumables/Food/Item_White_Texture.png",
  "Icon": "Icons/ItemsGenerated/Food_Item_White.png"
}
```

| Interaction Field   | Description                                                                                                                                                                                                                                                                                                                                                                                                         |
|---------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `EntityId`          | NPC role id, must match both the role file and the familiar asset id.                                                                                                                                                                                                                                                                                                                                               |
| `SpawnOffset`       | Optional per-item override of the offset relative to the player when no block is targeted. When omitted (the recommended default), the asset's `TeleportOffset` is used instead, so the pet spawns at the same place the leash teleport would put it, never directly on top of the player. The same offset is also used by the rejoin path so the pet doesn't materialize inside the player when they log back in. |
| `SpawnParticleId`   | Optional particle system id played at the familiar's spawn position. The standard `Effects.Particles` block on the interaction is anchored to the player's hand (`TargetEntityPart` only supports `Self`/`Entity`/`PrimaryItem`/`SecondaryItem`) so it cannot reach the targeted block, this field plays the particle at the actual spawn location instead.                                                         |
| `SpawnSoundId`      | Optional sound event id played at the familiar's spawn position via 3D positional audio. Same reasoning as `SpawnParticleId`, `Effects.WorldSoundEventId` is centered on the player rather than the spawn point.                                                                                                                                                                                                   |
| `DespawnParticleId` | Optional particle system id played at the **familiar's current world position** when the player dismisses (right-clicks the same item again). The replace flow (right-clicking a different familiar item while one is already out) intentionally suppresses despawn FX so only the new familiar's spawn effects play, doubling up would be visual noise.                                                           |
| `DespawnSoundId`    | Optional sound event id played at the familiar's current world position on dismiss. Same replace-flow suppression as `DespawnParticleId`.                                                                                                                                                                                                                                                                           |

> Use the four fields above (not `Effects.Particles` / `Effects.WorldSoundEventId`) for spawn/despawn FX, the standard `Effects` block always anchors to the player or held item and cannot reach the spawn location.

**Spawn position**: when the player right-clicks while looking at a block, the familiar spawns on top of that block (centered on the block face). When right-clicking in open air, or rejoining the world with a previously summoned familiar, it spawns at the player's position plus the asset's `TeleportOffset` (or the item's `SpawnOffset` if explicitly set).

**Use animation**: add `"Effects": { "ItemAnimationId": "Interact" }` to the interaction (as shown above) to trigger a
soft pickup-style arm motion on right-click, the same animation used for interacting with crops and other gentle block
actions. Other valid ids defined by the standard `Item` animation set include `Throw` (overhand toss, more aggressive),
`SwingRight`, `SwingLeft`, `SwingDown`, `Build`, and `Consume`.

The item is **not** consumed by `SpawnFamiliar`. Use the same item again to dismiss the familiar, use a different summon
item to swap to a new familiar.

## Behavior Summary

- **Summon**: right-click the item, the familiar spawns on the targeted block when looking at one, otherwise next to the player.
- **Dismiss**: right-click the same item again to despawn it.
- **Swap**: right-click a different familiar item to replace the current one.
- **Leash teleport**: when the familiar drifts further than `MaxFollowDistance` on either axis (XZ or Y), it teleports back to the owner using `TeleportOffset`.
- **Walking pets** follow the owner along the ground using the role's motion controller, so the role's `MaxWalkSpeed`, `Gravity`, step-up height, and idle/walk animation transitions all apply. They aim for `player.y + HoverHeightOffset` and stop chasing inside a `FollowRadius` dead-zone.
- **Walking pets when the owner flies or glides**: the pet enters a levitation mode and hovers next to the airborne owner instead of pancaking to the ground. Set `GravityCompensation` to around `5` to let it slowly settle when the owner lands.
- **Flying pets** push velocity directly on each tick, capped by the asset's `MaxHorizontalSpeed`, `MaxVerticalSpeed`, and biased by `GravityCompensation` while airborne so the pet can hover at altitude. Works in every game mode.
- **Facing direction**: the body's yaw lerps toward the player at the role's `MaxTurnSpeed` / `MaxRotationSpeed`. Only yaw is touched, never pitch or roll.
- **Logout / login**: the familiar despawns on leave, respawns on join.
- **World change**: despawns on leave, respawns on enter.
- **Death / respawn**: despawns on death, respawns when the player revives.
- **Damage / hitbox**: familiars cannot die and player attacks/movement pass through them.

## Examples

Two complete file sets showing how the role-file motion controller and the asset tuning fields combine to produce flying
vs walking pets. The summon item is identical in both cases, only the role and asset differ.

### Flying Familiar

A floating lantern that hovers around the player's head, follows them through the air, and ignores gravity.

`Server/NPC/Roles/MyFlyingFamiliar.json`

```json
{
  "Type": "Generic",
  "StartState": "Idle",
  "DefaultSubState": "Default",
  "Appearance": "Wraith_Lantern",
  "Invulnerable": true,
  "MaxHealth": 50,
  "MotionControllerList": [
    {
      "Type": "Fly",
      "MaxHorizontalSpeed": 12,
      "MaxClimbSpeed": 8,
      "MaxSinkSpeed": 8,
      "MaxFallSpeed": 20,
      "MaxClimbAngle": 60,
      "MaxSinkAngle": 60,
      "Acceleration": 6,
      "Deceleration": 6,
      "Gravity": 1,
      "MaxTurnSpeed": 360,
      "MaxRollAngle": 30,
      "MaxRollSpeed": 180,
      "RollDamping": 0.9,
      "MinHeightOverGround": 0,
      "MaxHeightOverGround": 256,
      "DesiredAltitudeWeight": 0,
      "AutoLevel": true
    }
  ],
  "Instructions": [
    {
      "Instructions": [
        {
          "Sensor": {
            "Type": "State",
            "State": "Idle"
          },
          "Instructions": [
            {
              "Sensor": {
                "Type": "State",
                "State": ".Default"
              },
              "Instructions": []
            }
          ]
        }
      ]
    }
  ],
  "NameTranslationKey": "MyMod.npcRoles.MyFlyingFamiliar.name"
}
```

`Server/NPC/Familiars/MyFlyingFamiliar.json`

```json
{
  "MaxFollowDistance": 20,
  "TeleportOffset": {
    "X": 1.5,
    "Y": 0.5,
    "Z": 1.5
  },
  "HoverHeightOffset": 0.5,
  "GravityCompensation": 1.5
}
```

Key choices:

- **`Fly` motion controller** handles 3D motion.
- **`HoverHeightOffset: 0.5`** parks the lantern at the player's head height (player transform sits at the feet, `+0.5` lands around eye level).
- **`TeleportOffset.Y: 0.5`** matches `HoverHeightOffset` so leash-teleports land at the same altitude the follow path targets.
- **`GravityCompensation: 1.5`** counters the role's `Gravity: 1` so the lantern holds its altitude instead of sinking. Roughly match the role's gravity, set lower to drift down when idle.
- **`MinHeightOverGround: 0` / `MaxHeightOverGround: 256`** keeps the vertical envelope wide so the controller does not fight the altitude target.

### Walking Familiar

A ground-bound pet that runs along behind the player, falls when the ground drops out, and never tries to lift off.

`Server/NPC/Roles/MyWalkingFamiliar.json`

```json
{
  "Type": "Generic",
  "StartState": "Idle",
  "DefaultSubState": "Default",
  "Appearance": "Test_Pet_Lantern",
  "Invulnerable": true,
  "MaxHealth": 50,
  "MotionControllerList": [
    {
      "Type": "Walk",
      "MaxWalkSpeed": 6,
      "Gravity": 10,
      "MaxFallSpeed": 12,
      "Acceleration": 6,
      "MaxRotationSpeed": 360,
      "MaxClimbHeight": 1.1
    }
  ],
  "Instructions": [
    {
      "Instructions": [
        {
          "Sensor": {
            "Type": "State",
            "State": "Idle"
          },
          "Instructions": [
            {
              "Sensor": {
                "Type": "State",
                "State": ".Default"
              },
              "Instructions": []
            }
          ]
        }
      ]
    }
  ],
  "NameTranslationKey": "MyMod.npcRoles.MyWalkingFamiliar.name"
}
```

`Server/NPC/Familiars/MyWalkingFamiliar.json`

```json
{
  "MaxFollowDistance": {
    "Horizontal": 20,
    "Vertical": 8
  },
  "TeleportOffset": {
    "X": 1.5,
    "Y": 0,
    "Z": 1.5
  },
  "GravityCompensation": 5
}
```

Key choices:

- **`Walk` motion controller** with vanilla-style `Gravity: 10` so the pet falls and lands on terrain like a normal NPC.
- **No `HoverHeightOffset`** needed, the default (`0`) targets foot level. Only flying familiars need to set this.
- **`TeleportOffset.Y: 0`** matches the foot-level target so leash-teleports drop the pet next to the player rather than above.
- **`MaxClimbHeight: 1.1`** in the role lets the pet step up single blocks, raise it for chunky terrain.
- **Asymmetric `MaxFollowDistance`** (`Horizontal: 20, Vertical: 8`) so the pet teleports up sooner when the player jumps onto a ledge.
- **`GravityCompensation: 5`** lets the pet hover alongside the player when the owner flies or glides, then settle gently when the player lands. Lower values drop faster, higher values resist gravity more.

If the pet is too fast or too slow, change the role's `MaxWalkSpeed`. If you want it to move slower than its role
allows (e.g., a "lazy" walking pet), add `"FollowSpeedRatio": 0.5` to the asset to clamp the relative speed at half max.

## Built-in Familiars

The mod ships several reference familiars you can use directly or copy as a starting point:

| Id                          | Item                             | Style   | Notes                                                                                                           |
|-----------------------------|----------------------------------|---------|-----------------------------------------------------------------------------------------------------------------|
| `MD_Familiar_Hawk_Lantern`  | `MD_Familiar_Item_Hawk_Lantern`  | Flying  | Living lantern that hovers near the player. Demonstrates the `Fly` motion controller.                          |
| `MD_Familiar_Frog`          | `MD_Familiar_Item_Frog`          | Walking | Hopping frog that runs behind the player on the ground. Demonstrates `Walk` and an asymmetric leash.           |
| `MD_Familiar_Frog_Red`      | `MD_Familiar_Item_Frog_Red`      | Walking | Red Dragon Frog variant. Exclusive drop from the Hardcore Dungeon of Fear I bag.                               |
| `MD_Familiar_Frog_Purple`   | `MD_Familiar_Item_Frog_Purple`   | Walking | Purple Wizard Frog variant. Exclusive drop from the Hardcore Dungeon of Fear III bag.                          |
| `MD_Familiar_Frog_Headphones` | `MD_Familiar_Item_Frog_Headphones` | Walking | Frog with headphones. Drops from Hardcore chests.                                                         |
| `MD_Familiar_Bat`           | `MD_Familiar_Item_Bat`           | Flying  | Bat familiar.                                                                                                   |
| `MD_Familiar_Bat_Vampire`   | `MD_Familiar_Item_Bat_Vampire`   | Flying  | Vampire bat. Exclusive drop from the Hardcore Dungeon of Fear II bag.                                          |
| `MD_Familiar_Boar`          | `MD_Familiar_Item_Boar`          | Walking | Boar familiar.                                                                                                  |
| `MD_Familiar_Cat_Black`     | `MD_Familiar_Item_Cat_Black`     | Walking | Black cat familiar.                                                                                             |
| `MD_Familiar_Cat_Orange`    | `MD_Familiar_Item_Cat_Orange`    | Walking | Orange cat familiar.                                                                                            |
| `MD_Familiar_Crab`          | `MD_Familiar_Item_Crab`          | Walking | Crab familiar.                                                                                                  |
| `MD_Familiar_Mimic`         | `MD_Familiar_Item_Mimic`         | Walking | Mimic familiar. Drops from Hardcore chests.                                                                    |

All familiar assets live under `Server/NPC/Familiars/`, roles under `Server/NPC/Roles/`, and summon items under
`Server/Item/Items/Familiars/`. Diff them against the [Examples](#examples) above to see how the flying and walking
patterns map to real assets.

## Summary of Files

```
Server/
├── NPC/
│   ├── Roles/
│   │   └── MyFamiliar.json        ← role (motion controller, appearance)
│   └── Familiars/
│       └── MyFamiliar.json        ← familiar asset (follow tuning)
└── Item/
    └── Items/
        └── MyFamiliarItem.json    ← summon item
```

The id `MyFamiliar` is shared by the role file, the familiar asset, and the `EntityId` field on the item interaction.
