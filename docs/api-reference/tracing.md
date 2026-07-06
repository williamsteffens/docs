---
title: "Tracing"
sidebar_label: "Tracing"
---

# Tracing

> **Namespace:** `DeadworksManaged.Api`

Cast rays and shapes through the world using the VPhys2 physics system. Useful for hit detection, line of sight, and collision checks.

## Trace (Static Class)

All methods are no-ops if the physics query system is not yet ready.

### Simple Ray Cast

Fire a line ray from start to end:

```csharp
var result = Trace.Ray(
    start: pawn.EyePosition,
    end: pawn.EyePosition + forwardDirection * 1000f,
    mask: MaskTrace.Solid,
    ignore: pawn  // skip the casting entity
);

if (result.DidHit)
{
    Console.WriteLine($"Hit at {result.HitPosition}, fraction: {result.Fraction}");
}
```

### TraceShape (Advanced)

Full control with custom ray shape and filter:

```csharp
var ray = new Ray_t();
// Initialize ray shape...

var filter = new CTraceFilter();
// Configure filter...

var trace = CGameTrace.Create();
Trace.TraceShape(start, end, ray, filter, ref trace);

if (trace.DidHit)
{
    var hitEntity = trace.HitEntity;
    Console.WriteLine($"Hit: {hitEntity?.Classname} at {trace.HitPoint}");
}
```

### SimpleTrace

Convenience method that builds the filter and ray from parameters:

```csharp
var trace = CGameTrace.Create();
Trace.SimpleTrace(
    start, end,
    RayType_t.Line,
    RnQueryObjectSet.All,
    interactWith: MaskTrace.Solid, interactExclude: MaskTrace.Empty, interactAs: MaskTrace.Empty,
    CollisionGroup.Always,
    ref trace,
    filterEntity: pawn,
    filterSecondEntity: null
);
```

### SimpleTraceAngles

Like `SimpleTrace` but uses pitch/yaw angles instead of an end point:

```csharp
var trace = CGameTrace.Create();
Trace.SimpleTraceAngles(
    origin, angles,
    RayType_t.Line,
    RnQueryObjectSet.All,
    interactWith: MaskTrace.Solid, interactExclude: MaskTrace.Empty, interactAs: MaskTrace.Empty,
    CollisionGroup.Always,
    ref trace,
    filterEntity: pawn,
    filterSecondEntity: null,
    maxDistance: 8192f  // default
);
```

## Trace Methods Summary

| Method | Description |
|--------|-------------|
| `Ray(Vector3 start, Vector3 end, MaskTrace mask = Solid\|Hitbox, CBaseEntity? ignore = null)` | Simple line ray, returns `TraceResult` |
| `TraceShape(Vector3, Vector3, Ray_t, CTraceFilter, ref CGameTrace)` | Full shape trace with custom ray and filter |
| `SimpleTrace(start, end, rayKind, objectQuery, interactWith, interactExclude, interactAs, collision, ref trace, filterEntity?, filterSecondEntity?)` | Convenience with individual parameters |
| `SimpleTraceAngles(start, angles, rayKind, objectQuery, interactWith, interactExclude, interactAs, collision, ref trace, filterEntity?, filterSecondEntity?, maxDistance = 8192f)` | Like SimpleTrace but with angles and max distance |

## TraceResult

Simplified result from `Trace.Ray()`:

| Property | Type | Description |
|----------|------|-------------|
| `DidHit` | `bool` | Whether the ray hit something |
| `HitPosition` | `Vector3` | World position of the hit |
| `Fraction` | `float` | 0.0 (at start) to 1.0 (at end) — how far the ray traveled |
| `Trace` | `CGameTrace` | Full trace data |

## CGameTrace

Full result of a VPhys2 shape trace:

| Member | Type | Description |
|--------|------|-------------|
| `DidHit` | `bool` | Whether the trace hit something |
| `HitPoint` | `Vector3` | World position of the hit |
| `HitNormal` | `Vector3` | Surface normal at hit point |
| `HitEntity` | `CBaseEntity?` | Entity that was hit |

## Ray Shapes

### RayType_t

Shape type for trace queries:

| Value | Description |
|-------|-------------|
| Line | Line ray (default) |
| Sphere | Sphere shape |
| Hull | AABB hull |
| Capsule | Capsule shape |

### Shape Data Structs

| Struct | Description |
|--------|-------------|
| `LineTrace` | Line ray with optional radius (swept sphere) |
| `SphereTrace` | Sphere at a fixed center point |
| `HullTrace` | AABB hull swept along a ray |
| `CapsuleTrace` | Capsule between two center points with radius |
| `MeshTrace` | Convex mesh with bounds and vertices |

## Collision Filtering

### CollisionGroup

Determines which objects interact in physics simulation.

### InteractionLayer

Individual content/interaction layers for building trace bitmasks.

### MaskTrace

Bitmask combining `InteractionLayer` values:

| Value | Description |
|-------|-------------|
| `Solid` | Solid world geometry |
| *(others)* | Various content layer combinations |

### RnQueryObjectSet

Controls which object sets are included:

| Value | Description |
|-------|-------------|
| `All` | All object sets (static, dynamic, locatable) |

### CTraceFilter

Full trace filter with entity-aware filtering:

```csharp
var filter = new CTraceFilter(true) {
    IterateEntities = true,
    QueryShapeAttributes = new RnQueryShapeAttr_t {
        ObjectSetMask = RnQueryObjectSet.All,
        InteractsWith = MaskTrace.Solid,
        InteractsExclude = MaskTrace.Empty,
        InteractsAs = MaskTrace.Empty,
        CollisionGroup = CollisionGroup.CitadelBullet,
        HitSolid = true,
    }
};
// Ignore specific entities by index (requires unsafe)
unsafe {
    filter.QueryShapeAttributes.EntityIdsToIgnore[0] = (uint)pawn.EntityIndex;
}
```

### RnQueryShapeAttr_t

Query attributes for shape traces:

| Property | Type | Description |
|----------|------|-------------|
| `ObjectSetMask` | `RnQueryObjectSet` | Which object sets to query |
| `InteractsWith` | `MaskTrace` | What the ray interacts with |
| `InteractsExclude` | `MaskTrace` | What to exclude from interaction |
| `InteractsAs` | `MaskTrace` | What the ray acts as |
| `CollisionGroup` | `CollisionGroup` | Collision group for filtering |
| `HitSolid` | `bool` | Whether to hit solid objects |
| `HitTrigger` | `bool` | Whether to hit trigger volumes |
| `EntityIdsToIgnore` | `fixed uint[2]` | Up to 2 entity indices to skip (requires `unsafe`) |

## Example: Line-of-Sight Check

Bullet-style LOS check that ignores trigger volumes and specific entities:

```csharp
var trace = CGameTrace.Create();
var ray = new Ray_t { Type = RayType_t.Line };
var filter = new CTraceFilter(true) {
    IterateEntities = true,
    QueryShapeAttributes = new RnQueryShapeAttr_t {
        ObjectSetMask = RnQueryObjectSet.All,
        InteractsWith = MaskTrace.Solid,
        InteractsExclude = MaskTrace.Empty,
        InteractsAs = MaskTrace.Empty,
        CollisionGroup = CollisionGroup.CitadelBullet,
        HitSolid = true,
    }
};
unsafe {
    filter.QueryShapeAttributes.EntityIdsToIgnore[0] = (uint)sourcePawn.EntityIndex;
    filter.QueryShapeAttributes.EntityIdsToIgnore[1] = (uint)targetPawn.EntityIndex;
}
Trace.TraceShape(sourcePawn.EyePosition, targetPawn.EyePosition, ray, filter, ref trace);

bool hasLos = !trace.DidHit;
// trace.HitEntity?.Classname tells you what blocked (e.g. "CWorld" for walls)
```

:::note
`EntityIdsToIgnore` is a fixed-size buffer — accessing it requires `unsafe` code and `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>` in your `.csproj`.
:::

## Computing a Forward Vector

Deadworks doesn't ship a built-in angles-to-vector helper. Use this conversion — it matches the Source engine convention where pitch is negated because positive Z is up:

```csharp
static Vector3 ForwardFromAngles(Vector3 angles)
{
    float pitch = angles.X * MathF.PI / 180f;
    float yaw   = angles.Y * MathF.PI / 180f;
    return new Vector3(
        MathF.Cos(pitch) * MathF.Cos(yaw),
        MathF.Cos(pitch) * MathF.Sin(yaw),
        -MathF.Sin(pitch));
}
```

## Example: Player Eye Trace

```csharp
[Command("trace")]
public void CmdTrace(CCitadelPlayerController caller)
{
    var pawn = caller.GetHeroPawn();
    if (pawn == null) return;

    var result = Trace.Ray(
        pawn.EyePosition,
        pawn.EyePosition + ForwardFromAngles(pawn.EyeAngles) * 5000f,
        MaskTrace.Solid,
        pawn
    );

    if (result.DidHit)
    {
        caller.PrintToConsole($"Hit at {result.HitPosition}");
        caller.PrintToConsole($"Distance: {result.Fraction * 5000f} units");
    }
}
```

## See Also

- [Entities](entities) — Entity references in trace results
- [Players](players) — `EyePosition`, `EyeAngles`, `ViewAngles`
