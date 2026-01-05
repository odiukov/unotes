---
tags:
  - component
  - transform
  - advanced
---

#### Description
- **Additional 4x4 matrix** applied after [[LocalTransform]] in transform hierarchy - enables non-uniform scale (different X/Y/Z values) and other matrix operations not supported by LocalTransform alone

- **Composed after LocalTransform** - final [[LocalToWorld]] = Parent.LocalToWorld × LocalTransform × **PostTransformMatrix**, allowing LocalTransform to remain simple with uniform scale

- **Primarily for non-uniform scale** - most common use case is stretching entities (ellipsoids, rectangular boxes) where X/Y/Z scale differs

- **Optional component** - only add when needed for non-uniform transformations, most entities use LocalTransform uniform scale alone

#### Example
```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

// Creating entity with non-uniform scale
public partial class NonUniformScaleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        EntityManager em = EntityManager;

        // Create entity with standard transform components
        Entity entity = em.CreateEntity(
            typeof(LocalTransform),
            typeof(LocalToWorld),
            typeof(PostTransformMatrix)  // Add for non-uniform scale
        );

        // Set uniform transform (position, rotation, uniform scale)
        em.SetComponentData(entity, LocalTransform.FromPositionRotationScale(
            new float3(0, 0, 0),
            quaternion.identity,
            1f  // Uniform scale = 1
        ));

        // Set non-uniform scale in PostTransformMatrix
        em.SetComponentData(entity, new PostTransformMatrix
        {
            Value = float4x4.Scale(2f, 1f, 0.5f)  // Stretch X, normal Y, squash Z
        });

        // Final LocalToWorld will have non-uniform scale (2, 1, 0.5)
    }
}

// Animating non-uniform scale
[BurstCompile]
public partial struct StretchAnimationSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float time = (float)SystemAPI.Time.ElapsedTime;

        foreach (var postMatrix in SystemAPI.Query<RefRW<PostTransformMatrix>>())
        {
            // Animate scale on X axis
            float scaleX = 1f + math.sin(time) * 0.5f;  // Oscillate 0.5 to 1.5

            postMatrix.ValueRW.Value = float4x4.Scale(scaleX, 1f, 1f);
        }
    }
}

// Using PostTransformMatrix for shear/skew transformations
[BurstCompile]
public partial struct ShearSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        foreach (var (postMatrix, shearComponent) in
            SystemAPI.Query<RefRW<PostTransformMatrix>, RefRO<ShearAmount>>())
        {
            float shear = shearComponent.ValueRO.Value;

            // Create shear matrix (skew along X based on Y)
            postMatrix.ValueRW.Value = new float4x4(
                new float4(1, 0, 0, 0),       // X basis
                new float4(shear, 1, 0, 0),   // Y basis (sheared)
                new float4(0, 0, 1, 0),       // Z basis
                new float4(0, 0, 0, 1)        // Translation
            );
        }
    }
}

// Baking non-uniform scale from GameObject
class NonUniformScaleAuthoring : MonoBehaviour
{
    public Vector3 scale = new Vector3(1, 1, 1);

    class Baker : Baker<NonUniformScaleAuthoring>
    {
        public override void Bake(NonUniformScaleAuthoring authoring)
        {
            var entity = GetEntity(TransformUsageFlags.Dynamic);

            // Check if scale is non-uniform
            Vector3 scale = authoring.transform.localScale;
            bool isUniform = Mathf.Approximately(scale.x, scale.y) &&
                           Mathf.Approximately(scale.y, scale.z);

            if (!isUniform)
            {
                // Add PostTransformMatrix for non-uniform scale
                AddComponent(entity, new PostTransformMatrix
                {
                    Value = float4x4.Scale(scale.x, scale.y, scale.z)
                });

                // LocalTransform will have uniform scale = 1
                // PostTransformMatrix handles actual non-uniform scale
            }
            // If uniform, no PostTransformMatrix needed - use LocalTransform.Scale
        }
    }
}

// Combining rotation with non-uniform scale
[BurstCompile]
public partial struct CombinedTransformSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        float deltaTime = SystemAPI.Time.DeltaTime;

        foreach (var (transform, postMatrix) in
            SystemAPI.Query<RefRW<LocalTransform>, RefRW<PostTransformMatrix>>())
        {
            // Rotate via LocalTransform (uniform)
            transform.ValueRW.Rotation = math.mul(
                transform.ValueRW.Rotation,
                quaternion.RotateY(deltaTime)
            );

            // Non-uniform scale via PostTransformMatrix
            postMatrix.ValueRW.Value = float4x4.Scale(2f, 1f, 1f);

            // Final result: rotated entity with non-uniform scale
            // LocalToWorld = LocalTransform (position + rotation + uniform scale 1) × PostTransformMatrix (non-uniform scale)
        }
    }
}
```

#### Pros
- **Non-uniform scale support** - enables different X/Y/Z scale values (ellipsoids, rectangular boxes, stretched objects)

- **Preserves LocalTransform simplicity** - LocalTransform stays simple with uniform scale, complexity isolated to PostTransformMatrix

- **Full matrix control** - supports shear, skew, and other matrix transformations beyond basic TRS (translate-rotate-scale)

- **Optional addition** - only adds memory cost when needed, most entities don't require it

#### Cons
- **Additional memory** - adds 64 bytes (float4x4) per entity, significant overhead if many entities use it

- **Composition complexity** - final transform is LocalTransform × PostTransformMatrix, must understand multiplication order

- **Performance cost** - extra matrix multiplication per entity in `LocalToWorldSystem`, slower than uniform scale alone

- **Less intuitive** - matrix-based transformations harder to understand than simple position/rotation/scale values

#### Best use
- **Non-uniform scale objects** - rectangular boxes, ellipsoids, walls, platforms with different X/Y/Z dimensions

- **Stretched meshes** - UI elements, ribbons, beams where one axis scales differently than others

- **Advanced transformations** - shear, skew, or custom matrix operations not representable as position/rotation/uniform-scale

#### Avoid if
- **Uniform scale sufficient** - if entity scales equally in all directions, use [[LocalTransform]].Scale instead

- **Many entities** - if most entities in game need non-uniform scale, consider alternative approaches (bake into mesh, separate entity types)

- **Simple transformations** - basic position/rotation/uniform-scale should use LocalTransform alone

- **Performance critical** - extra matrix multiplication overhead may be unacceptable for thousands of entities

#### Extra tip
- **Transform composition order:**
  ```csharp
  // LocalToWorld computation with PostTransformMatrix:
  LocalToWorld = Parent.LocalToWorld × LocalTransform × PostTransformMatrix

  // Without PostTransformMatrix:
  LocalToWorld = Parent.LocalToWorld × LocalTransform

  // PostTransformMatrix always applied AFTER LocalTransform
  ```

- **PostTransformMatrix structure:**
  ```csharp
  public struct PostTransformMatrix : IComponentData
  {
      public float4x4 Value;  // 4x4 matrix applied after LocalTransform
  }
  ```

- **Creating non-uniform scale matrix:**
  ```csharp
  // Non-uniform scale
  PostTransformMatrix post = new PostTransformMatrix
  {
      Value = float4x4.Scale(2f, 1f, 0.5f)  // X=2, Y=1, Z=0.5
  };

  // Or using component-wise scale
  PostTransformMatrix post2 = new PostTransformMatrix
  {
      Value = float4x4.Scale(new float3(scaleX, scaleY, scaleZ))
  };
  ```

- **Common transformations:**
  ```csharp
  // Non-uniform scale
  float4x4.Scale(2, 1, 0.5)

  // Shear (skew X based on Y)
  new float4x4(
      new float4(1, 0, 0, 0),
      new float4(shear, 1, 0, 0),
      new float4(0, 0, 1, 0),
      new float4(0, 0, 0, 1)
  )

  // Flip/mirror on X axis
  float4x4.Scale(-1, 1, 1)

  // Custom matrix (rotation + non-uniform scale)
  float4x4.TRS(
      float3.zero,                    // No translation (in LocalTransform)
      quaternion.RotateY(math.PI/4),  // Extra rotation
      new float3(2, 1, 1)            // Non-uniform scale
  )
  ```

- **When to use PostTransformMatrix vs LocalTransform:**
  ```csharp
  // Use LocalTransform.Scale (uniform):
  - Spheres, cubes with equal dimensions
  - Characters, props that scale uniformly
  - Most game entities

  // Use PostTransformMatrix (non-uniform):
  - Rectangular boxes (width ≠ height ≠ depth)
  - Ellipsoids (sphere stretched on one axis)
  - Walls, platforms with specific dimensions
  - UI elements with aspect ratio
  ```

- **Avoiding PostTransformMatrix overhead:**
  ```csharp
  // Option 1: Bake scale into mesh vertices
  // Create separate mesh assets with desired scale
  // No runtime PostTransformMatrix needed

  // Option 2: Use separate entity types
  // Entity1: Uniform scale entities (no PostTransformMatrix)
  // Entity2: Non-uniform scale entities (with PostTransformMatrix)
  // Avoids component on entities that don't need it

  // Option 3: Check if close to uniform
  float3 scale = new float3(1.01f, 1.0f, 0.99f);
  float epsilon = 0.1f;
  bool isApproxUniform = math.abs(scale.x - scale.y) < epsilon &&
                        math.abs(scale.y - scale.z) < epsilon;
  if (isApproxUniform)
  {
      // Use uniform scale, skip PostTransformMatrix
      float uniformScale = (scale.x + scale.y + scale.z) / 3f;
  }
  ```

- **LocalToWorldSystem processing:**
  ```csharp
  // LocalToWorldSystem checks for PostTransformMatrix:

  // If no PostTransformMatrix:
  LocalToWorld = LocalTransform.ToMatrix()

  // If has PostTransformMatrix:
  LocalToWorld = LocalTransform.ToMatrix() × PostTransformMatrix.Value

  // With parent:
  LocalToWorld = Parent.LocalToWorld × LocalTransform.ToMatrix() × PostTransformMatrix.Value
  ```

- **Memory layout:**
  ```csharp
  // PostTransformMatrix adds 64 bytes per entity (float4x4 = 16 floats × 4 bytes)

  // Chunk capacity impact:
  // Without PostTransformMatrix: ~500 entities per chunk (depending on archetype)
  // With PostTransformMatrix: ~400 entities per chunk
  // Fewer entities per chunk = more chunks = slight performance cost
  ```

- **Unity Editor visualization:**
  - PostTransformMatrix not visualized separately in Unity Editor
  - Final transform (with PostTransformMatrix applied) shown in Inspector
  - Use Entity Inspector to view component values directly

- **Gotcha with rotation:**
  ```csharp
  // Rotation in PostTransformMatrix applied AFTER LocalTransform rotation
  // Can produce unexpected results

  LocalTransform.Rotation = quaternion.RotateY(45°)
  PostTransformMatrix = float4x4.RotateX(90°)

  // Final rotation is NOT simple combination - matrix multiplication order matters
  // Usually put all rotation in LocalTransform, only scale in PostTransformMatrix
  ```

## See Also

- [[LocalTransform]] - Primary transform with position, rotation, uniform scale
- [[LocalToWorld]] - Computed world-space transform matrix
- [[Parent]] - Parent entity for transform hierarchy
- [[TransformUsageFlags]] - Baking transform component setup
- [[IComponentData]] - Base component interface
