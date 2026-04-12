# S.O.L.I.D Principles Checklist

## S -- Single Concern Principle (SCP) / Single Responsibility Principle (SRP)

**SRP (business)**: "A module should be responsible to one, and only one, actor."
**SCP (technical)**: "Each software unit should have one, and only one, technical, fine-grained concern."

### Class Categories (SCP)
Every class is exactly ONE of:
1. **Executor** -- applies logic or performs a task
2. **Delegator/Orchestrator** -- orchestrates flow and delegates to executors
3. **Data Container** -- holds data (models, POJOs, data classes)

If a class belongs to more than one category, it violates SCP.

### Red Flags
- [ ] Shared function serving different actors (accidental duplication)
- [ ] Merge conflicts from multiple developers in same file
- [ ] ViewModel serving multiple screens (1 ViewModel = 1 Screen)
- [ ] Repository mixing unrelated entities (1 Repository = 1 Entity)
- [ ] Middleman layer with zero responsibilities (redundant layer)
- [ ] Service handling both backend + database operations

---

## O -- Open-Closed Principle (OCP)

**"Open for extension, closed for modification."**

### When to Apply
Refactor if/else or switch blocks when:
1. Number of types grows
2. Logic per type becomes complex
3. Additional factors are introduced

### Technique
Use **Strategy Design Pattern**: interface + concrete implementations per type + factory.

### Red Flags
- [ ] Growing if/else or switch/when blocks based on types
- [ ] Modifying existing functions to add new behavior
- [ ] Multiple developers touching same conditional logic

---

## L -- Liskov Substitution Principle (LSP)

**"Any subtype should be substitutable for its parent without breaking behavior."**

### Design by Contract Rules
- **Input**: Subclass CANNOT be MORE restrictive. Can be more flexible.
- **Output**: Subclass CANNOT be MORE flexible. Can be more restrictive.

### Key Rules
- Prefer pure interfaces over concrete class inheritance
- Never add client-specific conditionals in shared implementations
- Unit test overridden functions in subclasses against all expected inputs
- Extra caution with third-party library subclasses

### Red Flags
- [ ] Subclass with stricter input validation than parent
- [ ] Subclass returning values outside parent's output range
- [ ] Client-specific conditionals in shared API/implementation
- [ ] Overriding third-party methods without unit tests

---

## I -- Interface Segregation Principle (ISP)

**"A client should not depend on stuff it doesn't need."**

### Multi-Level Application
- **Functions**: Should not accept input they don't use
- **Classes**: Should not depend on classes/functions they don't use
- **Modules**: Should not depend on interfaces they don't need
- **Systems**: Backend API should only expose what clients need

### Code Smells
- [ ] Bulky interfaces with too many functions
- [ ] Unused dependencies (passing null to functions)
- [ ] Functions throwing exceptions that some callers never trigger
- [ ] Empty overridden functions from inheritance

---

## D -- Dependency Inversion Principle (DIP)

**"High-level modules should not import from low-level modules. Both should depend on abstractions."**

### Code Level Hierarchy (highest to lowest)
1. **Domain** -- business rules (highest, most protected)
2. **Application** -- use case orchestration
3. **Presentation / Data** -- adapters
4. **Framework** -- platform code (lowest, most volatile)

### The DIP Technique
1. Create interface designed around Use Case's needs (not around data layer)
2. High-level class depends on interface
3. Low-level class implements interface
4. Dependency arrows point inward/upward

### Scaling Strategies
| Complexity | Strategy |
|---|---|
| Simple apps | Repository pattern (pragmatic, less protection) |
| Complex apps | Full DIP with DataSource interfaces + DataGateway implementations |

### Red Flags
- [ ] Domain layer importing from data layer concretions
- [ ] Interface that mirrors all Service methods (not tailored to Use Case)
- [ ] Service implementing too many DataSource interfaces
