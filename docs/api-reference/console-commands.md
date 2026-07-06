---
title: "Console Commands"
---

# Console Commands

> **Namespace:** `DeadworksManaged.Api`

Use [`[Command]`](commands) when you want to create a console command in your plugin.

If you name the command `heal`, Deadworks gives you:

- `/heal` in chat
- `!heal` in chat
- `dw_heal` in the console

The console version always starts with `dw_`.

If you only want the console version and do not want chat commands, set `ConsoleOnly = true`:

```csharp
[Command("heal", ConsoleOnly = true)]
public void CmdHeal()
{
    Console.WriteLine("Healing command ran.");
}
```

For plugin settings, see [ConVars](convars).
