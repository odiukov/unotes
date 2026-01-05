## EntityManager

**Primary interface for creating, destroying, and modifying [[Entity|entities]]** and their [[Component|components]] in a [[World]] - handles all [[Structural changes]] and entity data manipulation.

The EntityManager is the central authority for entity operations, managing the underlying [[Archetype|archetypes]] and [[Chunk|chunks]] automatically as entities are created and modified.

**Key characteristics:**
- **One per World** - each [[World]] has its own EntityManager accessed via `World.EntityManager`
- **Structural change authority** - only way to add/remove components or create/destroy entities
- **Main thread only** - most operations must run on main thread (use [[EntityCommandBuffer]] in jobs)
- **Automatic archetype management** - creates and manages archetypes transparently

**Common methods:**

**Entity creation/destruction:**
```csharp
Entity entity = entityManager.CreateEntity();
Entity copy = entityManager.Instantiate(entity);
entityManager.DestroyEntity(entity);
```

**Component operations:**
```csharp
entityManager.AddComponent<Health>(entity);
entityManager.RemoveComponent<Health>(entity);
bool has = entityManager.HasComponent<Health>(entity);

Health health = entityManager.GetComponent<Health>(entity);
entityManager.SetComponent(entity, new Health { Value = 100 });
```

**Buffer operations:**
```csharp
DynamicBuffer<Waypoint> buffer = entityManager.GetBuffer<Waypoint>(entity);
entityManager.AddBuffer<Waypoint>(entity);
```

**[[EntityQuery]] operations:**
```csharp
EntityQuery query = entityManager.CreateEntityQuery(typeof(Health), typeof(Transform));
NativeArray<Entity> entities = query.ToEntityArray(Allocator.Temp);
```

**Important:** Operations that add/remove components or create/destroy entities are [[Structural changes]] that can trigger [[Sync points]]. In jobs, use [[EntityCommandBuffer]] to defer these changes to the main thread.

**See also:** [[Entity]], [[World]], [[Component]], [[EntityCommandBuffer]], [[Structural changes]]
