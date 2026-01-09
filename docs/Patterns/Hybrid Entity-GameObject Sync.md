---
tags:
  - pattern
---
#### Description
- **Bidirectional lifecycle management** pattern synchronizing [[Entity]] and GameObject destruction to prevent dangling references in hybrid workflows

- MonoBehaviour tracks its corresponding Entity and World references, automatically destroying Entity when GameObject is destroyed

- Prevents "entity does not exist" errors in systems that reference GameObjects via managed components

- Essential for hybrid architectures mixing ECS logic with MonoBehaviour-based systems (UI, animation, legacy code)

#### Example
```csharp
// Core MonoBehaviour tracking Entity lifecycle
public class EntityGameObject : MonoBehaviour
{
    public Entity Entity;
    public World World;

    public void AssignEntity(Entity e, World world)
    {
        Entity = e;
        World = world;
    }

    // Bidirectional sync: GameObject destroyed → destroy Entity
    private void OnDestroy()
    {
        if (World != null && World.IsCreated &&
            World.EntityManager.Exists(Entity))
        {
            World.EntityManager.DestroyEntity(Entity);
        }
    }
}

// Creating hybrid entity
var entity = ecb.CreateEntity();
var go = new GameObject("HybridEntity");
var tracker = go.AddComponent<EntityGameObject>();
tracker.AssignEntity(entity, state.World);

// Store GameObject reference in managed component
ecb.AddComponent(entity, new GameObjectReference { Value = go });

// System syncing ECS to GameObject
Entities
    .WithoutBurst()
    .ForEach((in LocalTransform transform, in GameObjectReference goRef) =>
    {
        if (goRef.Value != null)
        {
            goRef.Value.transform.position = transform.Position;
        }
    }).Run();
```

**Lifecycle flow:**
1. Create ECS Entity
2. Create GameObject and add EntityGameObject MonoBehaviour
3. Call `AssignEntity()` to establish link
4. Store GameObject reference in managed component
5. OnDestroy automatically cleans up Entity when GameObject destroyed

**Validation checks:**
- Always check `World.IsCreated` before accessing EntityManager
- Always check `goRef.Value != null` before using GameObject
- Always check `EntityManager.Exists(entity)` before destroying

#### Pros
- **Lifecycle consistency** - prevents dangling references when GameObject destroyed unexpectedly (scene unload, user deletion)

- **Error prevention** - automatic Entity cleanup eliminates "entity does not exist" errors in systems

- **Bidirectional safety** - handles destruction from either side (GameObject.Destroy or EntityManager.DestroyEntity)

- **Simple API** - single AssignEntity call establishes sync, OnDestroy handles cleanup automatically

- **Debugging friendly** - clear ownership relationship visible in inspector and hierarchy

#### Cons
- **Main thread only** - GameObject operations not Burst-compatible, managed components break parallelism

- **Performance overhead** - managed components and GameObject references add CPU cost vs pure ECS

- **Memory overhead** - GameObject allocation much heavier than pure Entity (KB vs bytes)

- **Complexity** - hybrid architecture harder to reason about than pure ECS or pure GameObject

- **World transition issues** - requires careful handling when switching worlds or during domain reload

#### Best use
- **UI integration** - ECS gameplay logic driving MonoBehaviour UI (health bars, tooltips, menus)

- **Animation systems** - ECS controlling Animator/Animation components on GameObjects

- **Legacy code integration** - gradual ECS migration while maintaining MonoBehaviour systems

- **Third-party assets** - integrating asset store plugins requiring GameObjects with ECS logic

- **Visual debugging** - attaching debug GameObjects to ECS entities for scene view visualization

#### Avoid if
- **Pure ECS possible** - if gameplay logic can be 100% ECS, avoid GameObject overhead entirely

- **Performance-critical** - hybrid sync adds frame cost, pure ECS 10-100x faster for most operations

- **Large entity counts** - thousands of hybrid entities create garbage and slow down systems

- **Burst-critical code** - managed components prevent Burst compilation, eliminating main ECS performance benefit

#### Extra tip
- **Null checks essential**: Always validate GameObject exists before use - destroyed GameObjects = null reference

- **World validation**: Check `World.IsCreated` before accessing EntityManager to avoid crashes

- **Alternative: CompanionLink**: Unity provides built-in `CompanionGameObjectSystemGroup` for official hybrid support

- **Minimize hybrid entities**: Use hybrid only where necessary (visuals, audio), keep logic pure ECS - aim for 10 hybrid vs 1000 pure ECS

- **Cleanup component pattern**: Use [[Cleanup Component Pattern]] for automatic GameObject destruction when Entity destroyed

- **Pooling for performance**: Reuse GameObjects to avoid allocation cost - 1000 spawns/sec: pooling ~5ms/frame vs new ~50ms/frame + GC

- **Presentation layer only**: Keep hybrid systems in PresentationSystemGroup, simulation systems pure ECS

- **Entity.Null pattern**: Reset Entity reference when pooling/despawning to prevent OnDestroy from destroying invalid entity

- **Domain reload handling**: Clear static references on domain reload to avoid errors: `[RuntimeInitializeOnLoadMethod]`

- **Performance comparison**: Pure ECS 10,000 entities @ 0.1ms vs Hybrid @ 5ms (50x slower)

- **Memory comparison**: Pure ECS entity ~100 bytes vs Hybrid ~5KB (50x larger)

- **Best practices**: Hybrid sparingly, validate existence, presentation layer only, pool GameObjects, pure ECS alternatives first

- **Migration strategy**: GameObject → Hybrid Entity (logic in ECS) → Pure Entity + visual GameObject → Pure Entity + Entities Graphics
