## Baking

**Edit-time conversion process** that transforms GameObject authoring data (MonoBehaviours, GameObjects, prefabs) into optimized ECS runtime data ([[Entity|entities]], [[Component|components]], [[BlobAsset (immutable data)|BlobAssets]]).

Baking happens in a separate **baking world**, not the runtime world. When you modify a GameObject in the Editor, Unity automatically triggers incremental rebaking of affected entities, providing instant feedback.

The [[Baker (authoring conversion)|Baker<T>]] class defines the conversion logic, specifying how authoring data becomes runtime components.

**Key characteristics:**
- **Automatic and incremental** - only rebakes changed GameObjects
- **Edit-time only** - happens during authoring, not at runtime
- **Optimizes data** - converts Unity-friendly authoring format to cache-friendly runtime format
- **Dependency tracking** - tracks asset dependencies to trigger rebakes when assets change

**Example:** GameObject with `HealthAuthoring` MonoBehaviour → bakes to Entity with `Health` IComponentData

**See also:** [[Baker (authoring conversion)]], [[TransformUsageFlags]]
