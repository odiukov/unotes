## Entity

**Unique identifier (ID) for a game object** in Unity ECS - lightweight handle that represents an instance without storing any data itself.

An Entity is just an **integer index** combined with a **version number**, taking only 8 bytes of memory. All actual game data lives in [[Component|components]] attached to the entity.

**Key characteristics:**
- **ID only** - contains no data, just points to components
- **Lightweight** - 8 bytes (int Index + int Version)
- **Version tracking** - version increments when entity destroyed and ID reused, preventing stale references
- **Archetype-based** - entity's [[Archetype]] defines what components it has

**Example:**
```csharp
Entity playerEntity;  // Just an ID
```

The entity with components Health, Transform, and Player has archetype `[Health, Transform, Player]` and lives in a [[Chunk]] with other entities of the same archetype.

**See also:** [[Component]], [[Archetype]], [[IComponentData]]
