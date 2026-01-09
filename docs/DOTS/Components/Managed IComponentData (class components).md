---
tags:
  - component
---
#### Description
- **Class-based** [[IComponentData]] that can store managed references like `GameObject`, `Texture2D`, or other C# objects

- Stored **outside the chunk** in separate managed array, breaking [[SoA layout]] and [[Cache-friendly]] architecture

- Useful for **hybrid workflows** bridging between DOTS and traditional Unity systems (GameObjects, MonoBehaviours, Assets)

- Cannot be used with [[Burst]] or in jobs - requires `SystemAPI.ManagedAPI` for access in systems

#### Example
```csharp
// Managed component storing a GameObject reference
public class PresentationGo : IComponentData
{
    public GameObject Prefab;
}

// Another example with texture reference
public class MaterialData : IComponentData
{
    public Texture2D Texture;
    public Material Material;
}

// Accessing in a system (cannot use Burst)
public partial struct PresentationSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Must use ManagedAPI for managed components
        foreach (var presentationGo in
            SystemAPI.Query<PresentationGo>())
        {
            var instance = Object.Instantiate(presentationGo.Prefab);
        }
    }
}
```

#### Pros
- **Allows GameObject references** - bridge between ECS and traditional Unity

- **Stores complex managed types** - textures, materials, C# objects with references

- **Simplifies hybrid workflows** - companion GameObjects for rendering/animation while using ECS for logic

#### Cons
- **Breaks cache locality** - stored separately from chunk data, causing [[Cache miss]]

- **Cannot use Burst** - no job compilation, must access on main thread with `SystemAPI.ManagedAPI`

- **Causes [[Structural changes]]** - adding/removing managed components triggers archetype changes

- **Garbage collection overhead** - managed objects create GC pressure and allocations

#### Best use
- **GameObject companions** - storing prefab/instance references for rendering while ECS handles logic

- **Asset references** - materials, textures, or other Unity assets that can't be baked to pure data

- **Legacy integration** - bridging existing MonoBehaviour systems during gradual DOTS conversion

#### Avoid if
- **You can use [[BlobAsset (immutable data)|BlobAsset]]** - for read-only shared data, use blob assets instead

- **Performance is critical** - managed components destroy cache performance, use [[IComponentData]] with value types

- **You need parallel jobs** - managed data can't be accessed in [[Burst]] jobs, use [[IComponentData]] with blittable types

- **The data can be baked** - if you can convert data during baking to value types, do it instead

#### Extra tip
- **Cleanup pattern** - use [[ICleanupComponentData]] to track when managed component removed and clean up GameObject instances to prevent leaks

- **Minimize usage** - use managed components sparingly as last resort - they negate most DOTS performance benefits

- **Companion GameObject pattern** - common pattern: managed component stores GameObject reference, separate system syncs transform from [[Entity]] to GameObject using `UnityEngineComponent<Transform>`
