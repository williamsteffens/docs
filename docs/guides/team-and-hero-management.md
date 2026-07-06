---
title: "Team & Hero Management"
sidebar_label: "Team & Hero Management"
---

# Team & Hero Management Guide

This guide covers team balancing, hero selection, and player management patterns.

## Team Assignment

### Auto-Balance on Connect

Assign players to the team with fewer members:

```csharp
public override HookResult OnClientFullConnect(ClientFullConnectEvent ev)
{
    var controller = Players.FromSlot(ev.Slot);
    if (controller == null) return HookResult.Handled;

    // Count players per team
    int team0 = 0, team1 = 0;
    foreach (var c in Players.GetAll())
    {
        var pawn = c.GetHeroPawn();
        if (pawn == null) continue;
        if (pawn.TeamNum == 2) team0++;
        else if (pawn.TeamNum == 3) team1++;
    }

    // Assign to smaller team
    controller.ChangeTeam(team0 <= team1 ? 2 : 3);

    return HookResult.Handled;
}
```

### Changing Teams

```csharp
controller.ChangeTeam(2);  // Team 0 (Amber)
controller.ChangeTeam(3);  // Team 1 (Sapphire)
```

## Hero Selection

### Force a Specific Hero

```csharp
controller.SelectHero(Heroes.Inferno);
```

### Random Hero Assignment

```csharp
var heroes = Enum.GetValues<Heroes>()
                 .Where(h => h.GetHeroData()?.AvailableInGame == true)
                 .ToArray();
var randomHero = heroes[Random.Shared.Next(heroes.Length)];
controller.SelectHero(randomHero);
```

### Prevent Hero Changes

Block hero changes using `OnClientConCommand`:

```csharp
public override HookResult OnClientConCommand(ClientConCommandEvent ev)
{
    if (ev.Command == "selecthero")
    {
        return HookResult.Stop;
    }
    return HookResult.Continue;
}
```

## Hero Reset

Reset a pawn's hero including items, abilities, and level:

```csharp
pawn.ResetHero();
```

## Player Tracking

### Iterating Connected Players

```csharp
foreach (var controller in Players.GetAll())
{
    var pawn = controller.GetHeroPawn();
    if (pawn == null) continue;

    // Do something with each active player
}
```

### Player Cleanup on Disconnect

```csharp
public override void OnClientDisconnect(ClientDisconnectedEvent args) {
    var controller = args.Controller;
    if (controller == null) return;

    controller.GetHeroPawn()?.Remove();
    controller.Remove();
}
```

## Game Mode Setup Pattern

Configure server convars for custom game modes in `OnStartupServer`:

```csharp
public override void OnStartupServer()
{
    // Deathmatch configuration
    ConVar.Find("citadel_active_lane")?.SetInt(4);
    ConVar.Find("citadel_player_spawn_time_max_respawn_time")?.SetInt(5);
    ConVar.Find("citadel_allow_purchasing_anywhere")?.SetInt(1);
    ConVar.Find("citadel_item_sell_price_ratio")?.SetFloat(1.0f);
    ConVar.Find("citadel_voice_all_talk")?.SetInt(1);
    ConVar.Find("citadel_player_starting_gold")?.SetInt(0);
    ConVar.Find("citadel_trooper_spawn_enabled")?.SetInt(0);
    ConVar.Find("citadel_npc_spawn_enabled")?.SetInt(0);
    ConVar.Find("citadel_start_players_on_zipline")?.SetInt(0);
    ConVar.Find("citadel_allow_duplicate_heroes")?.SetInt(1);
}
```

## See Also

- [Players API](../api-reference/players) — Controller and pawn reference
- [Heroes API](../api-reference/heroes) — Hero enum and data
- [ConVars](../api-reference/convars) — ConVar manipulation
