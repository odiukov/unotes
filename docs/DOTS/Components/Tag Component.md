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

#### See also
- [[ECS Design Decisions]] - Tag component design patterns and architectural considerations
- [[Tag-Based Behavior Selection]] - Advanced tag patterns for behavior control
- [[Disable Tag Pattern]] - Exclusion filtering with disable tags