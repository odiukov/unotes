---
tags:
  - component
  - transform
  - rendering
---

#### Description
- **Read-only 4x4 transformation matrix** computed by `LocalToWorldSystem` from [[LocalTransform]] and parent hierarchy - the final world-space transform used by Unity's rendering system

- **Automatically updated** each frame by `TransformSystemGroup` - combines LocalTransform with [[Parent]] hierarchy and [[PostTransformMatrix]] if present

- **Required for rendering** - Unity's Entities Graphics package reads LocalToWorld matrices for rendering entities, not [[LocalTransform]] directly

- **Never write directly** - systems should modify [[LocalTransform]] instead, LocalToWorld is automatically recomputed next frame

#### Example
```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

// Reading LocalToWorld (never write to it)
[BurstCompile]
public partial struct RenderingSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Read LocalToWorld for rendering or world-space queries
        foreach (var (localToWorld, entity) in
            SystemAPI.Query<RefRO<LocalToWorld>>()
                .WithEntityAccess())
        {
            // Extract world position from matrix
            float3 worldPosition = localToWorld.ValueRO.Position;

            // Extract rotation from matrix
            quaternion worldRotation = localToWorld.ValueRO.Rotation;

            // Get forward vector in world space
            float3 worldForward = localToWorld.ValueRO.Forward;

            // Full 4x4 matrix for rendering
            float4x4 matrix = localToWorld.ValueRO.Value;
        }
    }
}

// Wrong: Don't modify LocalToWorld directly
[BurstCompile]
public partial struct WrongSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // WRONG - changes will be overwritten next frame
        foreach (var localToWorld in SystemAPI.Query<RefRW<LocalToWorld>>())
        {
            // Don't do this!
            localToWorld.ValueRW = float4x4.Translate(new float3(1, 0, 0));
        }
    }
}

// Correct: Modify LocalTransform instead
[BurstCompile]
public partial struct CorrectSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Correct - modify LocalTransform, LocalToWorld updates automatically
        foreach (var transform in SystemAPI.Query<RefRW<LocalTransform>>())
        {
            transform.ValueRW.Position += new float3(1, 0, 0);
        }
        // LocalToWorldSystem will compute new LocalToWorld from updated LocalTransform
    }
}

// Static entities with LocalToWorld only (no LocalTransform)
public partial class StaticSetupSystem : SystemBase
{
    protected override void OnUpdate()
    {
        EntityManager em = EntityManager;

        // Create static entity with only LocalToWorld (optimization)
        Entity staticEntity = em.CreateEntity(typeof(LocalToWorld));

        // Set world-space transform once
        em.SetComponentData(staticEntity, new LocalToWorld
        {
            Value = float4x4.TRS(
                new float3(10, 0, 5),        // Position
                quaternion.identity,         // Rotation
                new float3(1, 1, 1)         // Scale
            )
        });

        // Add Static tag to prevent systems from processing
        em.AddComponent<Static>(staticEntity);

        // LocalToWorld won't be updated since no LocalTransform exists
    }
}

// Using LocalToWorld for world-space queries
[BurstCompile]
public partial struct ProximitySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float3 targetPosition = new float3(0, 0, 0);

        foreach (var (localToWorld, entity) in
            SystemAPI.Query<RefRO<LocalToWorld>>()
                .WithEntityAccess())
        {
            // Get world position for distance check
            float3 worldPos = localToWorld.ValueRO.Position;
            float distance = math.distance(worldPos, targetPosition);

            if (distance < 10f)
            {
                // Entity within range
            }
        }
    }
}

// Baking - TransformUsageFlags determines which components added
class TransformAuthoring : MonoBehaviour
{
    class Baker : Baker<TransformAuthoring>
    {
        public override void Bake(TransformAuthoring authoring)
        {
            // TransformUsageFlags.Dynamic → LocalTransform + LocalToWorld
            var dynamicEntity = GetEntity(TransformUsageFlags.Dynamic);

            // TransformUsageFlags.WorldSpace → LocalToWorld only (static)
            var staticEntity = GetEntity(TransformUsageFlags.WorldSpace);

            // TransformUsageFlags.None → No transform components
            var noneEntity = GetEntity(TransformUsageFlags.None);
        }
    }
}
```

#### Pros
- **Rendering ready** - directly usable by Entities Graphics package without additional computation

- **World-space access** - provides final world-space position/rotation for queries, raycasts, distance checks

- **Automatic computation** - no manual matrix multiplication needed for hierarchies, `LocalToWorldSystem` handles it

- **Optimized updates** - only recomputed when LocalTransform changes (with change filtering in systems)

#### Cons
- **Read-only semantics** - cannot directly set world-space transform, must modify [[LocalTransform]] instead

- **One-frame delay** - changes to LocalTransform update LocalToWorld next frame after `LocalToWorldSystem` runs

- **Memory overhead** - storing both LocalTransform and LocalToWorld uses more memory than LocalTransform alone

- **Redundant for static** - static entities only need LocalToWorld, LocalTransform is wasted memory (use [[TransformUsageFlags]].WorldSpace)

#### Best use
- **All rendered entities** - required component for any entity rendered by Entities Graphics package

- **World-space queries** - distance checks, raycasting, proximity detection using final world positions

- **Static entities** - entities that never move can use LocalToWorld only without LocalTransform for optimization

#### Avoid if
- **No rendering or spatial queries** - entities that don't render and don't need world-space position don't need transform components

- **Pure logic entities** - UI data, game state, configuration entities typically don't need transforms

#### Extra tip
- **LocalToWorld is float4x4 matrix:**
  ```csharp
  public struct LocalToWorld : IComponentData
  {
      public float4x4 Value;  // 4x4 transformation matrix

      // Convenience properties (extract from matrix)
      public float3 Position { get; }     // Translation from matrix
      public quaternion Rotation { get; }  // Rotation from matrix
      public float3 Forward { get; }      // Forward vector (0, 0, 1) transformed
      public float3 Up { get; }          // Up vector (0, 1, 0) transformed
      public float3 Right { get; }       // Right vector (1, 0, 0) transformed
  }
  ```

- **How LocalToWorld is computed:**
  ```csharp
  // No parent (world-space entity):
  LocalToWorld = LocalTransform.ToMatrix()

  // With parent (hierarchy):
  LocalToWorld = Parent.LocalToWorld × LocalTransform.ToMatrix()

  // With PostTransformMatrix (non-uniform scale):
  LocalToWorld = Parent.LocalToWorld × LocalTransform.ToMatrix() × PostTransformMatrix

  // Computed by LocalToWorldSystem automatically
  ```

- **TransformUsageFlags control which components added:**
  ```csharp
  // Dynamic entities (moving, rotating, scaling)
  GetEntity(TransformUsageFlags.Dynamic)
  // → Adds: LocalTransform + LocalToWorld

  // Static entities (never move)
  GetEntity(TransformUsageFlags.WorldSpace)
  // → Adds: LocalToWorld only

  // Renderable (similar to Dynamic)
  GetEntity(TransformUsageFlags.Renderable)
  // → Adds: LocalTransform + LocalToWorld

  // No transform needed
  GetEntity(TransformUsageFlags.None)
  // → Adds: Nothing

  // Manual override - add components yourself
  GetEntity(TransformUsageFlags.ManualOverride)
  ```

- **Static entities optimization:**
  ```csharp
  // Static entities don't need LocalTransform
  Entity staticProp = em.CreateEntity(typeof(LocalToWorld), typeof(Static));

  em.SetComponentData(staticProp, new LocalToWorld
  {
      Value = float4x4.TRS(position, rotation, scale)
  });

  // LocalToWorld won't be updated (no LocalTransform to read from)
  // Saves memory and avoids unnecessary LocalToWorldSystem processing
  ```

- **Transform system update order:**
  ```csharp
  // TransformSystemGroup runs each frame:
  1. ParentSystem - Updates Parent/Child relationships
  2. LocalToWorldSystem - Computes LocalToWorld from LocalTransform + hierarchy
  3. Rendering systems read LocalToWorld

  // Changes to LocalTransform visible in LocalToWorld next frame
  ```

- **Extracting data from matrix:**
  ```csharp
  // Position (4th column)
  float3 position = localToWorld.Value.c3.xyz;
  // Or use property:
  float3 position = localToWorld.Position;

  // Rotation (upper-left 3x3)
  float3x3 rotationMatrix = new float3x3(
      localToWorld.Value.c0.xyz,
      localToWorld.Value.c1.xyz,
      localToWorld.Value.c2.xyz
  );
  quaternion rotation = new quaternion(rotationMatrix);
  // Or use property:
  quaternion rotation = localToWorld.Rotation;

  // Scale (magnitude of basis vectors)
  float scaleX = math.length(localToWorld.Value.c0.xyz);
  float scaleY = math.length(localToWorld.Value.c1.xyz);
  float scaleZ = math.length(localToWorld.Value.c2.xyz);
  ```

- **Change filtering (avoid redundant updates):**
  ```csharp
  [BurstCompile]
  public partial struct OptimizedSystem : ISystem
  {
      private uint lastSystemVersion;

      [BurstCompile]
      public void OnUpdate(ref SystemState state)
      {
          // Only process entities whose LocalToWorld changed
          var query = SystemAPI.QueryBuilder()
              .WithAll<LocalToWorld>()
              .Build();

          query.SetChangedVersionFilter(ComponentType.ReadOnly<LocalToWorld>());

          foreach (var localToWorld in SystemAPI.Query<RefRO<LocalToWorld>>())
          {
              // Only processes entities with changed LocalToWorld
          }
      }
  }
  ```

- **LocalToWorld for hierarchy:**
  ```csharp
  // Parent entity
  Parent LocalToWorld = float4x4.Translate(10, 0, 0)

  // Child entity with Parent component
  Child LocalTransform = float4x4.Translate(0, 5, 0)  // Relative to parent

  // LocalToWorldSystem computes:
  Child LocalToWorld = Parent.LocalToWorld × Child.LocalTransform
                     = Translate(10,0,0) × Translate(0,5,0)
                     = Translate(10,5,0)  // Final world position
  ```

- **Writing LocalToWorld manually (rare):**
  ```csharp
  // Only for static entities or special cases
  // Most systems should modify LocalTransform instead
  em.SetComponentData(entity, new LocalToWorld
  {
      Value = float4x4.TRS(
          new float3(x, y, z),      // Position
          quaternion.Euler(rx, ry, rz),  // Rotation
          new float3(sx, sy, sz)    // Scale (can be non-uniform)
      )
  });
  ```

## See Also

- [[LocalTransform]] - Writable local-space transform (position, rotation, uniform scale)
- [[Parent]] - Parent entity reference for hierarchies
- [[PostTransformMatrix]] - Additional matrix for non-uniform scale
- [[TransformUsageFlags]] - Baking flags controlling transform component setup
- [[Static]] - Tag for entities that never move
- [[IComponentData]] - Base component interface
