---
tags:
  - component
  - advanced
---

#### Description
- **Single value per [[Chunk]]** component type belonging to chunk itself rather than individual entities - all entities in chunk share access to same value

- Different from [[ISharedComponentData]] - shared components logically belong to entities (changing value moves entity), chunk components belong to chunk itself

- **Stored in chunk** - unmanaged chunk components stored directly in 16KiB chunk block, managed chunk components stored separately

- Defined as struct or class implementing `IComponentData`, but added/accessed through special EntityManager chunk methods

#### Example
```csharp
// Define chunk component (same as regular IComponentData)
public struct ChunkWorldBounds : IComponentData
{
    public AABB WorldBounds;  // Bounding box for all entities in chunk
}

// System using chunk components
foreach (var (transform, chunk) in
    SystemAPI.Query<RefRO<LocalTransform>>()
        .WithEntityAccess())
{
    // Get the chunk this entity belongs to
    ArchetypeChunk archetypeChunk = em.GetChunk(entity);

    // Check if chunk has chunk component
    if (em.HasChunkComponent<ChunkWorldBounds>(archetypeChunk))
    {
        // Get chunk component value
        ChunkWorldBounds bounds = em.GetChunkComponentData<ChunkWorldBounds>(archetypeChunk);

        if (bounds.WorldBounds.Contains(transform.ValueRO.Position))
        {
            // Process entity...
        }
    }
}

// Adding chunk component when creating archetype
Entity entity = em.CreateEntity(typeof(LocalTransform), typeof(Health));
ArchetypeChunk chunk = em.GetChunk(entity);

// Add chunk component to the chunk
em.AddChunkComponentData<ChunkWorldBounds>(chunk);

// Set chunk component value
em.SetChunkComponentData(chunk, new ChunkWorldBounds
{
    WorldBounds = new AABB
    {
        Center = float3.zero,
        Extents = new float3(10, 10, 10)
    }
});

// Accessing in IJobChunk
[BurstCompile]
public struct MyChunkJob : IJobChunk
{
    public ComponentTypeHandle<ChunkWorldBounds> ChunkBoundsHandle;

    public void Execute(in ArchetypeChunk chunk, int unfilteredChunkIndex,
                       bool useEnabledMask, in v128 chunkEnabledMask)
    {
        // Access chunk component
        if (chunk.Has(ChunkBoundsHandle))
        {
            ChunkWorldBounds bounds = chunk.GetChunkComponentData(ref ChunkBoundsHandle);
            // Process using bounds...
        }
    }
}
```

#### Pros
- **Efficient shared data** - one value per chunk rather than duplicating per entity, saves memory

- **Direct chunk storage** - unmanaged chunk components stored in chunk's 16KiB block, no indirection

- **Useful for metadata** - perfect for storing chunk-level information like bounding boxes, timestamps, or processing flags

- **No entity movement** - unlike [[ISharedComponentData]], setting chunk component value doesn't move entities between chunks

#### Cons
- **Limited use cases** - most data belongs to entities or is global, chunk-level data relatively rare

- **Awkward API** - requires obtaining ArchetypeChunk first, then using special EntityManager methods

- **Not query-friendly** - cannot query for entities based on chunk component values like regular components

- **Chunk lifetime** - chunk component exists only while chunk exists, destroyed when chunk destroyed

#### Best use
- **Chunk-level metadata** - bounding boxes, spatial hashing keys, processing timestamps for entire chunk

- **Batch processing flags** - marking chunks as "needs update", "dirty", "processed" for optimization

- **Chunk-wide constants** - data that applies uniformly to all entities in chunk

#### Avoid if
- **Entity-specific data** - use regular [[IComponentData]] for per-entity values

- **Shared entity data** - use [[ISharedComponentData]] when multiple entities share value and should be grouped

- **Global data** - use singleton components for world-wide configuration or state

#### Extra tip
- **EntityManager chunk component methods:**
  ```csharp
  em.AddChunkComponentData<T>(ArchetypeChunk chunk);
  em.RemoveChunkComponentData<T>(ArchetypeChunk chunk);
  bool has = em.HasChunkComponent<T>(ArchetypeChunk chunk);
  T value = em.GetChunkComponentData<T>(ArchetypeChunk chunk);
  em.SetChunkComponentData<T>(ArchetypeChunk chunk, T value);
  ```

- **Getting entity's chunk:**
  ```csharp
  ArchetypeChunk chunk = entityManager.GetChunk(entity);
  ```

- **Chunk component vs Shared component:**
  - **Chunk component:** Belongs to chunk, changing value doesn't move entities
  - **Shared component:** Belongs to entities, changing value moves entity to different chunk
  - Use shared components when value determines chunk grouping
  - Use chunk components when value is consequence of chunk's contents

- **Managed chunk components** - can create from classes (managed IComponentData), stored separately with GC overhead

- **Chunk creation and destruction:**
  - Chunk components created when chunk created (entity added to full archetype)
  - Chunk components destroyed when chunk destroyed (last entity removed)
  - Don't persist independently of chunks

## See Also

- [[IComponentData]] - Regular per-entity components
- [[ISharedComponentData]] - Shared value components
- [[Chunk]] - 16KiB memory blocks storing entities
- [[Archetype]] - Unique component type combinations
- [[EntityManager]] - Entity and component operations
