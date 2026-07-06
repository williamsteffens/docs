---
title: "Chat & HUD Messaging"
sidebar_label: "Chat & HUD"
---

# Chat & HUD Messaging Guide

This guide covers sending messages to players through chat, HUD announcements, console output, and world text.

## What You *Can't* Do From the Server

Before diving in, the ceiling is worth knowing — it's asked about often:

- **No custom HUD panels.** Panorama (the Deadlock UI system) is locked down server-side. You can't inject new UI elements, listen for game events from Panorama, or mutate the chrome.
- **No worldtext pinned to the screen.** The camera is not a networked entity. You can only parent to the pawn. However, you can configure the reorient mode of a worldtext to face the camera (billboarding).
- **No per-recipient rendering.** `OnCheckTransmit` lets you hide an entity from specific players, but you can't give the *same* entity different text/color per viewer.
- **No colored minimap lines.** `CCitadelUserMsg_MapLine` renders green only.

What you *can* do:

- **HUD announcements** (`CCitadelUserMsg_HudGameAnnouncement`) — a single title/description popup per-player
- **Chat messages** (`CCitadelUserMsg_ChatMsg`) — supports per-recipient text by sending one message per player
- **Console output** (`PrintToConsole`)
- **World text** (`point_worldtext`) — 3D panels parented to entities
- **Minimap lines** (`CCitadelUserMsg_MapLine`) — short-lived green strokes

## HUD Announcements

Large on-screen announcements visible to targeted players:

```csharp
var msg = new CCitadelUserMsg_HudGameAnnouncement
{
    TitleLocstring = "ANNOUNCEMENT TITLE",
    DescriptionLocstring = "Description text here"
};

// Send to all players
NetMessages.Send(msg, RecipientFilter.All);

// Send to one player
NetMessages.Send(msg, RecipientFilter.Single(playerSlot));
```

## Console Messages

Print to a specific player's console:

```csharp
controller.PrintToConsole("Hello, player!");
```

Print to all players' consoles:

```csharp
CCitadelPlayerController.PrintToConsoleAll("Server message for everyone");
```

Or use `Server.ClientCommand`:

```csharp
Server.ClientCommand(playerSlot, "echo Hello from server!");
```

## Chat Messages

### Intercepting Chat

Override `OnChatMessage` to intercept all chat:

```csharp
public override HookResult OnChatMessage(ChatMessage msg)
{
    Console.WriteLine($"[Chat] Slot {msg.SenderSlot}: {msg.ChatText}");
    return HookResult.Continue;
}
```

### Rebroadcasting Chat

Hook outgoing chat messages for custom formatting:

```csharp
[NetMessageHandler]
public HookResult OnChatMsgOutgoing(OutgoingMessageContext<CCitadelUserMsg_ChatMsg> ctx)
{
    var sender = Players.FromSlot(ctx.Message.PlayerSlot);
    if (sender == null) return HookResult.Handled;

    // Send personalized chat to each recipient
    foreach (var controller in Players.GetAll())
    {
        var personalMsg = new CCitadelUserMsg_ChatMsg
        {
            // Customize per-recipient
            Text = $"[{sender.PlayerName}]: {ctx.Message.Text}"
        };
        NetMessages.Send(personalMsg, RecipientFilter.Single(controller.Slot));
    }

    // Block the original message
    return HookResult.Stop;
}
```

## Sounds

Play sound effects on entities:

```csharp
// Basic sound
pawn.EmitSound("Mystical.Piano.AOE.Warning");

// With parameters
pawn.EmitSound("Damage.Send.Crit", pitch: 100, volume: 0.1f, delay: 0f);
```

For global or positional playback without an entity, see the `Sounds`/`SoundEvent` API in [Sound](../api-reference/sound).

## World Text

3D text panels in the world:

```csharp
var text = CPointWorldText.Create(
    "GAME OVER",
    position: new Vector3(0, 0, 500),
    fontSize: 200f,
    r: 255, g: 0, b: 0, a: 255  // Red text
);

// Update later
text.SetMessage("ROUND 2");

// Clean up
text.Remove();
```

See [World Text API](../api-reference/world-text).

## Targeting Players

### All Players

```csharp
NetMessages.Send(msg, RecipientFilter.All);
```

### Single Player

```csharp
NetMessages.Send(msg, RecipientFilter.Single(slot));
```

### Custom Selection

```csharp
var filter = new RecipientFilter();
foreach (var controller in Players.GetAll())
{
    if (IsEligible(controller))
        filter.Add(controller.Slot);
}
NetMessages.Send(msg, filter);
```

### By Team

```csharp
var filter = new RecipientFilter();
foreach (var controller in Players.GetAll())
{
    var pawn = controller.GetHeroPawn();
    if (pawn?.TeamNum == 2)
        filter.Add(controller.Slot);
}
NetMessages.Send(msg, filter);
```

## See Also

- [Networking API](../api-reference/networking) — Full message sending reference
- [World Text API](../api-reference/world-text) — 3D text panels
- [Players API](../api-reference/players) — Player enumeration
