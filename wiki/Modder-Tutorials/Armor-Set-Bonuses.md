---
title: "Armor Set Bonuses"
order: 12
published: true
draft: false
---

# Armor Set Bonuses

Set bonuses let you tie a status effect to wearing a specific group of armor pieces, optionally gated on world conditions like time of day or moon phase. While the bonus is active the framework applies the effect you point at, when the conditions stop holding it removes the effect again. Everything lives in JSON.

The most common use is a damage boost, but the effect can do anything an `EntityEffect` can do: tint the player, attach particles, modify stats, regen, screen effects, you name it.

## How It Works

The framework scans each player's worn armor once per second. If the player wears the items declared on a set bonus AND any world conditions on the bonus are met, the bonus's `EntityEffect` is applied. If a condition stops being true (player removes a piece, sunrise, moon phase changes) the effect is removed.

The bonus tracks itself per-player, so the apply/remove only fires on transitions, not every tick. If the player dies and respawns the effect is re-applied automatically.

## What You Author

A set bonus needs two files:

1. An **EntityEffect** at `Server/Entity/Effects/Status/<effect-id>.json`, the actual buff (tint, particles, stat modifiers, etc.).
2. A **Set Bonus** at `Server/SetBonuses/<bonus-id>.json`, the rules for when the effect applies.

## Step 1 - Create the Entity Effect

This is a standard Hytale `EntityEffect`. The framework just calls `addEffect` with it, so any field the base game supports works here.

`Server/Entity/Effects/Status/MyMod_Mark_Of_The_Moon.json`

```json
{
  "Infinite": true,
  "Debuff": false,
  "StatusEffectIcon": "UI/StatusEffects/Burn.png",
  "ApplicationEffects": {
    "EntityTopTint": "#c79bff",
    "EntityBottomTint": "#7a3fbf"
  },
  "RawStatModifiers": {
    "DungeonFramework_OutgoingDamageMultiplier": [
      { "Amount": 0.5, "CalculationType": "Additive" }
    ]
  }
}
```

| Field | Description |
|-------|-------------|
| `Infinite` | Set `true` so the effect never expires on its own. The framework decides when to remove it based on the bonus conditions. |
| `Debuff` | `false` routes the HUD icon to the buff (top) tray, `true` to the debuff tray. |
| `StatusEffectIcon` | Path to the icon shown next to the hotbar. Vanilla paths like `UI/StatusEffects/Burn.png` work, or ship your own under `Common/UI/StatusEffects/`. |
| `ApplicationEffects` | Standard Hytale block. `EntityTopTint` / `EntityBottomTint` recolor the player, `Particles` attach VFX, etc. |
| `RawStatModifiers` | Map of stat id → modifier list. See the [Damage Boost](#damage-boost) section below. |

## Step 2 - Create the Set Bonus

`Server/SetBonuses/MyMod_Mark_Of_The_Moon.json`

```json
{
  "RequiredItems": [
    ["MyMod_Helmet"],
    ["MyMod_Chestplate", "MyMod_Chestplate_Cape"],
    ["MyMod_Leggings"],
    ["MyMod_Boots"]
  ],
  "EffectId": "MyMod_Mark_Of_The_Moon",
  "RequireNight": true,
  "MoonPhases": [0]
}
```

| Field | Description |
|-------|-------------|
| `RequiredItems` | List of OR-groups. Each inner array is a single armor slot, and the player must have **at least one** of the listed item ids equipped to satisfy that group. All groups must be satisfied together. The example needs the helmet AND boots AND leggings AND **either** of the two chestplate variants. |
| `EffectId` | The `EntityEffect` id from Step 1. |
| `RequireNight` | `true` requires the world's time of day to be in the night portion. Omit or set `false` to ignore time of day. |
| `MoonPhases` | Optional list of allowed moon phase indexes (0-indexed, as defined by Hytale's moon cycle). Omit or use an empty array for any phase. |

The two-level `RequiredItems` shape is what lets you support visual variants of the same slot (a chestplate with and without a cape, a head piece with two skin colors, etc.) without duplicating the bonus.

## Damage Boost

The framework ships a generic outgoing-damage-multiplier stat at `DungeonFramework_OutgoingDamageMultiplier` with a base value of `1.0`. Any effect (whether from a set bonus, a potion, an ability, anything) that adds a modifier to this stat will boost the wearer's outgoing damage while it's active. The stat is read once per damage event applied to the source, so no per-effect Java is needed.

In the example above, `+0.5 Additive` makes the stat's effective value `1.5`, so all outgoing damage from the wearer is multiplied by 1.5.

| Amount | CalculationType | Result | Effect |
|--------|-----------------|--------|--------|
| `+0.5` | `Additive` | base 1.0 + 0.5 = 1.5 | +50% damage |
| `+1.0` | `Additive` | base 1.0 + 1.0 = 2.0 | x2 damage |
| `-0.3` | `Additive` | base 1.0 - 0.3 = 0.7 | -30% damage (debuff) |
| `1.5` | `Multiplicative` | base 1.0 \* 1.5 = 1.5 | +50% damage |

**Stacking note:** if two effects both add modifiers to this stat, their amounts sum before being applied. Two `+0.5 Additive` modifiers stack to `+1.0` (x2 damage). Two `1.5 Multiplicative` modifiers stack to a single `x3.0` multiplication (the amounts sum first, then multiply once), which is rarely what you want. Stick to `Additive` unless you have a specific reason to compound.

## Examples

A simple two-piece bonus that always applies whenever both pieces are worn, no world conditions:

```json
{
  "RequiredItems": [
    ["MyMod_Ring_Of_Vigor"],
    ["MyMod_Amulet_Of_Vigor"]
  ],
  "EffectId": "MyMod_Vigor_Buff"
}
```

A full four-piece set that only triggers during a new moon at night:

```json
{
  "RequiredItems": [
    ["MyMod_Shadow_Helm"],
    ["MyMod_Shadow_Chest"],
    ["MyMod_Shadow_Legs"],
    ["MyMod_Shadow_Boots"]
  ],
  "EffectId": "MyMod_Shadowstep",
  "RequireNight": true,
  "MoonPhases": [4]
}
```

If you want the bonus to apply on a specific moon phase regardless of vanilla time-of-day drift, see the [Instance Config](./instance-config) tutorial for the gameplay-config technique that pins an instance world to a single moon phase forever.

## Behavior Summary

- **Tick rate**: once per second per player who has armor and an effect controller. Cheap, no work done when nothing has changed.
- **Apply**: the first time all conditions hold, the effect is added with the asset's full settings (Infinite, OverlapBehavior, etc.).
- **Remove**: when any condition stops holding (armor change, time of day, moon phase), the previously-applied effect is removed.
- **Death**: on respawn the framework resets its tracking so the effect is re-applied on the next tick if conditions still hold.
- **Multiple bonuses**: each player can have at most one active set bonus. If two configured bonuses match simultaneously, the first one found wins. Make sure your `RequiredItems` are specific enough to avoid overlap.

## Summary of Files

```
Server/
├── Entity/
│   └── Effects/
│       └── Status/
│           └── MyMod_Mark_Of_The_Moon.json
└── SetBonuses/
    └── MyMod_Mark_Of_The_Moon.json
```

The two files share a logical name but live in different directories. The `EffectId` field on the set bonus is what links them, the file names don't have to match.
