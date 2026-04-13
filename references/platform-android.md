# Clean Mobile Architecture -- Android (Kotlin) Platform Reference

**Source**: "Clean Mobile Architecture" by Petros Efthymiou (2022) -- mapped to Android/Kotlin conventions

---

## 1. How Clean Architecture Maps to Android

### The Four Layers

| Clean Architecture Layer | Android Mapping | Key Classes |
|---|---|---|
| **Domain** (innermost) | Pure Kotlin module (`:domain`) | Data classes (entities), business rule functions, repository interfaces |
| **Application** (Use Cases) | Pure Kotlin module or package | Use Case classes (`operator fun invoke()`), orchestration logic |
| **Interface Adapters** | Platform-agnostic Kotlin | Platform-agnostic ViewModels, Presenters, DataGateway implementations, Mappers |
| **Framework & Drivers** (outermost) | Android SDK code | Activities, Fragments, Composables, Android `ViewModel`, Navigation, Room, Retrofit, Hilt modules |

### The Critical ViewModel Distinction

The book draws a hard line between two "ViewModel" concepts:

```kotlin
// FRAMEWORK LAYER -- extends Android's ViewModel (lifecycle-aware, survives config changes)
// This is a framework class. It belongs in the outermost layer.
class LoginAndroidViewModel(
    private val loginViewModel: LoginViewModel // delegates to the real ViewModel
) : androidx.lifecycle.ViewModel() {

    val uiState: StateFlow<LoginUiState> = loginViewModel.state
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), LoginUiState.Idle)
}

// INTERFACE ADAPTERS LAYER -- plain Kotlin class, no Android dependency
// This is the "real" ViewModel per the book. Platform-agnostic.
class LoginViewModel(
    private val loginUseCase: LoginUseCase
) {
    private val _state = MutableStateFlow<LoginUiState>(LoginUiState.Idle)
    val state: StateFlow<LoginUiState> = _state.asStateFlow()

    suspend fun login(email: String, password: String) {
        _state.value = LoginUiState.Loading
        loginUseCase(email, password)
            .onSuccess { _state.value = LoginUiState.Success(it) }
            .onFailure { _state.value = LoginUiState.Error(it.message) }
    }
}
```

**Rule**: If it extends `androidx.lifecycle.ViewModel`, it lives in the Framework layer. The book treats Android ViewModel as a framework convenience wrapper, not the place for logic.

**Pragmatic note (MVA)**: For startups and small teams, the book acknowledges merging these into one class is acceptable -- but the logic should still be delegated to Use Cases, not placed directly in the Android ViewModel.

### Jetpack Compose as the "Dumb" View Layer

Composables must contain zero logic -- no conditionals based on data, no business decisions:

```kotlin
// CORRECT: Composable is "dumb" -- renders state, fires intents
@Composable
fun LoginScreen(
    state: LoginUiState,
    onLogin: (String, String) -> Unit
) {
    when (state) {
        is LoginUiState.Idle -> LoginForm(onLogin = onLogin)
        is LoginUiState.Loading -> CircularProgressIndicator()
        is LoginUiState.Success -> WelcomeMessage(state.user)
        is LoginUiState.Error -> ErrorBanner(state.message)
    }
}

// WRONG: Logic in Composable (book's anti-pattern)
@Composable
fun LoginScreen(viewModel: LoginAndroidViewModel) {
    val user = viewModel.user
    if (user != null && user.isAdmin) {       // Business logic leaked into view
        AdminDashboard()
    } else if (user?.isExpired == true) {      // More logic in view
        RenewalScreen()
    }
}
```

---

## 2. MV* Patterns on Android

### MVVM with Kotlin Flow/StateFlow

The dominant Android pattern. Maps directly to the book's recommended starting point (MVA):

```kotlin
// ViewModel exposes StateFlow (single source of truth for screen state)
class ProfileViewModel(
    private val getProfileUseCase: GetProfileUseCase
) : ViewModel() {

    private val _state = MutableStateFlow<ProfileState>(ProfileState.Loading)
    val state: StateFlow<ProfileState> = _state.asStateFlow()

    fun loadProfile(userId: String) {
        viewModelScope.launch {
            getProfileUseCase(userId)
                .onSuccess { _state.value = ProfileState.Loaded(it) }
                .onFailure { _state.value = ProfileState.Error(it.message) }
        }
    }
}

// View collects state
@Composable
fun ProfileScreen(viewModel: ProfileViewModel = koinViewModel()) {
    val state by viewModel.state.collectAsStateWithLifecycle()
    ProfileContent(state = state)
}
```

### MVI with Sealed Classes

The book's recommendation for complex applications:

```kotlin
// State -- sealed class with ALL possible screen states
sealed class OrderState {
    data object Loading : OrderState()
    data class Loaded(val orders: List<Order>, val filter: OrderFilter) : OrderState()
    data class Error(val message: String) : OrderState()
    data object Empty : OrderState()
}

// Intent -- sealed class with ALL user actions
sealed class OrderIntent {
    data object LoadOrders : OrderIntent()
    data class FilterBy(val filter: OrderFilter) : OrderIntent()
    data class DeleteOrder(val orderId: String) : OrderIntent()
    data object Retry : OrderIntent()
}

// Reducer -- pure function, easy to test
fun reduce(currentState: OrderState, intent: OrderIntent): OrderState {
    return when (intent) {
        is OrderIntent.LoadOrders -> OrderState.Loading
        is OrderIntent.FilterBy -> {
            if (currentState is OrderState.Loaded) {
                currentState.copy(filter = intent.filter)
            } else currentState
        }
        // ...
    }
}

// ViewModel wires it together
class OrderViewModel(
    private val getOrdersUseCase: GetOrdersUseCase,
    private val deleteOrderUseCase: DeleteOrderUseCase
) : ViewModel() {

    private val _state = MutableStateFlow<OrderState>(OrderState.Loading)
    val state: StateFlow<OrderState> = _state.asStateFlow()

    fun dispatch(intent: OrderIntent) {
        viewModelScope.launch {
            _state.value = reduce(_state.value, intent)
            // Side effects handled here (the book skips unidirectional streams
            // for driving->driven direction, uses function calls instead)
            when (intent) {
                is OrderIntent.LoadOrders -> loadOrders()
                is OrderIntent.DeleteOrder -> deleteOrder(intent.orderId)
                else -> Unit
            }
        }
    }
}
```

### MVI Frameworks

- **Orbit MVI**: Lightweight, Kotlin-first. Aligns well with the book's "skip unidirectional for driving->driven" approach.
- **MVIKotlin**: Multiplatform (KMP-compatible), more opinionated. Full unidirectional data flow.

### Google's Official Architecture vs. the Book

| Aspect | Google's Guide | The Book |
|---|---|---|
| Layers | UI -> Domain (optional) -> Data | Framework -> Interface Adapters -> Application -> Domain |
| ViewModel role | Contains UI logic + calls repositories | Framework wrapper; logic lives in Use Cases |
| Domain layer | "Optional" | Always present (even if thin) |
| Repository | Data layer entry point | Interface Adapters or Data layer |
| DI | Hilt (first choice) | Koin (preferred for module independence) |
| Boundary enforcement | Convention-based | Gradle modules for complex apps |

Both agree on: unidirectional data flow, separation of concerns, reactive streams, immutable UI state.

### XML-based MVC: The Book's Anti-Pattern

Activity-as-controller (classic Android MVC) violates every principle the book cares about:
- Activity handles UI, navigation, business logic, and data fetching (zero SCP)
- Tightly coupled to Android framework (untestable without emulator)
- No separation between presentation and business logic
- The book explicitly states: "Better alternatives exist for every scenario"

---

## 3. SOLID on Android

### SRP / SCP: Single Concern Principle

Each class belongs to exactly one category:

```kotlin
// DATA CONTAINER -- holds data, nothing else
data class User(
    val id: String,
    val name: String,
    val email: String,
    val tier: UserTier
)

// EXECUTOR -- applies logic or performs a task (Use Case)
class ValidateEmailUseCase {
    operator fun invoke(email: String): Result<String> {
        return if (email.matches(EMAIL_REGEX)) Result.success(email)
        else Result.failure(InvalidEmailException())
    }
}

// ORCHESTRATOR -- delegates to executors, never contains logic itself
class RegistrationViewModel(
    private val validateEmail: ValidateEmailUseCase,
    private val registerUser: RegisterUserUseCase,
    private val sendWelcomeEmail: SendWelcomeEmailUseCase
) : ViewModel() {
    // Orchestrates flow, does NOT contain validation/registration logic
}
```

**Key rules**:
- 1 ViewModel per screen
- 1 Repository per entity
- data classes as Data Containers (no logic methods beyond what Kotlin generates)

### OCP: Open-Closed Principle

Sealed classes + `when` expressions for type-safe Strategy pattern:

```kotlin
// Sealed class defines the type hierarchy
sealed class PaymentMethod {
    data class CreditCard(val number: String, val cvv: String) : PaymentMethod()
    data class BankTransfer(val iban: String) : PaymentMethod()
    data class Crypto(val walletAddress: String) : PaymentMethod()
}

// Strategy interface -- open for extension
fun interface PaymentProcessor {
    suspend fun process(amount: Money): PaymentResult
}

// Factory selects strategy -- adding a new payment method means adding a new
// class + implementation, NOT modifying existing code
class PaymentProcessorFactory(
    private val creditCardProcessor: CreditCardProcessor,
    private val bankTransferProcessor: BankTransferProcessor,
    private val cryptoProcessor: CryptoProcessor
) {
    fun create(method: PaymentMethod): PaymentProcessor = when (method) {
        is PaymentMethod.CreditCard -> creditCardProcessor
        is PaymentMethod.BankTransfer -> bankTransferProcessor
        is PaymentMethod.Crypto -> cryptoProcessor
        // Kotlin compiler forces handling new sealed subclasses
    }
}
```

### LSP: Liskov Substitution Principle

The book recommends **Kotlin interfaces over abstract classes**:

```kotlin
// PREFERRED: Interface -- no inherited state, clean contract
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    suspend fun saveUser(user: User): Result<Unit>
}

// Implementations are freely substitutable
class RemoteUserRepository(private val api: UserApi) : UserRepository {
    override suspend fun getUser(id: String) = runCatching { api.fetchUser(id).toDomain() }
    override suspend fun saveUser(user: User) = runCatching { api.updateUser(user.toApi()) }
}

class CachedUserRepository(
    private val remote: UserRepository,
    private val cache: UserCache
) : UserRepository {
    override suspend fun getUser(id: String): Result<User> {
        return cache.get(id) ?: remote.getUser(id).also { it.onSuccess { user -> cache.put(user) } }
    }
    override suspend fun saveUser(user: User) = remote.saveUser(user)
}
```

**Why interfaces over abstract classes**: No inherited state = no side effects = easier substitution = fewer LSP violations.

### ISP: Interface Segregation Principle

Kotlin interface delegation (`by` keyword) as ISP enabler:

```kotlin
// SEGREGATED interfaces -- clients depend only on what they need
interface Readable<T> {
    suspend fun getById(id: String): Result<T>
    suspend fun getAll(): Result<List<T>>
}

interface Writable<T> {
    suspend fun save(item: T): Result<Unit>
    suspend fun delete(id: String): Result<Unit>
}

// Implementation composes both
class ProductRepository(
    private val reader: ProductReader,
    private val writer: ProductWriter
) : Readable<Product> by reader, Writable<Product> by writer
// Use Case that only reads never sees write methods

// Use Case depends on narrow interface
class GetProductsUseCase(private val products: Readable<Product>) {
    suspend operator fun invoke() = products.getAll()
}
```

### DIP: Dependency Inversion Principle

`fun interface` (SAM) enables lightweight abstractions:

```kotlin
// fun interface = SAM = single abstract method = minimal abstraction cost
fun interface GetUserById {
    suspend operator fun invoke(id: String): Result<User>
}

// Can be implemented as lambda -- zero boilerplate
val getUserById: GetUserById = GetUserById { id ->
    api.fetchUser(id).toDomain()
}

// Or as full class when complexity warrants it
class GetUserByIdImpl(
    private val repository: UserRepository
) : GetUserById {
    override suspend fun invoke(id: String): Result<User> {
        return repository.getUser(id)
    }
}
```

**Interface-based architecture**: Inner layers define interfaces. Outer layers implement them. Dependency arrows always point inward.

---

## 4. Reactive Programming on Android

### Kotlin Flow Taxonomy

```kotlin
// COLD STREAM -- flow {} -- emits only when collected, each collector gets fresh data
fun observePrices(): Flow<Price> = flow {
    while (true) {
        emit(api.getLatestPrice())
        delay(5000)
    }
}

// HOT STREAM -- StateFlow -- always has a value, shares latest state with all collectors
// Maps to the book's "single state stream" in MVI
private val _uiState = MutableStateFlow(DashboardState())
val uiState: StateFlow<DashboardState> = _uiState.asStateFlow()

// HOT STREAM -- SharedFlow -- no initial value, broadcasts events
// Maps to the book's hot streams for one-shot events
private val _events = MutableSharedFlow<UiEvent>()
val events: SharedFlow<UiEvent> = _events.asSharedFlow()
```

### StateFlow for UI State (MVI Single State Stream)

```kotlin
// Single state object holds ALL screen information
data class CheckoutState(
    val items: List<CartItem> = emptyList(),
    val total: Money = Money.ZERO,
    val isLoading: Boolean = false,
    val error: String? = null,
    val promoApplied: Boolean = false
)

class CheckoutViewModel(/*...*/) : ViewModel() {
    private val _state = MutableStateFlow(CheckoutState())
    val state: StateFlow<CheckoutState> = _state.asStateFlow()

    fun applyPromo(code: String) {
        _state.update { it.copy(isLoading = true) } // Immutable update via copy()
        viewModelScope.launch {
            applyPromoUseCase(code)
                .onSuccess { discount ->
                    _state.update { it.copy(
                        total = it.total - discount,
                        promoApplied = true,
                        isLoading = false
                    )}
                }
                .onFailure { e ->
                    _state.update { it.copy(error = e.message, isLoading = false) }
                }
        }
    }
}
```

### SharedFlow for One-Shot Events

```kotlin
// Events that should NOT be replayed (navigation, toasts, snackbars)
sealed class UiEvent {
    data class ShowSnackbar(val message: String) : UiEvent()
    data class Navigate(val route: String) : UiEvent()
    data object GoBack : UiEvent()
}

// In ViewModel
private val _events = MutableSharedFlow<UiEvent>() // replay = 0 by default
val events: SharedFlow<UiEvent> = _events.asSharedFlow()

// In Composable
LaunchedEffect(Unit) {
    viewModel.events.collect { event ->
        when (event) {
            is UiEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
            is UiEvent.Navigate -> navController.navigate(event.route)
            is UiEvent.GoBack -> navController.popBackStack()
        }
    }
}
```

### Coroutines + Suspend Functions

The async/await equivalent in Kotlin:

```kotlin
// suspend = the book's async operation building block
suspend fun syncUserData(userId: String): Result<Unit> = coroutineScope {
    val profileDeferred = async { profileRepository.fetch(userId) }
    val settingsDeferred = async { settingsRepository.fetch(userId) }

    val profile = profileDeferred.await().getOrThrow()
    val settings = settingsDeferred.await().getOrThrow()

    localStore.save(profile, settings)
    Result.success(Unit)
}
```

### The Book's Immutability Rule

```kotlin
// data class + copy() = immutable updates (mandatory in reactive contexts)
data class CartItem(val productId: String, val quantity: Int, val price: Money)

// Sealed classes for exhaustive state modeling
sealed class ScreenState<out T> {
    data object Loading : ScreenState<Nothing>()
    data class Success<T>(val data: T) : ScreenState<T>()
    data class Error(val message: String) : ScreenState<Nothing>()
}

// NEVER mutate -- always create new instances
fun updateQuantity(items: List<CartItem>, productId: String, newQty: Int): List<CartItem> {
    return items.map { item ->
        if (item.productId == productId) item.copy(quantity = newQty) else item
    }
}
```

### Flow Operators -- Use With Restraint

```kotlin
// The book warns: powerful but hurts readability when chained excessively

// GOOD -- simple, readable transformation chain
val activeUsers: Flow<List<User>> = usersFlow
    .map { users -> users.filter { it.isActive } }

// GOOD -- combining two state sources
val screenState: StateFlow<ScreenState> = combine(
    userFlow,
    settingsFlow
) { user, settings ->
    ScreenState(user = user, darkMode = settings.darkMode)
}.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), ScreenState())

// GOOD -- flatMapLatest for search-as-you-type (cancels previous query)
val searchResults: Flow<List<Product>> = searchQuery
    .flatMapLatest { query -> searchUseCase(query) }

// BAD -- too many chained operators (the book's warning)
val result = sourceFlow
    .map { transform1(it) }
    .filter { it.isValid }
    .flatMapLatest { fetchDetails(it) }
    .map { transform2(it) }
    .combine(otherFlow) { a, b -> merge(a, b) }
    .map { transform3(it) }
    .distinctUntilChanged()
    // 7+ operators = hard to read, hard to debug. Break into intermediate Flows.
```

---

## 5. Dependency Injection on Android

### Koin (Book's Recommendation)

```kotlin
// Feature module -- self-contained, no annotation processing
val loginModule = module {
    factory { LoginValidator() }
    factory { LoginUseCase(get(), get()) }
    viewModel { LoginViewModel(get()) }
}

val networkModule = module {
    single { provideOkHttpClient() }
    single { provideRetrofit(get()) }
    single<UserApi> { get<Retrofit>().create(UserApi::class.java) }
}

// Application setup
class App : Application() {
    override fun onCreate() {
        super.onCreate()
        startKoin {
            androidContext(this@App)
            modules(networkModule, loginModule, profileModule)
        }
    }
}
```

**Why the book prefers Koin**:
- Works cleanly with feature module segregation (no annotation processing across modules)
- Simple DSL -- lower learning curve
- Runtime resolution = no generated code, no build time hit
- Aligns with MVA (minimum viable architecture) philosophy

### Hilt (Google's Recommendation)

```kotlin
@HiltAndroidApp
class App : Application()

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}

@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val getProfileUseCase: GetProfileUseCase
) : ViewModel() { /* ... */ }
```

**Book's concern with Hilt**: Annotation processing requires the domain/application modules to depend on Hilt (a framework library), which violates the Dependency Rule for inner layers. Workarounds exist (manual `@Provides`) but add friction.

### Dagger 2

Powerful but the book considers it overcomplicated for most mobile apps. The learning curve violates the "simplify complex code, don't complexify simple code" principle for small-to-medium teams.

### Manual DI with Factory Pattern

```kotlin
// Acceptable for small apps (MVA starting point)
class AppContainer(context: Context) {
    private val retrofit = Retrofit.Builder().baseUrl(BASE_URL).build()
    private val userApi = retrofit.create(UserApi::class.java)
    private val userRepository: UserRepository = RemoteUserRepository(userApi)
    val getUserUseCase = GetUserUseCase(userRepository)
}

// In Activity/Fragment
val viewModel = LoginViewModel(appContainer.getUserUseCase)
```

---

## 6. Testing on Android

### The Book's Testing Pyramid Mapped to Android

```
        /  E2E  \              Espresso / Compose UI Tests
       /----------\            (~10%)
      / Integration \          Robolectric
     /----------------\        (~20%)
    /     Unit Tests    \      JUnit 5 + MockK + Turbine
   /----------------------\    (~70%)
```

### Unit Tests: JUnit + MockK

```kotlin
// Testing a Use Case (pure Kotlin, no Android dependencies)
class GetUserUseCaseTest {

    private val repository = mockk<UserRepository>()
    private val useCase = GetUserUseCase(repository)

    @Test
    fun `returns user when repository succeeds`() = runTest {
        // Arrange
        val expected = User(id = "1", name = "Alice", email = "alice@test.com")
        coEvery { repository.getUser("1") } returns Result.success(expected)

        // Act
        val result = useCase("1")

        // Assert
        assertEquals(Result.success(expected), result)
    }

    @Test
    fun `returns failure when repository fails`() = runTest {
        coEvery { repository.getUser("1") } returns Result.failure(IOException("Network error"))

        val result = useCase("1")

        assertTrue(result.isFailure)
    }
}
```

### Testing Kotlin Flow with Turbine

```kotlin
// Turbine makes Flow testing readable and deterministic
class OrderViewModelTest {

    @Test
    fun `emits loading then success when orders load`() = runTest {
        val useCase = mockk<GetOrdersUseCase>()
        coEvery { useCase() } returns Result.success(listOf(testOrder))

        val viewModel = OrderViewModel(useCase)

        viewModel.state.test {
            assertEquals(OrderState.Loading, awaitItem())       // Initial state

            viewModel.dispatch(OrderIntent.LoadOrders)

            assertEquals(OrderState.Loading, awaitItem())       // Loading emitted
            assertEquals(
                OrderState.Loaded(listOf(testOrder), OrderFilter.All),
                awaitItem()                                     // Success emitted
            )
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### Robolectric for Integration Tests

Tests that need Android context without an emulator:

```kotlin
@RunWith(RobolectricTestRunner::class)
class RoomUserDaoTest {

    private lateinit var db: AppDatabase
    private lateinit var dao: UserDao

    @Before
    fun setup() {
        db = Room.inMemoryDatabaseBuilder(
            ApplicationProvider.getApplicationContext(),
            AppDatabase::class.java
        ).build()
        dao = db.userDao()
    }

    @Test
    fun `insert and retrieve user`() = runTest {
        dao.insert(UserEntity(id = "1", name = "Alice"))
        val result = dao.getById("1")
        assertEquals("Alice", result?.name)
    }
}
```

### Compose UI Testing

```kotlin
@get:Rule
val composeRule = createComposeRule()

@Test
fun `shows error message when state is error`() {
    composeRule.setContent {
        LoginScreen(
            state = LoginUiState.Error("Invalid credentials"),
            onLogin = { _, _ -> }
        )
    }

    composeRule.onNodeWithText("Invalid credentials").assertIsDisplayed()
}
```

### Espresso for Legacy XML UI

```kotlin
@Test
fun `clicking login button shows progress`() {
    onView(withId(R.id.loginButton)).perform(click())
    onView(withId(R.id.progressBar)).check(matches(isDisplayed()))
}
```

### Enterprise vs. Startup Testing (from the book)

| Aspect | Enterprise | Startup |
|---|---|---|
| Unit tests | White-box, mock all deps, test each layer | Black-box, ViewModel to Service, mock only boundaries |
| Integration | Robolectric for DB + framework-adjacent code | Optional |
| E2E | Espresso/Compose for regression | Skip |
| Mocking | Extensive (MockK) | Minimal (real objects where possible) |
| Test location | Inside each Gradle module | Single test source set |

---

## 7. Boundaries & Modules on Android

### Gradle Modules as Hard Boundaries

The book's recommendation for complex apps:

```
project/
  :app                    # Framework entry point (Application, Activities, DI setup)
  :core                   # Shared utilities, base classes
  :domain                 # Entities, repository interfaces, business rules (ZERO Android deps)
  :data                   # Repository implementations, API models, Room entities, mappers
  :feature:login          # Login feature (own ViewModel, Use Cases, UI)
  :feature:profile        # Profile feature
  :feature:orders         # Orders feature
```

### build.gradle Dependency Declarations = Boundary Contracts

```kotlin
// :domain/build.gradle.kts -- ZERO framework dependencies
plugins { id("kotlin") } // Pure Kotlin module, NOT Android library

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.3")
    // NO Android, NO Room, NO Retrofit, NO Hilt
}

// :data/build.gradle.kts -- depends inward on domain
plugins { id("com.android.library") }

dependencies {
    implementation(project(":domain"))
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("androidx.room:room-ktx:2.6.0")
    // Can NOT depend on :feature:* or :app
}

// :feature:login/build.gradle.kts
dependencies {
    implementation(project(":domain"))
    implementation(project(":data"))    // Or inject via DI
    implementation(project(":core"))
    // Can NOT depend on :feature:profile (features don't cross-depend)
}

// :app/build.gradle.kts -- wires everything together
dependencies {
    implementation(project(":domain"))
    implementation(project(":data"))
    implementation(project(":feature:login"))
    implementation(project(":feature:profile"))
    implementation(project(":core"))
}
```

### The :app Module as Framework Entry Point

`:app` is the outermost layer:
- `Application` class with DI initialization
- `MainActivity` (single Activity in single-Activity architecture)
- Navigation graph
- DI module wiring (Koin `startKoin` or Hilt `@HiltAndroidApp`)

### Dynamic Feature Modules

For large apps (50+ developers), the book's "microservices" equivalent:

```kotlin
// Dynamic feature module -- downloaded on demand
// Enforces hard boundaries at the Play Store delivery level
dynamicFeatures = [":feature:premium", ":feature:analytics"]
```

### Boundary Strategy by Complexity (from the book)

| Complexity | Android Implementation |
|---|---|
| Medium | Single module, feature packages + layer packages (soft boundaries) |
| Complex / Enterprise | Feature Gradle modules + layer packages within each |
| Very Large (50+ devs) | Dynamic feature modules, each team owns a module |

---

## 8. Common Android Anti-Patterns (from the Book's Perspective)

### God Activity / God Fragment

```kotlin
// ANTI-PATTERN: Activity handles everything
class OrderActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        // Fetches data from API directly
        // Validates input
        // Transforms data for display
        // Handles navigation
        // Manages state
        // 800+ lines of code
    }
}
```
Violates: SCP (multiple concerns), low cohesion, high coupling, untestable.

### Business Logic in ViewModel or Activity

```kotlin
// ANTI-PATTERN: Business logic in ViewModel
class OrderViewModel : ViewModel() {
    fun calculateDiscount(order: Order): Money {
        // This is domain logic -- belongs in a Use Case or Domain entity
        return if (order.total > Money(100) && order.user.tier == UserTier.PREMIUM) {
            order.total * 0.15
        } else {
            Money.ZERO
        }
    }
}
```

### Room @Entity Leaking into Domain Layer

```kotlin
// ANTI-PATTERN: Domain layer uses Room entity directly
@Entity(tableName = "users")  // Framework annotation on domain model
data class User(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "user_name") val name: String
)

// CORRECT: Separate models with mappers
// Domain layer
data class User(val id: String, val name: String)

// Data layer
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "user_name") val name: String
) {
    fun toDomain() = User(id = id, name = name)
}
```

### Using Android ViewModel for Business Logic Instead of Use Cases

```kotlin
// ANTI-PATTERN: ViewModel IS the use case
class CheckoutViewModel @Inject constructor(
    private val cartRepository: CartRepository,
    private val paymentApi: PaymentApi,
    private val shippingService: ShippingService,
    private val taxCalculator: TaxCalculator,
    private val promoValidator: PromoValidator
) : ViewModel() {
    // 500 lines of business logic -- ViewModel is an Orchestrator,
    // not an Executor. This logic belongs in Use Cases.
}

// CORRECT: ViewModel orchestrates, Use Cases execute
class CheckoutViewModel(
    private val calculateTotals: CalculateTotalsUseCase,
    private val processPayment: ProcessPaymentUseCase,
    private val applyPromo: ApplyPromoUseCase
) : ViewModel() { /* delegates to use cases */ }
```

### Hilt Modules in Domain Layer

```kotlin
// ANTI-PATTERN: Domain module depends on Hilt (framework in inner layer)
// :domain/build.gradle.kts
dependencies {
    implementation("com.google.dagger:hilt-core:2.48")  // VIOLATION of Dependency Rule
}

// Domain class with Hilt annotation
class GetUserUseCase @Inject constructor(  // @Inject is a framework annotation
    private val repository: UserRepository
)
```

The book's recommendation: Domain layer should have zero framework dependencies. DI wiring happens in the outermost layer.

### LiveData in Domain/Data Layers

```kotlin
// ANTI-PATTERN: LiveData (Android framework class) in domain interface
interface UserRepository {
    fun getUser(id: String): LiveData<User>  // Framework type crossing boundary
}

// CORRECT: Use Kotlin Flow (platform-agnostic) or suspend functions
interface UserRepository {
    suspend fun getUser(id: String): Result<User>
    fun observeUser(id: String): Flow<User>
}
```

---

## 9. Model Transformation Chain on Android

### Enterprise (Complex App)

```
API (JSON)
  |
  v
ApiModel (@SerializedName / @Json)     -- Retrofit/Gson or Retrofit/Moshi
  |  mapper
  v
RoomEntity (@Entity, @PrimaryKey)      -- Room database model
  |  mapper
  v
PlainModel (data class)                -- No annotations, no framework deps
  |  mapper
  v
DomainModel (data class)               -- Business rules, domain logic
  |  mapper
  v
UiModel (data class)                   -- Formatted for display (strings, colors, visibility)
```

```kotlin
// API Model -- outermost, coupled to JSON structure
data class UserApiModel(
    @SerializedName("user_id") val userId: String,
    @SerializedName("full_name") val fullName: String,
    @SerializedName("account_type") val accountType: Int,
    @SerializedName("created_at") val createdAt: String
)

// Room Entity -- coupled to database schema
@Entity(tableName = "users")
data class UserEntity(
    @PrimaryKey val id: String,
    @ColumnInfo(name = "full_name") val fullName: String,
    @ColumnInfo(name = "account_type") val accountType: Int,
    @ColumnInfo(name = "created_at") val createdAt: Long
)

// Plain Model -- no annotations
data class UserPlain(
    val id: String,
    val fullName: String,
    val accountType: Int,
    val createdAt: Long
)

// Domain Model -- business meaning
data class User(
    val id: UserId,
    val name: UserName,
    val tier: UserTier,
    val memberSince: Instant
)

// UI Model -- ready for rendering
data class UserUiModel(
    val displayName: String,
    val tierBadge: String,        // "Premium" / "Free"
    val tierColor: Color,         // Gold / Gray
    val memberSinceText: String   // "Member since Jan 2023"
)

// Mappers live in the layer that knows both models
fun UserApiModel.toEntity() = UserEntity(
    id = userId, fullName = fullName,
    accountType = accountType, createdAt = parseIso8601(createdAt)
)

fun UserEntity.toPlain() = UserPlain(
    id = id, fullName = fullName,
    accountType = accountType, createdAt = createdAt
)

fun UserPlain.toDomain() = User(
    id = UserId(id), name = UserName(fullName),
    tier = UserTier.fromCode(accountType),
    memberSince = Instant.ofEpochMilli(createdAt)
)

fun User.toUiModel() = UserUiModel(
    displayName = name.value,
    tierBadge = tier.displayName,
    tierColor = if (tier == UserTier.PREMIUM) Color.Gold else Color.Gray,
    memberSinceText = "Member since ${memberSince.formatMonthYear()}"
)
```

### Startup (Pragmatic)

```
API (JSON)
  |
  v
ApiModel (@Json -- Moshi preferred for Kotlin)
  |  mapper
  v
PlainModel (data class)               -- No annotations
  |  mapper
  v
Model (merged domain + presentation)  -- Single model, business + display
```

```kotlin
// API Model
@JsonClass(generateAdapter = true)
data class UserApiModel(
    @Json(name = "user_id") val userId: String,
    @Json(name = "full_name") val fullName: String,
    @Json(name = "account_type") val accountType: Int
)

// Plain Model (no framework annotations)
data class UserPlain(val id: String, val fullName: String, val accountType: Int)

// Merged Model -- pragmatic, fewer mappers, faster to ship
data class User(
    val id: String,
    val name: String,
    val tier: UserTier,
    val tierDisplayName: String   // Presentation concern merged in
)

fun UserApiModel.toPlain() = UserPlain(id = userId, fullName = fullName, accountType = accountType)

fun UserPlain.toModel() = User(
    id = id, name = fullName,
    tier = UserTier.fromCode(accountType),
    tierDisplayName = UserTier.fromCode(accountType).displayName
)
```

**The book's guidance**: Start with the startup chain. Add layers (Room entities, separate UI models) only when complexity demands it -- preparatory refactoring, not speculative architecture.

---

## Quick Reference: Layer Placement Cheat Sheet

| Component | Layer | Rationale |
|---|---|---|
| `Activity` / `Fragment` | Framework | Android SDK class |
| `@Composable` | Framework | Jetpack Compose (framework) |
| `androidx.lifecycle.ViewModel` | Framework | Android lifecycle class |
| Navigation (`NavController`) | Framework | Jetpack Navigation |
| Platform-agnostic ViewModel | Interface Adapters | Pure Kotlin, no Android imports |
| Presenter | Interface Adapters | Transforms data for presentation |
| Mapper (UI) | Interface Adapters | Converts domain -> UI model |
| DataGateway implementation | Interface Adapters | Implements domain-defined interface |
| Repository implementation | Interface Adapters / Data | Concrete data access |
| Use Case | Application | Orchestrates business operations |
| Domain Entity | Domain | Core business data + rules |
| Repository interface | Domain | Defines data contract |
| Business rule function | Domain | Pure logic, zero dependencies |
| Room `@Dao` / `@Entity` | Framework | Room is a framework library |
| Retrofit `@GET` / `@POST` | Framework | Retrofit is a framework library |
| Koin `module {}` | Framework | DI configuration |
| Hilt `@Module` | Framework | DI configuration |

## Quick Reference: Android Library Choices (2025-2026 Community Consensus)

| Concern | Recommended | Alternatives | Notes |
|---|---|---|---|
| DI | Hilt (Google's official) | Koin (simpler, better for KMP) | Hilt dominates Android-only. Koin for KMP or simpler projects. Book recommends Koin. |
| Networking | Retrofit + OkHttp | Ktor (KMP-compatible) | Always behind Repository interface |
| Persistence | Room (Jetpack) | SQLDelight (KMP), DataStore | Room @Entity stays in Framework layer, never leaks to domain |
| Reactive | Kotlin Flow + Coroutines | RxJava (legacy only) | StateFlow for UI state. SharedFlow for events. |
| Navigation | Jetpack Navigation / Compose Navigation | Voyager, Appyx | Use official unless specific need |
| Testing | JUnit 5 + MockK + Turbine | Mockito-Kotlin (legacy) | MockK dominates Kotlin-first. Turbine is the standard for Flow testing. |
| UI Testing | Compose Testing | Espresso (View-based legacy) | Compose test rules are simpler than Espresso |
| Architecture | MVI/UDF + Compose (Google's direction) | MVVM + StateFlow | MVI with sealed classes is the dominant pattern. Google calls it "UDF". |
