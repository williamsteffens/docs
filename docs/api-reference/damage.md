---
title: "Damage"
sidebar_label: "Damage"
---

# Damage

> **Namespace:** `DeadworksManaged.Api`

Intercept, modify, and apply damage to entities.

## OnTakeDamage Hook

Override in your plugin to intercept all damage on the server:

```csharp
public override HookResult OnTakeDamage(TakeDamageEvent ev)
{
    // ev.Entity — the entity taking damage
    // ev.Info — full damage descriptor

    // Block damage to the Patrons
    if (ev.Entity.DesignerName == "npc_boss_tier3")
        return HookResult.Stop;

    return HookResult.Continue;
}
```

### TakeDamageEvent

| Property | Type | Description |
|----------|------|-------------|
| `Entity` | `CBaseEntity` | The entity taking damage |
| `Info` | `CTakeDamageInfo` | Full damage descriptor (attacker, inflictor, amount, flags) |


## Applying Damage

### Simple: Hurt()

Convenience wrapper for applying damage. All parameters except `damage` are optional:

```csharp
// Minimal — damage is self-inflicted with no attacker credit
pawn.Hurt(50f);

// Attacker credited — shows in the kill feed
target.Hurt(100f, attacker: shooter);

// Full control
entity.Hurt(
    100f,           // damage amount
    attacker,       // attacking entity
    inflictor,      // entity that caused damage (weapon, projectile)
    ability,        // ability entity
    damageType: 0   // damage type bits (int, see below)
);
```

> If `attacker` is null, it falls back to the `inflictor`; if that is also null, the victim itself is credited (self-damage). `Hurt` always sets `TakeDamageFlags.AllowSuicide`, so self-damage can kill.

### Advanced: CTakeDamageInfo

For full control, create a `CTakeDamageInfo`:

```csharp
using var damageInfo = new CTakeDamageInfo(
    damage: 200f,
    attacker: attackerEntity,
    inflictor: inflictorEntity,
    ability: abilityEntity,
    damageType: 0
);

targetEntity.TakeDamage(damageInfo);
// damageInfo is disposed automatically via using
```

### CTakeDamageInfo

| Method/Constructor | Description |
|-------------------|-------------|
| `new CTakeDamageInfo(float damage, CBaseEntity? attacker, CBaseEntity? inflictor, CBaseEntity? ability, int damageType)` | Create new (must dispose) |
| `FromExisting(nint)` | Wrap existing pointer (non-owning, e.g. from hook) — *internal* |

**Important:** When creating via constructor, always use `using` or call `Dispose()`.

#### Properties (Verified)

| Property | Type | Description |
|----------|------|-------------|
| `Damage` | `float` | Damage amount (get/set) |
| `TotalledDamage` | `float` | Totalled damage (mirrors Damage on creation) |
| `DamageFlags` | `TakeDamageFlags` | Damage flags (get/set) |
| `DamageType` | `int` | Damage type bits (get/set) |
| `Attacker` | `CBaseEntity?` | The attacking entity |
| `Inflictor` | `CBaseEntity?` | The inflictor entity |
| `Ability` | `CBaseEntity?` | The ability entity |
| `Originator` | `CBaseEntity?` | The originator (usually null on creation) |

## TakeDamageFlags

Managed `[Flags]` enum (`TakeDamageFlags : ulong`) that modifies how damage is applied. Commonly used values:

| Flag | Value | Description |
|------|-------|-------------|
| `None` | 0 | No flags |
| `SuppressHealthChanges` | 0x1 | Don't change health |
| `SuppressPhysicsForce` | 0x2 | No knockback |
| `SuppressEffects` | 0x4 | No visual effects |
| `PreventDeath` | 0x8 | Cannot kill (clamp to 1HP) |
| `ForceDeath` | 0x10 | Guarantee kill |
| `AlwaysGib` | 0x20 | Always gib on death |
| `NeverGib` | 0x40 | Never gib on death |
| `SuppressDamageModification` | 0x100 | Ignore armor/resist |
| `RadiusDmg` | 0x400 | Area/splash damage |
| `AllowSuicide` | 0x40000 | Allow self-kill |
| `SuppressKillCredit` | 0x400000 | No kill credit |
| `SuppressDeathCredit` | 0x800000 | No death credit |
| `HeavyMelee` | 0x200000000 | Heavy melee hit |
| `LightMelee` | 0x400000000 | Light melee hit |

The enum has ~70 members in total (`IgnoreResistances`, `DoNotCrit`, `SuppressCritResistance`, `Ricochet`, `BonusDamage`, `IsHealthTransfer`, …) — see `TakeDamageFlags` in the SDK for the full list.

:::tip
Combine `TakeDamageFlags.ForceDeath | TakeDamageFlags.AllowSuicide` for guaranteed kills.
:::

## Damage Type Bits

There is **no managed enum for damage types** — `Hurt(...)` and the `CTakeDamageInfo` constructor take the raw engine bits as an `int` (`damageType` parameter). Values from the game's `DamageTypes_t`, for reference:

| Engine flag | Value | Description |
|------|-------|-------------|
| generic | 0 | Generic damage |
| bullet | 2 | Bullet damage |
| slash | 4 | Slash/melee |
| burn | 8 | Fire/burn |
| fall | 32 | Fall damage |
| blast | 64 | Explosion |
| shock | 256 | Shock/electric |
| headshot | 524288 | Headshot |
| crit | 1048576 | Critical hit |
| dot | 4194304 | Damage over time |
| lethal | 16777216 | Lethal flag |

## See Also

- [Entities](entities) — `Hurt()` and `TakeDamage()` on `CBaseEntity`
- [Players](players) — Currency types and management
- [Scourge Example](../examples/scourge) — Complete DOT implementation
