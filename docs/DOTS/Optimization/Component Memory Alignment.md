# Component Memory Alignment

Practical guide to memory alignment and struct layout optimization in Unity ECS components for minimizing memory footprint and maximizing [[Cache-friendly]] performance.

---

## Understanding Memory Alignment

### What is Memory Alignment?

Memory alignment is the practice of arranging data in memory so the CPU can read values efficiently as complete "words" without multiple memory accesses.

**Key concept**: CPUs cannot read misaligned data directly. The compiler adds padding (empty bytes) to ensure each field starts at an address divisible by its size.

---

## The Problem: Wasted Memory

### Example 1: Simple Struct

```csharp
// Naive assumption: 5 bytes total
struct BadLayout : IComponentData
{
    public int Health;     // 4 bytes
    public bool IsDead;    // 1 byte
}
```

**Expected size**: 5 bytes (4 + 1)
**Actual size**: 8 bytes

**Memory footprint**:
```
[Health|Health|Health|Health]
[IsDead|-------|-------|-------]
```

The compiler adds 3 bytes of padding after `IsDead` to align the struct to 4-byte boundaries.

---

### Example 2: Complex Struct

```csharp
// ❌ Bad: Poorly ordered fields
struct PoorlyOrganized : IComponentData
{
    public bool IsDead;      // a - 1 byte
    public int Health;       // b - 4 bytes
    public bool NeedsHelp;   // c - 1 byte
    public int XP;           // d - 4 bytes
}
```

**Memory footprint** (16 bytes):
```
[a|-|-|-]
[b|b|b|b]
[c|-|-|-]
[d|d|d|d]
```

- 10 bytes of actual data
- 6 bytes of padding (37.5% waste!)

---

### Example 3: Optimized Layout

```csharp
// ✅ Good: Fields ordered by size
struct WellOrganized : IComponentData
{
    public int Health;       // a - 4 bytes
    public int XP;           // b - 4 bytes
    public bool IsDead;      // c - 1 byte
    public bool NeedsHelp;   // d - 1 byte
}
```

**Memory footprint** (12 bytes):
```
[a|a|a|a]
[b|b|b|b]
[c|d|-|-]
```

- 10 bytes of actual data
- 2 bytes of padding
- **25% memory reduction** (from 16 to 12 bytes)

---

## Nested Structs: Amplified Problem

### Bad Nested Structure

```csharp
struct NestedBad
{
    public bool a;        // 1 byte
    public long b;        // 8 bytes
    public bool c;        // 1 byte
}

struct ParentBad : IComponentData
{
    public NestedBad nested;
    public int d;
    public bool e;
}

// Inline expansion:
struct ParentBad_Expanded
{
    public bool a;
    public long b;
    public bool c;
    public int d;
    public bool e;
}
```

**Memory footprint** (32 bytes):
```
[a|-|-|-|-|-|-|-]
[b|b|b|b|b|b|b|b]
[c|-|-|-|-|-|-|-]
[d|d|d|d|e|-|-|-]
```

---

### Optimized Nested Structure

```csharp
struct NestedGood
{
    public long b;        // 8 bytes
    public int d;         // 4 bytes
    public bool a;        // 1 byte
    public bool c;        // 1 byte
    public bool e;        // 1 byte
}
```

**Memory footprint** (16 bytes):
```
[b|b|b|b|b|b|b|b]
[d|d|d|d|a|c|e|-]
```

**Result**: **50% memory reduction** (from 32 to 16 bytes)

---

## Enum Optimization

### Problem: Default Enum Size

```csharp
// ❌ Bad: Uses int (4 bytes) by default
enum State
{
    Idle,
    Walking,
    Running
}

struct Component : IComponentData
{
    public State CurrentState;  // 4 bytes for 3 values!
}
```

Default enums use `int` (4 bytes), supporting 4+ billion values—far more than needed.

---

### Solution: Byte-Sized Enums

```csharp
// ✅ Good: Uses byte (1 byte) for up to 256 values
enum State : byte
{
    Idle,
    Walking,
    Running
}

struct Component : IComponentData
{
    public State CurrentState;  // 1 byte (75% reduction)
}
```

**Impact**: With 10,000 entities, this saves 30KB immediately.

---

## Measuring Struct Size

Use Unity's `UnsafeUtility` to inspect component sizes:

```csharp
using Unity.Collections.LowLevel.Unsafe;

// Check alignment (how struct should be aligned in memory)
int alignment = UnsafeUtility.AlignOf<MyComponent>();

// Check total size (including padding)
int size = UnsafeUtility.SizeOf<MyComponent>();

Debug.Log($"MyComponent: {size} bytes, {alignment}-byte alignment");
```

### Example Output

```csharp
struct Example : IComponentData
{
    public long a;    // 8 bytes
    public int b;     // 4 bytes
    public bool c;    // 1 byte
}

// Output: "Example: 16 bytes, 8-byte alignment"
```

---

## Optimization Best Practices

### 1. Sort Fields by Size (Largest First)

```csharp
// ✅ Optimal ordering
struct OptimalComponent : IComponentData
{
    // 8-byte types first
    public double Timestamp;
    public long EntityID;

    // 4-byte types
    public float Speed;
    public int Health;

    // 2-byte types
    public ushort Level;

    // 1-byte types last
    public byte TeamID;
    public bool IsActive;
}
```

**Field size reference**:
- 8 bytes: `long`, `ulong`, `double`
- 4 bytes: `int`, `uint`, `float`, `enum` (default)
- 2 bytes: `short`, `ushort`
- 1 byte: `byte`, `sbyte`, `bool`, `enum : byte`

---

### 2. Avoid Complex Nested Structs

```csharp
// ❌ Bad: Nested struct makes alignment difficult to predict
struct PlayerComponent : IComponentData
{
    public HealthData Health;
    public MovementData Movement;
    public InventoryData Inventory;
}

// ✅ Better: Flatten or separate into multiple components
struct PlayerHealthComponent : IComponentData
{
    public float MaxHealth;
    public float CurrentHealth;
}

struct PlayerMovementComponent : IComponentData
{
    public float Speed;
    public float Acceleration;
}
```

**Benefits**:
- Predictable memory layout
- Better [[Archetype]] granularity
- Systems query only needed data

---

### 3. Minimize Field Count

```csharp
// ❌ Bad: Kitchen sink component
struct PlayerComponent : IComponentData
{
    public float Health;
    public float MaxHealth;
    public float Speed;
    public float JumpForce;
    public float AttackDamage;
    public int Level;
    public int XP;
    public bool IsGrounded;
    public bool IsAttacking;
    // ... 20 more fields
}

// ✅ Good: Focused components
struct HealthComponent : IComponentData
{
    public float Max;
    public float Current;
}

struct MovementComponent : IComponentData
{
    public float Speed;
    public bool IsGrounded;
}
```

**See also**: [[ECS Design Decisions]] for component composition strategies

---

### 4. Use Byte-Sized Enums

```csharp
// ✅ Always specify enum size when < 256 values
public enum AIState : byte
{
    Idle,
    Patrol,
    Chase,
    Attack
}

public enum Team : byte
{
    Red,
    Blue,
    Green,
    Neutral
}

struct AIComponent : IComponentData
{
    public AIState CurrentState;   // 1 byte
    public Team Team;               // 1 byte
}
```

---

### 5. Group Boolean Fields

```csharp
// If you have many bools, consider bit packing
struct FlagsComponent : IComponentData
{
    public byte Flags;  // 1 byte can hold 8 booleans

    public bool IsActive
    {
        get => (Flags & 0b00000001) != 0;
        set => Flags = value ? (byte)(Flags | 0b00000001) : (byte)(Flags & ~0b00000001);
    }

    public bool IsVisible
    {
        get => (Flags & 0b00000010) != 0;
        set => Flags = value ? (byte)(Flags | 0b00000010) : (byte)(Flags & ~0b00000010);
    }

    // ... up to 8 flags
}
```

**Trade-off**: CPU instructions vs memory—use sparingly for frequently accessed flags.

---

## Real-World Impact

### Scaling Analysis

```csharp
// Example: Character component
struct CharacterBad : IComponentData
{
    public bool IsDead;        // 1 byte
    public int Health;         // 4 bytes
    public bool NeedsHelp;     // 1 byte
    public int XP;             // 4 bytes
    // Actual: 16 bytes
}

struct CharacterGood : IComponentData
{
    public int Health;         // 4 bytes
    public int XP;             // 4 bytes
    public bool IsDead;        // 1 byte
    public bool NeedsHelp;     // 1 byte
    // Actual: 12 bytes
}
```

**Memory savings at scale**:

| Entities | Bad Layout | Good Layout | Savings |
|----------|-----------|-------------|---------|
| 1,000 | 16 KB | 12 KB | 4 KB (25%) |
| 10,000 | 160 KB | 120 KB | 40 KB (25%) |
| 100,000 | 1.6 MB | 1.2 MB | 400 KB (25%) |
| 1,000,000 | 16 MB | 12 MB | 4 MB (25%) |

---

## Cache Line Considerations

Modern CPUs load data in **64-byte cache lines**. Smaller components mean:

1. **More entities per cache line**:
   - 12-byte component: ~5 entities per cache line
   - 16-byte component: ~4 entities per cache line

2. **Fewer [[Cache miss|cache misses]]** during iteration

3. **Better [[Burst]] performance** with SIMD processing

**See also**: [[Cache-friendly]], [[Chunk]]

---

## Component Size Analyzer Tool

Maxim Zaks created a component size analyzer tool:

**Gist**: https://gist.github.com/mzaks/ec261ac853621af8503b73391ebd18f1

This tool analyzes your project's components and suggests optimizations.

---

## Common Type Sizes Reference

### Primitive Types

| Type | Size | Alignment |
|------|------|-----------|
| `bool` | 1 byte | 1 byte |
| `byte`, `sbyte` | 1 byte | 1 byte |
| `short`, `ushort` | 2 bytes | 2 bytes |
| `int`, `uint`, `float` | 4 bytes | 4 bytes |
| `long`, `ulong`, `double` | 8 bytes | 8 bytes |
| `enum` (default) | 4 bytes | 4 bytes |
| `enum : byte` | 1 byte | 1 byte |

### Unity Types

| Type | Size | Notes |
|------|------|-------|
| `float2` | 8 bytes | 2 floats |
| `float3` | 12 bytes | 3 floats, padded to 16 |
| `float4` | 16 bytes | 4 floats |
| `quaternion` | 16 bytes | 4 floats |
| `Entity` | 8 bytes | 2 ints |
| `FixedString32Bytes` | 32 bytes | Fixed-size string |
| `FixedString64Bytes` | 64 bytes | Fixed-size string |

---

## Anti-Patterns to Avoid

### 1. Alternating Small/Large Fields

```csharp
// ❌ Creates maximum padding
struct Worst : IComponentData
{
    public bool a;     // 1 byte + 3 padding
    public int b;      // 4 bytes
    public bool c;     // 1 byte + 3 padding
    public int d;      // 4 bytes
    // Total: 16 bytes (50% waste!)
}
```

### 2. Unnecessary Nesting

```csharp
// ❌ Nested structs inherit alignment requirements
struct Inner
{
    public long Value;  // Forces 8-byte alignment
}

struct Outer : IComponentData
{
    public bool Flag;   // 1 byte + 7 padding due to Inner
    public Inner Data;  // 8 bytes
    // Total: 16 bytes (1 byte + 7 padding + 8 bytes)
}
```

### 3. Using Default Enums

```csharp
// ❌ Wastes 3 bytes per enum
enum Status { Active, Inactive }  // 4 bytes

// ✅ Use byte-sized enums
enum Status : byte { Active, Inactive }  // 1 byte
```

---

## Debugging Memory Layout

### Manual Calculation

1. **Sort fields by size** (largest first)
2. **Calculate cumulative offset**:
   - Each field starts at next multiple of its size
3. **Round struct size** to largest field's alignment

### Example Calculation

```csharp
struct Example : IComponentData
{
    public int a;      // Offset 0, size 4
    public bool b;     // Offset 4, size 1
    public bool c;     // Offset 5, size 1
    // Next field would start at offset 6
}

// Size calculation:
// - Last field ends at offset 6 (5 + 1)
// - Largest field alignment: 4 bytes (int)
// - Round up to 8 bytes (next multiple of 4 >= 6)
// Total size: 8 bytes
```

---

## Performance Testing

```csharp
using Unity.Collections.LowLevel.Unsafe;
using UnityEngine;

public static class ComponentProfiler
{
    public static void LogComponentInfo<T>() where T : struct, IComponentData
    {
        int size = UnsafeUtility.SizeOf<T>();
        int alignment = UnsafeUtility.AlignOf<T>();

        Debug.Log($"{typeof(T).Name}:");
        Debug.Log($"  Size: {size} bytes");
        Debug.Log($"  Alignment: {alignment} bytes");
        Debug.Log($"  Entities per 64B cache line: {64 / size}");
    }
}

// Usage in editor script or test
ComponentProfiler.LogComponentInfo<MyComponent>();
```

---

## Optimization Checklist

Before finalizing component design:

- [ ] Fields sorted by size (8-byte → 4-byte → 2-byte → 1-byte)
- [ ] Enums use `:byte` for < 256 values
- [ ] Nested structs minimized or flattened
- [ ] Component size checked with `UnsafeUtility.SizeOf<T>()`
- [ ] Total field count kept minimal (consider splitting)
- [ ] Boolean flags grouped or bit-packed if many
- [ ] No alternating large/small field patterns
- [ ] Alignment verified with `UnsafeUtility.AlignOf<T>()`

---

## Related Topics

- [[Cache-friendly]] - How memory layout affects cache performance
- [[Chunk]] - How components are stored in 16KB chunks
- [[SoA layout]] - Structure of Arrays layout in ECS
- [[ECS Design Decisions]] - When to split vs combine components
- [[IComponentData]] - Basic component type

---

## Further Reading

- **Original article**: [Memory alignment and Component design in Unity ECS](https://mzaks.medium.com/memory-alignment-and-component-design-in-unity-ecs-84f05f21ccc8) by Maxim Zaks (July 2019)
- **Component Size Analyzer**: https://gist.github.com/mzaks/ec261ac853621af8503b73391ebd18f1
- **Follow-up article**: [Slicing and Dicing Components for Unity ECS](https://mzaks.medium.com/slicing-and-dicing-components-for-unity-ecs-221f8a181850)
