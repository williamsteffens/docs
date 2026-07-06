---
title: "World Text"
sidebar_label: "World Text"
---

# World Text

> **Namespace:** `DeadworksManaged.Api`

Display text in the 3D world or anchored to a player's screen.

## CPointWorldText

Wraps the `point_worldtext` entity — a world-space text panel rendered in 3D.

### Creating World Text

```csharp
var text = CPointWorldText.Create(
    message: "Hello World",
    position: new Vector3(100, 200, 300),
    fontSize: 100f,
    worldUnitsPerPx: null,     // null = auto-calculated from fontSize
    fontName: null,            // null = default font
    r: 255, g: 255, b: 255, a: 255,  // RGBA color
    reorientMode: 0            // 0 = no reorientation
);
```

### Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `message` | `string` | — | Text to display |
| `position` | `Vector3` | — | World-space position |
| `fontSize` | `float` | `100f` | Font size in pixels |
| `worldUnitsPerPx` | `float?` | `null` | World units per pixel (`null` = auto-calculated as `0.25 / 1050 * fontSize`) |
| `fontName` | `string?` | `null` | Font name (`null` = default, max 63 bytes UTF-8) |
| `r, g, b, a` | `byte` | `255` | RGBA color components |
| `reorientMode` | `int` | `0` | Reorientation mode (0 = none) |

### Instance Properties

| Property | Type | Description |
|----------|------|-------------|
| `Enabled` | `bool` | Enable/disable the text entity |
| `Fullbright` | `bool` | Whether text is fullbright (ignores lighting) |
| `FontSize` | `float` | Font size in pixels |
| `WorldUnitsPerPx` | `float` | World units per pixel |
| `DepthOffset` | `float` | Depth offset for rendering |
| `FontName` | `string` | Font name (max 63 bytes UTF-8) |
| `ColorABGR` | `uint` | Color as ABGR uint32 |
| `JustifyHorizontal` | `HorizontalJustify` | Horizontal text justification |
| `JustifyVertical` | `VerticalJustify` | Vertical text justification |

### Updating Text

```csharp
text.SetMessage("Updated text!");
text.SetColor(255, 0, 0);           // RGB (alpha defaults to 255)
text.SetColor(255, 0, 0, 128);      // RGBA
```

### Cleanup

```csharp
text.Remove();  // Inherited from CBaseEntity
```

### Positioning Tips

| Parameter | Recommended Value | Notes |
|-----------|-------------------|-------|
| `fwdOffset` | `-50f` | Places text close to camera (behind eye position) |
| `posX` | `2.341f` | Horizontal center (accounting for shoulder camera) |
| `posY` | `2.6f` | Vertical center |
| `PixelSize` | `50f` | Good size for pixel art blocks |
| `SpacingX` | `0.023f` | Horizontal character spacing |
| `SpacingY` | `0.06f` | Vertical character spacing (~2.6x taller than wide for "█") |

:::tip Pixel Art
Use the full block character `"█"` for pixel-art style displays. The `"."` character is invisible at small sizes. The block character is approximately 2.6x taller than wide, so adjust `SpacingX` and `SpacingY` accordingly for square pixels.
:::

## The EKV Workflow (Full Control)

`CPointWorldText.Create` covers the common case. When you need access to keys the helper doesn't expose (reorientation, justify, depth offset, font name, fullbright), spawn the entity directly with `CEntityKeyValues`:

```csharp
var text = CBaseEntity.CreateByDesignerName("point_worldtext");
if (text == null) return;

var ekv = new CEntityKeyValues();
ekv.SetString("message_text", controller.PlayerName);
ekv.SetInt("font_size", 128);
ekv.SetString("font_name", "Comic Sans MS");
ekv.SetFloat("world_units_per_pixel", 0.15f);
ekv.SetInt("enabled", 1);
ekv.SetColor("color", 255, 255, 255, 255);
ekv.SetFloat("depth_render_offset", 0.125f);
ekv.SetInt("justify_horizontal", 1);   // 0 left, 1 center, 2 right
ekv.SetInt("reorient_mode", 1);        // 1 = always face the camera (around up axis)
ekv.SetInt("fullbright", 1);
text.Spawn(ekv);

// Belt and braces: also push the text via the SetMessage input after spawn,
// which is what CPointWorldText.Create does internally.
text.AcceptInput("SetMessage", value: controller.PlayerName);

text.Teleport(pawn.Position + new Vector3(0, 0, 96), new Vector3(0, 180, 90));
text.SetParent(pawn);
```

A few gotchas worth knowing before you burn hours on them:

- **The EKV key is `message_text`, not `message`.** The SDK's own `Create` helper sets `message_text` and additionally fires `SetMessage` after spawn.
- **The EKV key is `font_name`, not `font`.** Passing `font` silently does nothing.
- **`fullbright: 1` is required for the color to show unfiltered** — without it, text is tinted by world lighting and often reads as dim gray indoors.
- **`reorient_mode: 1`** rotates the text around its up axis so it faces each viewer's camera. Mode `0` is a fixed-orientation billboard.
- **`depth_render_offset`** nudges the text toward the camera in world units — use a small positive value (`0.125`) to push text in front of geometry it would otherwise z-fight with.
- **Fonts are looked up from the player's OS.** Deadlock does not ship fonts as usable by this entity. Stick to Windows 10 default fonts (Arial, Segoe UI, Consolas, Comic Sans MS, Verdana, …). Custom font files in the game's `pak01_dir` do **not** work.

## Nametag Pattern

Parent a worldtext to the pawn and it follows them around, bright side facing camera:

```csharp
public override void OnClientFullConnect(ClientFullConnectEvent args)
{
    var pawn = args.Controller?.GetHeroPawn();
    if (pawn == null) return;

    var text = CBaseEntity.CreateByDesignerName("point_worldtext");
    var ekv = new CEntityKeyValues();
    ekv.SetString("message_text", args.Controller.PlayerName);
    ekv.SetInt("font_size", 128);
    ekv.SetFloat("world_units_per_pixel", 0.12f);
    ekv.SetInt("justify_horizontal", 1);
    ekv.SetInt("reorient_mode", 1);
    ekv.SetInt("fullbright", 1);
    ekv.SetColor("color", 255, 255, 255, 255);
    text.Spawn(ekv);
    text.AcceptInput("SetMessage", value: args.Controller.PlayerName);

    text.Teleport(pawn.Position + new Vector3(0, 0, 96), new Vector3(0, 180, 90));
    text.SetParent(pawn);
    // No need to track: when the pawn is removed, parented children die with it.
}
```

## Attaching to the Camera

You **can't** parent a worldtext to the player's camera directly. The camera is not a networked entity, and the player controller sits at `(0,0,0)` on the server. Common workarounds:

- Attach to the pawn and accept that the text moves with the model (fine for nametags, timers, debug overlays)
- Recompute position every tick in `OnGameFrame` based on `EyePosition + forward * distance` (visible stutter because client prediction can't help)
- Use `CCitadelUserMsg_HudGameAnnouncement` for real HUD text (see [chat-and-hud](../guides/chat-and-hud))

## See Also

- [Entities](entities) — Base entity operations (`Remove`, `SetParent`)
- [Networking](networking) — HUD announcements as an alternative
- [Chat & HUD Messaging](../guides/chat-and-hud) — When to use worldtext vs announcements
