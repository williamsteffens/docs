---
title: "ConVars"
sidebar_label: "ConVars"
---

# ConVars

> **Namespace:** `DeadworksManaged.Api`

A ConVar is a named setting you can read or change from the console.

Use `[ConVar]` when you want a simple plugin setting that server admins can tweak without rebuilding your plugin.

For custom ConVars, start the name with `dw_`.

## Quick Start

```csharp
using DeadworksManaged.Api;

namespace MyPlugin;

public class MyPlugin : DeadworksPluginBase
{
    [ConVar("dw_my_plugin_enabled", Description = "Turn the plugin on or off")]
    public bool Enabled { get; set; } = true;

    [ConVar("dw_my_plugin_damage", Description = "Damage multiplier", ServerOnly = true)]
    public float DamageMultiplier { get; set; } = 1.0f;
}
```

That gives you console settings named:

- `dw_my_plugin_enabled`
- `dw_my_plugin_damage`

If you type the name by itself, Deadworks prints the current value.

If you type the name followed by a value, Deadworks updates it.

For example:

- `dw_my_plugin_enabled`
- `dw_my_plugin_enabled false`
- `dw_my_plugin_damage 1.5`

## ConVarAttribute

Use `[ConVar]` on a property in your plugin.

Supported property types are:

- `int`
- `float`
- `bool`
- `string`

| Setting | Type | Description |
|---------|------|-------------|
| `Name` | `string` | The console name, like `dw_my_plugin_enabled` |
| `Description` | `string` | Short help text shown in console |
| `ServerOnly` | `bool` | Only let the server console change it |

## ConVar Class

Use `ConVar` when you want to read or change console variables from code.

This is useful when you want to change built-in game settings in `OnStartupServer`.

| Method | Returns | Description |
|--------|---------|-------------|
| `Find(string name)` | `ConVar?` | Find an existing ConVar |
| `Create(string name, string defaultValue, string description = "", bool serverOnly = false)` | `ConVar?` | Create a new ConVar |
| `GetInt()` | `int` | Read the value as an integer |
| `GetFloat()` | `float` | Read the value as a float |
| `GetString()` | `string` | Read the value as a string |
| `SetInt(int value)` | `void` | Set the value as an integer |
| `SetFloat(float value)` | `void` | Set the value as a float |
| `IsValid` | `bool` | Whether the underlying native ConVar is alive |

```csharp
public override void OnStartupServer()
{
    ConVar.Find("citadel_allow_duplicate_heroes")?.SetInt(1);
    ConVar.Find("citadel_player_starting_gold")?.SetInt(0);
    ConVar.Find("citadel_voice_all_talk")?.SetInt(1);
}
```
