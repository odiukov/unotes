---
tags:
  - attribute
---
#### Description
- **Performance compilation attribute** that compiles systems and jobs to highly optimized native code using the Burst compiler

- Applied to [[ISystem]] structs, job structs ([[IJobEntity]], [[IJobChunk]], etc.), and their methods for maximum performance

- Provides **10-100x performance improvement** over standard C# through SIMD vectorization, optimizations, and removal of managed code overhead

- Requires code to be **safe C# subset** - no managed objects, garbage collection, or exceptions in Burst-compiled code

#### Example
```csharp
// System with BurstCompile
[BurstCompile]
public partial struct HealthRegenSystem : ISystem
{
    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Burst-compiled initialization
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Burst-compiled update - runs at maximum performance
        foreach (var (health, regen) in
            SystemAPI.Query<RefRW<Health>, RefRO<HealthRegen>>())
        {
            health.ValueRW.Current += regen.ValueRO.PointPerSec * SystemAPI.Time.DeltaTime;
        }
    }
}

// IJobEntity with BurstCompile
[BurstCompile]
public partial struct ProjectileMoveJob : IJobEntity
{
    public float DeltaTime;
    [ReadOnly] public ComponentLookup<LocalToWorld> Positions;

    // Execute is automatically Burst-compiled when job struct has [BurstCompile]
    private void Execute(ProjectileAspect projectile)
    {
        projectile.Move(DeltaTime, Positions);
    }
}

// Burst options for fine-tuning
[BurstCompile(CompileSynchronously = true)]  // Compile immediately (Editor only, for testing)
public partial struct DebugSystem : ISystem
{
    public void OnUpdate(ref SystemState state) { }
}

[BurstCompile(FloatMode = FloatMode.Fast)]  // Faster but less precise math
public partial struct FastMathJob : IJobEntity
{
    private void Execute(ref Position pos, in Velocity vel) { }
}
```

#### Pros
- **Massive performance gains** - 10-100x faster than equivalent C# code for data processing

- **SIMD auto-vectorization** - compiler automatically uses CPU vector instructions (SSE, AVX, NEON)

- **Zero garbage collection** - no GC allocations or pauses, perfect for real-time gameplay

- **Multi-platform optimization** - generates optimal code for each target platform automatically

- **Compile-time safety** - errors if you use unsupported features (managed objects, exceptions)

#### Cons
- **Restricted C# subset** - can't use managed objects, strings, LINQ, exceptions, virtual calls

- **Longer compile times** - Burst compilation adds time to builds and domain reloads

- **Debugging complexity** - can be harder to debug than regular C#, though Unity provides Burst Inspector

- **Learning curve** - need to understand what's allowed/disallowed in Burst-compiled code

#### Best use
- **All [[ISystem]] systems** - nearly always use `[BurstCompile]` on systems for maximum performance

- **All jobs** - [[IJobEntity]], [[IJobChunk]], [[ITriggerEventsJob]] should be Burst-compiled

- **Performance-critical code** - math-heavy, data transformation, AI pathfinding, physics queries

#### Avoid if
- **Managed component access** - systems using [[Managed IComponentData (class components)|managed components]] with `SystemAPI.ManagedAPI` can't be fully Burst-compiled

- **Debug logging in jobs** - Burst doesn't support `Debug.Log` or exceptions, use in non-Burst code only

- **Prototyping phase** - during early prototyping, you might skip Burst for faster iteration, add later for performance

#### Extra tip
- **Required for [[ISystem]]** - `ISystem` is designed for Burst, always use `[BurstCompile]` on the struct and all lifecycle methods (OnCreate, OnUpdate, OnDestroy)

- **CompileSynchronously option** - use `[BurstCompile(CompileSynchronously = true)]` in Editor for immediate compilation to test performance (build times increase, only for testing)

- **Burst Inspector** - use Window > Burst > Burst Inspector to see generated assembly code and verify optimizations

- **Safety checks** - Burst enables safety checks (bounds checking, null checks) by default in Editor, disabled in builds for maximum performance

- **FloatMode options:**
  - `FloatMode.Default` - precise IEEE 754 compliance
  - `FloatMode.Fast` - faster math, less precise (allows optimizations like reciprocal approximations)
  - `FloatMode.Strict` - strictest IEEE compliance

- **OptimizeFor option** - use `OptimizeFor.FastCompilation` to speed up iteration in Editor, `OptimizeFor.Performance` for final builds (default)

- **Allowed in Burst:**
  - Blittable types (primitives, structs with primitives)
  - [[IComponentData]], [[IBufferElementData (dynamic buffers)|DynamicBuffer]], [[BlobAsset (immutable data)|BlobAsset]]
  - Math operations (Unity.Mathematics)
  - NativeArray, NativeList, and other native collections
  - [[ComponentLookup and BufferLookup|ComponentLookup/BufferLookup]]

- **Not allowed in Burst:**
  - Managed objects (class instances, strings)
  - Garbage collection allocations
  - Exceptions (try/catch/throw)
  - Virtual method calls
  - `Debug.Log` (use Burst-compatible logging alternatives)
