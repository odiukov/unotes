## Archetype

**Unique combination of [[Component]] types** that defines an [[Entity]]'s structure - entities with the same set of components share the same archetype.

Archetypes determine memory layout and enable efficient querying. Entities are grouped into [[Chunk|chunks]] by archetype, allowing systems to iterate over entities with specific components at maximum speed.

**Key characteristics:**
- **Component type set** - defined by which components an entity has (order doesn't matter)
- **Memory grouping** - all entities with same archetype stored together in chunks
- **Query matching** - systems query by archetype to find entities to process
- **Structural identity** - changing components changes archetype (moves entity to different chunk)

**Examples:**
```csharp
// Archetype A: [Health, Transform, Player]
Entity player;  // Has Health, Transform, Player components

// Archetype B: [Health, Transform, Enemy]
Entity enemy;   // Has Health, Transform, Enemy components

// Different archetypes - stored in different chunks!
```

**Archetype changes ([[Structural changes]]):**
- Adding component: entity moves to archetype with that component
- Removing component: entity moves to archetype without that component
- Structural changes trigger [[Sync points]] - avoid in hot paths

**Query example:**
```csharp
// Queries archetype: [Health, Transform, *]
// Matches any archetype containing Health AND Transform
foreach (var (health, transform) in
    SystemAPI.Query<RefRW<Health>, RefRW<Transform>>())
{
    // Only processes entities with both components
}
```

**Performance implications:**
- **Good:** Many entities per archetype = efficient iteration
- **Bad:** Many archetypes with few entities = poor cache utilization, sparse [[Chunk|chunks]]

**See also:** [[Entity]], [[Component]], [[Chunk]], [[Structural changes]]
