---
tags:
  - component
  - transform
  - advanced
---

#### Description
- **Additional 4x4 matrix** applied after [[LocalTransform]] in transform hierarchy - enables non-uniform scale (different X/Y/Z values) not supported by LocalTransform

- **Composed after LocalTransform** - final [[LocalToWorld]] = Parent.LocalToWorld × LocalTransform × **PostTransformMatrix**

- **Primarily for non-uniform scale** - most common use is stretching entities (ellipsoids, boxes) where X/Y/Z scale differs

- **Optional component** - only add when needed, most entities use LocalTransform uniform scale alone

#### Example
```csharp
// Creating entity with non-uniform scale
Entity entity = em.CreateEntity(
    typeof(LocalTransform),
    typeof(LocalToWorld),
    typeof(PostTransformMatrix)
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

// Animating non-uniform scale
foreach (var postMatrix in SystemAPI.Query<RefRW<PostTransformMatrix>>())
{
    float scaleX = 1f + math.sin(time) * 0.5f;
    postMatrix.ValueRW.Value = float4x4.Scale(scaleX, 1f, 1f);
}

// Shear/skew transformations
float shear = 0.5f;
postMatrix.ValueRW.Value = new float4x4(
    new float4(1, 0, 0, 0),       // X basis
    new float4(shear, 1, 0, 0),   // Y basis (sheared)
    new float4(0, 0, 1, 0),       // Z basis
    new float4(0, 0, 0, 1)        // Translation
);

// Combining rotation with non-uniform scale
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
}
```

#### Pros
- **Non-uniform scale support** - enables different X/Y/Z scale values (ellipsoids, boxes, stretched objects)

- **Preserves LocalTransform simplicity** - LocalTransform stays simple, complexity isolated to PostTransformMatrix

- **Full matrix control** - supports shear, skew, and other matrix transformations beyond basic TRS

- **Optional addition** - only adds memory cost when needed, most entities don't require it

#### Cons
- **Additional memory** - adds 64 bytes (float4x4) per entity, significant overhead

- **Composition complexity** - must understand LocalTransform × PostTransformMatrix multiplication order

- **Performance cost** - extra matrix multiplication per entity in `LocalToWorldSystem`

- **Less intuitive** - matrix-based transformations harder to understand than simple position/rotation/scale

#### Best use
- **Non-uniform scale objects** - rectangular boxes, ellipsoids, walls, platforms with different X/Y/Z dimensions

- **Stretched meshes** - UI elements, ribbons, beams where one axis scales differently

- **Advanced transformations** - shear, skew, or custom matrix operations not representable as position/rotation/uniform-scale

#### Avoid if
- **Uniform scale sufficient** - if entity scales equally in all directions, use [[LocalTransform]].Scale

- **Many entities** - if most entities need non-uniform scale, consider alternative approaches (bake into mesh)

- **Simple transformations** - basic position/rotation/uniform-scale should use LocalTransform alone

- **Performance critical** - extra matrix multiplication overhead may be unacceptable for thousands of entities

#### Extra tip
- **Transform composition order:**
  ```csharp
  // With PostTransformMatrix:
  LocalToWorld = Parent.LocalToWorld × LocalTransform × PostTransformMatrix

  // Without PostTransformMatrix:
  LocalToWorld = Parent.LocalToWorld × LocalTransform
  ```

- **PostTransformMatrix structure:**
  ```csharp
  public struct PostTransformMatrix : IComponentData
  {
      public float4x4 Value;  // Applied after LocalTransform
  }
  ```

- **Creating non-uniform scale:**
  ```csharp
  new PostTransformMatrix { Value = float4x4.Scale(2f, 1f, 0.5f) }
  new PostTransformMatrix { Value = float4x4.Scale(new float3(x, y, z)) }
  ```

- **Common transformations:**
  ```csharp
  float4x4.Scale(2, 1, 0.5)                  // Non-uniform scale
  float4x4.Scale(-1, 1, 1)                   // Flip/mirror on X axis
  new float4x4(...)                          // Custom shear/skew matrix
  ```

- **When to use PostTransformMatrix vs LocalTransform:**
  - **Use LocalTransform.Scale (uniform):** Spheres, cubes, characters, most props
  - **Use PostTransformMatrix (non-uniform):** Boxes (width ≠ height), ellipsoids, walls, UI with aspect ratio

- **Avoiding PostTransformMatrix overhead:**
  - **Option 1:** Bake scale into mesh vertices during authoring
  - **Option 2:** Use separate entity types (with/without PostTransformMatrix)
  - **Option 3:** Check if close to uniform, use uniform scale if within epsilon

- **LocalToWorldSystem processing:**
  ```csharp
  // If no PostTransformMatrix:
  LocalToWorld = LocalTransform.ToMatrix()

  // If has PostTransformMatrix:
  LocalToWorld = LocalTransform.ToMatrix() × PostTransformMatrix.Value
  ```

- **Memory impact** - PostTransformMatrix adds 64 bytes per entity, reduces chunk capacity from ~500 to ~400 entities

- **Rotation gotcha** - rotation in PostTransformMatrix applied AFTER LocalTransform rotation, can produce unexpected results. Usually put all rotation in LocalTransform, only scale in PostTransformMatrix

## See Also

- [[LocalTransform]] - Primary transform with position, rotation, uniform scale
- [[LocalToWorld]] - Computed world-space transform matrix
- [[Parent]] - Parent entity for transform hierarchy
- [[TransformUsageFlags]] - Baking transform component setup
- [[IComponentData]] - Base component interface
