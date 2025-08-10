---
tags:
  - component
---
**Description**
- Components that **remain** on an entity after `EntityManager.DestroyEntity` is called.
- Removed only **manually** or during world cleanup.
- Used for “final actions” after entity destruction.

**Example**
```csharp
public struct NetworkDisconnectTag : ICleanupComponentData {}
```

**Pros**
- Great for finalisation logic.
- Allows systems to process “entity death” in the next frame.

**Cons**
- Not automatically removed (must be handled).
- Consumes memory until cleared.

**Best use**
- Freeing GPU/network resources after entity deletion.
- Post-processing logic (death effects, final events).

**Avoid if**
- Data must disappear instantly on deletion.