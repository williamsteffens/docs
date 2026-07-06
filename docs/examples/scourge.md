---
title: "Scourge DOT"
sidebar_label: "Scourge DOT"
---

# Example: Scourge DOT Plugin

**Source:** `ScourgePlugin.cs`

A focused plugin that applies a configurable damage-over-time (DOT) effect when players are hit by a specific ability.

## Overview

- **Trigger:** Player hit by `upgrade_discord` ability
- **Effect:** Periodic damage as a fraction of max health
- **Concepts demonstrated:** `OnTakeDamage` hook, `Timer.Sequence`, config with validation, VData checking, entity data cleanup

## Architecture

```
Player takes damage
    │
    ├── OnTakeDamage hook fires
    ├── Check if ability is "upgrade_discord"
    ├── Cancel any existing DOT on this target
    └── Start Timer.Sequence:
        │
        └── Every 200ms (configurable):
            ├── Check if target is alive
            ├── Calculate damage (fraction of max HP)
            ├── Apply damage via Hurt()
            ├── Play sound
            └── Repeat until duration expires
```

## Configuration

Full JSON config with validation, using `IConfig` and `[PluginConfig]`:

```csharp
public class ScourgeConfig : IConfig
{
    [JsonPropertyName("DurationSeconds")]
    public float DurationSeconds { get; set; } = 15f;

    [JsonPropertyName("DamageIntervalMs")]
    public int DamageIntervalMs { get; set; } = 200;

    [JsonPropertyName("DamageFraction")]
    public float DamageFraction { get; set; } = 0.005f;

    [JsonPropertyName("DamageSound")]
    public string DamageSound { get; set; } = "Damage.Send.Crit";

    [JsonPropertyName("DamageSoundVolume")]
    public float DamageSoundVolume { get; set; } = 0.1f;

    public void Validate()
    {
        if (DurationSeconds < 0.1f) DurationSeconds = 0.1f;
        if (DamageIntervalMs < 50) DamageIntervalMs = 50;
        if (DamageFraction <= 0f) DamageFraction = 0.005f;
        DamageSoundVolume = Math.Clamp(DamageSoundVolume, 0f, 1f);
    }
}
```

`Validate()` runs automatically after the config is loaded or reloaded. The plugin exposes the config through a `[PluginConfig]` property:

```csharp
public class ScourgePlugin : DeadworksPluginBase
{
    [PluginConfig]
    public ScourgeConfig Config { get; set; } = new();
}
```

## Damage Interception

Uses `OnTakeDamage` to detect a specific ability and start the DOT:

```csharp
public override HookResult OnTakeDamage(TakeDamageEvent args)
{
    // Only trigger for upgrade_discord ability
    if (args.Info.Ability?.SubclassVData?.Name != "upgrade_discord")
        return HookResult.Continue;

    var pawn = args.Entity.As<CCitadelPlayerPawn>();
    if (pawn == null) return HookResult.Continue;

    var attacker = args.Info.Attacker;
    uint victimHandle = pawn.EntityHandle;

    // Cancel existing DOT on this target
    if (_dotTimers.TryGet(pawn, out var existing))
        existing.Cancel();

    // Start new DOT sequence...
    return HookResult.Continue;
}
```

**Key insight:** The hook returns `HookResult.Continue`, allowing the original damage to still be applied. The DOT is an additional effect on top.

## Timer.Sequence for DOT

The DOT uses `Timer.Sequence` for a stateful, repeating effect with automatic termination:

```csharp
int maxTicks = (int)(Config.DurationSeconds * 1000 / Config.DamageIntervalMs);

var handle = Timer.Sequence(step =>
{
    // Stop after max ticks
    if (step.Run > maxTicks)
        return step.Done();

    // Verify target still exists and is alive
    var ent = CBaseEntity.FromHandle(victimHandle);
    if (ent == null || !ent.IsAlive)
        return step.Done();

    // Calculate damage as fraction of max health
    var healthMax = pawn.Controller?.PlayerDataGlobal.HealthMax ?? 0;
    if (healthMax <= 0)
        return step.Done();

    // Apply damage and sound
    ent.Hurt(healthMax * damageFraction, attacker: attacker);
    ent.EmitSound(sound, volume: volume);

    return step.Wait(intervalMs.Milliseconds());
});

_dotTimers[pawn] = handle;
```

### Why Timer.Sequence?

- **Stateful:** `step.Run` tracks how many ticks have passed
- **Self-terminating:** Returns `step.Done()` when target dies or duration expires
- **Variable pacing:** `step.Wait(duration)` allows configurable tick intervals
- **Clean API:** No need to manage counters externally

## Safety Patterns

### Entity Handle Validation

Uses `CBaseEntity.FromHandle(victimHandle)` to verify the entity still exists before each tick. This prevents crashes from accessing deleted entities.

### Cancel on Re-application

If the same target gets hit again, the existing DOT is cancelled before starting a new one:

```csharp
if (_dotTimers.TryGet(pawn, out var existing))
    existing.Cancel();
```

### Cleanup on Unload

```csharp
public override void OnUnload()
{
    _dotTimers.Clear();  // Cancel all active DOTs
}
```

## API Features Used

| Feature | Reference |
|---------|-----------|
| `IConfig`, `[PluginConfig]` | [Configuration](../api-reference/configuration) |
| `OnTakeDamage` | [Damage](../api-reference/damage) |
| `Timer.Sequence`, `IStep` | [Timers](../api-reference/timers) |
| `EntityData<IHandle>` | [Entities](../api-reference/entities) |
| `CBaseEntity.FromHandle` | [Entities](../api-reference/entities) |
| `Hurt()`, `EmitSound()` | [Entities](../api-reference/entities) |
| `SubclassVData` | [Entities](../api-reference/entities) |
| `PlayerDataGlobal.HealthMax` | [Players](../api-reference/players) |
