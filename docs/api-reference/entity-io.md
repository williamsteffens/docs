---
title: "Entity I/O"
sidebar_label: "Entity I/O"
---

# Entity I/O

> **Namespace:** `DeadworksManaged.Api`

Hook into the Valve entity I/O system to observe or intercept output firings and input dispatches by entity designer name.

## EntityIO

### Hooking Outputs

Subscribe to outputs fired by entities with a given designer name:

```csharp
var handle = EntityIO.HookOutput("trigger_multiple", "OnTrigger", ev =>
{
    Console.WriteLine($"Output {ev.OutputName} fired by {ev.CallerClass} (entity {ev.Caller?.EntityIndex})");
    Console.WriteLine($"Activator: {ev.Activator?.Classname}");
});

// Cancel later
handle.Cancel();
```

### Hooking Inputs

Subscribe to inputs dispatched to entities:

```csharp
var handle = EntityIO.HookInput("func_button", "Kill", ev =>
{
    Console.WriteLine($"Kill input received by {ev.Entity.Classname}");
    Console.WriteLine($"Value: {ev.Value.AsString()}");
});
```

### Blocking Inputs and Outputs

Each hook has two overloads. Pass a handler that returns `HookResult` to run **pre** (before the original) — returning `HookResult.Stop` blocks the input/output entirely. Pass an `Action<...>` handler to run **post** (observe-only, after the original):

```csharp
// Pre hook: veto the Kill input so the button never kills its activator
EntityIO.HookInput("func_button", "Kill", ev =>
{
    return HookResult.Stop;
});

// Post hook: observe only, cannot block
EntityIO.HookInput("func_button", "Kill", ev =>
{
    Console.WriteLine("Kill input already processed");
});
```

Class and input/output names support `"*"` as a wildcard. Hooks are matched against four keys in priority order: `(class, name)`, `(class, "*")`, `("*", name)`, `("*", "*")`.

### Methods

| Method | Returns | Description |
|--------|---------|-------------|
| `HookOutput(string className, string outputName, Func<EntityOutputEvent, HookResult> handler, HookMode mode = HookMode.Pre)` | `IHandle` | Pre-mode hook; return `HookResult.Stop` to block the output |
| `HookOutput(string className, string outputName, Action<EntityOutputEvent> handler)` | `IHandle` | Post-mode hook; observe-only |
| `HookInput(string className, string inputName, Func<EntityInputEvent, HookResult> handler, HookMode mode = HookMode.Pre)` | `IHandle` | Pre-mode hook; return `HookResult.Stop` to block the input |
| `HookInput(string className, string inputName, Action<EntityInputEvent> handler)` | `IHandle` | Post-mode hook; observe-only |

All return an `IHandle` that cancels the hook when cancelled. Hooks are also auto-cancelled on plugin unload.

### Attribute Registration

Instead of calling `HookOutput`/`HookInput` in `OnLoad`, you can annotate plugin methods — they are auto-registered on load and dropped on unload. Pre handlers must return `HookResult`; post handlers must return `void`:

```csharp
[EntityOutputHook("trigger_multiple", "OnStartTouch")]
public HookResult OnTriggerTouched(EntityOutputEvent e)
{
    return HookResult.Continue;
}

[EntityInputHook("*", "Toggle", HookMode.Post)]
public void OnToggleObserved(EntityInputEvent e) { /* observer only */ }
```

:::caution Limited Entity I/O Support
Entity I/O hooks work with map-placed entities that use the Source 2 I/O system (e.g. `trigger_multiple`, `func_button`). Player entities (`"player"`) do not fire standard I/O outputs like `"OnDeath"` — use `OnTakeDamage` or `GameEvents.AddListener("player_death")` instead for player death detection.
:::

## EntityOutputEvent

Data for a fired entity output.

| Property | Type | Description |
|----------|------|-------------|
| `CallerClass` | `string` | Designer name of the entity firing the output (e.g. `"trigger_multiple"`) |
| `OutputName` | `string` | Name of the output being fired (e.g. `"OnTrigger"`) |
| `Activator` | `CBaseEntity?` | The entity that triggered the output (typically a player pawn) |
| `Caller` | `CBaseEntity?` | The entity that fired the output |
| `Value` | `EntityIOValue` | The typed variant value carried by the output |
| `Delay` | `float` | Delay (seconds) before the output's connected inputs fire |

## EntityInputEvent

Data for a received entity input.

| Property | Type | Description |
|----------|------|-------------|
| `Entity` | `CBaseEntity` | The entity receiving the input |
| `ClassName` | `string` | Designer name of the receiving entity (cheaper than reading `Entity.DesignerName`) |
| `InputName` | `string` | Name of the input (e.g. `"Kill"`) |
| `Activator` | `CBaseEntity?` | The activating entity, if any |
| `Caller` | `CBaseEntity?` | The calling entity, if any |
| `Value` | `EntityIOValue` | The typed variant value carried by the input |

## EntityIOValue

Typed accessor for the variant value carried by an input or output. Call the accessor matching the value's type — or `AsString()`, which formats any variant type:

| Member | Type / Returns | Description |
|--------|----------------|-------------|
| `Type` | `FieldType` | Source 2 field type of the variant (`FieldType.Void` for null) |
| `IsNull` | `bool` | True if the variant is null or holds no value |
| `AsString()` | `string` | Formats any variant type as a string |
| `AsInt()` / `AsInt64()` / `AsUInt()` | `int` / `long` / `uint` | Integer conversions; `0` if unsupported |
| `AsFloat()` / `AsDouble()` | `float` / `double` | Float conversions; `0` if unsupported |
| `AsBool()` | `bool` | Boolean conversion |
| `AsVector2()` / `AsVector3()` / `AsVector4()` | `Vector2/3/4` | Vector reads; zero for other types |
| `AsColor()` | `Color` | RGBA color; transparent black for non-color variants |
| `AsEntity()` | `CBaseEntity?` | Resolves ehandle variants; null otherwise |

:::warning Do not retain past the callback
`EntityIOValue` wraps a native pointer that is only valid for the duration of the hook callback. If you need the value later, call the typed accessor (e.g. `AsString()`) inside the callback and store the result.
:::

## AcceptInput

You can also fire inputs directly on entities. See [Entities](entities):

```csharp
entity.AcceptInput("Start", activator: null, caller: null, value: "");
entity.AcceptInput("Kill", activator: null, caller: null, value: "");
```

## See Also

- [Entities](entities) — `AcceptInput` method
