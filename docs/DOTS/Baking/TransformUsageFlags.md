---
tags:
  - baking
---
#### Description
- **Flags enum** passed to `GetEntity()` during [[Baker (authoring conversion)|baking]] to control what transform-related components are added to the baked entity

- Determines the **transform capabilities** of the entity: static position, movable, renderable, uniform/non-uniform scale, etc.

- Unity **automatically sets flags** based on GameObject setup (MeshRenderer adds Renderable, Collider adds Dynamic, static checkbox adds WorldSpace)

- Choosing correct flags **optimizes memory and performance** by only adding required transform components

#### Example
```csharp
public class ExampleAuthoring : MonoBehaviour
{
    public class Baker : Baker<ExampleAuthoring>
    {
        public override void Bake(ExampleAuthoring authoring)
        {
            // None - no transform components added
            // Use for pure data entities (game manager, singleton configs)
            Entity noneEntity = GetEntity(TransformUsageFlags.None);

            // Renderable - adds LocalToWorld (read-only world position)
            // Use for static visible objects that don't move
            Entity renderableEntity = GetEntity(TransformUsageFlags.Renderable);

            // Dynamic - adds LocalToWorld + LocalTransform
            // Use for movable objects (characters, projectiles, enemies)
            Entity dynamicEntity = GetEntity(TransformUsageFlags.Dynamic);

            // WorldSpace - adds LocalToWorld, removes Parent
            // Use for static objects, prevents hierarchy processing
            Entity worldSpaceEntity = GetEntity(TransformUsageFlags.WorldSpace);

            // NonUniformScale - adds PostTransformMatrix
            // Automatically set for GameObjects with non-uniform scale
            Entity nonUniformEntity = GetEntity(TransformUsageFlags.NonUniformScale);

            // Combine flags with bitwise OR
            Entity combined = GetEntity(
                TransformUsageFlags.Dynamic | TransformUsageFlags.Renderable);
        }
    }
}

// Practical examples
public class TowerAuthoring : MonoBehaviour
{
    public class Baker : Baker<TowerAuthoring>
    {
        public override void Bake(TowerAuthoring authoring)
        {
            // Tower is visible but doesn't move - Renderable only
            Entity entity = GetEntity(TransformUsageFlags.Renderable);
            AddComponent<TowerData>(entity);
        }
    }
}

public class ProjectileAuthoring : MonoBehaviour
{
    public class Baker : Baker<ProjectileAuthoring>
    {
        public override void Bake(ProjectileAuthoring authoring)
        {
            // Projectile moves and is visible - Dynamic
            Entity entity = GetEntity(TransformUsageFlags.Dynamic);
            AddComponent<Projectile>(entity);
        }
    }
}
```

#### Pros
- **Memory optimization** - only adds transform components that are actually needed

- **Performance optimization** - fewer components means better [[Cache-friendly]] iteration

- **Explicit control** - you decide exact transform capabilities rather than automatic "everything"

- **Prevents bugs** - can't accidentally move entities that shouldn't move if they lack LocalTransform

#### Cons
- **Requires knowledge** - must understand what each flag does and which components it adds

- **Combination complexity** - combining flags with bitwise OR can be confusing

- **Auto-overrides** - Unity may override your flags based on other components (MeshRenderer, Collider, etc.)

#### Best use
- **Pure data entities** - use `None` for managers, configs, singletons that don't need position

- **Static rendered objects** - use `Renderable` for decorations, static environment that's visible but never moves

- **Dynamic gameplay entities** - use `Dynamic` for anything that moves (characters, projectiles, enemies, pickups)

- **Static optimizations** - use `WorldSpace` for static objects to bypass hierarchy transform system

#### Avoid if
- **Prefab conversion** - when converting prefabs with `GetEntity(prefabGameObject, flags)`, usually want `Dynamic` for runtime spawning flexibility

#### Extra tip
- **Component mapping:**
  - `None` → no transform components
  - `Renderable` → `LocalToWorld`
  - `Dynamic` → `LocalToWorld` + `LocalTransform`
  - `WorldSpace` → `LocalToWorld`, removes `Parent` component
  - `NonUniformScale` → adds `PostTransformMatrix`

- **Automatic flags** - Unity adds flags automatically:
  - `Renderable` if MeshRenderer present
  - `Dynamic` if Collider present or GameObject is a prefab
  - `WorldSpace` if GameObject marked static in Inspector
  - `NonUniformScale` if scale is non-uniform (e.g., (1, 2, 1))

- **LocalTransform vs LocalToWorld** - `LocalTransform` is writable local transform (position/rotation/scale), `LocalToWorld` is computed world transform matrix (read-only result)

- **Prefab pattern** - for prefabs intended for spawning: `GetEntity(prefabGameObject, TransformUsageFlags.Dynamic)` ensures spawned instances can move

- **ManualOverride** - use `ManualOverride` to bypass automatic flag detection, giving you full control (advanced use only)
