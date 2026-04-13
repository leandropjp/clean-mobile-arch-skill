# Clean Mobile Architecture -- Kotlin Multiplatform (KMP) Platform Reference

**Source**: Mapping "Clean Mobile Architecture" by Petros Efthymiou to KMP conventions

---

## 1. How Clean Architecture Maps to KMP

KMP's module structure naturally enforces the book's four Clean Architecture layers. The shared module IS the inner layers; platform-specific code IS the outer layer. This is not a convention -- it is an architectural constraint enforced by the build system.

### Layer Mapping

| Clean Architecture Layer | KMP Location | What Lives Here |
|---|---|---|
| **Domain Model & Logic** (center) | `shared/commonMain` | Entities, business rules, domain models (pure Kotlin, zero platform deps) |
| **Application Layer** (use cases) | `shared/commonMain` | Use Cases, orchestrators, application-specific rules |
| **Interface Adapters** (presentation + data) | `shared/commonMain` + platform source sets | ViewModels, mappers, repositories, data sources |
| **Frameworks & Drivers** (outermost) | `iosMain` / `androidMain` / platform app modules | SwiftUI, Compose, UIKit, platform SDKs |

### Project Structure

```
project/
  shared/
    src/
      commonMain/          # Domain + Application + shared Interface Adapters
        kotlin/
          domain/          # Entities, business rules (ZERO dependencies)
          application/     # Use Cases
          data/            # Repository interfaces, mappers, expect declarations
          presentation/    # Shared ViewModels, state models, intent models
      commonTest/          # Shared tests (run on ALL platforms)
      androidMain/         # Android-specific actual implementations
      iosMain/             # iOS-specific actual implementations
  androidApp/              # Framework layer: Compose UI, Activities
  iosApp/                  # Framework layer: SwiftUI/UIKit Views
```

### The Dependency Rule -- Enforced by KMP

The book's Dependency Rule states: "Source code dependencies can only point inwards." In KMP, this is **architecturally enforced**:

- `shared` module CANNOT depend on `androidApp` or `iosApp` modules
- `commonMain` CANNOT import from `androidMain` or `iosMain`
- Platform modules depend on `shared`, never the reverse

This means inner layers (domain, application) are physically incapable of referencing framework code. The book's #1 rule -- "Framework code must be isolated in the outermost layer" -- is a build-time guarantee.

### expect/actual as the DIP Mechanism

The `expect`/`actual` mechanism is Dependency Inversion built into the language:

```kotlin
// commonMain -- the interface (high-level policy)
expect class PlatformDateFormatter() {
    fun format(timestamp: Long): String
}

// androidMain -- the implementation (low-level detail)
actual class PlatformDateFormatter {
    actual fun format(timestamp: Long): String {
        return SimpleDateFormat("yyyy-MM-dd", Locale.getDefault())
            .format(Date(timestamp))
    }
}

// iosMain -- the implementation (low-level detail)
actual class PlatformDateFormatter {
    actual fun format(timestamp: Long): String {
        val formatter = NSDateFormatter()
        formatter.dateFormat = "yyyy-MM-dd"
        return formatter.stringFromDate(
            NSDate.dateWithTimeIntervalSince1970(timestamp.toDouble() / 1000)
        )
    }
}
```

The high-level module (`commonMain`) defines WHAT it needs. The low-level modules (`androidMain`, `iosMain`) provide HOW. Dependencies point inward. This is textbook DIP.

---

## 2. MV* Patterns with KMP

### Shared ViewModels

Using KMP-ViewModel (lifecycle-viewmodel-compose) or a custom base:

```kotlin
// commonMain -- Shared ViewModel
class OrderListViewModel(
    private val getOrdersUseCase: GetOrdersUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(OrderListState())
    val state: StateFlow<OrderListState> = _state.asStateFlow()

    fun loadOrders() {
        viewModelScope.launch {
            _state.update { it.copy(isLoading = true) }
            getOrdersUseCase()
                .onSuccess { orders ->
                    _state.update { it.copy(isLoading = false, orders = orders) }
                }
                .onFailure { error ->
                    _state.update { it.copy(isLoading = false, error = error.message) }
                }
        }
    }
}
```

### MVVM: StateFlow Consumed by Both Platforms

**Android (Compose) -- Framework layer:**

```kotlin
@Composable
fun OrderListScreen(viewModel: OrderListViewModel = koinViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()

    when {
        state.isLoading -> CircularProgressIndicator()
        state.error != null -> ErrorMessage(state.error!!)
        else -> OrderList(state.orders)
    }
}
```

**iOS (SwiftUI) -- Framework layer, using SKIE:**

```swift
struct OrderListScreen: View {
    @StateObject private var viewModel = OrderListViewModel()

    var body: some View {
        // SKIE converts StateFlow to Swift's AsyncSequence
        Observing(viewModel.state) { state in
            if state.isLoading {
                ProgressView()
            } else if let error = state.error {
                ErrorMessageView(error: error)
            } else {
                OrderListView(orders: state.orders)
            }
        }
    }
}
```

### MVI: Shared Reducer in commonMain

```kotlin
// commonMain -- State, Intent, Reducer
data class OrderListState(
    val isLoading: Boolean = false,
    val orders: List<Order> = emptyList(),
    val error: String? = null
)

sealed interface OrderListIntent {
    data object LoadOrders : OrderListIntent
    data class SelectOrder(val orderId: String) : OrderListIntent
    data object Retry : OrderListIntent
}

// Reducer: pure function, trivially testable
fun reduce(current: OrderListState, result: OrderResult): OrderListState =
    when (result) {
        is OrderResult.Loading -> current.copy(isLoading = true, error = null)
        is OrderResult.Success -> current.copy(isLoading = false, orders = result.orders)
        is OrderResult.Failure -> current.copy(isLoading = false, error = result.message)
    }
```

Both platforms subscribe to a single `StateFlow<OrderListState>`. The reducer is a pure function in `commonMain` -- tested once, works everywhere.

### Bridging Kotlin Flow to Swift

This is the central KMP challenge for MV* patterns. Options ranked by the book's principles:

| Approach | Pros | Cons |
|---|---|---|
| **SKIE** (recommended) | Native Swift async/await, transparent | Touchlab dependency |
| **KMP-NativeCoroutines** | Mature, well-tested | Annotation boilerplate |
| **Manual wrapping** | No dependencies | Verbose, error-prone lifecycle management |

The book says: "Select libraries LAST." But SKIE or KMP-NativeCoroutines solve a real architectural problem (bridging reactive primitives), not a convenience problem. They belong in the Framework layer boundary.

---

## 3. SOLID on KMP

### SRP / SCP: Shared Concerns, Platform Rendering

The book defines three class categories: **Executor**, **Delegator**, **Data Container**. In KMP:

- **Executors** (Use Cases, mappers, validators) live in `commonMain` -- shared across platforms
- **Delegators** (ViewModels, orchestrators) live in `commonMain` -- shared across platforms
- **Data Containers** (models, state, intents) live in `commonMain` -- shared across platforms
- **Platform code** has a single concern: **rendering**. Views in `androidApp`/`iosApp` are "dumb" -- no logic, no conditionals based on data

```kotlin
// commonMain -- Executor (single concern: format currency)
class CurrencyFormatter {
    fun format(amount: Double, currencyCode: String): String {
        // Pure business logic, no platform dependency
        return "${currencySymbol(currencyCode)}${"%.2f".format(amount)}"
    }
}
```

The book's rule: "1 ViewModel per screen, 1 Repository per entity" applies identically in KMP. The ViewModel just happens to be shared.

### OCP: Kotlin Interfaces + Platform Implementations

```kotlin
// commonMain -- open for extension via new implementations
interface PaymentProcessor {
    suspend fun process(payment: Payment): PaymentResult
}

// commonMain -- Strategy pattern, new processors added without modifying existing code
class PaymentOrchestrator(
    private val processors: Map<PaymentType, PaymentProcessor>
) {
    suspend fun process(payment: Payment): PaymentResult {
        val processor = processors[payment.type]
            ?: throw UnsupportedPaymentTypeException(payment.type)
        return processor.process(payment)
    }
}
```

New payment types = new `PaymentProcessor` implementation. No modification to `PaymentOrchestrator`. expect/actual adds another OCP dimension: new platforms = new `actual` implementations without touching `commonMain`.

### LSP: Contracts Defined in Shared Code

Pure interfaces in `commonMain` define the contract. Platform implementations MUST respect it:

```kotlin
// commonMain -- contract
interface LocalStorage {
    suspend fun save(key: String, value: String)
    suspend fun load(key: String): String?  // null = not found, never throws
    suspend fun delete(key: String)
}

// androidMain -- must respect the contract
actual class PlatformLocalStorage(context: Context) : LocalStorage {
    override suspend fun load(key: String): String? {
        // Must return null for missing keys, not throw
        return sharedPreferences.getString(key, null)
    }
    // ...
}
```

The book warns: "Unit test overridden functions in subclasses against all expected inputs." In KMP, write those tests in `commonTest` -- they validate the contract on ALL platforms.

### ISP: Small Interfaces in commonMain

```kotlin
// WRONG -- fat interface forces platforms to implement everything
interface DataSource {
    suspend fun fetchUsers(): List<User>
    suspend fun fetchOrders(): List<Order>
    suspend fun fetchProducts(): List<Product>
    suspend fun syncAll()
}

// RIGHT -- segregated interfaces, platforms implement only what they need
interface UserDataSource {
    suspend fun fetchUsers(): List<User>
}

interface OrderDataSource {
    suspend fun fetchOrders(): List<Order>
}
```

The book's ISP applies at the module level too: platform source sets should only depend on the `commonMain` interfaces they actually use.

### DIP: expect/actual IS Dependency Inversion

```kotlin
// commonMain -- high-level policy defines what it needs
expect interface DatabaseDriver {
    fun execute(query: String): QueryResult
}

class UserRepository(private val driver: DatabaseDriver) {
    // Depends on abstraction, not on SQLDelight or Room or CoreData
    suspend fun getUser(id: String): User? {
        return driver.execute("SELECT * FROM users WHERE id = ?")
            .mapToUser()
    }
}

// androidMain -- low-level detail provides the implementation
actual class SqlDelightDatabaseDriver : DatabaseDriver { /* ... */ }

// iosMain -- different low-level detail, same contract
actual class NativeDatabaseDriver : DatabaseDriver { /* ... */ }
```

The dependency arrow points from platform (`actual`) toward shared (`expect`). Inner layers are protected from outer-layer volatility -- exactly as the book prescribes.

---

## 4. Reactive Programming with KMP

### Kotlin Flow: The Universal Reactive Primitive

```kotlin
// commonMain -- Flow works identically on all platforms
class ObserveOrdersUseCase(
    private val orderRepository: OrderRepository
) {
    operator fun invoke(): Flow<List<Order>> =
        orderRepository.observeOrders()
            .map { orders -> orders.filter { it.isActive } }
}
```

### StateFlow for UI State

```kotlin
// commonMain -- single state stream (MVI-like, book's recommendation)
class DashboardViewModel(
    private val observeMetrics: ObserveMetricsUseCase
) : ViewModel() {

    private val _state = MutableStateFlow(DashboardState())
    val state: StateFlow<DashboardState> = _state.asStateFlow()

    init {
        viewModelScope.launch {
            observeMetrics().collect { metrics ->
                _state.update { it.copy(metrics = metrics, isLoading = false) }
            }
        }
    }
}
```

### Bridging to Platforms

**Android -- native, no bridging needed:**

```kotlin
@Composable
fun DashboardScreen(viewModel: DashboardViewModel) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    DashboardContent(state)
}
```

**iOS with SKIE -- transparent bridging:**

```swift
// SKIE makes StateFlow<T> work as Swift AsyncSequence
struct DashboardScreen: View {
    let viewModel: DashboardViewModel

    var body: some View {
        Observing(viewModel.state) { state in
            DashboardContent(state: state)
        }
    }
}
```

**iOS without SKIE -- manual collection (last resort):**

```swift
class DashboardObservable: ObservableObject {
    @Published var state: DashboardState = DashboardState()
    private var job: Kotlinx_coroutines_coreJob? = nil

    func observe(_ viewModel: DashboardViewModel) {
        job = FlowCollector.collect(viewModel.state) { [weak self] state in
            DispatchQueue.main.async {
                self?.state = state
            }
        }
    }

    deinit { job?.cancel(cause: nil) }
}
```

### Coroutines in Shared Code

`suspend` functions work cross-platform. They are the shared async primitive:

```kotlin
// commonMain -- suspend function, no platform dependency
class PlaceOrderUseCase(
    private val orderRepository: OrderRepository,
    private val paymentService: PaymentService
) {
    suspend operator fun invoke(order: Order): Result<OrderConfirmation> =
        runCatching {
            val payment = paymentService.charge(order.total)
            orderRepository.save(order.copy(paymentId = payment.id))
        }
}
```

### Immutability

The book states: "In RP, immutable objects are mandatory." Kotlin data classes with `copy()` in `commonMain` enforce this:

```kotlin
// commonMain -- immutable state, new instances via copy()
data class CartState(
    val items: List<CartItem> = emptyList(),
    val total: Double = 0.0,
    val isCheckingOut: Boolean = false
)

// State transitions always produce new instances
fun addItem(current: CartState, item: CartItem): CartState =
    current.copy(
        items = current.items + item,
        total = current.items.sumOf { it.price } + item.price
    )
```

---

## 5. Dependency Injection with KMP

### Koin Multiplatform (Recommended)

The book recommends Koin for Android. Koin Multiplatform extends this to KMP:

```kotlin
// commonMain -- shared module definition
val sharedModule = module {
    // Domain
    factory { GetOrdersUseCase(get()) }
    factory { PlaceOrderUseCase(get(), get()) }

    // Data
    single<OrderRepository> { OrderRepositoryImpl(get(), get()) }

    // Presentation
    viewModel { OrderListViewModel(get()) }
}

// androidMain -- platform-specific dependencies
val androidModule = module {
    single<DatabaseDriver> { AndroidSqliteDriver(AppDatabase.Schema, get(), "app.db") }
    single<HttpEngine> { OkHttp.create() }
}

// iosMain -- platform-specific dependencies
val iosModule = module {
    single<DatabaseDriver> { NativeSqliteDriver(AppDatabase.Schema, "app.db") }
    single<HttpEngine> { Darwin.create() }
}
```

**Initialization:**

```kotlin
// androidMain
fun initKoin(context: Context) {
    startKoin {
        modules(sharedModule, androidModule)
        androidContext(context)
    }
}

// iosMain (called from Swift)
fun initKoin() {
    startKoin {
        modules(sharedModule, iosModule)
    }
}
```

### Manual DI with expect/actual Factories

For simpler apps (MVA -- "the simplest architecture that can support clean implementation"):

```kotlin
// commonMain
expect class PlatformDependencies {
    val httpEngine: HttpEngine
    val databaseDriver: DatabaseDriver
}

class AppDependencies(platform: PlatformDependencies) {
    // Data layer
    private val apiClient = ApiClient(platform.httpEngine)
    private val database = AppDatabase(platform.databaseDriver)
    val orderRepository: OrderRepository = OrderRepositoryImpl(apiClient, database)

    // Domain layer
    val getOrders = GetOrdersUseCase(orderRepository)

    // Presentation layer
    fun orderListViewModel() = OrderListViewModel(getOrders)
}
```

### When to Use What

| Scenario | DI Approach | Book Alignment |
|---|---|---|
| Startup / small team | Manual DI with `expect`/`actual` factories | MVA -- simplest viable |
| Medium complexity | Koin Multiplatform | Book's Android recommendation, scales well |
| Platform-heavy features | Koin shared + Hilt (Android) / manual (iOS) | Platform DI for platform-only deps |
| Large enterprise | Koin Multiplatform + feature module scoping | Full DIP with hard boundaries |

---

## 6. Testing with KMP

### commonTest: Write Once, Verify Everywhere

This is the book's testing dream realized. Tests in `commonTest` run on ALL target platforms:

```kotlin
// commonTest -- this test runs on JVM, iOS Simulator, JS, etc.
class GetOrdersUseCaseTest {

    private val fakeRepository = FakeOrderRepository()
    private val useCase = GetOrdersUseCase(fakeRepository)

    @Test
    fun `returns active orders sorted by date`() = runTest {
        // Arrange
        fakeRepository.setOrders(listOf(
            Order(id = "1", date = yesterday, isActive = true),
            Order(id = "2", date = today, isActive = true),
            Order(id = "3", date = today, isActive = false),
        ))

        // Act
        val result = useCase()

        // Assert
        assertEquals(2, result.getOrNull()?.size)
        assertEquals("2", result.getOrNull()?.first()?.id)
    }
}
```

### Testing Pyramid for KMP

```
                    /  E2E per platform  \          ~10%
                   / (Espresso / XCUITest) \
                  /-------------------------\
                 /  Integration tests        \      ~20%
                / (androidTest / iosTest)      \
               /--------------------------------\
              /  commonTest unit tests            \  ~70%
             / (run on ALL platforms automatically) \
            /----------------------------------------\
```

The book's pyramid maps perfectly:
- **~70% Unit tests in `commonTest`**: Shared business logic, Use Cases, ViewModels, mappers, reducers. This is where most logic lives in KMP.
- **~20% Integration tests in platform test source sets**: Platform-specific implementations (`actual` classes), DI wiring, database/network integration.
- **~10% E2E per platform**: Espresso (Android), XCUITest (iOS). Only for critical user flows.

### Mock Shared Interfaces in commonTest

```kotlin
// commonTest -- fake implementation for testing
class FakeOrderRepository : OrderRepository {
    private var orders: List<Order> = emptyList()
    private var shouldFail = false

    fun setOrders(orders: List<Order>) { this.orders = orders }
    fun setShouldFail(fail: Boolean) { shouldFail = fail }

    override suspend fun getOrders(): Result<List<Order>> =
        if (shouldFail) Result.failure(IOException("Network error"))
        else Result.success(orders)
}
```

No mocking framework needed for most cases. The book's Chicago/Classicist TDD school aligns well: use fakes, test behavior, keep tests refactoring-friendly.

### Platform-Specific Tests Only for Platform Code

```kotlin
// androidTest -- only for Android-specific implementations
class AndroidSqliteOrderDaoTest {
    @Test
    fun `saves and retrieves order from SQLite`() {
        val driver = JdbcSqliteDriver(JdbcSqliteDriver.IN_MEMORY)
        val dao = AndroidOrderDao(driver)
        // Test actual SQLite behavior on Android
    }
}
```

### Enterprise vs. Startup Testing (from the book)

| Strategy | commonTest | Platform Tests | E2E |
|---|---|---|---|
| **Enterprise** | White-box per layer, all deps mocked | Integration tests per `actual` class | Full regression suites |
| **Startup** | Black-box (ViewModel to Repository boundary) | Minimal, only critical platform code | Skip or minimal |

---

## 7. Boundaries & Modules with KMP

### Source Sets as Natural Layer Boundaries

KMP source sets enforce the book's boundary strategies at the compiler level:

```
commonMain  (inner layers)  <--  androidMain  (outer layer)
                            <--  iosMain      (outer layer)
```

`commonMain` cannot see platform source sets. This is a **hard boundary** -- not a convention that can be violated, but a compiler-enforced wall.

### Gradle Modules for Feature Decomposition

For complex apps (the book's "Enterprise" tier), combine KMP source sets with Gradle modules:

```
:shared:core           # Domain models, base interfaces
:shared:feature-orders # Orders feature (domain + data + presentation)
:shared:feature-auth   # Auth feature
:androidApp            # Android Compose UI
:iosApp                # iOS SwiftUI (via Xcode project referencing shared framework)
```

Each `:shared:feature-*` module is itself a KMP module with `commonMain`, `androidMain`, `iosMain`.

### The Book's Boundary Spectrum Applied to KMP

| Boundary Type | KMP Implementation | When to Use |
|---|---|---|
| **Soft** (packages) | Packages within `commonMain` (`domain/`, `data/`, `presentation/`) | MVA starting point, small teams |
| **Medium** (source sets) | `commonMain` vs `androidMain` vs `iosMain` | Always present in KMP (free) |
| **Hard** (modules) | Separate Gradle modules (`:shared:feature-x`) | Complex apps, large teams, multiple features |

### Feature Modules: Shared vs. Platform-Specific

```
:shared:feature-orders/
  commonMain/       # Use Cases, Repository interfaces, ViewModel, State
  androidMain/      # Android-specific data source implementations
  iosMain/          # iOS-specific data source implementations

:androidApp/        # Compose screens for orders, navigation
:iosApp/            # SwiftUI screens for orders, navigation
```

The book's rule: "Avoid segregating into feature modules from the start unless needed." Start with packages inside a single `shared` module. Extract to feature modules only when:
- Multiple teams work on separate features
- Build times warrant modularization
- Features have genuinely independent lifecycles

### The Contain Cancer Principle

Platform-specific code is the "cancer" in KMP terms. It is **contained** by design:

- Cancer (platform specifics) lives ONLY in `androidMain`, `iosMain`, or platform app modules
- Shared code (`commonMain`) is cancer-free -- pure Kotlin with no platform imports
- If cancer needs to spread (e.g., platform capability needed in shared logic), use `expect`/`actual` to create a **firewall**: the interface stays clean, only the implementation touches platform code

---

## 8. Common KMP Anti-Patterns (from the book's perspective)

### 1. Platform-Specific Code in commonMain

```kotlin
// WRONG -- platform assumption leaking into shared code
// commonMain
class UserRepository {
    fun save(user: User) {
        // Android-specific! This will not compile for iOS.
        val context = ApplicationContext.get()
        context.getSharedPreferences("users", MODE_PRIVATE)
            .edit().putString(user.id, user.toJson()).apply()
    }
}

// RIGHT -- use expect/actual or interface
// commonMain
class UserRepository(private val storage: LocalStorage) {
    suspend fun save(user: User) {
        storage.save(user.id, user.toJson())
    }
}
```

**Violates**: Layer separation, Dependency Rule. Inner layer references framework code.

### 2. Exposing Raw Kotlin Flow to Swift

```kotlin
// WRONG -- Swift developers get an unusable KotlinFlow<NSArray> type
class OrderViewModel {
    val orders: StateFlow<List<Order>> = /* ... */
    // iOS sees: Kotlinx_coroutines_coreStateFlow -- unidiomatic, painful
}
```

Use SKIE or KMP-NativeCoroutines to bridge properly. The book's principle: "Data crossing boundaries must be in the form most convenient to the inner circle." Here, the iOS "circle" needs Swift-native types.

### 3. Duplicating Business Logic in Platform Modules

```kotlin
// WRONG -- same validation logic in both platforms
// androidMain
fun validateEmail(email: String): Boolean = email.contains("@") && email.length > 5

// iosMain
fun validateEmail(email: String): Boolean = email.contains("@") && email.length > 5

// RIGHT -- shared in commonMain
// commonMain
fun validateEmail(email: String): Boolean = email.contains("@") && email.length > 5
```

**Violates**: DRY, defeats the purpose of KMP. The book warns about "accidental duplication" -- identical code serving different actors is sometimes OK, but identical business rules must be shared.

### 4. expect/actual When Pure Kotlin Exists

```kotlin
// WRONG -- using expect/actual for something that has a pure Kotlin solution
expect fun parseJson(json: String): Map<String, Any>
// Then implementing with Gson on Android and NSJSONSerialization on iOS

// RIGHT -- use kotlinx.serialization (pure Kotlin, works in commonMain)
@Serializable
data class OrderResponse(val id: String, val total: Double)
val order = Json.decodeFromString<OrderResponse>(json)
```

The book says: "Select libraries LAST." But when a pure Kotlin multiplatform library exists (kotlinx.serialization, Ktor, SQLDelight), prefer it over platform-specific alternatives behind `expect`/`actual`.

### 5. God Shared Module

```
// WRONG -- everything in one shared module
shared/commonMain/
  kotlin/
    (200+ files across all features, all layers, all concerns)

// RIGHT -- decompose when complexity warrants
:shared:core
:shared:feature-auth
:shared:feature-orders
:shared:feature-payments
```

**Violates**: SRP/SCP at the module level. The book's MVA approach: start monolithic, decompose via Preparatory Refactoring when the module becomes unwieldy.

### 6. Not Testing in commonTest

Writing tests only in `androidTest` or as XCTests when the logic lives in `commonMain` means:
- Tests only verify one platform
- Bugs on the other platform go undetected
- You lose KMP's biggest architectural advantage

The book says: "A testable app is, by rule, a well-architected app." KMP makes shared code inherently testable -- not using `commonTest` is leaving the biggest benefit on the table.

---

## 9. Model Transformation Chain with KMP

The book defines the enterprise chain: `JSON -> ModelRaw -> ModelDB -> ModelPlain -> Model -> ModelPresentation`

### KMP Transformation Chain

```
Shared (commonMain):
  ApiModel  -->  PlainModel  -->  DomainModel
  (kotlinx.       (data class,    (business entity,
  serialization)   no annotations)  core rules)

Platform (androidMain/iosMain):
  DomainModel  -->  PresentationModel
                    (platform-specific formatting)
```

### Implementation

```kotlin
// commonMain -- API layer (Data)
@Serializable
data class OrderApiModel(
    @SerialName("order_id") val orderId: String,
    @SerialName("total_cents") val totalCents: Long,
    @SerialName("created_at") val createdAt: String,
    val status: String
)

// commonMain -- Domain layer
data class Order(
    val id: String,
    val total: Double,
    val createdAt: Long,  // Unix timestamp
    val status: OrderStatus
)

enum class OrderStatus { PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED }

// commonMain -- Mapper (Interface Adapter, Executor class per SCP)
class OrderApiMapper {
    fun toDomain(api: OrderApiModel): Order = Order(
        id = api.orderId,
        total = api.totalCents / 100.0,
        createdAt = Instant.parse(api.createdAt).toEpochMilliseconds(),
        status = OrderStatus.valueOf(api.status.uppercase())
    )
}
```

**Android Presentation (androidApp):**

```kotlin
// Android-specific presentation model
data class OrderUiModel(
    val id: String,
    val formattedTotal: String,   // "$12.99"
    val formattedDate: String,    // "Mar 15, 2026"
    val statusColor: Color,       // Compose Color
    val statusLabel: String       // "Shipped"
)

class OrderUiMapper(private val context: Context) {
    fun toUiModel(order: Order): OrderUiModel = OrderUiModel(
        id = order.id,
        formattedTotal = NumberFormat.getCurrencyInstance().format(order.total),
        formattedDate = DateFormat.getDateInstance().format(Date(order.createdAt)),
        statusColor = when (order.status) {
            OrderStatus.DELIVERED -> Color.Green
            OrderStatus.CANCELLED -> Color.Red
            else -> Color.Gray
        },
        statusLabel = order.status.name.lowercase().replaceFirstChar { it.uppercase() }
    )
}
```

**iOS Presentation (Swift):**

```swift
struct OrderUIModel {
    let id: String
    let formattedTotal: String
    let formattedDate: String
    let statusColor: Color
    let statusLabel: String
}

extension OrderUIModel {
    init(from order: Order) {
        self.id = order.id
        self.formattedTotal = order.total.formatted(.currency(code: "USD"))
        self.formattedDate = Date(timeIntervalSince1970: Double(order.createdAt) / 1000)
            .formatted(date: .abbreviated, time: .omitted)
        self.statusColor = switch order.status {
            case .delivered: .green
            case .cancelled: .red
            default: .gray
        }
        self.statusLabel = order.status.name.localizedCapitalized
    }
}
```

### Startup Simplification

For simple apps (MVA), skip `PlainModel` and collapse the chain:

```kotlin
// commonMain -- merged for simplicity
@Serializable
data class Order(
    @SerialName("order_id") val id: String,
    @SerialName("total_cents") val totalCents: Long,
    val status: String
) {
    val total: Double get() = totalCents / 100.0
    val orderStatus: OrderStatus get() = OrderStatus.valueOf(status.uppercase())
}
```

The book's MVA principle: "The simplest architecture that can support the clean implementation of the product." Add transformation layers only when complexity demands it.

---

## 10. KMP-Specific Architectural Advantages

### 1. Enforced Framework Isolation

The book's #1 rule: "Framework code must be isolated in the outermost layer." In KMP, this is not a guideline -- it is a **compiler constraint**. `commonMain` physically cannot import `android.content.Context` or `UIKit`. The architecture is enforced, not just recommended.

### 2. Shared Domain is Inherently Framework-Independent

Domain and Application layers in `commonMain` have zero framework dependencies by definition. The book's ideal of "domain models with zero external dependencies" is the default state in KMP, not something that requires discipline to maintain.

### 3. expect/actual IS Dependency Inversion

The book dedicates significant attention to DIP as a technique. In KMP, `expect`/`actual` makes DIP a language-level feature:
- `expect` = the abstraction (designed around the Use Case's needs)
- `actual` = the implementation (low-level platform detail)
- The compiler enforces the contract

### 4. Test Once, Validate Everywhere

The book's testing strategy emphasizes unit tests as the pyramid's base. In KMP, `commonTest` tests run on every target platform. One test suite validates business logic for Android, iOS, desktop, and web simultaneously. This is the maximum return on testing investment.

### 5. The "Contain Cancer" Principle -- Built In

Platform-specific code (the "cancer" that must be isolated) is structurally contained in platform source sets. It cannot leak into shared code. The `expect`/`actual` boundary is the firewall. You can "treat the cancer" later (replace platform implementations) without touching shared business logic.

### 6. Preparatory Refactoring is Cheaper

The book recommends Preparatory Refactoring: "refactor right before a feature the codebase can't absorb." In KMP, refactoring shared code automatically benefits all platforms. A single refactoring pass in `commonMain` improves the architecture for Android AND iOS simultaneously, halving the refactoring cost compared to maintaining separate codebases.

### 7. Technical Price Triangle Optimization

The book's Technical Price Triangle (People + Feature Complexity + Longevity) tilts favorably with KMP:
- **People**: One team writes shared logic instead of two platform teams duplicating effort
- **Feature Complexity**: Complex logic written and tested once
- **Longevity**: Shared code survives platform UI framework changes (SwiftUI replacing UIKit, Compose replacing Views)

---

## Quick Reference: Book Concept to KMP Mapping

| Book Concept | KMP Implementation |
|---|---|
| Dependency Rule | `commonMain` cannot import platform code (compiler-enforced) |
| DIP | `expect`/`actual` + Kotlin interfaces |
| Contain Cancer | Platform code isolated in `androidMain`/`iosMain` |
| MVA | Start with single `shared` module, soft boundaries (packages) |
| Preparatory Refactoring | Extract feature modules from shared when complexity grows |
| 1 ViewModel = 1 Screen | Shared ViewModel in `commonMain`, one per screen |
| 1 Repository = 1 Entity | Repository interface in `commonMain`, `actual` implementations |
| Testing Pyramid (70% unit) | `commonTest` -- runs on all platforms |
| MVI State-model | `data class` in `commonMain` with `StateFlow` |
| MVI Intent-model | `sealed interface` in `commonMain` |
| Reducer | Pure function in `commonMain` (trivially testable) |
| Strategy Pattern (OCP) | Kotlin interface + implementations + Koin DI |
| Model Transformation | Shared mappers in `commonMain`, presentation mappers per platform |
| Select Libraries LAST | Prefer pure Kotlin multiplatform libs (Ktor, SQLDelight, kotlinx.*) |
| Framework Must Be Dumb | Platform Views only render state, zero business logic |

## Quick Reference: KMP Library Choices (2025-2026 Community Consensus)

| Concern | Recommended | Alternatives | Notes |
|---|---|---|---|
| DI | Koin Multiplatform | Kodein, manual expect/actual factories | Koin has first-class KMP support. Hilt/Dagger are Android-only. |
| Networking | Ktor Client | - | Native KMP HTTP client. Plug platform engines (Darwin, OkHttp). |
| Persistence | SQLDelight | Realm KMP, DataStore | SQLDelight generates type-safe Kotlin from SQL. Room is Android-only. |
| Serialization | kotlinx.serialization | Moshi (Android-only) | Pure multiplatform. Use for JSON, Protobuf, CBOR. |
| Coroutine bridging (iOS) | SKIE (Touchlab) | KMP-NativeCoroutines | SKIE has won — seamless Swift interop with zero boilerplate. |
| ViewModel | AndroidX lifecycle-viewmodel KMP (official) | Rickclephas/KMP-ObservableViewModel | JetBrains official artifact is now stable. Use official. |
| Testing | kotlin.test (commonTest) | - | Write once, runs on all platforms. This is KMP's biggest testing advantage. |
| Date/Time | kotlinx-datetime | - | Pure KMP. Don't use java.time or Foundation.Date in shared code. |
| Async | kotlinx.coroutines | - | StateFlow, SharedFlow, suspend functions all work cross-platform. |
