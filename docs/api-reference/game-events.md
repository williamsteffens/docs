---
title: "Game Events"
sidebar_label: "Game Events"
---

# Game Events

> **Namespace:** `DeadworksManaged.Api`

Listen to and create Source 2 game events.

## GameEventHandlerAttribute

Auto-register a method as a game event handler:

```csharp
[GameEventHandler("player_hero_changed")]
public HookResult OnPlayerHeroChanged(PlayerHeroChangedEvent args)
{
    var pawn = args.Userid?.As<CCitadelPlayerPawn>();
    if (pawn != null)
    {
        // Do stuff
    }
    return HookResult.Continue;
}
```

### Typed Event Classes

For every event in the game's `.gameevents` schema, the SDK source-generates a typed class named after the event in PascalCase (`player_hero_changed` → `PlayerHeroChangedEvent`, `player_death` → `PlayerDeathEvent`), with one property per field. Your handler can take either the typed class or the untyped `GameEvent` (with `GetString`/`GetInt`/`GetFloat`/`GetBool`/`GetPlayerPawn`/`GetPlayerController`/`GetEHandle` accessors).

Fields typed `player_controller_and_pawn` in the schema (e.g. `player_death.userid`/`attacker`) generate **split properties** — `UseridController`/`UseridPawn`, `AttackerController`/`AttackerPawn` — there is no single `Userid` property on those events. Fields typed `player_pawn` (like `player_hero_changed.userid`) generate a single pawn-typed property.

You can also subscribe dynamically without an attribute via `GameEvents.AddListener(name, handler)` / `GameEvents.RemoveListener(...)`.

## GameEventWriter

Wraps a newly created game event for setting fields and firing.

```csharp
var ev = GameEvents.Create("my_custom_event");
if (ev != null)
{
    ev.SetInt("player_slot", 0);
    ev.SetString("action", "test");
    ev.Fire();
    // After firing, the event is owned by the engine — do not use it again
}
```

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `SetString(string key, string value)` | `GameEventWriter` | Set string field (chainable) |
| `SetInt(string key, int value)` | `GameEventWriter` | Set int field (chainable) |
| `SetFloat(string key, float value)` | `GameEventWriter` | Set float field (chainable) |
| `SetBool(string key, bool value)` | `GameEventWriter` | Set bool field (chainable) |
| `Fire(bool dontBroadcast)` | `bool` | Fires the event. Returns success. After firing, owned by engine — must not be used |

## Common Game Events

| Event Name | Typical fields | Notes |
|------------|----------------|-------|
| `player_death` | `userid`, `attacker` | Fires on hero death |
| `player_spawn` | `userid` | See warning below — fires **before** the pawn is fully materialized on the first spawn |
| `player_hero_changed` | `userid` | More reliable than `player_spawn` for post-hero-select logic |
| `player_used_ability` | `player`, `caster`, `abilityname`, `annotation` | Fires for every ability activation including shots (`citadel_weapon_*`) and melee (`ability_melee_*`) |
| `ability_added` | `userid`, `ability` (ehandle) | Fires when an ability/item is granted — use this to detect item purchases |
| `game_state_changed` | `game_state_new` | Fires at major match transitions, including end-of-match |

The full list of Source 2 events shipped by Deadlock is available at [SteamTracking-Deadlock/resource/core.gameevents](https://github.com/SteamTracking/GameTracking-Deadlock/blob/master/game/core/pak01_dir/resource/core.gameevents) and [game.gameevents](https://github.com/SteamTracking/GameTracking-Deadlock/blob/master/game/citadel/pak01_dir/resource/game.gameevents). These files list every event name and its field schema.

## Common Patterns

### Detecting a melee attack

```csharp
[GameEventHandler("player_used_ability")]
public HookResult OnAbility(GameEvent ev)
{
    var name = ev.GetString("abilityname", "");
    if (!name.StartsWith("citadel_ability_melee")) return HookResult.Continue;

    // annotation tells heavy vs light melee (event keys are case-sensitive)
    var kind = ev.GetString("annotation", ""); // "heavy_melee" or "light_melee"
    Console.WriteLine($"Melee attack: {kind}");
    return HookResult.Continue;
}
```

You could also use `OnAbilityAttempt` with `InputButton.Weapon1`, but `player_used_ability` fires only on successful activation and gives you the heavy/light distinction for free.

### Detecting a weapon shot

Same event, different prefix check:

```csharp
if (name.StartsWith("citadel_weapon_")) { /* shot fired */ }
```

### Detecting an item purchase

```csharp
[GameEventHandler("ability_added")]
public HookResult OnAbilityAdded(GameEvent ev)
{
    var pawn = ev.GetPlayerPawn("userid")?.As<CCitadelPlayerPawn>();
    if (pawn == null) return HookResult.Continue;

    // ability_added fires both for ability unlocks and for item purchases.
    // Walk the pawn's abilities on the next tick to see what's actually there.
    Timer.Once(1.Seconds(), () =>
    {
        foreach (var ab in pawn.AbilityComponent.Abilities)
            if (ab.IsItem && ab.AbilityName == "upgrade_unstoppable")
                Console.WriteLine("Bought Unstoppable!");
    });
    return HookResult.Continue;
}
```

### Detecting match end

The cleanest signal is `game_state_changed`.

## player_spawn Is Racy on First Spawn

The `player_spawn` event fires before the pawn is fully populated the first time a player connects. Subsequent spawns are fine. If your handler dereferences hero data (`GetHeroPawn`, `AbilityComponent`, etc.), either:

- Subscribe to `player_hero_changed` instead — it fires after hero data is ready, and
- Use `Timer.NextTick` or a short delay to re-check the pawn before touching it.

```csharp
[GameEventHandler("player_hero_changed")]
public HookResult OnPlayerHeroChanged(PlayerHeroChangedEvent args)
{
    var pawn = args.Userid?.As<CCitadelPlayerPawn>();
    if (pawn == null) return HookResult.Continue;
    // safe to inspect pawn's hero-specific state here
    return HookResult.Continue;
}
```

## See Also

- [Players](players) — Player pawn and controller access
