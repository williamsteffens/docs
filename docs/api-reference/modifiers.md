---
title: "Modifiers"
sidebar_label: "Modifiers"
---

# Modifiers

> **Namespace:** `DeadworksManaged.Api`

Modifiers are buffs/debuffs applied to entities. They can alter gameplay states, apply visual effects, and control movement.

## Adding Modifiers

Use `CBaseEntity.AddModifier()` with a VData name and `KeyValues3` parameters:

```csharp
using var kv = new KeyValues3();
kv.SetFloat("duration", 3.0f);
pawn.AddModifier("modifier_citadel_knockdown", kv);
```

### AddModifier Parameters

```csharp
entity.AddModifier(
    string name,            // Modifier VData name (e.g. "modifier_citadel_knockdown")
    KeyValues3? kv,         // Parameters (duration, etc.) (optional)
    CBaseEntity? caster,    // Entity that applied the modifier (optional)
    CBaseEntity? ability,   // Ability that caused the modifier (optional)
    int team                // (optional, defaults to 0)
);
```

### Overriding Ability Property Values

A second overload takes a `Dictionary<string, float>` of per-instance ability property overrides. Use it to apply modifiers that normally read values from a parent ability **without needing a real ability**, or to override the ability's default values. Property names match the VData's `m_vecAutoRegisterModifierValueFromAbilityPropertyName` entries:

```csharp
pawn.AddModifier("ability_doorman_bomb/debuff", kv: kv,
    abilityValues: new() { ["SlowPercent"] = 100.0f });
```

## KeyValues3

Wraps a native KeyValues3 handle. Create, set typed members, pass to `AddModifier`, then dispose.

### Important: Always Dispose

```csharp
// Using statement ensures proper cleanup
using var kv = new KeyValues3();
kv.SetFloat("duration", 5.0f);
kv.SetString("effect", "burning");
pawn.AddModifier("my_modifier", kv);
// kv is automatically disposed here
```

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `IsValid` | `bool` | `true` if underlying native handle is alive |

### Setter Methods

| Method | Description |
|--------|-------------|
| `SetString(string key, string value)` | Set a string value |
| `SetBool(string key, bool value)` | Set a boolean value |
| `SetInt(string key, int value)` | Set a signed 32-bit integer |
| `SetUInt(string key, uint value)` | Set an unsigned 32-bit integer |
| `SetInt64(string key, long value)` | Set a signed 64-bit integer |
| `SetUInt64(string key, ulong value)` | Set an unsigned 64-bit integer |
| `SetFloat(string key, float value)` | Set a single-precision float |
| `SetDouble(string key, double value)` | Set a double-precision float |
| `SetVector(string key, Vector3 value)` | Set a 3D vector |

## CModifierProperty

Manages modifier state bits and active modifier instances on an entity. Access via `entity.ModifierProp`.

```csharp
// Enable unlimited air jumps
pawn.ModifierProp.SetModifierState(EModifierState.UnlimitedAirJumps, true);
pawn.ModifierProp.SetModifierState(EModifierState.UnlimitedAirDashes, true);

// Check a state
bool hasState = pawn.ModifierProp.HasModifierState(EModifierState.UnlimitedAirJumps);

// Disable
pawn.ModifierProp.SetModifierState(EModifierState.UnlimitedAirJumps, false);

// Enumerate active modifiers
foreach (var mod in pawn.ModifierProp.Modifiers)
    Console.WriteLine(mod.SubclassVData?.Name);

// Check by name
if (pawn.ModifierProp.HasModifier("modifier_citadel_knockdown")) { /* ... */ }
```

| Method / Property | Returns | Description |
|--------|---------|-------------|
| `SetModifierState(EModifierState, bool)` | `void` | Sets or clears a modifier state bit |
| `HasModifierState(EModifierState)` | `bool` | Returns `true` if state bit is set |
| `HasModifier(string name)` | `bool` | Returns `true` if the entity has an active modifier with the given subclass VData name |
| `Modifiers` | `IReadOnlyList<CBaseModifier>` | All active modifier instances on the entity |
| `Owner` | `CBaseEntity?` | The entity this modifier property belongs to |

### Removing a modifier

Remove is a method on the entity itself, not on `ModifierProp`:

```csharp
// By name — removes the first instance matching that VData name
pawn.RemoveModifier("modifier_citadel_knockdown");

// By instance — use when you held on to a CBaseModifier from AddModifier
var mod = pawn.AddModifier("modifier_...", kv);
pawn.RemoveModifier(mod);
```

## EModifierState

Enum containing all modifier state flags (302 values). Key examples:

| Value | Description |
|-------|-------------|
| `UnlimitedAirJumps` | Allow unlimited air jumps |
| `UnlimitedAirDashes` | Allow unlimited air dashes |
| *(many more)* | Covers movement, visibility, combat, hero-specific states |

## CBaseModifier

Wraps a native CBaseModifier instance — a buff/debuff applied to an entity. Returned by `AddModifier`.

## EKnockDownTypes

Knockdown animation type applied to a hero.

## Common Modifier Patterns

### Knockdown with Duration

```csharp
using var kv = new KeyValues3();
kv.SetFloat("duration", 3.0f);
pawn.AddModifier("modifier_citadel_knockdown", kv);
```

### Toggling Movement States

```csharp
// Enable
var mp = pawn.ModifierProp;
mp.SetModifierState(EModifierState.UnlimitedAirJumps, true);
mp.SetModifierState(EModifierState.UnlimitedAirDashes, true);

// After duration, disable
Timer.Once(20.Seconds(), () =>
{
    mp.SetModifierState(EModifierState.UnlimitedAirJumps, false);
    mp.SetModifierState(EModifierState.UnlimitedAirDashes, false);
});
```

### AddAbility + AddModifier Pattern

Many game modifiers read properties from their parent ability (e.g., `ModelScaleGrowth`, `ActiveMoveSpeedPenalty`). The simplest way to supply those values is the `abilityValues` overload shown above — no real ability needed. Alternatively, you can add the actual ability first and pass the returned ability entity as the 4th parameter to `AddModifier`:

```csharp
// 1. Add the item ability — returns the ability entity
var ability = pawn.AddAbility("upgrade_shrink_ray", 0);
if (ability == null) return;

// 2. Apply modifier, passing the ability so it can read VData properties
using var kv = new KeyValues3();
kv.SetFloat("duration", 30.0f);
pawn.AddModifier("modifier_shrink_ray", kv, pawn, ability);

// 3. Remove the ability after the modifier is applied
pawn.RemoveAbility("upgrade_shrink_ray");
```

## EModifierState Reference

State behavior is inconsistent — some flags are honored by the client, others are only meaningful when set by internal C++ modifier code. When in doubt, try setting the flag every tick (from `OnGameFrame` or a 1-tick timer): several states that "don't work" when set once do work when sustained.

### Examples

| Raw Value | Name | Description |
|-----------|------|-------------|
| 11 | `Immobilized` | Prevents all movement |
| 38 | `AdditionalAirMoves` | Extra air moves |
| 39 | `UnlimitedAirDashes` | Allow unlimited air dashes |
| 40 | `UnlimitedAirJumps` | Allow unlimited air jumps |
| 68 | `VisibleToEnemy` | Reveal on the minimap to the enemy team. **Must be set every tick** to stay reliable for human players (bots work with a single call). |
| 69 | `InfiniteClip` | Infinite ammo, no reload needed |
| 115 | `UnitStatusHealthHidden` | Hides the healthbar above the hero. Set every tick. |
| 116 | `UnitStatusHidden` | Hides the whole unit status / nameplate panel. Set every tick. |
| 118 | `FriendlyFireEnabled` | Enables friendly-fire **for bullet damage only**. Abilities and melee still ignore teammates; see the FFA recipe in [team-and-hero-management](../guides/team-and-hero-management). |

:::tip Try setting every tick before giving up
The Discord dump has several cases of "doesn't work" turning into "works when set every tick." Before writing off a state, try:

```csharp
public override void OnGameFrame(bool simulating, bool firstTick, bool lastTick)
{
    if (!simulating) return;
    foreach (var pawn in Players.GetAllPawns())
        pawn.ModifierProp?.SetModifierState(EModifierState.VisibleToEnemy, true);
}
```

You can also cast raw integer values to `EModifierState` for states not in the managed enum:

```csharp
pawn.ModifierProp?.SetModifierState((EModifierState)69, true); // InfiniteClip
```
:::

:::note
`EModifierState` has 302 values total. The raw values follow the `MODIFIER_STATE_` prefix pattern from VData (e.g., `MODIFIER_STATE_INFINITE_CLIP` = 69).
:::

## Discovering Modifier Names

Deadlock has hundreds of modifiers. Useful sources:

- **`modifier_dump_list`** in the server console — lists currently-known modifiers. Results can vary between invocations; if it looks short, run it again after a map fully loads.
- **`modifier_dump`** — lists modifiers currently active on an entity. Useful for discovering subclassed modifiers.
- **[Deadlock modding modifier list](https://deadlockmodding.pages.dev/modifier-list)** — community-maintained dump.
- **`scripts/modifiers.vdata`** (extracted from `pak01_dir.vpk`) — the authoritative standalone modifier definitions.
- **`scripts/abilities.vdata`** — abilities contain inline modifier subclasses under `Modifiers` blocks. These are referenced as `upgrade_<item>/modifier_<name>` in VData but you call `AddModifier` with just the bare modifier name.

Tools for browsing the VDATA: [source2viewer-cli](https://github.com/ValveResourceFormat/ValveResourceFormat) (decompile and grep), and [s2v.app](https://s2v.app/) (web browser for game files).

## See Also

- [Entities](entities) — `AddModifier` on `CBaseEntity`
- [Players](players) — `ModifierProp` on `CCitadelPlayerPawn`
- [Timers](timers) — Timed modifier application
- [Roll The Dice Example](../examples/roll-the-dice) — Modifier effects in practice
