---
tags:
  - component
---
#### Description
- Empty [[IComponentData]] or [[ICleanupComponentData]].
      
- Used only for marking entities.
#### Example
```csharp
public struct PlayerTag : IComponentData {}
```
#### Pros
- Minimal memory overhead.
      
- Simple filtering.

#### Cons
- No data, only a flag.

#### Best use
- Mark entity type (enemy, player, projectile).
      
- Toggle behavior quickly.