## EntityQuery

**Efficient filtering system** that finds all [[Entity|entities]] having a specified set of [[Component]] types - core mechanism for processing entities in Unity DOTS systems.

An EntityQuery gathers [[Chunk|chunks]] from all matching [[Archetype|archetypes]], enabling fast iteration over large numbers of entities without searching the entire [[World]].

**Key characteristics:**
- **Archetype matching** - finds archetypes that include specified component types
- **Cached results** - matching archetypes are cached until new archetypes are added to the world
- **Flexible filtering** - supports inclusion, exclusion, and optional component types
- **[[Cache-friendly]]** - returns entire chunks for efficient sequential processing

**Query construction:**
```csharp
// All entities with Health AND Transform components
EntityQuery query = entityManager.CreateEntityQuery(
    typeof(Health),
    typeof(Transform)
);

// With exclusions - has Health and Transform, but NOT Dead
EntityQuery query2 = new EntityQueryBuilder(Allocator.Temp)
    .WithAll<Health, Transform>()
    .WithNone<Dead>()
    .Build(ref systemState);
```

**Querying behavior:**
- Query for `[Health, Transform]` matches entities with:
  - Exactly `[Health, Transform]` ✓
  - Also `[Health, Transform, Player]` ✓ (has more components)
  - But NOT `[Health]` alone ✗ (missing Transform)

**Common operations:**
```csharp
// Get entity count
int count = query.CalculateEntityCount();

// Get entities
NativeArray<Entity> entities = query.ToEntityArray(Allocator.Temp);

// Get component arrays
NativeArray<Health> healths = query.ToComponentDataArray<Health>(Allocator.Temp);

// Iterate through chunks
foreach (ArchetypeChunk chunk in query.ToArchetypeChunkArray(Allocator.Temp))
{
    // Process chunk...
}
```

**In systems:**
```csharp
[BurstCompile]
public partial struct HealthSystem : ISystem
{
    EntityQuery query;

    [BurstCompile]
    public void OnCreate(ref SystemState state)
    {
        // Create query once
        query = state.GetEntityQuery(typeof(Health), typeof(Transform));
    }

    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Use query for processing
        foreach (var (health, transform) in
            SystemAPI.Query<RefRW<Health>, RefRO<Transform>>())
        {
            // Process entities...
        }
    }
}
```

**Performance note:** Query caching makes repeated queries very cheap. The archetype cache is only invalidated when new archetypes are added, which stabilizes early in the program lifetime.

**See also:** [[Entity]], [[Component]], [[Archetype]], [[Chunk]], [[SystemAPI]]
