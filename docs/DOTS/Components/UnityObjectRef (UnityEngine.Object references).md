---
tags:
  - component
  - hybrid
---
#### Description
- **Blittable wrapper for UnityEngine.Object references** that allows storing Unity objects (GameObject, Texture, Material, etc.) in [[IComponentData]] while remaining [[Burst]] compatible

- Stores object reference as an **instance ID** internally, enabling [[Burst]] compilation unlike direct `UnityEngine.Object` references

- Provides **type-safe API** for getting/setting Unity objects while maintaining ECS performance characteristics

- Alternative to [[Managed IComponentData (class components)|Managed IComponentData]] for cases where you need Unity object references but want to keep components blittable

#### Example
```csharp
using Unity.Entities;
using UnityEngine;

// Component storing Unity object reference
public struct CharacterSkin : IComponentData
{
    // UnityObjectRef<T> instead of direct Material reference
    public UnityObjectRef<Material> SkinMaterial;
    public UnityObjectRef<Texture2D> SkinTexture;
}

// Baker creates UnityObjectRef from Unity objects
public class CharacterAuthoring : MonoBehaviour
{
    public Material CharacterMaterial;
    public Texture2D CharacterTexture;

    public class Baker : Baker<CharacterAuthoring>
    {
        public override void Bake(CharacterAuthoring authoring)
        {
            Entity entity = GetEntity(TransformUsageFlags.Renderable);

            AddComponent(entity, new CharacterSkin
            {
                // Implicit conversion from Material to UnityObjectRef<Material>
                SkinMaterial = authoring.CharacterMaterial,
                SkinTexture = authoring.CharacterTexture
            });

            // Track dependencies for rebaking
            DependsOn(authoring.CharacterMaterial);
            DependsOn(authoring.CharacterTexture);
        }
    }
}

// Using UnityObjectRef in systems (CANNOT be Burst compiled)
public partial struct CharacterRenderSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Access requires managed API - no Burst
        foreach (var (skin, entity) in
            SystemAPI.Query<RefRO<CharacterSkin>>()
                .WithEntityAccess())
        {
            // Get Unity object from UnityObjectRef
            Material material = skin.ValueRO.SkinMaterial.Value;
            Texture2D texture = skin.ValueRO.SkinTexture.Value;

            if (material != null && texture != null)
            {
                material.mainTexture = texture;
            }
        }
    }
}

// Checking if reference is valid
public partial struct ValidationSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        foreach (var skin in SystemAPI.Query<RefRO<CharacterSkin>>())
        {
            // Check if Unity object still exists
            if (skin.ValueRO.SkinMaterial.Value != null)
            {
                // Material is valid
            }
            else
            {
                // Material was destroyed or not set
            }
        }
    }
}

// Setting UnityObjectRef at runtime
public partial struct SkinChangerSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        Material newMaterial = Resources.Load<Material>("NewSkin");

        foreach (var skin in SystemAPI.Query<RefRW<CharacterSkin>>())
        {
            // Assign new Unity object
            skin.ValueRW.SkinMaterial = newMaterial;
        }
    }
}
```

#### Pros
- **Blittable component** - keeps [[IComponentData]] blittable, better [[Cache-friendly|cache performance]] than [[Managed IComponentData (class components)|managed components]]

- **[[Burst]] storage compatible** - component can be stored and moved by Burst code (though accessing `.Value` requires managed context)

- **Type safety** - `UnityObjectRef<Material>` ensures only Material references are stored

- **Null-safe** - can check `.Value != null` to verify object still exists

- **Instance ID based** - survives Unity domain reloads better than direct references

#### Cons
- **Accessing requires managed context** - cannot use `.Value` in [[Burst]] jobs, must access on main thread or in non-Burst systems

- **Performance overhead** - accessing `.Value` requires instance ID lookup, slower than direct reference

- **Not fully ECS** - still bridges to Unity object system, not pure data-oriented

- **Broken references possible** - if Unity object is destroyed, `.Value` returns null (no automatic cleanup)

#### Best use
- **Baked asset references** - materials, textures, prefabs converted during [[Baking]]

- **Rendering data** - storing meshes, materials, textures for hybrid rendering

- **Audio/VFX** - audio clips, particle system prefabs, visual effect assets

- **Configuration assets** - ScriptableObject references for game configuration

#### Avoid if
- **Pure ECS is goal** - if building fully data-oriented system, avoid Unity object references entirely

- **Need [[Burst]] access** - if jobs need to access the data, use [[BlobAsset (immutable data)|BlobAsset]] instead to convert data to blittable format

- **Frequent runtime changes** - if references change often, [[Managed IComponentData (class components)|managed components]] may be simpler

- **Can be baked to data** - if Unity object data can be converted to pure data during baking, do that instead

#### Extra tip
- **Implicit conversions** - UnityObjectRef supports implicit conversion from Unity objects:
  ```csharp
  UnityObjectRef<Material> materialRef = myMaterial;  // Implicit
  Material material = materialRef.Value;               // Explicit via .Value
  ```

- **Null handling:**
  ```csharp
  // Check before use
  if (materialRef.Value != null)
  {
      // Safe to use
  }

  // Or use null-conditional
  materialRef.Value?.SetFloat("_Metallic", 1.0f);
  ```

- **Instance ID access** - can get raw instance ID:
  ```csharp
  int id = materialRef.instanceID;
  // Useful for comparison or storage in blittable containers
  ```

- **Equality comparison:**
  ```csharp
  if (refA == refB)  // Compares instance IDs
      Debug.Log("Same object");
  ```

- **Works with Resources and AssetDatabase:**
  ```csharp
  // Load and assign
  var material = Resources.Load<Material>("MyMaterial");
  component.MaterialRef = material;

  // Or during baking from asset
  var texture = AssetDatabase.LoadAssetAtPath<Texture2D>("Assets/texture.png");
  ```

- **Cleanup pattern** - use [[ICleanupComponentData]] to track when entities with UnityObjectRef are destroyed if you need custom cleanup

- **Not serialized directly** - Unity serializes as instance ID, resolves to object on load (similar to how Unity serializes object references)

- **Performance vs Managed Components:**
  - ✅ Better cache performance (component stays in chunk)
  - ✅ Survives across Burst boundaries (can be in Burst-accessible memory)
  - ❌ Accessing `.Value` still requires managed context
  - ❌ Lookup overhead vs direct managed reference

## Comparison with Alternatives

| Approach | Blittable | Burst Storage | Burst Access | Use Case |
|----------|-----------|---------------|--------------|----------|
| `UnityObjectRef<T>` | ✅ | ✅ | ❌ | Asset references in baked data |
| [[Managed IComponentData (class components)\|Managed Component]] | ❌ | ❌ | ❌ | GameObject companions |
| [[BlobAsset (immutable data)\|BlobAsset]] | ✅ | ✅ | ✅ | Immutable configuration data |
| Direct `UnityEngine.Object` | ❌ | ❌ | ❌ | Not allowed in IComponentData |

## See Also

- [[Managed IComponentData (class components)]] - Alternative for managed references
- [[BlobAsset (immutable data)]] - Pure data alternative
- [[Baking]] - Converting Unity objects during authoring
- [[ICleanupComponentData]] - Cleanup when entity destroyed
