---
tags:
  - pattern
---
#### Description
- **Reversible operation pattern** where adding a request [[Component]] triggers work, and removing it reverses the operation, allowing repeatable on/off control

- Different from one-time request entities - request components persist on entities and can be added/removed multiple times

- Unity's `Unity.Scenes` uses `RequestSceneLoaded` component - adding it loads the scene, removing it unloads

- Enables declarative control: "this entity should have scene loaded" vs imperative "load this scene now"

#### Example
```csharp
// Request component for scene loading
public struct RequestSceneLoaded : IComponentData
{
    public Hash128 SceneGUID;
}

// System processes request component
[BurstCompile]
public partial struct ProcessSceneLoadRequestSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        // Load scenes for entities with request component
        foreach (var (request, entity) in
            SystemAPI.Query<RefRO<RequestSceneLoaded>>()
                .WithNone<SceneLoaded>()  // Not already loaded
                .WithEntityAccess())
        {
            // Trigger scene load
            LoadScene(request.ValueRO.SceneGUID);

            // Mark as loaded
            state.EntityManager.AddComponent<SceneLoaded>(entity);
        }

        // Unload scenes when request component removed
        foreach (var (loaded, entity) in
            SystemAPI.Query<RefRO<SceneLoaded>>()
                .WithNone<RequestSceneLoaded>()  // Request removed
                .WithEntityAccess())
        {
            // Trigger scene unload
            UnloadScene(loaded.ValueRO.SceneGUID);

            // Remove loaded marker
            state.EntityManager.RemoveComponent<SceneLoaded>(entity);
        }
    }
}

// Usage: Load scene
Entity sceneEntity = entityManager.CreateEntity();
entityManager.AddComponent(sceneEntity, new RequestSceneLoaded
{
    SceneGUID = sceneGuid
});

// Usage: Unload scene
entityManager.RemoveComponent<RequestSceneLoaded>(sceneEntity);

// Example: Audio request component
public struct RequestAudioPlay : IComponentData
{
    public Entity AudioClip;
    public float Volume;
}

public struct AudioPlaying : IComponentData
{
    public int AudioSourceID;
}

[BurstCompile]
public partial struct AudioRequestSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Start audio when request added
        foreach (var (request, entity) in
            SystemAPI.Query<RefRO<RequestAudioPlay>>()
                .WithNone<AudioPlaying>()
                .WithEntityAccess())
        {
            // Play audio
            int sourceId = PlayAudio(request.ValueRO.AudioClip, request.ValueRO.Volume);

            // Mark as playing
            ecb.AddComponent(entity, new AudioPlaying { AudioSourceID = sourceId });
        }

        // Stop audio when request removed
        foreach (var (playing, entity) in
            SystemAPI.Query<RefRO<AudioPlaying>>()
                .WithNone<RequestAudioPlay>()
                .WithEntityAccess())
        {
            // Stop audio
            StopAudio(playing.ValueRO.AudioSourceID);

            // Remove playing marker
            ecb.RemoveComponent<AudioPlaying>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Network connection request
public struct RequestNetworkConnection : IComponentData
{
    public FixedString64Bytes ServerAddress;
    public ushort Port;
}

public struct NetworkConnected : IComponentData
{
    public int ConnectionID;
}

[BurstCompile]
public partial struct NetworkConnectionSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Connect when request added
        foreach (var (request, entity) in
            SystemAPI.Query<RefRO<RequestNetworkConnection>>()
                .WithNone<NetworkConnected>()
                .WithEntityAccess())
        {
            // Initiate connection
            int connectionId = ConnectToServer(
                request.ValueRO.ServerAddress,
                request.ValueRO.Port);

            ecb.AddComponent(entity, new NetworkConnected
            {
                ConnectionID = connectionId
            });
        }

        // Disconnect when request removed
        foreach (var (connected, entity) in
            SystemAPI.Query<RefRO<NetworkConnected>>()
                .WithNone<RequestNetworkConnection>()
                .WithEntityAccess())
        {
            // Close connection
            DisconnectFromServer(connected.ValueRO.ConnectionID);

            ecb.RemoveComponent<NetworkConnected>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Subscription pattern
public struct RequestDataSubscription : IComponentData
{
    public FixedString64Bytes DataChannel;
}

public struct SubscriptionActive : IComponentData
{
    public Entity SubscriptionHandle;
}

[BurstCompile]
public partial struct DataSubscriptionSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Subscribe when request added
        foreach (var (request, entity) in
            SystemAPI.Query<RefRO<RequestDataSubscription>>()
                .WithNone<SubscriptionActive>()
                .WithEntityAccess())
        {
            Entity handle = Subscribe(request.ValueRO.DataChannel);
            ecb.AddComponent(entity, new SubscriptionActive { SubscriptionHandle = handle });
        }

        // Unsubscribe when request removed
        foreach (var (subscription, entity) in
            SystemAPI.Query<RefRO<SubscriptionActive>>()
                .WithNone<RequestDataSubscription>()
                .WithEntityAccess())
        {
            Unsubscribe(subscription.ValueRO.SubscriptionHandle);
            ecb.RemoveComponent<SubscriptionActive>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Resource loading
public struct RequestResourceLoad : IComponentData
{
    public FixedString128Bytes ResourcePath;
}

public struct ResourceLoaded : IComponentData
{
    public Entity ResourceHandle;
}

[BurstCompile]
public partial struct ResourceLoadingSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Load when request added
        foreach (var (request, entity) in
            SystemAPI.Query<RefRO<RequestResourceLoad>>()
                .WithNone<ResourceLoaded>()
                .WithEntityAccess())
        {
            Entity resource = LoadResource(request.ValueRO.ResourcePath);
            ecb.AddComponent(entity, new ResourceLoaded { ResourceHandle = resource });
        }

        // Unload when request removed
        foreach (var (loaded, entity) in
            SystemAPI.Query<RefRO<ResourceLoaded>>()
                .WithNone<RequestResourceLoad>()
                .WithEntityAccess())
        {
            UnloadResource(loaded.ValueRO.ResourceHandle);
            ecb.RemoveComponent<ResourceLoaded>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Enable/disable rendering
public struct RequestRendering : IComponentData { }
public struct RenderingActive : IComponentData
{
    public Entity RenderEntity;
}

[BurstCompile]
public partial struct RenderingRequestSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Enable rendering
        foreach (var entity in
            SystemAPI.QueryBuilder()
                .WithAll<RequestRendering>()
                .WithNone<RenderingActive>()
                .Build()
                .ToEntityArray(Allocator.Temp))
        {
            // Add rendering components
            Entity renderEntity = CreateRenderEntity(entity);
            ecb.AddComponent(entity, new RenderingActive { RenderEntity = renderEntity });
        }

        // Disable rendering
        foreach (var (active, entity) in
            SystemAPI.Query<RefRO<RenderingActive>>()
                .WithNone<RequestRendering>()
                .WithEntityAccess())
        {
            // Destroy render entity
            ecb.DestroyEntity(active.ValueRO.RenderEntity);
            ecb.RemoveComponent<RenderingActive>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}

// Example: Async operation tracking
public struct RequestAsyncOperation : IComponentData
{
    public int OperationID;
}

public struct OperationInProgress : IComponentData
{
    public float Progress;
}

public struct OperationCompleted : IComponentData
{
    public bool Success;
}

[BurstCompile]
public partial struct AsyncOperationSystem : ISystem
{
    [BurstCompile]
    public void OnUpdate(ref SystemState state)
    {
        var ecb = new EntityCommandBuffer(Allocator.Temp);

        // Start operation
        foreach (var (request, entity) in
            SystemAPI.Query<RefRO<RequestAsyncOperation>>()
                .WithNone<OperationInProgress, OperationCompleted>()
                .WithEntityAccess())
        {
            StartAsyncOperation(request.ValueRO.OperationID);
            ecb.AddComponent(entity, new OperationInProgress { Progress = 0f });
        }

        // Update progress
        foreach (var (inProgress, entity) in
            SystemAPI.Query<RefRW<OperationInProgress>>()
                .WithAll<RequestAsyncOperation>()
                .WithEntityAccess())
        {
            float progress = GetOperationProgress(entity);
            inProgress.ValueRW.Progress = progress;

            if (progress >= 1f)
            {
                ecb.RemoveComponent<OperationInProgress>(entity);
                ecb.AddComponent(entity, new OperationCompleted { Success = true });
            }
        }

        // Cancel operation when request removed
        foreach (var (inProgress, entity) in
            SystemAPI.Query<RefRO<OperationInProgress>>()
                .WithNone<RequestAsyncOperation>()
                .WithEntityAccess())
        {
            CancelAsyncOperation(entity);
            ecb.RemoveComponent<OperationInProgress>(entity);
        }

        ecb.Playback(state.EntityManager);
        ecb.Dispose();
    }
}
```

#### Pros
- **Reversible operations** - can easily undo by removing request component, natural on/off control

- **Declarative state** - presence of component declares desired state, system makes it happen

- **Repeatable** - can add/remove request multiple times for repeated operations

- **Clean API** - simple add/remove component API vs complex state machine

- **Testable** - easy to test by adding/removing component and verifying state changes

#### Cons
- **[[Structural changes]]** - adding/removing components triggers structural changes, can be expensive if frequent

- **Requires state tracking** - need additional component (e.g., SceneLoaded) to track operation completion

- **One-frame delay** - system processes request on next update, not immediately when component added

- **Cleanup complexity** - must handle cleanup when request removed, can be non-trivial for complex operations

- **Can be forgotten** - developers might add request but forget to remove it, causing resource leaks

#### Best use
- **Scene loading/unloading** - Unity.Scenes canonical example, load when added, unload when removed

- **Network connections** - connect when requested, disconnect when request removed

- **Resource loading** - load assets when needed, unload when no longer requested

- **Audio playback** - start audio on request, stop when request removed

- **Subscriptions** - subscribe to data feeds, unsubscribe when request removed

#### Avoid if
- **One-time operations** - for fire-and-forget operations, use request entities that are destroyed after processing

- **Frequent toggling** - if operation toggles every frame, structural changes are too expensive, use [[IEnableableComponent (toggleable components)|IEnableableComponent]] instead

- **Immediate execution needed** - request pattern has one-frame delay, use direct function calls for immediate execution

- **Complex state machines** - if operation has many states beyond on/off, use proper state machine components

#### Extra tip
- **Pair with state component** - use separate component to track operation state (Loading, Loaded, Failed):
  ```csharp
  public struct RequestSceneLoaded : IComponentData { }
  public struct SceneLoadState : IComponentData
  {
      public enum State { Loading, Loaded, Failed }
      public State Current;
  }
  ```

- **Unity.Scenes example** - `RequestSceneLoaded` is real component from Unity packages, check source for implementation details

- **Cleanup pattern integration** - combine with [[Cleanup Component Pattern]] to detect removal:
  ```csharp
  // Auto-add cleanup component when request added
  foreach (var entity in
      SystemAPI.QueryBuilder()
          .WithAll<RequestSceneLoaded>()
          .WithNone<PreviousRequestSceneLoaded>()
          .Build()
          .ToEntityArray(Allocator.Temp))
  {
      ecb.AddComponent<PreviousRequestSceneLoaded>(entity);
  }
  ```

- **Error handling** - track failures in state component:
  ```csharp
  public struct ResourceLoadState : IComponentData
  {
      public bool IsLoading;
      public bool IsLoaded;
      public bool HasError;
  }
  ```

- **Async operations** - request pattern works well with async operations:
  - Add request component
  - Add "in progress" component
  - Poll/wait for completion
  - Add "completed" component
  - Remove request to trigger cleanup

- **Multiple requests** - can have multiple request types on same entity:
  ```csharp
  entityManager.AddComponent<RequestSceneLoaded>(entity);
  entityManager.AddComponent<RequestNetworkConnection>(entity);
  ```

- **Request parameters** - include operation parameters in request component:
  ```csharp
  public struct RequestSceneLoaded : IComponentData
  {
      public Hash128 SceneGUID;
      public int LoadPriority;
      public bool BlockOnLoad;
  }
  ```

- **Reference counting** - for shared resources, track reference count:
  ```csharp
  public struct ResourceReferenceCount : IComponentData
  {
      public int Count;
  }
  // Only unload when Count reaches 0
  ```

- **Combining with [[Disable Tag Pattern]]** - can disable request processing without removing request:
  ```csharp
  .WithAll<RequestSceneLoaded>()
  .WithNone<DisableSceneLoading>()
  ```

- **[[EntityCommandBuffer]] usage** - always use ECB for structural changes in request systems

- **Performance tip** - use [[RequireMatchingQueriesForUpdate]] to skip system when no requests:
  ```csharp
  [RequireMatchingQueriesForUpdate]
  public partial struct ProcessRequestsSystem : ISystem { }
  ```

- **IEnableableComponent alternative** - for frequent toggling without structural changes:
  ```csharp
  public struct RequestRendering : IComponentData, IEnableableComponent { }

  // Toggle without structural changes
  SystemAPI.SetComponentEnabled<RequestRendering>(entity, true);
  ```

- **Naming conventions** - prefix request components with `Request` (RequestSceneLoaded, RequestNetworkConnection)

- **State component naming** - suffix state components with `State`, `Active`, `InProgress`, or `Loaded`

- **Documentation** - clearly document what happens when request is added vs removed

- **Testing patterns:**
  ```csharp
  // Test request addition
  entityManager.AddComponent<RequestSceneLoaded>(entity);
  world.Update();
  Assert.IsTrue(entityManager.HasComponent<SceneLoaded>(entity));

  // Test request removal
  entityManager.RemoveComponent<RequestSceneLoaded>(entity);
  world.Update();
  Assert.IsFalse(entityManager.HasComponent<SceneLoaded>(entity));
  ```

- **Best practices:**
  - Use request pattern for reversible operations
  - Track state with separate component
  - Handle cleanup properly when request removed
  - Document request behavior clearly
  - Consider IEnableableComponent for frequent toggling
  - Use ECB for structural changes
  - Test both add and remove paths

- **Difference from one-time requests:**
  - **Request components** - persist on entity, reversible, repeatable
  - **Request entities** - destroyed after processing, one-time, fire-and-forget

- **When to use request entities instead:**
  - One-time operations (spawn entity, send message)
  - Fire-and-forget commands
  - Don't need to reverse operation
  - Operation completes immediately
