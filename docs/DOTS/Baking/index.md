# Baking

Baking is Unity DOTS' authoring workflow that converts GameObject-based scene data into optimized ECS runtime data. It happens automatically in Edit mode and provides instant feedback when you modify GameObjects.

## Core Concepts

- **[[Baker (authoring conversion)|Baker<T>]]** - Conversion class that transforms authoring data to runtime components
- **[[TransformUsageFlags]]** - Controls which transform components are added during baking
- **[[Baking]]** (Glossary) - Overview of the baking process

## How Baking Works

1. You create a **MonoBehaviour authoring component** on a GameObject in the Editor
2. You define a **Baker<T> class** that specifies the conversion logic
3. Unity runs the baker in a **separate baking world** during Edit mode
4. Changes to GameObjects trigger **automatic incremental rebaking**
5. The result is optimized **Entity + Components** in the runtime world

## Key Features

- **Automatic** - Runs whenever you modify GameObjects in Edit mode
- **Incremental** - Only rebakes affected entities, not the entire scene
- **Dependency tracking** - Tracks asset references and rebakes when they change
- **Efficient** - Converts Unity-friendly authoring data to cache-friendly runtime format

## Common Patterns

```csharp
public class MyAuthoring : MonoBehaviour
{
    public float Speed;

    public class Baker : Baker<MyAuthoring>
    {
        public override void Bake(MyAuthoring authoring)
        {
            Entity entity = GetEntity(TransformUsageFlags.Dynamic);
            AddComponent(entity, new Speed { Value = authoring.Speed });
        }
    }
}
```

## Best Practices

1. Use correct **[[TransformUsageFlags]]** to minimize components
2. Use **[[BlobAsset (immutable data)|BlobAssets]]** for shared read-only configuration
3. Track dependencies with **DependsOn()** for assets
4. Separate authoring data from runtime data for optimal performance

## See Also

- [[IComponentData]] - Runtime component types
- [[BlobAsset (immutable data)]] - Immutable shared data
- [[Managed IComponentData (class components)]] - Hybrid workflow components
