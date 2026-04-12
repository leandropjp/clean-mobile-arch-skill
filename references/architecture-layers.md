# Clean Architecture Layers for Mobile

## The Four Clean Architecture Layers (inner to outer)

### 1. Domain Model & Logic (Center)
- Enterprise-wide business rules
- Domain models with core business functions
- Most stable -- changes only when business changes
- **Zero external dependencies**
- Reusable across applications

### 2. Application Layer (Use Cases)
- Application-specific logic and rules
- Orchestrate data flow to/from entities
- Direct entities to use business rules
- If a Use Case just wires ViewModel to Gateway with no logic = unnecessary (CA might be overkill)

### 3. Interface Adapters (Presentation & Data)
- **Presentation**: Data transformations for UI/UX, State-model, Intent-model
- **Data**: Data aggregation/manipulation, local DB access, remote server access
- HARD wall between Presentation and Data (neither depends on the other)
- Both depend only on Application layer
- Must be platform-agnostic (no Activities, Fragments, ViewControllers)

### 4. Frameworks & Drivers (Outermost)
- All mobile framework code (Activities, Fragments, ViewControllers, Widgets)
- Third-party libraries isolated here
- **Must be "dumb"** -- no logic, no conditionals based on data
- Hard to test (requires emulator)

## The Dependency Rule
- Source code dependencies can only point **inwards**
- Inner circles must NOT be aware of outer layers
- Data crossing boundaries must be in the form most convenient to the **inner** circle
- **Never** propagate framework/third-party data structures to inner layers

## Seven Types of Logic in Mobile Apps

| Type | Description | Where It Belongs |
|---|---|---|
| 1. User input validations | Validate input meets preconditions | Presentation |
| 2. User action business logic | Logic after user action | Domain (prefer backend as source of truth) |
| 3. Data transformation (outbound) | Format data for server | Data |
| 4. Data validation (inbound) | Validate/transform received data | Data |
| 5. Data storage | DB operations | Data |
| 6. Business/Application logic | Process, transform, aggregate data | Domain / Application |
| 7. Presentation logic | UI transformations (color, size, format) | Presentation |

## Model Transformation Chain

### Enterprise (Complex)
```
JSON -> ModelRaw -> ModelDB -> ModelPlain -> Model (domain) -> ModelPresentation
```

### Startup (Pragmatic)
```
JSON -> ModelRaw -> ModelPlain -> Model (merged business+presentation)
```

## Boundary Strategies

| App Complexity | Recommendation |
|---|---|
| Medium | Feature packages (soft) + layer packages (soft) = single module |
| Complex / Enterprise | Feature modules (hard) + layer packages (soft) |
| Very Large (50+ devs) | Microservices (hard feature + hard layer) |

## Mobile Architecture Layer Stack (SCP-derived)

```
View -> ViewModel -> [Presentation Mapper] -> [Aggregator/DataSource] -> [Domain Mapper] -> Repository -> [Service + DAO] -> [HTTP Client + JSON Parser + DB Client]
```

- From ViewModel leftward: serves users. **1 ViewModel per screen**.
- From Repository rightward: serves data. **1 Repository per entity**.

## Rich vs. Poor Models

- **Driving logic** (user-initiated): Add inside models. Minimal with decent backend.
- **Driven logic** (data transformation): Dedicated mapper classes for complex; in-model for simple apps.
- **Use Cases always needed** for orchestration. Models must have zero external dependencies.

## Workflow (Recommended Order)
1. Understand specifics: DB needs, API type, logic types, aggregation requirements
2. Define data models and transformations
3. Draw boundaries and layers
4. Select third-party libraries **last** (last responsible moment)
5. Continuously inspect and adapt the MVA
