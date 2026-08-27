---
title: "Modder Tutorials"
order: 14
published: true
draft: false
---

# Modder Tutorials

This section is for other modders who want to use the Major Dungeons framework in their own asset packs or plugins. Every content feature the mod adds is designed to be used by anyone, purely through JSON data files, no Java code required. The one exception is the [Plugin API](./plugin-api), which is there for Java plugins that want to react to dungeon runs.

## How to Add Major Dungeons as a Dependency

For most features, you'll want to add MajorDungeons as a dependency so that players make sure to install it for the features to work. Some of the features only kick in if MajorDungeons is installed (like Boss Bars), making it optional. In your mod's `manifest.json`, add `MAJOR76:MajorDungeons` to your `Dependencies` or `OptionalDependencies` block:

```json
{
  "Group": "MyGroup",
  "Name": "MyMod",
  "Version": "1.0.0",
  "Dependencies": {
    "MAJOR76:MajorDungeons": "*"
  }
}
```

## What is Covered

Each page in this section walks through one feature from scratch, using the bare minimum JSON needed to get it working:

[//]: # (- [Basic Dungeon Setup]&#40;./basic-dungeon-setup&#41; - instances, portal types, and portal keys)
- [Bosses](./bosses) - boss bars and kill rewards
- [Locked Doors and Keys](./locked-doors-and-keys) - blocks that require a key item to open
- [Loot Packs](./loot-packs) - items that open and roll drop lists
- [Summonable Familiars](./familiars) - items that summon cosmetic pets that follow their summoner around
- [Summonable Mounts](./summonable-mounts) - items that summon and mount an NPC
- [Mimic Blocks](./mimic-blocks) - blocks, like treasure chests, that spawn NPCs when used
- [Tabbed Barter Shops](./tabbed-barter-shops) - merchants with multiple tab categories
- [Mutating Barter Shops](./mutating-barter-shops) - merchants with randomly generated trades on server start
- [Armor Set Bonuses](./armor-set-bonuses) - status effects (damage boosts, tints, particles) that apply while a specific armor set is worn, optionally gated on time of day and moon phase
- [Instance Objectives](./instance-objectives) - contract-style "Devil's Deal" objectives that activate when a player enters an instance carrying a specific item
- [Instance Config](./instance-config) - custom HUD plus difficulty/rule modes (time limit, PvP toggle, mob/boss scaling, drop list and block spawner swaps, death overrides with item loss, server-owner editable values)
- [Plugin API](./plugin-api) - Java-side hooks for querying dungeon worlds and their active difficulty, plus listeners for entering, exiting, completing a run, and boss kills

[//]: # (- [Lore Pages]&#40;./lore-pages&#41; - items that teach readable lore chapters)
