---
title: "Item Rotation"
sidebar_label: "Item Rotation"
---

# Example: Item Rotation

A minimal plugin that swaps every player's loadout to a new item set on a timer. Demonstrates:

- Rotating set assignments on an interval
- Removing old items and giving new ones via `pawn.RemoveItem` / `pawn.AddItem`
- Playing a sound as feedback
- Showing a HUD announcement

## What It Does

- `/ir_start` or `!ir_start` begins rotation. Every player gets a starting set.
- Every 10 seconds, each player advances to the next set: items from the previous set are removed, items from the new set are given, a warning sound plays, and a HUD announcement names the new set.
- `/ir_reset` or `!ir_reset` stops rotation and clears any items that were given.

Extend the `_sets` array to add more loadouts.

## Key Pieces

| Concept | Where |
|---|---|
| Recurring work | `Timer.Every(10.Seconds(), ...)` - save the handle so `/ir_reset` can cancel it |
| Per-player state | `Dictionary<int, int>` mapping player slot to current set index |
| Item swap | `pawn.RemoveItem(oldItem)` then `pawn.AddItem(newItem)` |
| Sound feedback | `pawn.EmitSound("Mystical.Piano.AOE.Warning")` |
| HUD announcement | `CCitadelUserMsg_HudGameAnnouncement` via `NetMessages.Send(..., RecipientFilter.Single(slot))` |

## Full Source

```csharp
using DeadworksManaged.Api;

namespace ItemRotationPlugin;

public class ItemRotationPlugin : DeadworksPluginBase {
    public override string Name => "Item Rotation";

    public override void OnLoad(bool isReload) { }
    public override void OnUnload() => Stop();

    private record ItemSet(string Name, string[] Items);

    private static readonly ItemSet[] _sets = {
        new("Speed Demons", new[] { "upgrade_sprint_booster", "upgrade_kinetic_sash" }),
        new("Cardio Kings", new[] { "upgrade_fleetfoot_boots", "upgrade_cardio_calibrator" }),
        new("Warp Zone",    new[] { "upgrade_warp_stone",     "upgrade_sprint_booster" }),
    };

    private readonly Dictionary<int, int> _playerSet = new(); // slot -> set index
    private IHandle? _timer;

    [Command("ir_start", Description = "Start the item rotation game")]
    public void CmdStart(CCitadelPlayerController? caller) {
        if (_timer != null) return;

        // Initial assignment - each player starts on a different set
        int i = 0;
        foreach (var controller in Players.GetAll()) {
            var pawn = controller.GetHeroPawn();
            if (pawn == null) continue;

            int setIndex = i % _sets.Length;
            _playerSet[controller.Slot] = setIndex;
            GiveSet(pawn, _sets[setIndex], controller.Slot, announce: false);
            i++;
        }

        _timer = Timer.Every(10.Seconds(), Rotate);
    }

    [Command("ir_reset", Description = "Stop the item rotation game and clear all items")]
    public void CmdReset(CCitadelPlayerController? caller) {
        Stop();
    }

    private void Rotate() {
        foreach (var controller in Players.GetAll()) {
            var pawn = controller.GetHeroPawn();
            if (pawn == null) continue;
            int slot = controller.Slot;

            // Remove the old set
            if (_playerSet.TryGetValue(slot, out int oldIndex)) {
                foreach (var item in _sets[oldIndex].Items)
                    pawn.RemoveItem(item);
            }

            // Advance and give the new set
            int newIndex = _playerSet.TryGetValue(slot, out int current)
                ? (current + 1) % _sets.Length
                : 0;
            _playerSet[slot] = newIndex;
            GiveSet(pawn, _sets[newIndex], slot, announce: true);
        }
    }

    private static void GiveSet(CCitadelPlayerPawn pawn, ItemSet set, int slot, bool announce) {
        foreach (var item in set.Items)
            pawn.AddItem(item);

        if (!announce) return;

        pawn.EmitSound("Mystical.Piano.AOE.Warning");

        var msg = new CCitadelUserMsg_HudGameAnnouncement {
            TitleLocstring = "ITEMS ROTATED",
            DescriptionLocstring = set.Name,
        };
        NetMessages.Send(msg, RecipientFilter.Single(slot));
    }

    private void Stop() {
        _timer?.Cancel();
        _timer = null;

        // Clean up any items we gave out
        foreach (var (slot, setIndex) in _playerSet) {
            var pawn = Players.FromSlot(slot)?.GetHeroPawn();
            if (pawn == null) continue;
            foreach (var item in _sets[setIndex].Items)
                pawn.RemoveItem(item);
        }
        _playerSet.Clear();
    }

    public override void OnClientDisconnect(ClientDisconnectedEvent args) {
        _playerSet.Remove(args.Slot);
    }
}
```

## See Also

- [Commands](../api-reference/commands)
- [Players](../api-reference/players) - `AddItem` / `RemoveItem`
- [Timers](../api-reference/timers) - `Timer.Every` and `IHandle.Cancel`
- [Networking](../api-reference/networking) - HUD announcements
