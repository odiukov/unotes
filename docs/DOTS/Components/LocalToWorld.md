---
tags:
  - component
  - transform
  - rendering
---

#### Description
- **Read-only 4x4 transformation matrix** computed by `LocalToWorldSystem` from [[LocalTransform]] and parent hierarchy - the final world-space transform used by rendering

- **Automatically updated** each frame by `TransformSystemGroup` - combines LocalTransform with [[Parent]] hierarchy and [[PostTransformMatrix]] if present

- **Required for rendering** - Unity's Entities Graphics reads LocalToWorld matrices, not [[LocalTransform]] directly

- **Never write directly** - systems should modify [[LocalTransform]] instead, LocalToWorld is automatically recomputed

#### Example
```csharp
// Reading LocalToWorld (never write to it)
foreach (var localToWorld in SystemAPI.Query<RefRO<LocalToWorld>>())
{
    // Extract world position from matrix
    float3 worldPosition = localToWorld.ValueRO.Position;
    quaternion worldRotation = localToWorld.ValueRO.Rotation;
    float3 worldForward = localToWorld.ValueRO.Forward;
    float4x4 matrix = localToWorld.ValueRO.Value;
}

// Wrong: Don't modify LocalToWorld directly
foreach (var localToWorld in SystemAPI.Query<RefRW<LocalToWorld>>())
{
    // WRONG - changes overwritten next frame
    localToWorld.ValueRW = float4x4.Translate(new float3(1, 0, 0));
}

// Correct: Modify LocalTransform instead
foreach (var transform in SystemAPI.Query<RefRW<LocalTransform>>())
{
    transform.ValueRW.Position += new float3(1, 0, 0);
    // LocalToWorldSystem will update LocalToWorld automatically
}

// Static entities with LocalToWorld only (no LocalTransform)
Entity staticEntity = em.CreateEntity(typeof(LocalToWorld));
em.SetComponentData(staticEntity, new LocalToWorld
{
    Value = float4x4.TRS(
        new float3(10, 0, 5),
        quaternion.identity,
        new float3(1, 1, 1)
    )
});
em.AddComponent<Static>(staticEntity);
// LocalToWorld won't be updated since no LocalTransform exists

// Using LocalToWorld for world-space queries
float3 targetPosition = new float3(0, 0, 0);
foreach (var localToWorld in SystemAPI.Query<RefRO<LocalToWorld>>())
{
    float3 worldPos = localToWorld.ValueRO.Position;
    float distance = math.distance(worldPos, targetPosition);
}
```

#### Pros
- **Rendering ready** - directly usable by Entities Graphics without additional computation

- **World-space access** - provides final world position/rotation for queries, raycasts, distance checks

- **Automatic computation** - no manual matrix multiplication for hierarchies, system handles it

- **Optimized updates** - only recomputed when LocalTransform changes (with change filtering)

#### Cons
- **Read-only semantics** - cannot directly set world-space transform, must modify [[LocalTransform]]

- **One-frame delay** - changes to LocalTransform update LocalToWorld next frame

- **Memory overhead** - storing both LocalTransform and LocalToWorld uses more memory

- **Redundant for static** - static entities only need LocalToWorld, [[LocalTransform]] wasted (use [[TransformUsageFlags]].WorldSpace)

#### Best use
- **All rendered entities** - required component for any entity rendered by Entities Graphics

- **World-space queries** - distance checks, raycasting, proximity detection using final world positions

- **Static entities** - entities that never move can use LocalToWorld only without LocalTransform

#### Avoid if
- **No rendering or spatial queries** - entities without world-space position don't need transforms

- **Pure logic entities** - UI data, game state, configuration entities typically don't need transforms

#### Extra tip
- **LocalToWorld structure:**
  ```csharp
  public struct LocalToWorld : IComponentData
  {
      public float4x4 Value;  // 4x4 transformation matrix

      // Convenience properties
      public float3 Position { get; }
      public quaternion Rotation { get; }
      public float3 Forward/Up/Right { get; }
  }
  ```

- **How LocalToWorld is computed:**
  ```csharp
  // No parent (world-space):
  LocalToWorld = LocalTransform.ToMatrix()

  // With parent (hierarchy):
  LocalToWorld = Parent.LocalToWorld × LocalTransform.ToMatrix()

  // With PostTransformMatrix:
  LocalToWorld = Parent.LocalToWorld × LocalTransform × PostTransformMatrix
  ```

- **TransformUsageFlags control which components added:**
  ```csharp
  GetEntity(TransformUsageFlags.Dynamic)      // → LocalTransform + LocalToWorld
  GetEntity(TransformUsageFlags.WorldSpace)   // → LocalToWorld only
  GetEntity(TransformUsageFlags.None)         // → Nothing
  ```

- **Static entities optimization** - don't add LocalTransform to static entities, only LocalToWorld saves memory

- **Transform system update order:**
  ```csharp
  1. ParentSystem - Updates Parent/Child relationships
  2. LocalToWorldSystem - Computes LocalToWorld from LocalTransform + hierarchy
  3. Rendering systems read LocalToWorld
  ```

- **Extracting data from matrix:**
  ```csharp
  float3 position = localToWorld.Position;  // Use property
  quaternion rotation = localToWorld.Rotation;

  // Or manually from matrix columns:
  float3 position = localToWorld.Value.c3.xyz;  // 4th column
  ```

- **Change filtering (avoid redundant updates):**
  ```csharp
  var query = SystemAPI.QueryBuilder()
      .WithAll<LocalToWorld>()
      .Build();
  query.SetChangedVersionFilter(ComponentType.ReadOnly<LocalToWorld>());
  // Only processes entities with changed LocalToWorld
  ```

- **Hierarchy computation example:**
  ```csharp
  Parent LocalToWorld = Translate(10,0,0)
  Child LocalTransform = Translate(0,5,0)  // Relative

  // LocalToWorldSystem computes:
  Child LocalToWorld = Parent.LocalToWorld × Child.LocalTransform
                     = Translate(10,5,0)  // Final world position
  ```

## See Also

- [[LocalTransform]] - Writable local-space transform (position, rotation, uniform scale)
- [[Parent]] - Parent entity reference for hierarchies
- [[PostTransformMatrix]] - Additional matrix for non-uniform scale
- [[TransformUsageFlags]] - Baking flags controlling transform component setup
- [[Static]] - Tag for entities that never move
- [[IComponentData]] - Base component interface
