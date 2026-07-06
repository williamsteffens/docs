---
title: "Networking"
sidebar_label: "Networking"
---

# Networking

> **Namespace:** `DeadworksManaged.Api`

Send and intercept Source 2 network messages between server and clients.

## NetMessages

Entry point for sending and hooking protobuf network messages.

### Sending Messages

```csharp
// Create a protobuf message
var msg = new CCitadelUserMsg_HudGameAnnouncement
{
    TitleLocstring = "GAME OVER",
    DescriptionLocstring = "The match has ended"
};

// Send to all players
NetMessages.Send(msg, RecipientFilter.All);

// Send to one player
NetMessages.Send(msg, RecipientFilter.Single(playerSlot));
```

### Hooking Outgoing Messages (Server → Client)

```csharp
// Hook before a message is sent to clients
NetMessages.HookOutgoing<CCitadelUserMsg_ChatMsg>(ctx =>
{
    // ctx.Message — the protobuf message
    // ctx.Recipients — modifiable recipient set
    // ctx.MessageId — numeric message ID

    // Modify recipients. RecipientFilter is a struct, so mutate a copy
    // and assign it back — calling Remove on ctx.Recipients directly
    // would only change a temporary copy.
    var recipients = ctx.Recipients;
    recipients.Remove(someSlot);
    ctx.Recipients = recipients;

    return HookResult.Handled;
});
```

### Hooking Incoming Messages (Client → Server)

```csharp
// Hook when server receives a message from a client
NetMessages.HookIncoming<CCitadelUserMsg_ChatMsg>(ctx =>
{
    // ctx.Message — the protobuf message from client
    // ctx.SenderSlot — who sent it
    // ctx.MessageId — numeric message ID

    return HookResult.Handled;
});
```

### Unhooking

```csharp
NetMessages.UnhookOutgoing<CCitadelUserMsg_ChatMsg>(myHandler);
NetMessages.UnhookIncoming<CCitadelUserMsg_ChatMsg>(myHandler);
```

### Using the Attribute (Alternative)

Any registered message type can also be hooked with the `[NetMessageHandler]` attribute — the direction is inferred from the parameter type (`OutgoingMessageContext<T>` or `IncomingMessageContext<T>`):

```csharp
[NetMessageHandler]
public HookResult OnChatMsgOutgoing(OutgoingMessageContext<CCitadelUserMsg_ChatMsg> ctx)
{
    // Process outgoing chat
    return HookResult.Handled;
}
```

## RecipientFilter

Bitmask of player slots that should receive a message.

### Static Members

| Member | Description |
|--------|-------------|
| `RecipientFilter.All` | A filter targeting all 64 possible player slots |
| `RecipientFilter.Single(int slot)` | A filter targeting exactly one player |

### Instance Methods

| Method | Description |
|--------|-------------|
| `Add(int slot)` | Adds a player slot |
| `Remove(int slot)` | Removes a player slot |
| `HasRecipient(int slot)` | Returns `true` if slot is included |

All slot parameters are **0-based player slots** (`controller.Slot`, which equals `EntityIndex - 1`) — not entity indices.

### Properties

| Property | Type | Description |
|----------|------|-------------|
| `Mask` | `ulong` | Raw bitmask where bit i indicates slot i is included |

### Example: Custom Recipient List

```csharp
var filter = new RecipientFilter();
foreach (var controller in Players.GetAll())
{
    if (ShouldReceive(controller))
        filter.Add(controller.Slot);
}
NetMessages.Send(msg, filter);
```

## Message Contexts

### OutgoingMessageContext\<T\>

Carries a server→client message with destination recipients.

| Property | Type | Description |
|----------|------|-------------|
| `Message` | `T` | The protobuf message being sent |
| `Recipients` | `RecipientFilter` | Target players (modifiable) |
| `MessageId` | `int` | Numeric network message ID |

### IncomingMessageContext\<T\>

Carries a client→server message with sender info.

| Property | Type | Description |
|----------|------|-------------|
| `Message` | `T` | The protobuf message from client |
| `SenderSlot` | `int` | Player slot of the sender |
| `MessageId` | `int` | Numeric network message ID |

## NetMessageRegistry

Maps protobuf message types to network message IDs.

| Method | Returns | Description |
|--------|---------|-------------|
| `GetMessageId<T>()` | `int` | Message ID for type T, or `-1` |
| `GetMessageId(Type)` | `int` | Message ID for protobuf type, or `-1` |
| `RegisterManual<T>(int)` | `void` | Manually register type with specific ID |

## Common Message Types

### Set Client Camera Angles

Forces a player's camera to look in a specific direction. This is the only confirmed working method for server-side camera control — schema writes and Teleport angles only affect the hero model, not the client camera.

```csharp
NetMessages.Send(new CCitadelUserMsg_SetClientCameraAngles {
    PlayerSlot = slot,                    // target player slot (int)
    CameraAngles = new CMsgQAngle {
        X = pitch,                        // vertical angle (negative = look up)
        Y = yaw,                          // horizontal angle
        Z = 0                             // roll (usually 0)
    }
}, RecipientFilter.Single(slot));
```

### HUD Announcement Example

```csharp
var msg = new CCitadelUserMsg_HudGameAnnouncement
{
    TitleLocstring = "ANNOUNCEMENT TITLE",
    DescriptionLocstring = "Description text here"
};
NetMessages.Send(msg, RecipientFilter.All);
```

## See Also

- [Commands](commands) — Command registration and chat-triggered actions
- [Chat and HUD Guide](../guides/chat-and-hud) — Intercepting chat and sending HUD messages
