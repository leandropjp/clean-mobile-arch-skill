# Testing & TDD Reference

## Core Rule
"A testable app is, by rule, a well-architected app." Designing for testability forces proper separation of concerns, small focused units, and DI.

## Types of Tests

| Type | Scope | Mocking | Speed | Precision |
|---|---|---|---|---|
| Unit (white-box) | Single class, deps mocked | All dependencies | Fast | Exact defect location |
| Unit (black-box) | Multiple layers, boundaries mocked | Only system boundaries | Fast | Less precise location |
| Integration | Module groups | External systems/DB | Slow (emulator) | Moderate |
| E2E | Full user workflow | Nothing | Very slow | Low (which system broke?) |

## AAA Pattern
- **Arrange**: Set up data + dependencies. Extract to helpers if complex.
- **Act**: Execute function under test. Aim for single function call.
- **Assert**: Verify expected outcome. One assertion per test.
- Strive for one-liner per A.

## Testing Pyramid
~70% Unit | ~20% Integration | ~10% E2E

## Mobile Testing Strategies

### Enterprise (Complex Apps, Large Teams)
- White-box unit tests (mocks) for all framework-independent layers
- Each layer tested in isolation
- Integration + E2E tests for regression
- Tests reside within each module (team ownership)
- High mocking effort but team has capacity

### Startup (Pragmatic, Small Teams)
- Black-box unit tests (ViewModel to Service, mock only external systems)
- Flexible for refactoring (no mock code to update)
- Skip E2E and Integration (optional)
- Less safe but much less effort

## TDD Cycle
1. **Red** -- Write failing test
2. **Green** -- Minimum code to pass
3. **Refactor** -- Clean up, tests stay green

## TDD Schools

| Aspect | Chicago (Classicist) | London (Mockist) |
|---|---|---|
| Mocks | None (except boundaries) | Extensive |
| Tests | Behavior (black-box) | Interactions (white-box) |
| Design | Emerges from TDD cycle | Upfront architectural thinking |
| Refactoring | Easy (no mocks to update) | Rigid (mocks must change) |
| Defect location | Less precise | Precise |
| Advocates | Kent Beck, Uncle Bob | Sandro Mancuso, Steve Freeman |

## Anti-Patterns
- `if(test) -> logic` in production code (FORBIDDEN)
- Trying TDD first in a production project under deadline pressure
- Skipping tests because "manual testing is easier" (it isn't at scale)
- Not testing error/edge cases

## TDD Learning Path (Recommended)
1. Master platform testing tools (unit tests first)
2. Practice small TDD cycles in pet projects
3. Add Acceptance tests
4. Only then use in production work

## Reactive Programming & Immutability
- In RP, immutable objects are **mandatory** -- mutable objects in async contexts cause race conditions and unpredicted state
- Use `map` operator to create new immutable instances
- RP decouples producers from consumers (lower coupling)
- Avoid chaining multiple operators -- powerful but hurts readability

## Inversion of Control Summary
- IoC = Dependency Inversion + Dependency Injection
- **DI should be used widely** (modern libraries make it trivial)
- **DIP should be used only when needed** (to protect high-level modules, comes with cost)
- "Dependency Injection is the enabler of Dependency Inversion"
- Circular dependency is one of the worst anti-patterns in software engineering
