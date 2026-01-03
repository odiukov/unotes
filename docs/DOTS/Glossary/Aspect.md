## Aspect

**Component grouping wrapper** that bundles multiple related [[Component|components]] together with their behavior logic, implemented via [[IAspect (component grouping)|IAspect]] interface.

Aspects provide **reusable behavior** that can be shared across systems and jobs, and help **circumvent query limitations** when you need more components than the query parameter limit.

**Key characteristics:**
- **Component composition** - groups RefRO/RefRW fields for multiple components
- **Behavior encapsulation** - methods that operate on the grouped components
- **Automatic query generation** - when you query for an aspect, Unity generates the underlying entity query
- **Type safety** - ensures all required components are present at compile time

**Example:** `ProjectileAspect` groups `Transform`, `Speed`, `Target` components with a `Move()` method

**See also:** [[IAspect (component grouping)]], [[RefRO and RefRW]]
