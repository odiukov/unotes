---
tags:
  - baking
---
#### Description
- **Conversion system** that transforms GameObject authoring data into ECS [[Entity|entities]] and [[Component|components]] during the baking process

- Defined as nested class `Baker<T>` inside a MonoBehaviour authoring component, with `Bake(T authoring)` method performing the conversion

- Runs in a **separate baking world**, not the runtime world - changes to GameObject trigger automatic rebaking in Edit mode

- Provides methods like `GetEntity()`, `AddComponent()`, `AddBuffer()`, and `DependsOn()` for entity creation and dependency tracking

#### Example
```csharp
public class DamageableAuthoring : MonoBehaviour
{
    public float MaxHealth;
    public float HealthRegenPerSec;

    public class Baker : Baker<DamageableAuthoring>
    {
        public override void Bake(DamageableAuthoring authoring)
        {
            // Get or create entity for this GameObject
            // TransformUsageFlags determines what transform components to add
            Entity entity = GetEntity(TransformUsageFlags.None);

            // Add components with authoring data
            AddComponent(entity, new Health
            {
                Max = authoring.MaxHealth,
                Current = authoring.MaxHealth / 2f
            });

            AddComponent(entity, new HealthRegen
            {
                PointPerSec = authoring.HealthRegenPerSec
            });
        }
    }
}

// Advanced example with prefabs and buffers
public class SpawnerAuthoring : MonoBehaviour
{
    public GameObject EnemyPrefab;
    public float SpawnRate;

    public class Baker : Baker<SpawnerAuthoring>
    {
        public override void Bake(SpawnerAuthoring authoring)
        {
            Entity entity = GetEntity(TransformUsageFlags.Dynamic);

            // Convert prefab GameObject to Entity
            Entity prefabEntity = GetEntity(authoring.EnemyPrefab, TransformUsageFlags.Dynamic);

            AddComponent(entity, new SpawnerData
            {
                Prefab = prefabEntity,
                SpawnRate = authoring.SpawnRate
            });

            // Add dynamic buffer
            DynamicBuffer<SpawnPoint> buffer = AddBuffer<SpawnPoint>(entity);
            buffer.Add(new SpawnPoint { Position = float3.zero });
        }
    }
}
```

#### Pros
- **Automatic rebaking** - changes to GameObject in Edit mode trigger rebake, instant feedback

- **Incremental baking** - only affected entities rebake when changes occur, not the entire scene

- **Dependency tracking** - `DependsOn(asset)` ensures rebake when referenced assets change

- **Clean separation** - authoring data (GameObjects) stays separate from runtime data (Entities)

- **Type safety** - compile-time checks ensure components are properly configured

#### Cons
- **Separate world** - baking happens in baking world, can't access runtime world data

- **No runtime conversion** - baking is edit-time only, runtime GameObject spawning requires different approach

- **Learning curve** - requires understanding TransformUsageFlags and baking workflow

#### Best use
- **Scene authoring** - converting GameObjects placed in scenes to ECS entities

- **Prefab conversion** - baking GameObject prefabs into Entity prefabs for runtime spawning

- **Complex data transformation** - converting Unity-friendly authoring format to performance-optimized runtime format (e.g., BlobAssets)

#### Avoid if
- **Runtime GameObject conversion** - for converting spawned GameObjects at runtime, use different approach (managed components or direct entity creation)

- **Simple data** - if data is already in optimal format and doesn't need transformation, direct entity creation may be simpler

#### Extra tip
- **[[TransformUsageFlags]]** - choose correct flags: `None` (no transform), `Renderable` (LocalToWorld), `Dynamic` (LocalToWorld + LocalTransform), `WorldSpace` (static, no parent)

- **GetEntity variants** - `GetEntity(TransformUsageFlags)` for self, `GetEntity(GameObject, TransformUsageFlags)` for other GameObjects (prefabs, references)

- **Multiple bakers** - multiple Baker classes can bake the same GameObject, each adding different components

- **BlobAsset pattern** - use `BlobBuilder` in Baker to create [[BlobAsset (immutable data)|BlobAssets]], then `AddBlobAsset(ref bar, out hash)` to store efficiently

- **DependsOn** - call `DependsOn(asset)` to track asset dependencies, ensuring rebake when Texture, Material, ScriptableObject, etc. changes

- **AddComponentObject** - use for adding [[Managed IComponentData (class components)|managed components]] during baking

- **Primary entity** - `GetEntity()` gets the "primary" entity for the GameObject, additional entities can be created with `CreateAdditionalEntity()`
