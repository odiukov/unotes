---
tags:
  - component
  - transform
  - hierarchy
---

#### Description
- **Parent entity reference** component storing the Entity ID of this entity's parent - creates transform hierarchies where child [[LocalTransform]] is relative to parent's [[LocalToWorld]]

- Automatically maintained by `ParentSystem` which also updates the reverse [[Child]] buffer on parent entities when parent/child relationships change

- **Structural change** - adding/removing Parent component moves entity to different [[Archetype]], triggering chunk reallocation (use [[EntityCommandBuffer]] in jobs)

- Combined with [[LocalTransform]] by `LocalToWorldSystem` each frame: child's LocalToWorld = parent's LocalToWorld × child's LocalTransform

#### Example
```csharp
using Unity.Entities;
using Unity.Mathematics;
using Unity.Transforms;

// Creating parent-child hierarchy
public partial class HierarchyExampleSystem : SystemBase
{
    protected override void OnUpdate()
    {
        EntityManager em = EntityManager;

        // Create parent entity
        Entity parent = em.CreateEntity(typeof(LocalTransform), typeof(LocalToWorld));
        em.SetComponentData(parent, LocalTransform.FromPosition(new float3(10, 0, 0)));

        // Create child entity with Parent component
        Entity child = em.CreateEntity(typeof(LocalTransform), typeof(LocalToWorld), typeof(Parent));
        em.SetComponentData(child, new Parent { Value = parent });

        // Child's local position relative to parent
        em.SetComponentData(child, LocalTransform.FromPosition(new float3(0, 5, 0)));

        // Child's world position will be (10, 5, 0) = parent (10,0,0) + child local (0,5,0)
    }
}

// Changing parent at runtime (structural change - use ECB in jobs)
[BurstCompile]
public partial struct ReparentSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);

        foreach (var (parent, entity) in
            SystemAPI.Query<RefRO<Parent>>()
                .WithEntityAccess()
                .WithAll<NeedsReparent>())
        {
            Entity newParent = /* find new parent */;

            // Change parent value (structural change)
            ecb.SetComponent(entity, new Parent { Value = newParent });

            // Or remove parent entirely (make world-space)
            ecb.RemoveComponent<Parent>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Querying parent-child relationships
[BurstCompile]
public partial struct ParentQuerySystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Get all child entities with specific parent
        Entity targetParent = /* ... */;

        foreach (var (parent, transform, entity) in
            SystemAPI.Query<RefRO<Parent>, RefRW<LocalTransform>>()
                .WithEntityAccess())
        {
            if (parent.ValueRO.Value == targetParent)
            {
                // Process child entity
                transform.ValueRW.Position += new float3(0, 1, 0);
            }
        }

        // Access children through parent's Child buffer (reverse lookup)
        var childBufferLookup = SystemAPI.GetBufferLookup<Child>(isReadOnly: true);

        if (childBufferLookup.TryGetBuffer(targetParent, out DynamicBuffer<Child> children))
        {
            for (int i = 0; i < children.Length; i++)
            {
                Entity childEntity = children[i].Value;
                // Process each child...
            }
        }
    }
}

// Baking hierarchy from GameObject
class HierarchyAuthoring : MonoBehaviour
{
    class Baker : Baker<HierarchyAuthoring>
    {
        public override void Bake(HierarchyAuthoring authoring)
        {
            var entity = GetEntity(TransformUsageFlags.Dynamic);

            // Parent components automatically created from GameObject hierarchy
            // No manual Parent component addition needed - handled by baking system

            // To explicitly set parent during baking:
            Transform parentTransform = authoring.transform.parent;
            if (parentTransform != null)
            {
                Entity parentEntity = GetEntity(parentTransform, TransformUsageFlags.Dynamic);
                // Parent relationship preserved automatically
            }
        }
    }
}
```

#### Pros
- **Automatic hierarchy** - transform hierarchies work like GameObject parent-child relationships without manual matrix math

- **Efficient propagation** - `LocalToWorldSystem` updates all child transforms in parallel using jobs, leveraging [[Chunk]] memory layout

- **Bidirectional access** - Parent component for upward lookup, [[Child]] buffer for downward iteration of all children

- **Clean API** - single Entity reference is simple to understand and modify

#### Cons
- **Structural change** - adding/removing Parent moves entity to different archetype, causing [[Structural changes|structural change]] overhead

- **No immediate update** - parent changes processed by `ParentSystem`, then `LocalToWorldSystem` computes new LocalToWorld next frame

- **Query complexity** - finding all descendants requires recursive traversal or maintaining separate hierarchy data structure

- **Deep hierarchy cost** - very deep hierarchies (10+ levels) incur matrix multiplication overhead each frame for all descendants

#### Best use
- **Transform hierarchies** - characters with bones, vehicles with turrets, UI panels with child elements

- **Spatial attachment** - projectiles attached to moving platforms, weapons attached to character hands

- **Scene hierarchies** - building structures where rooms are children of buildings, furniture children of rooms

#### Avoid if
- **Static relationships** - if parent-child never changes at runtime, consider baking final world transforms instead

- **Frequent reparenting** - constantly changing parents causes repeated structural changes, consider alternative approaches (offsets, manual transforms)

- **Very deep hierarchies** - hierarchies deeper than 10 levels may benefit from flattening or caching computed transforms

- **No transform dependency** - if child doesn't need parent's transform (just logical grouping), use custom component instead of Parent

#### Extra tip
- **ParentSystem vs LocalToWorldSystem ordering:**
  ```csharp
  // Update order each frame:
  1. ParentSystem - Updates Parent/Child relationships (structural changes)
  2. LocalToWorldSystem - Computes LocalToWorld from Parent hierarchy + LocalTransform
  3. Runs in TransformSystemGroup
  ```

- **Child buffer component (reverse lookup):**
  ```csharp
  // Parent entity automatically gets Child buffer with all children
  DynamicBuffer<Child> children = em.GetBuffer<Child>(parentEntity);

  for (int i = 0; i < children.Length; i++)
  {
      Entity childEntity = children[i].Value;
      // Process child...
  }

  // Child buffer is automatically maintained by ParentSystem
  ```

- **Removing parent (make world-space):**
  ```csharp
  // Remove Parent component to make entity world-space
  ecb.RemoveComponent<Parent>(entity);

  // Entity's LocalTransform becomes world-space position
  // (Not relative to any parent anymore)
  ```

- **Setting parent during entity creation:**
  ```csharp
  Entity parent = em.CreateEntity(typeof(LocalTransform), typeof(LocalToWorld));

  Entity child = em.CreateEntity(typeof(LocalTransform), typeof(LocalToWorld), typeof(Parent));
  em.SetComponentData(child, new Parent { Value = parent });

  // ParentSystem will add Child buffer to parent automatically
  ```

- **Hierarchy traversal pattern:**
  ```csharp
  // Recursive function to process entire hierarchy
  void ProcessHierarchy(Entity entity, EntityManager em, ComponentLookup<LocalTransform> transforms)
  {
      // Process current entity
      if (transforms.TryGetComponent(entity, out LocalTransform transform))
      {
          // Do something with transform...
      }

      // Process children
      if (em.HasBuffer<Child>(entity))
      {
          DynamicBuffer<Child> children = em.GetBuffer<Child>(entity);
          for (int i = 0; i < children.Length; i++)
          {
              ProcessHierarchy(children[i].Value, em, transforms);
          }
      }
  }
  ```

- **Parent component is just Entity reference:**
  ```csharp
  public struct Parent : IComponentData
  {
      public Entity Value;  // Reference to parent entity
  }
  ```

- **Baking automatically preserves hierarchy:**
  - GameObject parent-child relationships automatically become Parent components
  - No manual Parent assignment needed during baking
  - Hierarchy flattening options available via baking settings

- **Parent changes are deferred:**
  ```csharp
  // Changes via ECB are deferred until playback
  ecb.SetComponent(entity, new Parent { Value = newParent });

  // Parent/Child buffers updated by ParentSystem after playback
  // LocalToWorld updated by LocalToWorldSystem after ParentSystem
  ```

- **Checking if entity has parent:**
  ```csharp
  // Entity with no Parent component is world-space
  bool isWorldSpace = !em.HasComponent<Parent>(entity);

  // Entity with Parent component is local-space relative to parent
  bool hasParent = em.HasComponent<Parent>(entity);
  ```

- **Performance consideration:**
  - Parent/Child relationship updates are batched in `ParentSystem`
  - LocalToWorld computation parallelized across all entities
  - Deep hierarchies (10+ levels) multiply matrices for each level each frame
  - Consider caching or flattening for performance-critical deep hierarchies

## See Also

- [[LocalTransform]] - Child's local transform relative to parent
- [[LocalToWorld]] - Computed world-space transform matrix
- [[Child]] - Buffer component on parent listing all children
- [[TransformUsageFlags]] - Baking flags for transform component setup
- [[EntityCommandBuffer]] - Deferred structural changes for Parent modification
- [[Structural changes]] - Understanding archetype changes and overhead
