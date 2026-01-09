---
tags:
  - component
  - hybrid
---
#### Description
- **Blittable wrapper for UnityEngine.Object references** allowing Unity objects (GameObject, Texture, Material, etc.) in [[IComponentData]] while remaining [[Burst]] compatible

- Stores object reference as **instance ID** internally, enabling [[Burst]] compilation unlike direct `UnityEngine.Object` references

- Provides **type-safe API** for getting/setting Unity objects while maintaining ECS performance characteristics

- Alternative to [[Managed IComponentData (class components)|Managed IComponentData]] for Unity object references while keeping components blittable

#### Example
```csharp
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
        foreach (var skin in SystemAPI.Query<RefRO<CharacterSkin>>())
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
foreach (var skin in SystemAPI.Query<RefRO<CharacterSkin>>())
{
    if (skin.ValueRO.SkinMaterial.Value != null)
    {
        // Material is valid
    }
}

// Setting UnityObjectRef at runtime
Material newMaterial = Resources.Load<Material>("NewSkin");
foreach (var skin in SystemAPI.Query<RefRW<CharacterSkin>>())
{
    skin.ValueRW.SkinMaterial = newMaterial;
}
```

#### Pros
- **Blittable component** - keeps [[IComponentData]] blittable, better [[Cache-friendly|cache performance]] than [[Managed IComponentData (class components)|managed components]]

- **[[Burst]] storage compatible** - component can be stored and moved by Burst code (though accessing `.Value` requires managed context)

- **Type safety** - `UnityObjectRef<Material>` ensures only Material references stored

- **Null-safe** - can check `.Value != null` to verify object still exists

- **Instance ID based** - survives Unity domain reloads better than direct references

#### Cons
- **Accessing requires managed context** - cannot use `.Value` in [[Burst]] jobs, must access on main thread

- **Performance overhead** - accessing `.Value` requires instance ID lookup, slower than direct reference

- **Not fully ECS** - still bridges to Unity object system, not pure data-oriented

- **Broken references possible** - if Unity object destroyed, `.Value` returns null (no automatic cleanup)

#### Best use
- **Baked asset references** - materials, textures, prefabs converted during [[Baking]]

- **Rendering data** - storing meshes, materials, textures for hybrid rendering

- **Audio/VFX** - audio clips, particle system prefabs, visual effect assets

- **Configuration assets** - ScriptableObject references for game configuration

#### Avoid if
- **Pure ECS is goal** - if building fully data-oriented system, avoid Unity object references

- **Need [[Burst]] access** - if jobs need to access data, use [[BlobAsset (immutable data)|BlobAsset]] to convert to blittable format

- **Frequent runtime changes** - if references change often, [[Managed IComponentData (class components)|managed components]] may be simpler

- **Can be baked to data** - if Unity object data can be converted to pure data during baking, do that instead

#### Extra tip
- **Implicit conversions:**
  ```csharp
  UnityObjectRef<Material> materialRef = myMaterial;  // Implicit
  Material material = materialRef.Value;               // Explicit via .Value
  ```

- **Null handling:**
  ```csharp
  if (materialRef.Value != null) { /* Safe to use */ }
  materialRef.Value?.SetFloat("_Metallic", 1.0f);  // Null-conditional
  ```

- **Instance ID access:**
  ```csharp
  int id = materialRef.instanceID;
  // Useful for comparison or storage
  ```

- **Equality comparison:**
  ```csharp
  if (refA == refB)  // Compares instance IDs
      Debug.Log("Same object");
  ```

- **Works with Resources and AssetDatabase:**
  ```csharp
  var material = Resources.Load<Material>("MyMaterial");
  component.MaterialRef = material;
  ```

- **Cleanup pattern** - use [[ICleanupComponentData]] to track when entities with UnityObjectRef destroyed if need custom cleanup

- **Not serialized directly** - Unity serializes as instance ID, resolves to object on load

- **Performance vs Managed Components:**
  - ✅ Better cache performance (component stays in chunk)
  - ✅ Survives across Burst boundaries
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
