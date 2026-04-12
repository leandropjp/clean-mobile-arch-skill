# MV* Pattern Decision Tree

## Pattern Comparison

| Pattern | Status | Reactive? | View Reference? | State Management | Best For |
|---|---|---|---|---|---|
| MVC (original) | Obsolete | No | Coupled | Manual | Nothing (historical) |
| MVC (mobile) | Outdated | No | Coupled | Manual | Low-complexity CRUD only |
| MVP | Outshined | No | Via interface | Manual | Teams unwilling to learn RP |
| MVVM | Current go-to | Yes | No (streams) | Multiple streams | Small to medium complexity |
| MVI | Author's pick | Yes | No (streams) | Single state stream | Complex applications |

## Decision Flow

```
START
  |
  v
Is your team willing to learn Reactive Programming?
  |
  NO --> Use MVP (enhancement of MVC, decent but dated)
  |
  YES
  |
  v
How complex is your app?
  |
  LOW --> MVVM (industry standard, good MVA starting point)
  |
  MEDIUM --> MVVM (works well, consider MVI if many data streams per screen)
  |
  HIGH --> MVI-like + Clean Architecture (author's recommendation)
         - Take from MVI: RP, State-model, Intent-model
         - Skip unidirectional reactive streams (use function calls for driving->driven)
         - Combine with CA layers and Dependency Rule
```

## MVC -- Why NOT
1. Better alternatives exist for every scenario
2. Considered outdated -- hurts hiring/retention
3. Controller tightly coupled to views (untestable)
4. No distinction between presentation and business logic

## MVP Concerns
- Presenter coupled to View (solvable with DI via view interface)
- Shared Presenters across screens = accidental duplication
- Superseded by MVVM in every way

## MVVM Concerns
- Multiple stream composition: View must handle all combinations of loader/error/data streams
- Steep Reactive Programming learning curve
- Model layer often too thick (needs decomposition)

## MVI Strengths
- **Single State-model**: ALL screen info in one class (loading, data, empty, failed)
- **Single Intent-model**: ALL user actions in one enum/sealed class
- **Reducer function**: Pure function assembling state -- easy to test
- **Single stream** to View simplifies rendering enormously

## MVI Concerns
- Steepest learning curve
- Does not define layering (must combine with CA)
- Unidirectional streams add boilerplate -- author recommends skipping for driving->driven direction

## MVs vs. Clean Architecture
- **NOT mutually exclusive**: Can use MVVM + CA or MVI + CA together
- **CA** = high-level architecture (layers, dependency rule, principles)
- **MVs** = low-level patterns (how to structure View/Model communication)
- MVs are a good kickstarter for MVA, upscalable to Clean later

## Context-Dependent Architecture

### Enterprise (Complex, Large Team, Long-Lived)
- MVI-like + Clean Architecture
- Hard boundaries (feature modules)
- Full DIP with DataSource interfaces + DataGateway
- White-box unit tests per layer

### Startup (Pragmatic, Small Team, Fast-Moving)
- MVVM or MVI-like with soft boundaries
- Repository pattern (pragmatic DIP)
- Merged Application + Presentation layers
- Black-box unit tests (test multiple layers together)

## Third-Party Libraries -- Author's Recommendations
- **Android DI**: Koin (works with module segregation)
- **iOS DI**: Resolver (simple, fast)
- **Rule**: Select libraries LAST. Architecture should banish them to the edge.
