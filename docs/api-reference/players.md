---
title: "Players"
sidebar_label: "Players"
---

# Players

> **Namespace:** `DeadworksManaged.Api`

The player system consists of **controllers** (managing the player slot) and **pawns** (the in-game physical entity).

## Players (Static Helpers)

```csharp
// Get all connected controllers
foreach (var controller in Players.GetAll()) { }

// Get all hero pawns
foreach (var pawn in Players.GetAllPawns()) { }

// Get controller by slot
var controller = Players.FromSlot(slotIndex);
```

| Member | Type | Description |
|--------|------|-------------|
| `MaxSlot` | `int` (const) | Maximum number of player slots on the server (31) |
| `GetAll()` | `IEnumerable<CCitadelPlayerController>` | All fully connected controllers |
| `GetAllControllers()` | `IEnumerable<CCitadelPlayerController>` | All existing controllers, including ones not fully connected |
| `GetAllPawns()` | `IEnumerable<CCitadelPlayerPawn>` | Hero pawn for every connected player that has one |
| `FromSlot(int)` | `CCitadelPlayerController?` | Controller in given slot, or `null` |
| `IsConnected(int slot)` | `bool` | Whether the given slot has a fully connected player |

## CCitadelPlayerController

Deadlock-specific player controller. Extends `CBasePlayerController`.

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `GetHeroPawn()` | `CCitadelPlayerPawn?` | Returns the player's current hero pawn, or `null` |
| `ChangeTeam(int team)` | `void` | Moves player to specified team. See the [Team & Hero Management](../guides/team-and-hero-management) guide for the visual-update caveat (prefer the `citadel_change_team` modifier for in-match switches). |
| `SelectHero(Heroes hero)` | `void` | Forces player to select specified hero |
| `PrintToConsole(string msg)` | `void` | Sends message to this player's console (via `echo` client command) |
| `PrintToConsoleAll(string msg)` | `void` | *Static* — Sends message to all connected players' consoles |

> **Running a server-only command as this player** is not exposed as a controller method. Use `Server.ExecuteCommand("...")` to run a server console command globally; there is no API surface for "run as if this client typed it" at the moment.

### Reading the SteamID

The SteamID64 is available as a first-class property on the controller (inherited from `CBasePlayerController`):

```csharp
ulong steamId = controller.PlayerSteamId;
```

During connection you can also use `ClientConnectEvent.SteamId`.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `PlayerDataGlobal` | `PlayerDataGlobal?` | Read-only access to networked stats (kills, gold, level, damage) |

### Example

```csharp
var controller = ctx.Controller;
if (controller == null) return HookResult.Handled;

// Force hero selection
controller.SelectHero(Heroes.Inferno);

// Change team
controller.ChangeTeam(2);

// Get pawn
var pawn = controller.GetHeroPawn();
```

## CBasePlayerController

Base player controller entity. Manages the link between a player slot and their pawn.

| Member | Type | Description |
|--------|------|-------------|
| `Slot` | `int` | 0-based player slot index (`EntityIndex - 1`) |
| `PlayerName` | `string` | Player display name (get/set, char[128] inline buffer) |
| `PlayerSteamId` | `ulong` | The player's SteamID64 |
| `Pawn` | `CBasePlayerPawn?` | The controller's current pawn |
| `SetPawn(pawn, retainOldPawnTeam, copyMovementState, allowTeamMismatch, preserveMovementState)` | `void` | Assigns a new pawn, optionally transferring team and movement state |

## CCitadelPlayerPawn

The in-game physical representation of a player (the hero). Extends `CBasePlayerPawn`.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `EyeAngles` | `Vector3` | Networked eye angles (quantized to 11 bits, ~0.18° precision) |
| `EyePosition` | `Vector3` | Eye position (AbsOrigin + ViewOffset) — where the camera sits |
| `CameraAngles` | `Vector3` | Client camera angles for SourceTV/spectating |
| `ViewAngles` | `Vector3` | Raw server-side view angles (full float precision, no quantization) |
| `Health` | `int` | Current health (inherited from `CBaseEntity`) |
| `Position` | `Vector3` | World position (inherited from `CBaseEntity`) |
| `AbilityComponent` | `CCitadelAbilityComponent` | Access to stamina and ability resources |
| `ModifierProp` | `CModifierProperty` | Access to modifier state flags |

:::tip Angles & Position — All Confirmed Working (Read)
- **`Position`** — World-space origin of the pawn (feet)
- **`EyePosition`** — Where the camera sits (origin + view offset, ~72 units above position)
- **`EyeAngles`** — Quantized eye angles (~0.18° precision, suitable for most checks)
- **`ViewAngles`** — Full-precision server-side view angles (use for accurate aim calculations)
- **`CameraAngles`** — Client camera angles (useful for spectating/SourceTV)
- **Velocity** — Read via `pawn.AbsVelocity`; write via `Teleport(velocity: …)` (see [Entities — Transform](entities#transform))
- **Setting camera** — Use `CCitadelUserMsg_SetClientCameraAngles` via `NetMessages.Send` (see [Networking](networking#set-client-camera-angles)). Schema writes and `Teleport` angles only move the model, not the camera.
:::

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `ModifyCurrency(ECurrencyType type, int amount, ECurrencySource source, bool silent, bool forceGain, bool spendOnly)` | `void` | Add/remove currency (gold, ability points). Negative to spend |
| `ResetHero(bool resetAbilities = true)` | `void` | Full reset: clears loadout, removes items, re-adds starting abilities |
| `RemoveAbility(string abilityName)` | `bool` | Removes ability by internal name. Returns `true` on success |
| `AddAbility(string abilityName, ushort slot)` | `CBaseEntity?` | Adds ability to given slot. Returns the new ability entity |
| `AddItem(string itemName, bool enhanced = false)` | `CBaseEntity?` | Gives an item. Pass `true` for the enhanced version |
| `RemoveItem(string itemName)` | `bool` | Removes item directly (no refund). Calls RemoveAbility internally |
| `SellItem(string itemName, bool fullRefund, bool forceSellPrice)` | `bool` | Sells item with gold refund |
| `GetCurrency(ECurrencyType type)` | `int` | Get current currency amount |
| `SetCurrency(ECurrencyType type, int value)` | `void` | Set currency directly |
| `Level` | `int` | Hero level (get/set) |
| `InRegenerationZone` | `bool` | Whether pawn is in base regen zone (read-only) |
| `ExecuteAbilityBySlot(EAbilitySlot, bool altCast, byte flags)` | `int` | Execute ability in slot (0 = success) |
| `ExecuteAbilityByID(int abilityID, bool altCast, byte flags)` | `int` | Execute ability by runtime ID |
| `GetAbilityBySlot(EAbilitySlot slot)` | `CBaseEntity?` | Get ability entity in slot |
| `ToggleActivate(CBaseEntity ability, bool activate)` | `void` | Activate/deactivate an ability |

### Currency Example

```csharp
// Give 15000 gold
pawn.ModifyCurrency(ECurrencyType.EGold, 15000, ECurrencySource.ECheats, false, false, false);

// Give 17 ability points
pawn.ModifyCurrency(ECurrencyType.EAbilityPoints, 17, ECurrencySource.ECheats, false, false, false);
```

### Ability Management

```csharp
// Remove an ability
pawn.RemoveAbility("ability_priest_weaponswap");

// Add an ability to slot 3
pawn.AddAbility("ability_familiar_ability01", slot: 3);
```

### Unlocking and Upgrading Abilities

Every ability has an `UpgradeBits` field that encodes its unlock and upgrade tier. The low bit is the unlock flag; the next 4 bits are the upgrade stars. Setting it to `0b11111` unlocks and fully upgrades a signature ability:

```csharp
foreach (var ability in pawn.AbilityComponent.Abilities) {
    if (ability.AbilitySlot < EAbilitySlot.Signature1
     || ability.AbilitySlot > EAbilitySlot.Signature4) continue;
    ability.UpgradeBits = ability.UpgradeBits | 0b11111;
}
```

> **Do not write `m_nUpgradeInfo` directly via `SchemaAccessor`.** Bypassing the `SetUpgradeBits` native call (which is what the `UpgradeBits` setter uses) leaves the client UI and cooldown state desynced — abilities will appear grayed-out or fire on wrong cooldowns.

| Ability property | Description |
|---|---|
| `UpgradeBits` | Raw unlock/upgrade bitmask (read/write) |
| `IsUnlocked` | Shorthand for `(UpgradeBits & 1) != 0` |
| `IsChanneling` | Whether the ability is currently being channeled |
| `CooldownStart` / `CooldownEnd` | Cooldown window in server time (read/write) |
| `AbilityName` | Internal name (`SubclassVData?.Name`) |
| `AbilitySlot` | Which slot the ability occupies |
| `IsSignature` / `IsActiveItem` / `IsInnate` / `IsWeapon` / `IsItem` | Slot-category shortcuts |

### EAbilitySlot

Managed enum (`EAbilitySlot : ushort`) matching the game's `EAbilitySlots_t`:

| Value | Raw | Description |
|-------|-----|-------------|
| `Signature1` | 0 | Signature ability 1 |
| `Signature2` | 1 | Signature ability 2 |
| `Signature3` | 2 | Signature ability 3 |
| `Signature4` | 3 | Signature ability 4 (ultimate) |
| `ActiveItem1` | 4 | Active item slot 1 |
| `ActiveItem2` | 5 | Active item slot 2 |
| `ActiveItem3` | 6 | Active item slot 3 |
| `ActiveItem4` | 7 | Active item slot 4 |
| `Ability_Held` | 8 | Held ability |
| `Ability_ZipLine` | 9 | Zipline ability |
| `Ability_Mantle` | 10 | Mantle ability |
| `Ability_ClimbRope` | 11 | Climb rope ability |
| `Ability_Jump` | 12 | Jump ability |
| `Ability_Slide` | 13 | Slide ability |
| `Ability_Teleport` | 14 | Teleport ability |
| `Ability_ZipLineBoost` | 15 | Zipline boost |
| `Cosmetic1` | 16 | Cosmetic slot |
| `Innate1` | 17 | Innate ability 1 |
| `Innate2` | 18 | Innate ability 2 |
| `Innate3` | 19 | Innate ability 3 |
| `WeaponSecondary` | 20 | Secondary weapon |
| `WeaponPrimary` | 21 | Primary weapon |
| `WeaponMelee` | 22 | Melee weapon |
| `None` | 23 | No slot |
| `Invalid` | 0xFFFF | Invalid slot |

## CCitadelAbilityComponent

Ability component on a player pawn. Provides access to stamina/ability resources.

### ResourceStamina (AbilityResource)

```csharp
var stamina = pawn.AbilityComponent.ResourceStamina;

// Read values (confirmed working)
float current = stamina.CurrentValue;  // e.g. 3
float max = stamina.MaxValue;          // e.g. 3
float regen = stamina.PrevRegenRate;   // e.g. 0.555555
float latch = stamina.LatchTime;       // server time of last latch
float latchVal = stamina.LatchValue;   // e.g. 3
```

| Property | Type | Read | Write | Description |
|----------|------|------|-------|-------------|
| `CurrentValue` | `float` | Yes | Yes* | Current stamina value |
| `MaxValue` | `float` | Yes | Yes* | Maximum stamina value |
| `PrevRegenRate` | `float` | Yes | Yes* | Stamina regeneration rate per second |
| `LatchTime` | `float` | Yes | Yes* | Server time of last latch event |
| `LatchValue` | `float` | Yes | Yes* | Latch value |

\* All properties have setters, but the engine recomputes stamina from the latch state each tick — a bare write to `CurrentValue` is overwritten. To change stamina durably, write `LatchValue`/`LatchTime` together with `CurrentValue` (see the stamina example in [First Plugin](../getting-started/first-plugin)).

## Ability Entities

### CCitadel_Ability_Jump

Jump ability entity tracking air jump/wall jump counters. Cast from the abilities list with `As<CCitadel_Ability_Jump>()`.

| Property | Type | Description |
|----------|------|-------------|
| `DesiredAirJumpCount` | `int` | Target air jump count |
| `ExecutedAirJumpCount` | `int` | Actual air jumps performed |
| `ConsecutiveAirJumps` | `sbyte` | Consecutive air jumps without landing |
| `ConsecutiveWallJumps` | `sbyte` | Consecutive wall jumps |

### CCitadel_Ability_Dash

Dash ability entity tracking consecutive air/down dash counters. Cast with `As<CCitadel_Ability_Dash>()`.

| Property | Type | Description |
|----------|------|-------------|
| `ConsecutiveAirDashes` | `sbyte` | Consecutive air dashes without landing |
| `ConsecutiveDownDashes` | `sbyte` | Consecutive downward dashes |

### Example: Reading Ability Counters

```csharp
var abilities = pawn.AbilityComponent.Abilities;
for (int i = 0; i < abilities.Count; i++)
{
    var jump = abilities[i].As<CCitadel_Ability_Jump>();
    if (jump != null)
        Console.WriteLine($"Air jumps: {jump.ConsecutiveAirJumps}");

    var dash = abilities[i].As<CCitadel_Ability_Dash>();
    if (dash != null)
        Console.WriteLine($"Air dashes: {dash.ConsecutiveAirDashes}");
}
```

## PlayerDataGlobal

Read-only access to networked player stats via `Controller.PlayerDataGlobal`.

The underlying schema type is `PlayerDataGlobal_t`. You can read fields via `SchemaAccessor<T>` on the controller's handle:

```csharp
private static readonly SchemaAccessor<int> _goldNetWorth =
    new("PlayerDataGlobal_t"u8, "m_iGoldNetWorth"u8);
private static readonly SchemaAccessor<int> _playerKills =
    new("PlayerDataGlobal_t"u8, "m_iPlayerKills"u8);
private static readonly SchemaAccessor<int> _heroDamage =
    new("PlayerDataGlobal_t"u8, "m_iHeroDamage"u8);

// Read from the controller
var controller = Players.FromSlot(slot);
int networth = _goldNetWorth.Get(controller.Handle);
int kills = _playerKills.Get(controller.Handle);
int damage = _heroDamage.Get(controller.Handle);
```

### All Fields (Verified)

All 24 fields confirmed readable via test:

| Property | Type | Description |
|----------|------|-------------|
| `Level` | `int` | Hero level |
| `MaxAmmo` | `int` | Maximum ammo count |
| `HealthMax` | `int` | Maximum health |
| `Health` | `int` | Current health |
| `GoldNetWorth` | `int` | Total gold (souls) networth |
| `APNetWorth` | `int` | Ability point networth |
| `CreepGold` | `int` | Total creep gold earned |
| `CreepGoldSoloBonus` | `int` | Solo lane bonus gold |
| `CreepGoldKill` | `int` | Gold from creep last hits |
| `CreepGoldAirOrb` | `int` | Gold from air orbs |
| `CreepGoldGroundOrb` | `int` | Gold from ground orbs |
| `CreepGoldDeny` | `int` | Gold from denies |
| `CreepGoldNeutral` | `int` | Gold from neutral camps |
| `FarmBaseline` | `int` | Farm baseline value |
| `PlayerKills` | `int` | Player kills |
| `PlayerAssists` | `int` | Player assists |
| `Deaths` | `int` | Deaths |
| `Denies` | `int` | Creep denies |
| `LastHits` | `int` | Creep last hits |
| `KillStreak` | `int` | Current kill streak |
| `HeroDraftPosition` | `int` | Position in hero draft |
| `HeroDamage` | `int` | Total hero damage dealt |
| `HeroHealing` | `int` | Hero healing done |
| `SelfHealing` | `int` | Self healing done |
| `ObjectiveDamage` | `int` | Damage dealt to objectives |

These can also be read via `SchemaAccessor<int>` on the controller handle using `"PlayerDataGlobal_t"` as the class name (e.g. `"m_iGoldNetWorth"`).

:::caution Slot Limit
Player slots above 31 contain garbage data (e.g. Level=1119879168). Always cap iteration to `slot < 32` when reading PlayerDataGlobal fields across all players.
:::

## Currency Types

### ECurrencyType

| Value | Raw | Description |
|-------|-----|-------------|
| `ECurrencyInvalid` | -1 | Invalid |
| `EGold` | 0 | In-game gold (souls) |
| `EAbilityPoints` | 1 | Ability upgrade points |
| `EAbilityUnlocks` | 2 | Ability unlock tokens |
| `EDeathPenaltyGold` | 3 | Gold lost on death |
| `EItemDraftRerolls` | 4 | Item draft reroll tokens |
| `EItemEnhancements` | 5 | Item enhancement tokens |

### ECurrencySource

| Value | Raw | Description |
|-------|-----|-------------|
| `EItemPurchase` | 0 | Item purchased |
| `EItemUpgrade` | 1 | Item upgraded |
| `EItemSale` | 2 | Item sold |
| `EStartingAmount` | 5 | Starting currency |
| `ELevelUp` | 6 | Level up reward |
| `ECheats` | 7 | Cheat/debug |
| `EPlayerKill` | 11 | Player kill reward |
| `EPlayerKillAssist` | 12 | Assist reward |
| `EBossKill` | 13 | Boss kill reward |
| `ELaneTrooperKill` | 14 | Lane trooper kill |
| `ENeutralTrooperKill` | 15 | Neutral camp kill |
| `EOrbPlayer` | 22 | Soul orb from player |
| `EOrbLaneTrooper` | 24 | Soul orb from lane trooper |

:::note
ECurrencySource has 45 values total. Only some are listed above.
:::

## LifeState

Entity life-cycle state (managed enum `LifeState : uint`):

| Value | Raw | Description |
|-------|-----|-------------|
| `Alive` | 0 | Entity is alive |
| `Dying` | 1 | Entity is in the dying process |
| `Dead` | 2 | Entity is dead |
| `Respawnable` | 3 | Entity can respawn |
| `Respawning` | 4 | Entity is respawning |

## See Also

- [Entities](entities) — Base entity system
- [Modifiers](modifiers) — Modifier states on pawns
- [Heroes](heroes) — Hero enum and data
- [Team & Hero Management Guide](../guides/team-and-hero-management) — Practical patterns
