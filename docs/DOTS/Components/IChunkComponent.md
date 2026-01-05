---
tags:
  - component
  - advanced
---

#### Description
- **Single value per [[Chunk]]** component type that belongs to the chunk itself rather than individual entities - all entities in chunk share access to same chunk component value

- Different from [[ISharedComponentData]] - shared components logically belong to entities (changing value moves entity to different chunk), while chunk components belong to the chunk itself

- **Stored in chunk** - unmanaged chunk components are stored directly in the 16KiB chunk block, managed chunk components stored separately

- Defined as struct or class implementing `IComponentData`, but added/accessed through special EntityManager chunk methods

#### Example
```csharp
using Unity.Entities;
using Unity.Mathematics;

// Define chunk component (same as regular IComponentData)
public struct ChunkWorldBounds : IComponentData
{
    public AABB WorldBounds;  // Bounding box for all entities in chunk
}

// System using chunk components
[BurstCompile]
public partial struct ChunkBoundsSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        EntityManager em = state.EntityManager;

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

                // Check if entity is within chunk bounds
                if (bounds.WorldBounds.Contains(transform.ValueRO.Position))
                {
                    // Process entity...
                }
            }
        }
    }
}

// Adding chunk component when creating archetype
public void OnCreate(ref SystemState state)
{
    EntityManager em = state.EntityManager;

    // Create entity
    Entity entity = em.CreateEntity(typeof(LocalTransform), typeof(Health));

    // Get chunk containing this entity
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
}
```

#### Pros
- **Efficient shared data** - one value per chunk rather than duplicating per entity, saves memory for chunk-wide metadata

- **Direct chunk storage** - unmanaged chunk components stored in chunk's 16KiB block, no indirection for access

- **Useful for metadata** - perfect for storing chunk-level information like bounding boxes, timestamps, or processing flags

- **No entity movement** - unlike [[ISharedComponentData]], setting chunk component value doesn't move entities between chunks

#### Cons
- **Limited use cases** - most data belongs to entities or is global, chunk-level data is relatively rare

- **Awkward API** - requires obtaining ArchetypeChunk first, then using special EntityManager methods (not as convenient as regular components)

- **Not query-friendly** - cannot query for entities based on chunk component values like regular components

- **Chunk lifetime** - chunk component exists only while chunk exists, destroyed when chunk destroyed by EntityManager

#### Best use
- **Chunk-level metadata** - bounding boxes, spatial hashing keys, processing timestamps for entire chunk

- **Batch processing flags** - marking chunks as "needs update", "dirty", "processed" for optimization

- **Chunk-wide constants** - data that applies uniformly to all entities in chunk (but consider [[ISharedComponentData]] instead)

#### Avoid if
- **Entity-specific data** - use regular [[IComponentData]] for per-entity values

- **Shared entity data** - use [[ISharedComponentData]] when multiple entities share same value and should be grouped in chunks

- **Global data** - use singleton components for world-wide configuration or state

#### Extra tip
- **EntityManager chunk component methods:**
  ```csharp
  // Add chunk component to chunk
  em.AddChunkComponentData<T>(ArchetypeChunk chunk);

  // Remove chunk component from chunk
  em.RemoveChunkComponentData<T>(ArchetypeChunk chunk);

  // Check if chunk has chunk component
  bool has = em.HasChunkComponent<T>(ArchetypeChunk chunk);

  // Get chunk component value
  T value = em.GetChunkComponentData<T>(ArchetypeChunk chunk);

  // Set chunk component value
  em.SetChunkComponentData<T>(ArchetypeChunk chunk, T value);
  ```

- **Getting entity's chunk:**
  ```csharp
  Entity entity = /* ... */;
  ArchetypeChunk chunk = entityManager.GetChunk(entity);
  ```

- **Accessing in IJobChunk:**
  ```csharp
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

- **Chunk component vs Shared component:**
  - **Chunk component**: Belongs to chunk, changing value doesn't move entities
  - **Shared component**: Belongs to entities, changing value moves entity to different chunk
  - Use shared components when value determines chunk grouping
  - Use chunk components when value is consequence of chunk's contents

- **Managed chunk components:**
  - Can create chunk components from classes (managed IComponentData)
  - Stored separately from chunk like managed entity components
  - Incurs GC overhead but allows managed types (strings, arrays, etc.)

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
