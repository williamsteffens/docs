---
title: "Sound"
sidebar_label: "Sound"
---

# Sound

> **Namespace:** `DeadworksManaged.Api`

Deadlock plays sounds through **soundevents** — named entries in the game's audio manifest (e.g. `Mystical.Piano.AOE.Explode`, `Male.AnnTemp.Core_Damaged`). You cannot play raw `.vsnd` files directly.

There are two ways to trigger a soundevent from a plugin:

- **`CBaseEntity.EmitSound`** — plays a sound attached to an entity, for everyone in range.
- **`Sounds` / `SoundEvent`** (namespace `DeadworksManaged.Api.Sounds`) — sends the soundevent directly to chosen recipients, with per-player targeting and GUID-based stop/update.

## Playing a Soundevent on an Entity

The simplest case — play a sound on a pawn, NPC, or world entity:

```csharp
pawn.EmitSound("Mystical.Piano.AOE.Warning");
pawn.EmitSound("Damage.Send.Crit", pitch: 100, volume: 0.5f, delay: 0f);
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `soundName` | `string` | — | Full soundevent name |
| `pitch` | `int` | `100` | 100 = normal pitch |
| `volume` | `float` | `1.0` | 0.0–1.0 |
| `delay` | `float` | `0.0` | Seconds before the sound actually starts |

The sound plays in 3D space attached to the entity. Nearby players hear it with positional falloff.

## Sounds (Static Helpers)

For playback without an entity — or targeted at specific players — use the `Sounds` class (namespace `DeadworksManaged.Api.Sounds`):

```csharp
using DeadworksManaged.Api.Sounds;

// Non-positional, heard by everyone (plays at each listener's own position)
uint guid = Sounds.Play("Mystical.Piano.AOE.Warning", RecipientFilter.All);

// Anchored to an entity's position, sent to one player only
Sounds.PlayAt("Damage.Send.Crit", pawn.EntityIndex, RecipientFilter.Single(slot), volume: 0.5f);

// Stop a playing sound by GUID
SoundEvent.Stop(guid, RecipientFilter.All);
```

| Method | Returns | Description |
|--------|---------|-------------|
| `Play(string name, RecipientFilter recipients, float volume = 1, float pitch = 1)` | `uint` | Plays at each listener's own position (non-positional) |
| `PlayAt(string name, int sourceEntityIndex, RecipientFilter recipients, float volume = 1, float pitch = 1)` | `uint` | Plays anchored to an entity's world position |

Both return the soundevent **GUID**, which you can use to stop or update the sound later.

## SoundEvent (Builder)

`Sounds.Play`/`PlayAt` are shortcuts over the full `SoundEvent` builder:

```csharp
uint guid = new SoundEvent("UI.SomeSoundName")
{
    Volume = 0.8f,
    Pitch = 1.2f,
    SourceEntityIndex = pawn.EntityIndex,  // -1 (default) = listener's own position
}
.SetFloat("public.custom_param", 2f)
.Emit(RecipientFilter.All);

// Update a playing sound's params
new SoundEvent("UI.SomeSoundName") { Volume = 0.2f }
    .SetParams(guid, RecipientFilter.All);

// Stop by GUID, or stop all instances by name for a source entity
SoundEvent.Stop(guid, RecipientFilter.All);
SoundEvent.StopByName("UI.SomeSoundName", pawn.EntityIndex, RecipientFilter.All);
```

| Member | Description |
|--------|-------------|
| `SoundEvent(string name)` | Create a builder for the named soundevent |
| `Name` | Soundevent name from the `.vsndevts` manifest |
| `SourceEntityIndex` | Entity the sound emits from; `-1` plays at the listener's own position |
| `StartTime` | Optional start-time offset |
| `Volume` / `Pitch` | Write-only shortcuts for `SetFloat("public.volume"/"public.pitch", ...)` |
| `SetBool/SetInt32/SetUInt32/SetUInt64/SetFloat/SetFloat3(field, value)` | Set a typed field (chainable) |
| `HasField` / `TryGetFloat` / `TryGetInt32` / `TryGetBool` | Inspect fields already set on the builder |
| `Emit(RecipientFilter)` | Sends the sound; returns its GUID |
| `SetParams(uint guid, RecipientFilter)` | Updates a playing sound's params with this builder's fields |
| `Stop(uint guid, RecipientFilter)` | *Static* — stops a specific playing sound |
| `StopByName(string name, int sourceEntityIndex, RecipientFilter)` | *Static* — stops all instances of a soundevent on a source entity |

## Playing from a World Position (point_soundevent)

`EmitSound` and `PlayAt` both anchor to an entity. For a sound that plays from an arbitrary world position, spawn a `point_soundevent` and let it clean itself up:

```csharp
void PlayAt(string soundName, Vector3 position)
{
    var sound = CBaseEntity.CreateByDesignerName("point_soundevent");
    if (sound == null) return;

    var ekv = new CEntityKeyValues();
    ekv.SetString("soundName", soundName);
    ekv.SetBool("startOnSpawn", true);
    ekv.SetVector("origin", position);

    // Auto-remove after playback finishes so we don't leak entities.
    sound.AcceptInput("addoutput", value: "OnSoundFinished>!self>Kill>>0>-1");
    sound.Spawn(ekv);
}
```

## Silent Soundevent Problems

If `EmitSound` appears to return success but you hear nothing:

- **Check the volume on the soundevent itself.** Some internal soundevents are authored at very low dB (`Music.Base.Attack` is `-6` dB). The `volume` parameter multiplies an already-quiet source.
- **Verify with `soundinfo <soundname>` in the console** — it will tell you whether the soundevent resolved to a playable sound and at what volume.
- **Soundevents that are 3D-positional are inaudible when the listener is far from the entity.** Attach to the player pawn or use the global recipe above.

## Finding Soundevent Names

- `soundinfo <name>` in the server console — tests whether a name resolves and prints its sound file
- Extract `soundevents_*.vsndevts_c` with [source2viewer-cli](https://github.com/ValveResourceFormat/ValveResourceFormat) or browse via [s2v.app](https://s2v.app/)
- In-game community references sometimes list popular ones (announcer lines, hero abilities)

## See Also

- [Entities](entities) — `EmitSound` is defined on `CBaseEntity`
- [Precaching](precaching) — Soundevents usually don't need precaching, but the underlying `.vsnd` files sometimes do
