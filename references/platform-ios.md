# Clean Mobile Architecture -- iOS Platform Reference (Swift / SwiftUI)

**Source**: Mapped from "Clean Mobile Architecture" by Petros Efthymiou (2022) to iOS conventions.

---

## 1. How Clean Architecture Maps to iOS

### The 4 Layers on iOS

```
+---------------------------------------------------------+
|  Framework & Drivers (outermost)                        |
|  SwiftUI Views, UIViewControllers, UIKit widgets,       |
|  third-party SDKs (Alamofire, Realm, Firebase)          |
+---------------------------------------------------------+
|  Interface Adapters                                      |
|  Presentation: ViewModels (@Observable), Mappers         |
|  Data: Repositories, DataSources, CodableModels          |
+---------------------------------------------------------+
|  Application (Use Cases)                                 |
|  Interactors / Use Case classes                          |
|  Orchestrate domain logic, no framework imports          |
+---------------------------------------------------------+
|  Domain (center)                                         |
|  Plain Swift structs/enums, business rules               |
|  Zero dependencies -- not even Foundation if avoidable   |
+---------------------------------------------------------+
```

### Where iOS Types Live in the Stack

| iOS Type | Layer | Notes |
|---|---|---|
| `SwiftUI.View` | Framework | Must be "dumb" -- rendering only |
| `UIViewController` | Framework | Same rule: delegate everything to ViewModel |
| `@Observable` / `@ObservableObject` class | Presentation (Interface Adapters) | The ViewModel. 1 per screen. |
| Use Case / Interactor | Application | Protocol + concrete class |
| Repository (protocol) | Application boundary | Defined by Use Case needs (DIP) |
| Repository (impl) | Data (Interface Adapters) | Implements protocol, calls Services |
| Service / DAO | Data (Interface Adapters) | HTTP client wrappers, DB access |
| Domain Model (struct) | Domain | Pure value types, zero imports |
| `Codable` model | Data (Interface Adapters) | JSON parsing only, never leaks inward |

### The "Framework Layer Must Be Dumb" Rule for SwiftUI

**What goes in the View:**
- Layout declarations (`VStack`, `HStack`, `List`)
- Binding to ViewModel properties (`viewModel.title`, `viewModel.isLoading`)
- Forwarding user actions to ViewModel (`viewModel.onTap()`, `viewModel.send(.loadData)`)
- Navigation triggers

**What does NOT go in the View:**
- `if/else` based on business data (move to ViewModel, expose a computed presentation model)
- Date formatting, number formatting (Presentation Mapper or ViewModel)
- Network calls, database queries
- Any `import` of data-layer types

```swift
// GOOD -- dumb View
struct OrderListView: View {
    @State private var viewModel = OrderListViewModel()

    var body: some View {
        List(viewModel.orders) { order in
            OrderRow(presentation: order)
        }
        .overlay {
            if viewModel.isLoading { ProgressView() }
        }
        .task { await viewModel.load() }
    }
}

// BAD -- logic in View
struct OrderListView: View {
    @State private var orders: [Order] = []

    var body: some View {
        List(orders) { order in
            // Business logic leaked into framework layer
            Text(order.total > 100 ? "Premium: \(order.total)" : "\(order.total)")
                .foregroundStyle(order.isOverdue ? .red : .primary)
        }
        .task {
            // Data-layer concern leaked into framework layer
            let url = URL(string: "https://api.example.com/orders")!
            orders = try! await URLSession.shared.decode([Order].self, from: url)
        }
    }
}
```

### Data Flow: SwiftUI Property Wrappers and Boundary Crossing

| Wrapper | Book Concept | Direction |
|---|---|---|
| `@State` | Local UI state (framework layer only) | View-internal |
| `@Binding` | Parent passes control to child View | View <-> View (framework) |
| `@Observable` (iOS 17+) | ViewModel observed by View | Presentation -> Framework |
| `@ObservedObject` / `@StateObject` | Legacy ViewModel observation | Presentation -> Framework |
| `@Environment` | Framework-level DI (bounded to SwiftUI) | Outer -> Framework |
| `@EnvironmentObject` | Shared state injection in view tree | Outer -> Framework |

**Rule**: Data crossing a boundary must be in the form most convenient to the **inner** circle. The ViewModel exposes presentation-ready data; the View never transforms it.

---

## 2. MV* Patterns on iOS

### MVVM with Combine / async-await

MVVM is the book's recommended starting point (MVA for low-to-medium complexity). On iOS it maps naturally:

```swift
// ViewModel (Presentation layer -- Interface Adapter)
@Observable
final class ProfileViewModel {
    private(set) var name: String = ""
    private(set) var isLoading: Bool = false
    private(set) var error: String?

    private let getProfileUseCase: GetProfileUseCaseProtocol

    init(getProfileUseCase: GetProfileUseCaseProtocol) {
        self.getProfileUseCase = getProfileUseCase
    }

    func load() async {
        isLoading = true
        defer { isLoading = false }
        do {
            let profile = try await getProfileUseCase.execute()
            name = profile.displayName  // presentation-ready
        } catch {
            self.error = error.localizedDescription
        }
    }
}

// Use Case (Application layer)
protocol GetProfileUseCaseProtocol {
    func execute() async throws -> Profile
}

final class GetProfileUseCase: GetProfileUseCaseProtocol {
    private let repository: ProfileRepositoryProtocol
    func execute() async throws -> Profile {
        try await repository.fetchProfile()
    }
}
```

### MVI-Like on iOS: State-Model + Intent-Model

The book's recommendation for complex apps. On iOS, model state as an enum and intents as an enum:

```swift
// State-model: ALL screen state in one type
enum ProfileState {
    case idle
    case loading
    case loaded(ProfilePresentation)
    case failed(String)
}

// Intent-model: ALL user actions in one enum
enum ProfileIntent {
    case load
    case refresh
    case editName(String)
    case save
}

@Observable
final class ProfileViewModel {
    private(set) var state: ProfileState = .idle  // single stream

    private let getProfileUseCase: GetProfileUseCaseProtocol
    private let updateProfileUseCase: UpdateProfileUseCaseProtocol

    func send(_ intent: ProfileIntent) {
        switch intent {
        case .load, .refresh:
            Task { await load() }
        case .editName(let name):
            // Reduce into new state
            if case .loaded(var presentation) = state {
                presentation.name = name
                state = .loaded(presentation)
            }
        case .save:
            Task { await save() }
        }
    }

    private func load() async {
        state = .loading
        do {
            let profile = try await getProfileUseCase.execute()
            state = .loaded(ProfilePresentation(from: profile))
        } catch {
            state = .failed(error.localizedDescription)
        }
    }
}

// View consumes single state
struct ProfileView: View {
    @State private var viewModel: ProfileViewModel

    var body: some View {
        switch viewModel.state {
        case .idle:
            Color.clear.task { viewModel.send(.load) }
        case .loading:
            ProgressView()
        case .loaded(let profile):
            ProfileContent(profile: profile, onSave: { viewModel.send(.save) })
        case .failed(let message):
            ErrorView(message: message, onRetry: { viewModel.send(.refresh) })
        }
    }
}
```

### The Composable Architecture (TCA) as MVI on iOS

TCA by Point-Free is the most mature MVI implementation on iOS:

| TCA Concept | Book Concept |
|---|---|
| `@Reducer` | ViewModel + Reducer logic |
| `State` struct | State-model |
| `Action` enum | Intent-model |
| `reduce(into:action:)` | Pure reducer function |
| `Effect` | Side effects (Use Case calls) |
| `Store` | Single source of truth |
| `@Dependency` | DI container |

TCA enforces unidirectional data flow. The book notes this adds boilerplate but is justified for complex apps with many data streams per screen.

### Why MVC Is the Book's Anti-Pattern on iOS

Apple's original MVC led to "Massive View Controller" -- the canonical iOS anti-pattern:
- `UIViewController` handles View + Presentation + Business logic
- Untestable (coupled to UIKit lifecycle)
- No separation of concerns
- Violates SCP: the controller is simultaneously an Executor, Delegator, and partial Data Container

---

## 3. SOLID on iOS

### SRP/SCP: 1 ViewModel per Screen

```
ProfileScreen/
  ProfileView.swift          -- SwiftUI View (framework, dumb)
  ProfileViewModel.swift     -- @Observable class (presentation)
  ProfilePresentation.swift  -- Presentation model (data container)
```

**Rule**: 1 ViewModel = 1 Screen. If a screen has sub-sections with independent data needs, compose child ViewModels rather than bloating one ViewModel.

**How `@Observable` changes things** (iOS 17+):
- `@Observable` uses macro-generated observation, so Views automatically track only the properties they read.
- This makes fine-grained updates free, but does NOT change the 1 ViewModel per screen rule.
- Avoid scattering `@Observable` models throughout -- the ViewModel is still the single entry point for the View.

### OCP: Protocol-Oriented Programming

Swift protocols are the natural mechanism for OCP. Use the Strategy pattern via protocols:

```swift
// Closed for modification
protocol PaymentProcessor {
    func process(amount: Decimal) async throws -> PaymentResult
}

// Open for extension -- add new types without touching existing code
struct StripeProcessor: PaymentProcessor {
    func process(amount: Decimal) async throws -> PaymentResult { /* ... */ }
}

struct ApplePayProcessor: PaymentProcessor {
    func process(amount: Decimal) async throws -> PaymentResult { /* ... */ }
}

// Factory resolves the right implementation
struct PaymentProcessorFactory {
    static func make(for method: PaymentMethod) -> PaymentProcessor {
        switch method {
        case .stripe: return StripeProcessor()
        case .applePay: return ApplePayProcessor()
        }
    }
}
```

Protocol extensions provide default implementations without modifying existing types.

### LSP: Prefer Protocols Over Class Inheritance

Swift's strength aligns perfectly with the book's preference:

```swift
// GOOD -- protocol-based (LSP-safe, substitutable)
protocol OrderRepository {
    func fetchOrders() async throws -> [Order]
}

// BAD -- class inheritance (LSP risk: subclass may violate parent contract)
class BaseRepository {
    func fetch() async throws -> [Any] { /* ... */ }
}
class OrderRepository: BaseRepository {
    override func fetch() async throws -> [Any] {
        // Risk: may add stricter preconditions or wider output
    }
}
```

Use `final class` by default. Reach for class inheritance only when genuinely modeling an "is-a" hierarchy.

### ISP: Protocol Composition

Swift's `&` composition is the ISP enabler:

```swift
// Segregated protocols
protocol Readable {
    func fetch(id: String) async throws -> Entity
}

protocol Writable {
    func save(_ entity: Entity) async throws
}

protocol Deletable {
    func delete(id: String) async throws
}

// Compose only what a client needs
func syncEntities(using repo: Readable & Writable) async throws {
    // Only depends on fetch + save, not delete
}

// Full implementation composes all three
final class EntityRepository: Readable, Writable, Deletable {
    func fetch(id: String) async throws -> Entity { /* ... */ }
    func save(_ entity: Entity) async throws { /* ... */ }
    func delete(id: String) async throws { /* ... */ }
}
```

### DIP: Protocol-Based Dependency Injection

```swift
// 1. Protocol defined at the APPLICATION layer (inner)
protocol ProfileRepositoryProtocol {
    func fetchProfile() async throws -> Profile
}

// 2. Use Case depends on abstraction
final class GetProfileUseCase {
    private let repository: ProfileRepositoryProtocol
    init(repository: ProfileRepositoryProtocol) {
        self.repository = repository
    }
}

// 3. Concrete implementation at DATA layer (outer)
final class ProfileRepository: ProfileRepositoryProtocol {
    private let apiService: APIServiceProtocol
    func fetchProfile() async throws -> Profile {
        let raw = try await apiService.get("/profile", as: ProfileDTO.self)
        return raw.toDomain()
    }
}
```

The protocol is designed around the Use Case's needs, NOT mirroring the Service's API shape.

---

## 4. Reactive Programming on iOS

### Combine Framework Mapped to the Book

| Book RP Concept | Combine Equivalent |
|---|---|
| Observable / Stream | `Publisher` |
| Observer / Subscriber | `Subscriber`, `.sink`, `.assign` |
| Operators | `.map`, `.filter`, `.flatMap`, `.combineLatest`, etc. |
| Subscription / Disposal | `AnyCancellable`, stored in `Set<AnyCancellable>` |
| Hot Stream | `CurrentValueSubject`, `PassthroughSubject` |
| Cold Stream | `Future`, `Deferred` |
| Scheduler | `DispatchQueue`, `RunLoop`, `ImmediateScheduler` |

### async/await and AsyncSequence as Alternatives

```swift
// Combine approach
profileSubject
    .map { $0.displayName }
    .receive(on: DispatchQueue.main)
    .sink { [weak self] name in self?.name = name }
    .store(in: &cancellables)

// async/await approach (simpler for single-shot)
func load() async {
    let profile = try await repository.fetchProfile()
    name = profile.displayName  // already on @MainActor
}

// AsyncSequence (stream equivalent)
func observeUpdates() async {
    for await profile in repository.profileStream() {
        name = profile.displayName
    }
}
```

### When to Use Combine vs async/await

The book says both work; RP is more potent but steeper:

| Scenario | Recommendation |
|---|---|
| Single-shot API call | `async/await` -- simpler, native |
| Real-time stream (WebSocket, DB changes) | `AsyncSequence` or `Combine` |
| Complex multi-stream composition | `Combine` (`combineLatest`, `merge`, `zip`) |
| Debounce, throttle, retry with backoff | `Combine` operators are mature |
| New project, iOS 17+ target | Lean toward `async/await` + `AsyncSequence` |
| Existing Combine codebase | Keep Combine; do not mix paradigms needlessly |

### Hot vs. Cold Streams on iOS

```swift
// Hot -- emits whether or not anyone is subscribed
let currentProfile = CurrentValueSubject<Profile?, Never>(nil)   // has current value
let events = PassthroughSubject<AppEvent, Never>()               // no buffered value

// Cold -- only executes when subscribed
let fetchProfile = Deferred {
    Future<Profile, Error> { promise in
        // work starts here, on subscription
    }
}
```

### Immutability: Swift Structs

Swift value types naturally enforce the book's immutability rule:

```swift
struct Profile {                    // Immutable value type
    let id: UUID
    let name: String
    let email: String
}

// New instance via mutation = safe in async contexts
let updated = Profile(id: profile.id, name: "New Name", email: profile.email)

// Or use copy pattern:
extension Profile {
    func with(name: String? = nil, email: String? = nil) -> Profile {
        Profile(id: id, name: name ?? self.name, email: email ?? self.email)
    }
}
```

**Rule**: Domain models and state-models must be structs (value types). Classes are reserved for reference-semantic objects like ViewModels and Services.

---

## 5. Dependency Injection on iOS

### Resolver (Book's Recommendation)

```swift
import Resolver

extension Resolver: ResolverRegistering {
    public static func registerAllServices() {
        // Data layer
        register { ProfileService() as ProfileServiceProtocol }
        register { ProfileRepository(service: resolve()) as ProfileRepositoryProtocol }

        // Application layer
        register { GetProfileUseCase(repository: resolve()) as GetProfileUseCaseProtocol }

        // Presentation layer
        register { ProfileViewModel(useCase: resolve()) }
    }
}

// Usage in View
struct ProfileView: View {
    @State private var viewModel: ProfileViewModel = Resolver.resolve()
    // ...
}
```

### Swinject

```swift
let container = Container()
container.register(ProfileRepositoryProtocol.self) { r in
    ProfileRepository(service: r.resolve(ProfileServiceProtocol.self)!)
}
container.register(ProfileViewModel.self) { r in
    ProfileViewModel(useCase: r.resolve(GetProfileUseCaseProtocol.self)!)
}
```

### Manual DI with Factory Pattern

```swift
// AppDependencies acts as the composition root
final class AppDependencies {
    lazy var profileService: ProfileServiceProtocol = ProfileService()
    lazy var profileRepository: ProfileRepositoryProtocol =
        ProfileRepository(service: profileService)
    lazy var getProfileUseCase: GetProfileUseCaseProtocol =
        GetProfileUseCase(repository: profileRepository)

    func makeProfileViewModel() -> ProfileViewModel {
        ProfileViewModel(useCase: getProfileUseCase)
    }
}
```

### Swift Native: Protocol + Init Injection

The cleanest approach when DI containers feel like overkill:

```swift
final class ProfileViewModel {
    private let useCase: GetProfileUseCaseProtocol

    init(useCase: GetProfileUseCaseProtocol) {  // protocol in, concretion resolved elsewhere
        self.useCase = useCase
    }
}
```

### @Environment and @EnvironmentObject as DI

These are SwiftUI-specific DI mechanisms (framework layer only):

```swift
// Custom EnvironmentKey
struct ProfileViewModelKey: EnvironmentKey {
    static let defaultValue: ProfileViewModel = .init(/* defaults */)
}

extension EnvironmentValues {
    var profileViewModel: ProfileViewModel {
        get { self[ProfileViewModelKey.self] }
        set { self[ProfileViewModelKey.self] = newValue }
    }
}

// Injection at a parent level
ContentView()
    .environment(\.profileViewModel, dependencies.makeProfileViewModel())
```

**Caution**: `@Environment` / `@EnvironmentObject` are tied to the SwiftUI view tree. They work for framework-layer DI but should NOT replace proper constructor injection in Use Cases and Repositories (which are framework-independent).

---

## 6. Testing on iOS

### XCTest (Standard)

```swift
import XCTest

final class GetProfileUseCaseTests: XCTestCase {
    // Arrange
    private var mockRepository: MockProfileRepository!
    private var sut: GetProfileUseCase!

    override func setUp() {
        mockRepository = MockProfileRepository()
        sut = GetProfileUseCase(repository: mockRepository)
    }

    func testExecuteReturnsProfile() async throws {
        // Arrange
        let expected = Profile(id: UUID(), name: "Test", email: "t@t.com")
        mockRepository.stubbedProfile = expected
        // Act
        let result = try await sut.execute()
        // Assert
        XCTAssertEqual(result.name, expected.name)
    }
}
```

### Swift Testing Framework (iOS 18+ / Xcode 16+)

```swift
import Testing

struct GetProfileUseCaseTests {
    @Test("Returns profile from repository")
    func executeReturnsProfile() async throws {
        let expected = Profile(id: UUID(), name: "Test", email: "t@t.com")
        let mock = MockProfileRepository(stubbedProfile: expected)
        let useCase = GetProfileUseCase(repository: mock)

        let result = try await useCase.execute()

        #expect(result.name == expected.name)
    }

    @Test("Throws when repository fails", arguments: [
        NetworkError.timeout,
        NetworkError.serverError(500)
    ])
    func executeThrowsOnError(error: NetworkError) async {
        let mock = MockProfileRepository(stubbedError: error)
        let useCase = GetProfileUseCase(repository: mock)

        await #expect(throws: error) {
            try await useCase.execute()
        }
    }
}
```

### ViewInspector for SwiftUI Unit Testing

```swift
import ViewInspector

@Test("Shows loading indicator when loading")
func showsLoading() throws {
    let vm = ProfileViewModel(useCase: MockUseCase())
    vm.isLoading = true
    let view = ProfileView(viewModel: vm)

    let progress = try view.inspect().find(ViewType.ProgressView.self)
    XCTAssertNotNil(progress)
}
```

### XCUITest for UI/E2E Tests

```swift
final class ProfileUITests: XCTestCase {
    func testProfileLoadsAndDisplaysName() {
        let app = XCUIApplication()
        app.launch()
        app.tabBars.buttons["Profile"].tap()

        let nameLabel = app.staticTexts["profileName"]
        XCTAssertTrue(nameLabel.waitForExistence(timeout: 5))
        XCTAssertFalse(nameLabel.label.isEmpty)
    }
}
```

### Testing Pyramid on iOS

```
        /  E2E  \            ~10% -- XCUITest (slow, flaky, broad)
       /----------\
      / Integration \        ~20% -- XCTest with real DB/network stubs
     /----------------\
    /    Unit Tests     \    ~70% -- XCTest / Swift Testing (fast, precise)
   /____________________\
```

### Mocking with Protocols (No Framework Needed)

If architecture is clean, dependencies are behind protocols. Mocks are trivial:

```swift
final class MockProfileRepository: ProfileRepositoryProtocol {
    var stubbedProfile: Profile?
    var stubbedError: Error?
    var fetchCallCount = 0

    func fetchProfile() async throws -> Profile {
        fetchCallCount += 1
        if let error = stubbedError { throw error }
        return stubbedProfile!
    }
}
```

No Mockingbird, no Cuckoo, no runtime swizzling. Clean architecture makes mocking frameworks unnecessary.

### Enterprise vs. Startup Testing Strategy

| | Enterprise | Startup |
|---|---|---|
| Style | White-box (mock all deps per layer) | Black-box (ViewModel -> Service, mock boundaries) |
| Scope | Each layer in isolation | Multiple layers together |
| Mocking | Heavy (every protocol mocked) | Light (only external systems) |
| E2E | Yes, for regression | Skip or minimal |
| Refactoring | Mocks must update with internals | Flexible -- only API changes break tests |

---

## 7. Boundaries & Modules on iOS

### Swift Package Manager (Hard Boundaries)

```
MyApp/
  App/                          -- Main target (Framework layer)
  Packages/
    FeatureProfile/             -- SPM package (hard boundary)
      Sources/
        Domain/                 -- Profile domain models
        Application/            -- Use cases
        Presentation/           -- ViewModels
        Data/                   -- Repository impls
      Tests/
    FeatureOrders/              -- Another feature module
    CoreNetworking/             -- Shared infra
    CoreDesignSystem/           -- Shared UI components
```

Each SPM package enforces the dependency rule at compile time. `FeatureProfile` cannot import `FeatureOrders` unless explicitly declared.

### Frameworks / Dynamic Libraries

For large teams (50+ devs), use Xcode frameworks:
- Each team owns one or more `.framework` targets
- Binary dependencies reduce build times
- `@testable import` still works within the framework's test target

### Access Control as Soft Boundaries

| Modifier | Scope | Book Use |
|---|---|---|
| `private` | Same declaration | Hide implementation details |
| `fileprivate` | Same file | Rarely needed |
| `internal` (default) | Same module/package | Soft boundary within a module |
| `package` (Swift 5.9+) | Same SPM package | Expose between targets in one package |
| `public` | Anywhere that imports | API surface of a module |

**Rule**: Default to `internal`. Mark types `public` only when they cross a module boundary. Use `private` aggressively inside classes.

### Xcode Workspace with Multiple Targets

```
MyApp.xcworkspace
  MyApp.xcodeproj
    MyApp (iOS target)
    MyAppTests (unit tests)
    MyAppUITests (UI tests)
  FeatureProfile (SPM or framework)
  FeatureOrders (SPM or framework)
  Core (shared utilities)
```

---

## 8. Common iOS Anti-Patterns (From the Book's Perspective)

### 1. Massive View Controller / Massive ViewModel

**Symptom**: ViewModel has 500+ lines, handles multiple unrelated concerns.
**Fix**: 1 ViewModel = 1 Screen. Extract sub-ViewModels or Use Cases for independent concerns.

### 2. Business Logic in SwiftUI Views

**Symptom**: `if order.total > 100` inside `body`.
**Fix**: Move logic to ViewModel or Presentation Mapper. View only reads presentation-ready properties.

### 3. Core Data @FetchRequest Directly in Views

```swift
// ANTI-PATTERN: framework leaking into presentation
struct OrderListView: View {
    @FetchRequest(sortDescriptors: [SortDescriptor(\.date)])
    private var orders: FetchedResults<CDOrder>  // Core Data entity in View!
}
```

**Fix**: Repository wraps Core Data. ViewModel exposes domain/presentation models. View knows nothing about the persistence framework.

### 4. Singleton Abuse Instead of Proper DI

```swift
// ANTI-PATTERN
class NetworkManager {
    static let shared = NetworkManager()
    // Used everywhere, untestable, hidden dependency
}

// FIX: protocol + injection
protocol NetworkClient {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
}
```

### 5. Not Using Protocols for Dependencies (Tight Coupling)

```swift
// ANTI-PATTERN: ViewModel depends on concrete Repository
class ProfileViewModel {
    let repo = ProfileRepository()  // concrete, untestable
}

// FIX: depend on protocol, inject via init
class ProfileViewModel {
    private let repo: ProfileRepositoryProtocol
    init(repo: ProfileRepositoryProtocol) { self.repo = repo }
}
```

---

## 9. Model Transformation Chain on iOS

### Enterprise (Complex App)

```
Network Response (Data)
    |  JSONDecoder
    v
CodableModel (struct, Codable)          -- Raw API shape, Data layer
    |  manual mapping
    v
NSManagedObject / SwiftData @Model      -- Persistence, Data layer
    |  toDomain()
    v
PlainModel (struct)                     -- Clean struct, no framework deps
    |  business rules applied
    v
DomainModel (struct)                    -- Domain layer, pure logic
    |  toPresentation()
    v
PresentationModel (struct)             -- Formatted strings, colors, flags
    |  consumed by
    v
SwiftUI View                           -- Framework layer, renders only
```

```swift
// CodableModel (Data layer -- matches JSON shape)
struct ProfileDTO: Codable {
    let user_name: String
    let avatar_url: String
    let created_at: String

    func toDomain() -> Profile {
        Profile(
            name: user_name,
            avatarURL: URL(string: avatar_url),
            memberSince: ISO8601DateFormatter().date(from: created_at) ?? .distantPast
        )
    }
}

// DomainModel (Domain layer -- pure)
struct Profile {
    let name: String
    let avatarURL: URL?
    let memberSince: Date
}

// PresentationModel (Presentation layer -- display-ready)
struct ProfilePresentation: Identifiable {
    let id: UUID
    let displayName: String
    let avatarURL: URL?
    let memberSinceText: String   // "Member since Jan 2023"
    let isLongTimeMember: Bool    // for badge display
}
```

### Startup (Pragmatic)

```
Network Response (Data)
    |  JSONDecoder
    v
CodableModel (struct, Codable)         -- Raw API shape
    |  toDomain()
    v
Model (struct)                          -- Merged domain + presentation
    |  consumed by
    v
SwiftUI View
```

```swift
// Startup: single model serves domain + presentation
struct Profile: Codable, Identifiable {
    let id: UUID
    let name: String
    let email: String
    let createdAt: Date

    // Presentation concern merged in for pragmatism
    var displayName: String { name.isEmpty ? email : name }
    var initials: String { String(name.prefix(2)).uppercased() }
}
```

The startup approach trades separation for speed. The book says this is fine for small teams / short-lived apps, as long as you know when to refactor toward the enterprise chain (Preparatory Refactoring when complexity demands it).

---

## Quick Reference: iOS Library Choices (2025-2026 Community Consensus)

| Concern | Recommended | Alternatives | Notes |
|---|---|---|---|
| DI | Manual protocol + init injection | swift-dependencies (Point-Free), Factory | Resolver/Swinject are legacy. Manual DI is simplest and most common. swift-dependencies suits TCA projects. |
| Networking | URLSession (native) | Alamofire (isolate behind protocol) | Native is sufficient for most apps |
| Persistence | SwiftData / Core Data (behind Repository) | Realm, GRDB, UserDefaults | Always behind a Repository protocol |
| Reactive | async/await + AsyncSequence | Combine (maintained but not evolving) | async/await has effectively replaced Combine for new code. Combine fine for legacy. |
| Navigation | NavigationStack (native) | Coordinator pattern | Use native SwiftUI navigation |
| Testing | Swift Testing (@Test, #expect) | XCTest (still works, dominant in existing codebases) | Swift Testing is Apple's direction (WWDC 2024+). |
| UI Testing | XCUITest | swift-snapshot-testing, ViewInspector | Snapshot testing is more reliable than ViewInspector across OS versions |
| Architecture | MVVM (mainstream) | TCA (opinionated, steep curve, best testability) | TCA is production-ready but not mainstream. Plain MVVM + CA is most common. |

**Rule from the book**: Select libraries LAST. Architecture should banish them to the outermost layer. Every third-party library is accessed through a protocol so it can be replaced.
