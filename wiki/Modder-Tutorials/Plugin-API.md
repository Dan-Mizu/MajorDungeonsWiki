---
title: "Plugin API"
order: 13
published: true
draft: false
---

# Plugin API

Every other page in this section is pure JSON. This one is not. The Plugin API is for modders writing a **Java plugin** who want to react to dungeon runs, or line their own difficulty up with ours, without parsing world names or guessing at our internals.

If you only want to add content (bosses, loot, familiars, shops), you do not need any of this.

## Getting the API

Add `MAJOR76:MajorDungeons` to your `manifest.json` as shown on the [section index](./), then add the Major Dungeons jar to your build as a `compileOnly` dependency so you compile against the interfaces without shading them in.

Everything is reached through one entry point:

```java
import com.danbagh.dungeonframework.api.DungeonFrameworkAPI;
import com.danbagh.dungeonframework.api.DungeonFrameworkAPIProvider;

DungeonFrameworkAPI api = DungeonFrameworkAPIProvider.get();
if (api == null) return; // Major Dungeons is not installed or not started yet
```

`get()` returns `null` when the framework is absent, so declare us as an **optional** dependency and null-check, and your mod keeps working on servers that do not run Major Dungeons. Call it when you need it rather than caching it in a static, the binding is cleared on plugin shutdown so a stale reference will not survive a reload.

## Queries

All four take a world name and answer for that world only.

| Method | Returns |
| --- | --- |
| `isDungeonWorld(String worldName)` | `true` when the world is a dungeon instance this framework tracks. Normal survival worlds, and instances created by other systems, return `false`. |
| `getDungeonId(String worldName)` | The dungeon id behind the world, for example `"Dungeon_Of_Fear_III"` for `instance-Dungeon_Of_Fear_III-<uuid>`. `null` when the world is not one of ours. |
| `getDungeonMode(String worldName)` | The mode picked at the portal, for example `"Hardcore"`, or `"Default"` for a normal run. `null` when the world is not one of ours. Compare against `DungeonFrameworkAPI.DEFAULT_MODE` rather than hardcoding the string. |
| `getActiveModifiers(String worldName)` | A [`DungeonModifiersView`](#dungeonmodifiersview) snapshot of the scaling this run is using. `null` when the world is not one of ours. |

Prefer `getActiveModifiers` over `getDungeonMode` when you want to match our difficulty **numerically**. Branching on a mode name breaks the moment a server owner renames a mode or adds their own, the multipliers stay meaningful either way.

```java
DungeonModifiersView mods = api.getActiveModifiers(worldName);
if (mods != null && !mods.isUnscaled()) {
    myMobHealth *= mods.mobHealthMultiplier();
}
```

### DungeonModifiersView

An immutable snapshot taken at call time. It is never updated in place, so call again after a mode change rather than holding onto one.

| Field | Meaning |
| --- | --- |
| `dungeonId()` | Tracked instance id, e.g. `"Dungeon_Of_Fear_III"`. |
| `modeName()` | Active mode, `"Default"` or e.g. `"Hardcore"`. |
| `mobHealthMultiplier()` | Scales `MaxHealth` on regular NPCs at spawn. `1.0` means unchanged. |
| `mobDamageMultiplier()` | Scales outgoing damage from regular NPCs. `1.0` means unchanged. |
| `bossHealthMultiplier()` | Scales `MaxHealth` on entities tagged as bosses. |
| `bossDamageMultiplier()` | Scales outgoing damage from bosses. |
| `timeLimitSeconds()` | Run timer in seconds, `null` when the portal key supplies it instead. |
| `pvpEnabled()` | PvP state for the instance, `null` when inherited from the main world. |

`isUnscaled()` is a cheap check that returns `true` when all four multipliers are `1.0`, useful for skipping work on a default run.

See [Instance Config](./instance-config) for how server owners define these modes in the first place.

## Events

Listeners are registered and removed in pairs. Removing a listener that was never added is a no-op.

```java
Consumer<DungeonEnteredEvent> onEnter = event -> {
    LOGGER.atInfo().log("%s entered %s on %s", event.playerUuid(), event.dungeonId(), event.modeName());
};
api.addDungeonEnteredListener(onEnter);

// later, on your own plugin shutdown
api.removeDungeonEnteredListener(onEnter);
```

| Event | Fires when |
| --- | --- |
| `DungeonEnteredEvent` | A player is **fully spawned** inside a tracked dungeon. The world, its mode, and the player's entity all exist by this point, so it is safe to read state or apply per-player changes from the listener. |
| `DungeonExitedEvent` | A player leaves a tracked dungeon, for **any** reason. |
| `DungeonCompletedEvent` | A player finishes a run through the exit portal. Always followed by an exit event, so do not treat the two as alternatives. |
| `BossKilledEvent` | A tracked boss dies with at least one attacker. |

The three lifecycle events carry the same four fields: `playerUuid()`, `worldName()`, `dungeonId()`, and `modeName()`.

`BossKilledEvent` carries more, because a boss can die outside a dungeon:

| Field | Meaning |
| --- | --- |
| `bossTypeId()` | The boss role / `DungeonFrameworkBossAsset` id, e.g. `"Azaroth"`. |
| `worldUuid()` | The world the boss died in. |
| `worldName()` | That world's name, e.g. `instance-Dungeon_Of_Fear_I-<uuid>`. |
| `dungeonId()` | Tracked instance id, or `null` when the boss died outside any tracked dungeon (overworld, untracked instance). |
| `modeName()` | Active mode, `null` whenever `dungeonId()` is `null`. |
| `attackerUuids()` | Unmodifiable snapshot of every player UUID that contributed damage. |

### Listener rules

- Listeners run **synchronously on the world thread** that raised the event. Keep them short, and never block or sleep in one.
- A listener that throws is caught and logged, it will not break the framework or stop the other listeners. It will spam your server log, so handle your own errors.
- Register in your plugin's `start()`, and remove in shutdown. Listeners are not cleared for you.
