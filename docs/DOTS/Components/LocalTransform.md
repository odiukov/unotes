---
tags:
  - component
  - transform
---

#### Description
- **Primary transform component** storing an entity's local position, rotation, and uniform scale in 3D space - the writable transform that systems modify directly

- Represents transform **relative to parent** if [[Parent]] component exists, otherwise relative to world origin

- **Uniform scale only** - single float value scales equally in all directions. For non-uniform scale (different X/Y/Z values), must add [[PostTransformMatrix]]

- Automatically processed by `LocalToWorldSystem` each frame to compute the read-only [[LocalToWorld]] matrix for rendering

#### Example
```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

// Basic usage - create entity with transform
Entity entity = entityManager.CreateEntity(typeof(LocalTransform));
entityManager.SetComponentData(entity, new LocalTransform
{
    Position = new float3(0, 1, 0),
    Rotation = quaternion.identity,
    Scale = 1f
});

// Helper methods for common operations
LocalTransform transform = LocalTransform.FromPosition(new float3(10, 0, 5));
LocalTransform transform2 = LocalTransform.FromRotation(quaternion.RotateY(math.radians(90)));
LocalTransform transform3 = LocalTransform.FromScale(2f);

// Combined initialization
LocalTransform playerTransform = LocalTransform.FromPositionRotationScale(
    new float3(0, 0, 0),
    quaternion.Euler(0, math.radians(45), 0),
    1.5f
);

// System modifying transforms
[BurstCompile]
public partial struct MovementSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        foreach (var (transform, velocity) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRO<Velocity>>())
        {
            // Modify position
            transform.ValueRW.Position += velocity.ValueRO.Value * deltaTime;

            // Rotate entity
            transform.ValueRW.Rotation = math.mul(
                transform.ValueRW.Rotation,
                quaternion.RotateY(deltaTime)
            );

            // Scale entity
            transform.ValueRW.Scale *= 1.01f;
        }
    }
}

// Using helper methods
foreach (var transform in SystemAPI.Query<RefRW<LocalTransform>>())
{
    // Apply translation
    transform.ValueRW = transform.ValueRO.Translate(new float3(0, deltaTime, 0));

    // Rotate around axis
    transform.ValueRW = transform.ValueRO.Rotate(quaternion.RotateY(deltaTime));

    // Look at target
    float3 targetPos = new float3(10, 0, 0);
    transform.ValueRW.Rotation = quaternion.LookRotation(
        math.normalize(targetPos - transform.ValueRO.Position),
        math.up()
    );
}
```

#### Pros
- **Direct modification** - systems can write to this component directly without special APIs, unlike [[LocalToWorld]]

- **Hierarchical transforms** - automatically combines with [[Parent]] component for transform hierarchies without manual matrix math

- **Uniform scale performance** - single float scale is cheaper than 3-component scale, better for most use cases

- **[[Cache-friendly]]** - small size (28 bytes: float3 + quaternion + float) fits efficiently in [[Chunk]] memory

#### Cons
- **Uniform scale limitation** - cannot represent non-uniform scale (stretched box, ellipsoid) without adding [[PostTransformMatrix]]

- **Requires transform systems** - relies on `TransformSystemGroup` running to update [[LocalToWorld]], not automatic

- **Not the rendered transform** - rendering uses [[LocalToWorld]], not LocalTransform directly. LocalToWorld is computed from LocalTransform

- **Must exist with LocalToWorld** - Unity's rendering requires [[LocalToWorld]] component, so LocalTransform alone is insufficient for rendering

#### Best use
- **Dynamic entities** - any moving, rotating, or scaling entity (players, enemies, projectiles, cameras)

- **Transform hierarchies** - entities with [[Parent]] component automatically use LocalTransform for relative positioning

- **Uniform scaling objects** - characters, spheres, UI elements, anything that scales equally in all directions

#### Avoid if
- **Static non-moving entities** - if entity never moves/rotates/scales, can use only [[LocalToWorld]] with [[Static]] tag for optimization

- **Non-uniform scale needed** - stretched objects, ellipsoids, or different X/Y/Z scales require [[PostTransformMatrix]] addition

- **No rendering needed** - if entity doesn't render and doesn't need spatial position, avoid transform components entirely

#### Extra tip
- **Helper methods available:**
  ```csharp
  // Creation
  LocalTransform.Identity                      // Default: (0,0,0), identity rotation, scale 1
  LocalTransform.FromPosition(float3)          // Position only
  LocalTransform.FromRotation(quaternion)      // Rotation only
  LocalTransform.FromScale(float)              // Scale only
  LocalTransform.FromPositionRotation(float3, quaternion)
  LocalTransform.FromPositionRotationScale(float3, quaternion, float)

  // Transformation
  transform.Translate(float3 offset)           // Add to position
  transform.Rotate(quaternion rotation)        // Apply rotation
  transform.RotateX/Y/Z(float radians)         // Rotate around axis
  transform.TransformPoint(float3 point)       // Convert local to world point
  transform.InverseTransformPoint(float3 point) // Convert world to local point

  // Properties
  float3 up = transform.Up()         // Up vector (0, 1, 0) in world space
  float3 forward = transform.Forward() // Forward vector (0, 0, 1) in world space
  float3 right = transform.Right()   // Right vector (1, 0, 0) in world space
  ```

- **TransformUsageFlags in baking:**
  ```csharp
  // Authoring MonoBehaviour
  class MyAuthoring : MonoBehaviour
  {
      class Baker : Baker<MyAuthoring>
      {
          public override void Bake(MyAuthoring authoring)
          {
              var entity = GetEntity(TransformUsageFlags.Dynamic);
              // Adds LocalTransform + LocalToWorld
          }
      }
  }
  ```
  - `TransformUsageFlags.Dynamic` → Adds LocalTransform (for moving entities)
  - `TransformUsageFlags.WorldSpace` → Adds LocalToWorld only (for static rendering)
  - See [[TransformUsageFlags]] for all options

- **LocalTransform is computed FROM LocalToWorld during baking** - when baking GameObjects, Unity converts `UnityEngine.Transform` → `LocalToWorld` → `LocalTransform`

- **Hierarchy behavior:**
  - **No [[Parent]]:** LocalTransform IS world-space transform
  - **Has [[Parent]]:** LocalTransform is relative to parent's LocalToWorld
  - `LocalToWorldSystem` combines parent's LocalToWorld × child's LocalTransform → child's LocalToWorld

- **Quaternion normalization** - Unity automatically normalizes rotation quaternions, no need to manually normalize

- **Transform system ordering:**
  1. `ParentSystem` - Updates [[Parent]]/[[Child]] relationships
  2. `LocalToWorldSystem` - Computes [[LocalToWorld]] from LocalTransform + parent transforms
  3. Runs in `TransformSystemGroup` every frame

- **Performance tip** - batch transform changes in jobs with [[IJobEntity]] for thousands of entities:
  ```csharp
  [BurstCompile]
  public partial struct MoveJob : IJobEntity
  {
      public float DeltaTime;
      private void Execute(ref LocalTransform transform, in Velocity velocity)
      {
          transform.Position += velocity.Value * DeltaTime;
      }
  }
  ```

## See Also

- [[LocalToWorld]] - Read-only world-space transform matrix computed from LocalTransform
- [[Parent]] - Parent entity reference for transform hierarchies
- [[PostTransformMatrix]] - Additional matrix for non-uniform scale
- [[TransformUsageFlags]] - Baking flags controlling which transform components are added
- [[IComponentData]] - Base component interface
