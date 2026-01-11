---
tags:
  - pattern
---
#### Description
- **Reversible operation pattern** where adding request [[Component]] triggers work, removing it reverses the operation - allows repeatable on/off control

- Request components persist on entities and can be added/removed multiple times, unlike one-time request entities

- Unity's `Unity.Scenes` uses `RequestSceneLoaded` - adding loads scene, removing unloads

- Enables declarative control: "this entity should have scene loaded" vs imperative "load this scene now"

#### Example
```csharp
// Request component
public struct RequestSceneLoaded : IComponentData
{
    public Hash128 SceneGUID;
}

// State tracking component
public struct SceneLoaded : IComponentData { }

// System processes requests
[BurstCompile]
public partial struct ProcessSceneLoadRequestSystem : ISystem
{
    public void OnUpdate(ref SystemState state)
    {
        // Load when request added
        foreach (var (request, entity) in
            SystemAPI.Query<RefRO<RequestSceneLoaded>>()
                .WithNone<SceneLoaded>()
                .WithEntityAccess())
        {
            LoadScene(request.ValueRO.SceneGUID);
            state.EntityManager.AddComponent<SceneLoaded>(entity);
        }

        // Unload when request removed
        foreach (var (loaded, entity) in
            SystemAPI.Query<RefRO<SceneLoaded>>()
                .WithNone<RequestSceneLoaded>()  // Request removed
                .WithEntityAccess())
        {
            UnloadScene(loaded.ValueRO.SceneGUID);
            state.EntityManager.RemoveComponent<SceneLoaded>(entity);
        }
    }
}

// Usage
entityManager.AddComponent(entity, new RequestSceneLoaded { SceneGUID = guid });  // Load
entityManager.RemoveComponent<RequestSceneLoaded>(entity);  // Unload
```

**Pattern structure:**
1. **Request component** - added by user code
2. **State component** - added by system when operation completes
3. **Two queries** - `.WithAll<Request>().WithNone<State>()` for new requests, `.WithAll<State>().WithNone<Request>()` for removal

**Common use cases:**
- Scene loading/unloading
- Network connect/disconnect
- Resource load/unload
- Audio play/stop
- Subscriptions
- Rendering enable/disable

#### Pros
- **Reversible operations** - easily undo by removing request component, natural on/off control

- **Declarative state** - presence of component declares desired state, system makes it happen

- **Repeatable** - can add/remove request multiple times for repeated operations

- **Clean API** - simple add/remove component API vs complex state machine

- **Testable** - easy to test by adding/removing component and verifying state changes

#### Cons
- **[[Structural changes]]** - adding/removing components expensive if frequent

- **Requires state tracking** - need additional component to track operation completion

- **One-frame delay** - system processes request on next update, not immediately

- **Cleanup complexity** - must handle cleanup when request removed

- **Can be forgotten** - developers might add request but forget to remove, causing resource leaks

#### Best use
- **Scene loading/unloading** - Unity.Scenes canonical example

- **Network connections** - connect when requested, disconnect when removed

- **Resource loading** - load assets when needed, unload when no longer requested

- **Audio playback** - start audio on request, stop when removed

- **Subscriptions** - subscribe to data feeds, unsubscribe when removed

#### Avoid if
- **One-time operations** - for fire-and-forget, use request entities destroyed after processing

- **Frequent toggling** - if toggling every frame, [[IEnableableComponent (toggleable components)|IEnableableComponent]] avoids structural changes

- **Immediate execution needed** - request pattern has one-frame delay

- **Complex state machines** - if operation has many states beyond on/off, use proper state machine

#### Extra tip
- **Pair with state component**: Use separate component for state (Loading, Loaded, Failed) beyond just presence/absence

- **Unity.Scenes example**: `RequestSceneLoaded` is real component from Unity packages - check source

- **Cleanup pattern integration**: Combine with [[Cleanup Component Pattern]] to detect removal via cleanup components

- **Async operations**: Request pattern works well for async - add request, add "in progress" component, poll completion, remove request to cleanup

- **Multiple requests**: Can have multiple request types on same entity simultaneously

- **Request parameters**: Include operation parameters in request component struct

- **Reference counting**: For shared resources, track reference count - only unload when count reaches 0

- **Disable without removing**: Combine with [[Disable Tag Pattern]] - `.WithAll<Request>().WithNone<DisableProcessing>()`

- **ECB usage**: Always use [[EntityCommandBuffer]] for structural changes in request systems

- **Performance tip**: Use `[RequireMatchingQueriesForUpdate]` to skip system when no requests

- **IEnableableComponent alternative**: For frequent toggling, use IEnableableComponent to avoid structural changes:
  ```csharp
  public struct RequestRendering : IComponentData, IEnableableComponent { }
  SystemAPI.SetComponentEnabled<RequestRendering>(entity, true);  // No structural change
  ```

- **Naming conventions**: Prefix with `Request`, suffix state with `State`/`Active`/`InProgress`/`Loaded`

- **Documentation critical**: Clearly document what happens when request added vs removed

- **Testing both paths**: Test request addition and removal in unit tests

- **Request vs request entities**:
  - **Request components**: Persist on entity, reversible, repeatable
  - **Request entities**: Destroyed after processing, one-time, fire-and-forget

- **Architectural context**: See [[ECS Design Decisions]] for command component pattern and when to choose request patterns vs immediate execution
