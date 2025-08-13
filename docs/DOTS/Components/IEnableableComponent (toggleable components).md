---
tags:
  - component
---
#### Description
- Components that can be **enabled/disabled** without [[Structural changes]].
#### Example
```csharp
public struct Stunned : IComponentData, IEnableableComponent {}
```
#### Pros
- No [[Structural changes]] when toggling.
      
- Easy filtering (`.WithDisabled<>, .WithPresent<>, .WithAll<>, WithNone<>`).
#### Cons
- **Extra bitmask** stored in chunk to track state.
      
- If **all enabled or all disabled** in a chunk → filtering is fast (like a normal component).
      
- If **mixed states** (e.g., 50% enabled, 50% disabled) → filtering is slower because each entity is checked individually.
      
- Still consumes memory in disabled state.
#### Best use
- Quick activity flags (“frozen”, “in combat”, “visible/invisible”).
#### Avoid if
- Large data structures (memory still used when disabled).
      
- Frequent toggling in [[Chunk|chunks]] with mixed states (perf loss).