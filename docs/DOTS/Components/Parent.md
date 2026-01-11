---
tags:
  - component
  - transform
  - hierarchy
---

#### Description
- **Parent entity reference** storing Entity ID of this entity's parent - creates transform hierarchies where child [[LocalTransform]] is relative to parent's [[LocalToWorld]]

- Automatically maintained by `ParentSystem` which also updates reverse [[Child]] buffer on parent entities

- **Structural change** - adding/removing Parent moves entity to different [[Archetype]], use [[EntityCommandBuffer]] in jobs

- Combined with [[LocalTransform]] by `LocalToWorldSystem` each frame: child's LocalToWorld = parent's LocalToWorld × child's LocalTransform

#### Example
```csharp
// Creating parent-child hierarchy
Entity parent = em.CreateEntity(typeof(LocalTransform), typeof(LocalToWorld));
em.SetComponentData(parent, LocalTransform.FromPosition(new float3(10, 0, 0)));

Entity child = em.CreateEntity(typeof(LocalTransform), typeof(LocalToWorld), typeof(Parent));
em.SetComponentData(child, new Parent { Value = parent });
em.SetComponentData(child, LocalTransform.FromPosition(new float3(0, 5, 0)));
// Child's world position will be (10, 5, 0) = parent (10,0,0) + child local (0,5,0)

// Changing parent at runtime (structural change - use ECB in jobs)
EntityCommandBuffer ecb = new EntityCommandBuffer(Allocator.TempJob);
foreach (var (parent, entity) in
    SystemAPI.Query<RefRO<Parent>>()
        .WithEntityAccess()
        .WithAll<NeedsReparent>())
{
    Entity newParent = /* find new parent */;
    ecb.SetComponent(entity, new Parent { Value = newParent });

    // Or remove parent entirely (make world-space)
    ecb.RemoveComponent<Parent>(entity);
}
ecb.Playback(state.EntityManager);
ecb.Dispose();

// Querying parent-child relationships
Entity targetParent = /* ... */;
foreach (var (parent, transform, entity) in
    SystemAPI.Query<RefRO<Parent>, RefRW<LocalTransform>>()
        .WithEntityAccess())
{
    if (parent.ValueRO.Value == targetParent)
    {
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
```

#### Pros
- **Automatic hierarchy** - transform hierarchies work like GameObject parent-child relationships without manual matrix math

- **Efficient propagation** - `LocalToWorldSystem` updates all child transforms in parallel using jobs

- **Bidirectional access** - Parent component for upward lookup, [[Child]] buffer for downward iteration

- **Clean API** - single Entity reference is simple to understand and modify

#### Cons
- **Structural change** - adding/removing Parent moves entity to different archetype, causing overhead

- **No immediate update** - parent changes processed by `ParentSystem`, then `LocalToWorldSystem` computes new LocalToWorld next frame

- **Query complexity** - finding all descendants requires recursive traversal or separate hierarchy data structure

- **Deep hierarchy cost** - very deep hierarchies (10+ levels) incur matrix multiplication overhead each frame

#### Best use
- **Transform hierarchies** - characters with bones, vehicles with turrets, UI panels with child elements

- **Spatial attachment** - projectiles attached to moving platforms, weapons attached to character hands

- **Scene hierarchies** - building structures where rooms are children of buildings, furniture children of rooms

#### Avoid if
- **Static relationships** - if parent-child never changes at runtime, consider baking final world transforms

- **Frequent reparenting** - constantly changing parents causes repeated structural changes, consider alternatives

- **Very deep hierarchies** - hierarchies deeper than 10 levels may benefit from flattening or caching

- **No transform dependency** - if child doesn't need parent's transform (just logical grouping), use custom component

#### Extra tip
- **ParentSystem vs LocalToWorldSystem ordering:**
  ```csharp
  // Update order each frame:
  1. ParentSystem - Updates Parent/Child relationships (structural changes)
  2. LocalToWorldSystem - Computes LocalToWorld from hierarchy + LocalTransform
  3. Runs in TransformSystemGroup
  ```

- **Child buffer component (reverse lookup):**
  ```csharp
  // Parent entity automatically gets Child buffer with all children
  DynamicBuffer<Child> children = em.GetBuffer<Child>(parentEntity);
  for (int i = 0; i < children.Length; i++)
  {
      Entity childEntity = children[i].Value;
  }
  // Child buffer automatically maintained by ParentSystem
  ```

- **Removing parent (make world-space):**
  ```csharp
  ecb.RemoveComponent<Parent>(entity);
  // Entity's LocalTransform becomes world-space position
  ```

- **Setting parent during entity creation:**
  ```csharp
  Entity child = em.CreateEntity(typeof(LocalTransform), typeof(LocalToWorld), typeof(Parent));
  em.SetComponentData(child, new Parent { Value = parent });
  // ParentSystem will add Child buffer to parent automatically
  ```

- **Hierarchy traversal pattern:**
  ```csharp
  void ProcessHierarchy(Entity entity, EntityManager em)
  {
      // Process current entity

      // Process children
      if (em.HasBuffer<Child>(entity))
      {
          DynamicBuffer<Child> children = em.GetBuffer<Child>(entity);
          for (int i = 0; i < children.Length; i++)
          {
              ProcessHierarchy(children[i].Value, em);
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

- **Baking automatically preserves hierarchy** - GameObject parent-child relationships automatically become Parent components, no manual assignment needed

- **Parent changes are deferred** - changes via ECB are deferred until playback, Parent/Child buffers updated by ParentSystem, LocalToWorld updated by LocalToWorldSystem

- **Checking if entity has parent:**
  ```csharp
  // Entity with no Parent component is world-space
  bool isWorldSpace = !em.HasComponent<Parent>(entity);
  ```

- **Performance consideration** - Parent/Child relationship updates batched in ParentSystem, LocalToWorld computation parallelized, deep hierarchies (10+ levels) multiply matrices for each level each frame

