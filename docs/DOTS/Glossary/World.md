## World

**Collection of [[Entity|entities]] and systems** that forms an isolated ECS container in Unity DOTS - entities exist within a specific world and systems process entities from their world.

A World provides **entity isolation** - an entity ID is only unique within its own world. The same ID number in different worlds refers to completely different entities.

**Key characteristics:**
- **Owns entities** - manages creation, storage, and destruction of entities through [[EntityManager]]
- **Owns systems** - contains systems that process entities, typically running once per frame
- **Isolated namespace** - entity IDs from one world are unrelated to IDs in another world
- **Multiple worlds possible** - applications can create multiple worlds for different purposes

**Common worlds:**
- **Default World** - main runtime world created automatically on play mode
- **Editor World** - separate world for editor-specific functionality
- **Streaming Worlds** - additional worlds for scene streaming or multi-world scenarios

**Example:**
```csharp
// Access the default world
World defaultWorld = World.DefaultGameObjectInjectionWorld;

// Create a custom world
World customWorld = new World("MyCustomWorld");

// Get the world's EntityManager
EntityManager entityManager = defaultWorld.EntityManager;

// Dispose when done
customWorld.Dispose();
```

**See also:** [[Entity]], [[EntityManager]], [[Component]], [[Archetype]]
