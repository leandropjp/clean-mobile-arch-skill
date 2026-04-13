# Clean Mobile Architecture -- Flutter (Dart) Platform Reference

**Source**: "Clean Mobile Architecture" by Petros Efthymiou, mapped to Flutter/Dart conventions.

---

## 1. How Clean Architecture Maps to Flutter

### The Four Layers in Flutter

| Layer (inner to outer) | Flutter/Dart Implementation |
|---|---|
| **Domain** (center) | Plain Dart classes. Entities, value objects, business rules. Zero dependencies on Flutter SDK or any package. |
| **Application** (use cases) | Plain Dart classes orchestrating domain logic. No Flutter imports. |
| **Interface Adapters** | Cubits/Blocs (presentation side), Repository implementations, DataSource classes (data side). Platform-agnostic Dart code -- no `BuildContext`, no `Widget`. |
| **Frameworks & Drivers** (outermost) | `Widget` subclasses, `MaterialApp`, `Navigator`/`GoRouter`, `MethodChannel`/`EventChannel`, `http`/`dio`, `drift`/`hive`, `SharedPreferences`. |

### The Dependency Rule in Practice

```
lib/
  features/
    auth/
      domain/         # Zero imports from data/ or presentation/
        entities/
        repositories/  # Abstract classes (interfaces)
        usecases/
      data/            # Imports from domain/ only
        models/
        datasources/
        repositories/  # Concrete implementations
      presentation/    # Imports from domain/ only
        bloc/          # or cubit/, provider/, notifier/
        widgets/
        pages/
```

Domain defines abstract repository classes. Data implements them. Presentation depends on domain abstractions, never on data concretions. This is the Dependency Rule.

### Widgets as "Dumb" Views

The book mandates the Framework layer be **dumb** -- no logic, no conditionals based on data. In Flutter, this means Widgets should only describe UI:

```dart
// CORRECT: Widget is dumb, delegates everything to Bloc
class LoginPage extends StatelessWidget {
  const LoginPage({super.key});

  @override
  Widget build(BuildContext context) {
    return BlocBuilder<LoginCubit, LoginState>(
      builder: (context, state) => switch (state) {
        LoginInitial()  => const LoginForm(),
        LoginLoading()  => const CircularProgressIndicator(),
        LoginSuccess()  => const HomeRedirect(),
        LoginFailure(:final message) => LoginError(message: message),
      },
    );
  }
}

// WRONG: Business logic in Widget
class LoginPage extends StatefulWidget { /* ... */ }
class _LoginPageState extends State<LoginPage> {
  Future<void> _login() async {
    final response = await http.post(Uri.parse('/login'), body: credentials);
    if (response.statusCode == 200) { /* navigate */ }
    else { /* show error */ }
  }
}
```

### Flutter's Immutability and the Book's Rule

The book states: "In RP, immutable objects are mandatory -- mutable objects in async contexts cause race conditions and unpredicted state."

Flutter Widgets are immutable by design. Every `StatelessWidget` and `StatefulWidget` class is `@immutable`. When state changes, Flutter creates a **new widget tree** rather than mutating the old one. This is architecturally aligned with the book's immutability rule -- the framework itself enforces it at the view layer.

---

## 2. MV* Patterns on Flutter

### BLoC as MVI Implementation

BLoC maps directly to the book's MVI pattern -- the author's recommended approach for complex apps:

| MVI Concept | BLoC Equivalent |
|---|---|
| Intent-model (all user actions in one sealed class) | `sealed class LoginEvent` |
| State-model (all screen info in one class) | `sealed class LoginState` |
| Reducer (pure function assembling state) | `on<Event>` handlers inside Bloc |
| Single stream to View | `Stream<LoginState>` (one stream per Bloc) |

```dart
// Intent-model
sealed class LoginEvent {}
class LoginSubmitted extends LoginEvent {
  final String email;
  final String password;
  LoginSubmitted({required this.email, required this.password});
}
class LoginEmailChanged extends LoginEvent {
  final String email;
  LoginEmailChanged(this.email);
}

// State-model: ALL screen info in one class
sealed class LoginState {}
class LoginInitial extends LoginState {}
class LoginLoading extends LoginState {}
class LoginSuccess extends LoginState {
  final User user;
  LoginSuccess(this.user);
}
class LoginFailure extends LoginState {
  final String message;
  LoginFailure(this.message);
}

// Reducer: Bloc processes events into states
class LoginBloc extends Bloc<LoginEvent, LoginState> {
  final LoginUseCase _loginUseCase;

  LoginBloc(this._loginUseCase) : super(LoginInitial()) {
    on<LoginSubmitted>((event, emit) async {
      emit(LoginLoading());
      final result = await _loginUseCase(event.email, event.password);
      result.fold(
        (failure) => emit(LoginFailure(failure.message)),
        (user) => emit(LoginSuccess(user)),
      );
    });
  }
}
```

### Cubit as Simplified MVI

Cubit is a subset of Bloc without events. It still produces a single state stream, but uses direct method calls instead of an Intent-model. Good for simpler screens where the full event-driven approach is overkill (book's MVA principle -- don't complexify simple code):

```dart
class LoginCubit extends Cubit<LoginState> {
  final LoginUseCase _loginUseCase;
  LoginCubit(this._loginUseCase) : super(LoginInitial());

  Future<void> login(String email, String password) async {
    emit(LoginLoading());
    final result = await _loginUseCase(email, password);
    result.fold(
      (failure) => emit(LoginFailure(failure.message)),
      (user) => emit(LoginSuccess(user)),
    );
  }
}
```

### Riverpod as MVVM

Riverpod providers act as ViewModels with multiple observable state pieces. This maps to the book's MVVM pattern -- multiple streams, the View composes them:

```dart
// ViewModel-like provider
@riverpod
class LoginNotifier extends _$LoginNotifier {
  @override
  LoginState build() => LoginInitial();

  Future<void> login(String email, String password) async {
    state = LoginLoading();
    final useCase = ref.read(loginUseCaseProvider);
    final result = await useCase(email, password);
    state = result.fold(
      (failure) => LoginFailure(failure.message),
      (user) => LoginSuccess(user),
    );
  }
}

// View observes with ref.watch (reactive)
class LoginPage extends ConsumerWidget {
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final state = ref.watch(loginNotifierProvider);
    // render based on state
  }
}
```

### Provider (Simple MVVM-like)

`ChangeNotifier` + `Provider` is a simpler MVVM. Adequate for low-complexity apps (the book says MVVM is the "industry standard, good MVA starting point"):

```dart
class LoginViewModel extends ChangeNotifier {
  LoginState _state = LoginInitial();
  LoginState get state => _state;

  final LoginUseCase _loginUseCase;
  LoginViewModel(this._loginUseCase);

  Future<void> login(String email, String password) async {
    _state = LoginLoading();
    notifyListeners();
    // ...
  }
}
```

### GetX -- Why It Violates Book Principles

GetX is pragmatic but conflicts with multiple book principles:

1. **Violates SCP**: `GetxController` often mixes navigation, state, DI, and business logic in one class (multiple concerns).
2. **Violates DIP**: `Get.find<T>()` is a global service locator called anywhere -- domain/data layers become aware of the framework.
3. **Violates Dependency Rule**: Navigation via `Get.to()` from controllers couples presentation logic to the framework layer.
4. **Mutable state**: `Rx` observables use mutable `.value` assignment, conflicting with the book's immutability rule.
5. **Tight coupling**: The "all-in-one" philosophy means replacing GetX requires rewriting almost everything (the "spread cancer" problem).

### StatefulWidget + setState as MVC Anti-Pattern

The book says MVC is obsolete because the Controller is tightly coupled to the View and mixes presentation with business logic. `StatefulWidget` with complex `setState` is exactly this anti-pattern:

```dart
// This IS the MVC anti-pattern: Widget = View + Controller
class _LoginPageState extends State<LoginPage> {
  bool _isLoading = false;
  String? _error;

  Future<void> _login() async {
    setState(() => _isLoading = true);
    try {
      final response = await http.post(/*...*/); // data layer concern
      if (response.statusCode == 200) {
        final user = User.fromJson(/*...*/);      // domain concern
        Navigator.pushNamed(context, '/home');     // navigation concern
      }
    } catch (e) {
      setState(() => _error = e.toString());       // presentation concern
    }
    setState(() => _isLoading = false);
  }
}
```

Four different concerns in one class. Untestable without a running Flutter environment.

---

## 3. SOLID on Flutter

### SRP / SCP

**1 Bloc/Cubit per screen. 1 Repository per entity. Widgets for presentation only. Use Cases as executors.**

The book defines three class categories. Every Dart class should be exactly one:

| Category | Flutter Examples |
|---|---|
| **Executor** | Use Cases, Mappers, Validators |
| **Delegator/Orchestrator** | Bloc/Cubit (delegates to Use Cases), Repository (delegates to DataSources) |
| **Data Container** | Entity, State, Event, DTO/JsonModel |

```dart
// Executor: does one thing
class ValidateEmailUseCase {
  bool call(String email) => RegExp(r'^[\w-\.]+@[\w-\.]+\.\w+$').hasMatch(email);
}

// Delegator: orchestrates, no logic of its own
class LoginCubit extends Cubit<LoginState> {
  final LoginUseCase _login;
  final ValidateEmailUseCase _validateEmail;
  // ...
}

// Data Container: holds data, nothing else
@freezed
class User with _$User {
  const factory User({required String id, required String name}) = _User;
}
```

### OCP: Open-Closed Principle

Dart 3 sealed classes enable type-safe extension without modifying existing code:

```dart
// Sealed class for exhaustive pattern matching (compiler enforces completeness)
sealed class PaymentMethod {}
class CreditCard extends PaymentMethod { /* ... */ }
class BankTransfer extends PaymentMethod { /* ... */ }
class Pix extends PaymentMethod { /* ... */ }

// Adding a new payment method = add a new class, compiler flags all
// unhandled cases. Existing code is never modified (OCP).

// Strategy pattern via abstract class + mixins
abstract class PaymentProcessor {
  Future<PaymentResult> process(Payment payment);
}

class CreditCardProcessor implements PaymentProcessor {
  @override
  Future<PaymentResult> process(Payment payment) async { /* ... */ }
}

// Mixins for cross-cutting extension
mixin Loggable {
  void log(String message) => debugPrint('[$runtimeType] $message');
}

class CreditCardProcessor with Loggable implements PaymentProcessor {
  @override
  Future<PaymentResult> process(Payment payment) async {
    log('Processing credit card payment');
    // ...
  }
}
```

### LSP: Liskov Substitution Principle

In Dart, every class is implicitly an interface. Prefer composition via abstract classes:

```dart
// Abstract class as interface
abstract class AuthRepository {
  Future<Either<Failure, User>> login(String email, String password);
  Future<void> logout();
}

// Any implementation must honor the contract
class FirebaseAuthRepository implements AuthRepository {
  @override
  Future<Either<Failure, User>> login(String email, String password) async {
    // MUST NOT be more restrictive on inputs (e.g., rejecting valid emails)
    // MUST NOT return types outside the declared return type
  }

  @override
  Future<void> logout() async { /* ... */ }
}
```

The book warns: "Unit test overridden functions in subclasses against all expected inputs." In Flutter, test every `implements`/`extends` against the parent's contract.

### ISP: Interface Segregation Principle

Keep abstract classes small. Use mixin composition to add capabilities:

```dart
// WRONG: Fat interface
abstract class UserRepository {
  Future<User> getUser(String id);
  Future<void> updateUser(User user);
  Future<void> deleteUser(String id);
  Future<List<User>> searchUsers(String query);
  Future<void> uploadAvatar(String userId, File file);
  Future<UserStats> getUserStats(String id);
}

// CORRECT: Segregated interfaces
abstract class UserReader {
  Future<User> getUser(String id);
}

abstract class UserWriter {
  Future<void> updateUser(User user);
  Future<void> deleteUser(String id);
}

abstract class UserSearch {
  Future<List<User>> searchUsers(String query);
}

// Implementation composes only what it needs
class UserRepositoryImpl implements UserReader, UserWriter {
  // Only implements read + write, not search or avatar
}
```

### DIP: Dependency Inversion Principle

Abstract class-based injection. High-level modules define the interface; low-level modules implement it:

```dart
// Domain layer defines the interface (high-level)
abstract class AuthRepository {
  Future<Either<Failure, User>> login(String email, String password);
}

// Data layer implements it (low-level)
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource _remote;
  final AuthLocalDataSource _local;

  AuthRepositoryImpl(this._remote, this._local);

  @override
  Future<Either<Failure, User>> login(String email, String password) async {
    try {
      final model = await _remote.login(email, password);
      await _local.cacheUser(model);
      return Right(model.toDomain());
    } on ServerException catch (e) {
      return Left(ServerFailure(e.message));
    }
  }
}

// Use Case depends on abstraction, not concrete repository
class LoginUseCase {
  final AuthRepository _repository; // abstract class, not AuthRepositoryImpl
  LoginUseCase(this._repository);

  Future<Either<Failure, User>> call(String email, String password) {
    return _repository.login(email, password);
  }
}
```

---

## 4. Reactive Programming on Flutter

### Dart Streams

```dart
// Stream<T>: cold by default (new subscription = new execution)
Stream<int> countStream(int max) async* {
  for (int i = 0; i < max; i++) {
    yield i;
  }
}

// StreamController: hot stream (shared, broadcast)
final controller = StreamController<String>.broadcast();
controller.sink.add('event');
controller.stream.listen((data) => print(data));
```

### BLoC as RP Implementation

BLoC is literally "Business Logic Component" built on streams:

```
Events (Stream in) --> Bloc (transforms) --> States (Stream out)
```

The Bloc framework handles subscription, disposal, and error propagation. The View subscribes to the state stream via `BlocBuilder`/`BlocListener`.

### Riverpod's Reactive Observation

```dart
// ref.watch: rebuilds widget when provider state changes (reactive)
final state = ref.watch(loginNotifierProvider);

// ref.listen: side effects on state change (navigation, snackbar)
ref.listen(loginNotifierProvider, (prev, next) {
  if (next is LoginSuccess) {
    Navigator.pushNamed(context, '/home');
  }
});

// ref.read: one-time read (for event handlers, not build methods)
onPressed: () => ref.read(loginNotifierProvider.notifier).login(email, password),
```

### async/await with Future

`Future<T>` is the async primitive in Dart. Unlike Streams (multiple values over time), a Future resolves once:

```dart
Future<Either<Failure, User>> login(String email, String password) async {
  try {
    final response = await _client.post('/login', body: {'email': email, 'password': password});
    return Right(UserModel.fromJson(response.data).toDomain());
  } on DioException catch (e) {
    return Left(ServerFailure(e.message));
  }
}
```

### Immutability in Dart

The book mandates immutable state. Dart supports this with:

```dart
// @immutable annotation (compile-time warning if fields aren't final)
@immutable
class User {
  final String id;
  final String name;
  const User({required this.id, required this.name});
}

// freezed package: generates immutable data classes with copyWith
@freezed
class LoginState with _$LoginState {
  const factory LoginState.initial() = _Initial;
  const factory LoginState.loading() = _Loading;
  const factory LoginState.success(User user) = _Success;
  const factory LoginState.failure(String message) = _Failure;
}

// Creating new state = new instance, never mutation
emit(state.copyWith(isLoading: true)); // new object
```

### RxDart for Advanced Stream Operations

The book warns: "Avoid chaining multiple operators -- powerful but hurts readability."

```dart
// Acceptable: simple debounce for search
final searchResults = searchQueryStream
    .debounceTime(const Duration(milliseconds: 300))
    .switchMap((query) => _searchUseCase(query));

// Book caution: don't over-chain
// BAD -- hard to read, hard to debug, hard to test
final result = stream
    .debounce(/*...*/)
    .where(/*...*/)
    .map(/*...*/)
    .switchMap(/*...*/)
    .combineLatest(/*...*/)
    .distinctUntilChanged()
    .doOnData(/*...*/);
```

---

## 5. Dependency Injection on Flutter

### get_it + injectable (Service Locator Pattern)

Most popular DI approach. `injectable` generates the registration code:

```dart
// Setup (generated by injectable)
@InjectableInit()
void configureDependencies() => getIt.init();

// Registration
@module
abstract class AppModule {
  @lazySingleton
  Dio get dio => Dio(BaseOptions(baseUrl: 'https://api.example.com'));
}

@injectable
class LoginUseCase {
  final AuthRepository _repository;
  LoginUseCase(this._repository); // constructor injection
}

@LazySingleton(as: AuthRepository) // binds to abstract class
class AuthRepositoryImpl implements AuthRepository {
  final AuthRemoteDataSource _remote;
  AuthRepositoryImpl(this._remote);
}

// Usage (service locator call -- keep this at the Framework layer ONLY)
final loginUseCase = getIt<LoginUseCase>();
```

### Riverpod as DI

Providers ARE dependency injection. No separate DI container needed:

```dart
@riverpod
Dio dio(Ref ref) => Dio(BaseOptions(baseUrl: 'https://api.example.com'));

@riverpod
AuthRepository authRepository(Ref ref) =>
    AuthRepositoryImpl(remote: ref.watch(authRemoteDataSourceProvider));

@riverpod
LoginUseCase loginUseCase(Ref ref) =>
    LoginUseCase(ref.watch(authRepositoryProvider));
```

### Manual DI via Constructor Injection

The simplest form. No library needed. Good for small apps (MVA principle):

```dart
void main() {
  final dio = Dio();
  final remoteDataSource = AuthRemoteDataSource(dio);
  final repository = AuthRepositoryImpl(remoteDataSource);
  final loginUseCase = LoginUseCase(repository);
  final loginCubit = LoginCubit(loginUseCase);

  runApp(MyApp(loginCubit: loginCubit));
}
```

### Provider as Widget-Tree DI

Uses the widget tree for scoping. Simple but limited to the widget lifecycle:

```dart
MultiProvider(
  providers: [
    Provider<AuthRepository>(create: (_) => AuthRepositoryImpl(/*...*/)),
    ProxyProvider<AuthRepository, LoginUseCase>(
      update: (_, repo, __) => LoginUseCase(repo),
    ),
  ],
  child: const MyApp(),
)
```

### Choosing a DI Approach

The book recommends evaluating: **ease of use, performance, power, stability, industry adoption**.

| Approach | Ease | Power | Testing | Industry Adoption |
|---|---|---|---|---|
| get_it + injectable | High | High | Excellent (swap impls) | Very high |
| Riverpod | Medium | Very high | Excellent (overrides) | High and growing |
| Provider | High | Medium | Good | High (Flutter team endorsed) |
| Manual constructor | Very high | Low | Good (pass mocks directly) | Universal |

---

## 6. Testing on Flutter

### The Testing Pyramid Mapped

The book's pyramid: ~70% Unit | ~20% Integration | ~10% E2E

| Book Layer | Flutter Tool | What It Tests |
|---|---|---|
| **Unit** (70%) | `flutter_test` + `mocktail`/`mockito` | Use Cases, Blocs, Repositories, Mappers, Entities |
| **Widget** (part of 20%) | `flutter_test` (`testWidgets`) | Individual widgets rendering correctly for given states |
| **Integration** (20%) | `integration_test` package | Feature flows across multiple screens |
| **E2E** (10%) | `integration_test` on real device | Full user journeys |
| **Visual regression** | Golden tests (`matchesGoldenFile`) | UI does not change unexpectedly |

### AAA Pattern with bloc_test

The book's AAA maps to bloc_test's `given/when/then`:

```dart
blocTest<LoginCubit, LoginState>(
  'emits [loading, success] when login succeeds',
  // Arrange
  build: () {
    when(() => mockLoginUseCase(any(), any()))
        .thenAnswer((_) async => Right(testUser));
    return LoginCubit(mockLoginUseCase);
  },
  // Act
  act: (cubit) => cubit.login('test@test.com', 'password'),
  // Assert
  expect: () => [
    isA<LoginLoading>(),
    isA<LoginSuccess>(),
  ],
);
```

### Unit Testing Use Cases (mocktail)

```dart
class MockAuthRepository extends Mock implements AuthRepository {}

void main() {
  late LoginUseCase useCase;
  late MockAuthRepository mockRepository;

  setUp(() {
    mockRepository = MockAuthRepository();
    useCase = LoginUseCase(mockRepository);
  });

  test('returns User when repository login succeeds', () async {
    // Arrange
    when(() => mockRepository.login(any(), any()))
        .thenAnswer((_) async => Right(testUser));

    // Act
    final result = await useCase('email@test.com', 'password');

    // Assert
    expect(result, Right(testUser));
    verify(() => mockRepository.login('email@test.com', 'password')).called(1);
  });
}
```

### Widget Testing

```dart
testWidgets('LoginPage shows error when login fails', (tester) async {
  final cubit = MockLoginCubit();
  when(() => cubit.state).thenReturn(LoginFailure('Invalid credentials'));
  whenListen(cubit, Stream<LoginState>.empty());

  await tester.pumpWidget(
    MaterialApp(
      home: BlocProvider<LoginCubit>.value(
        value: cubit,
        child: const LoginPage(),
      ),
    ),
  );

  expect(find.text('Invalid credentials'), findsOneWidget);
});
```

### Enterprise vs. Startup Testing (from the book)

| Strategy | Approach | Mock Scope |
|---|---|---|
| **Enterprise** | White-box: test each layer in isolation (Bloc, UseCase, Repository, DataSource separately) | Mock all dependencies |
| **Startup** | Black-box: test from Cubit/Bloc to Service, mock only external boundaries (HTTP, DB) | Mock only `http.Client`, DB |

---

## 7. Boundaries & Modules on Flutter

### Hard Boundaries: Dart Packages

Each package has its own `pubspec.yaml`, enforcing the Dependency Rule at the package manager level:

```
my_app/
  packages/
    domain/           # pubspec.yaml: NO dependencies on data/ or presentation/
      lib/
        entities/
        repositories/  # abstract classes
        usecases/
    data/             # pubspec.yaml: depends on domain/
      lib/
        models/
        datasources/
        repositories/  # concrete implementations
    presentation/     # pubspec.yaml: depends on domain/
      lib/
        blocs/
        pages/
    app/              # pubspec.yaml: depends on all (composition root)
      lib/
        main.dart
        di/
```

### Monorepo with Melos

For multi-package management:

```yaml
# melos.yaml
name: my_app
packages:
  - packages/**

scripts:
  analyze:
    run: melos exec -- dart analyze .
  test:
    run: melos exec -- flutter test
```

### Soft Boundaries: Library Visibility

```dart
// Use `part` / `part of` for file-level privacy within a library
library auth_feature;
part 'src/auth_bloc.dart';
part 'src/auth_state.dart';

// Use `show` / `hide` for controlled exports
export 'src/entities/user.dart' show User;
export 'src/repositories/auth_repository.dart' show AuthRepository;
// Internal implementations are NOT exported
```

### Feature-First vs. Layer-First

The book recommends feature-first with layers inside each feature. This aligns with the "1 ViewModel per screen" and "1 Repository per entity" rules:

```
# Feature-first (recommended)
lib/
  features/
    auth/
      data/
        models/
        datasources/
        repositories/
      domain/
        entities/
        repositories/   # abstract
        usecases/
      presentation/
        bloc/
        pages/
        widgets/
    products/
      data/
      domain/
      presentation/
  core/                  # shared utilities, error handling, DI setup
    error/
    network/
    di/
```

```
# Layer-first (simpler, for small apps)
lib/
  data/
    models/
    datasources/
    repositories/
  domain/
    entities/
    repositories/
    usecases/
  presentation/
    blocs/
    pages/
    widgets/
```

The book's boundary strategy table:

| Complexity | Flutter Approach |
|---|---|
| Medium | Feature-first folders (soft boundaries) in a single package |
| Complex / Enterprise | Feature packages (hard boundaries) with `melos` |
| Very Large (50+ devs) | Separate repos or heavily isolated packages with strict API surfaces |

---

## 8. Common Flutter Anti-Patterns (from the book's perspective)

### 1. Business Logic in Widgets

```dart
// ANTI-PATTERN: setState with business logic
class _OrderPageState extends State<OrderPage> {
  void _placeOrder() {
    setState(() {
      if (cart.total > user.balance) { // business rule in widget
        _error = 'Insufficient balance';
      } else {
        _order = Order(items: cart.items, total: cart.total * 1.1); // tax logic in widget
      }
    });
  }
}
```

Move to a Use Case. The Widget only calls `cubit.placeOrder()` and renders the resulting state.

### 2. God Bloc / God Cubit

```dart
// ANTI-PATTERN: One Bloc handling everything
class AppBloc extends Bloc<AppEvent, AppState> {
  // handles login, logout, profile, settings, navigation, theme...
  // Violates SCP: 1 Bloc = 1 Screen (or 1 feature concern)
}
```

Split into `LoginBloc`, `ProfileCubit`, `SettingsCubit`.

### 3. BuildContext in Domain/Data Layers

```dart
// ANTI-PATTERN: Framework leaking inward
class UserRepository {
  Future<User> getUser(BuildContext context) async { // BuildContext = framework concept
    final locale = Localizations.localeOf(context);  // framework dependency
    // ...
  }
}
```

Pass locale as a plain `String` parameter. Domain/data layers must never import `package:flutter`.

### 4. Direct HTTP Calls in Widgets or Blocs

```dart
// ANTI-PATTERN: Bloc calling HTTP directly
class ProductBloc extends Bloc<ProductEvent, ProductState> {
  on<LoadProducts>((event, emit) async {
    final response = await Dio().get('/products'); // data layer concern
    final products = (response.data as List).map(/*...*/);
    emit(ProductLoaded(products));
  });
}
```

Bloc should call a Use Case, which calls a Repository, which calls a DataSource. The Bloc never sees `Dio`.

### 5. Mutable State in Global Singletons

```dart
// ANTI-PATTERN: Mutable global state
class AppState {
  static final instance = AppState._();
  AppState._();

  User? currentUser;        // mutable
  List<Product> cart = [];   // mutable
  bool isLoggedIn = false;   // mutable
}
```

Use immutable state streams (Bloc states, Riverpod providers with `@freezed` models).

### 6. JSON Models as Domain Models

```dart
// ANTI-PATTERN: Using the API model everywhere
@JsonSerializable()
class UserModel {
  final String id;
  final String first_name;  // server naming convention leaks into domain
  @JsonKey(name: 'avatar_url')
  final String avatarUrl;

  // fromJson, toJson -- serialization concern in domain
}

// This class is now used in Widgets, Blocs, Use Cases...
// When the API changes, EVERYTHING breaks.
```

Separate `UserModel` (data layer, has `fromJson`/`toJson`) from `User` (domain layer, plain Dart class).

---

## 9. Model Transformation Chain on Flutter

### Enterprise (Complex)

```
HTTP/Dio Response
    |
    v
JsonModel (fromJson/toJson -- json_serializable or freezed)
    |
    v
DBModel (drift entity or hive object -- @HiveType)
    |
    v
PlainModel (intermediate, no framework annotations)
    |
    v
DomainModel (pure Dart, business rules, zero dependencies)
    |
    v
PresentationModel / UI State (used by Widget, formatted for display)
```

```dart
// Data layer: JsonModel
@JsonSerializable()
class UserResponse {
  @JsonKey(name: 'user_id')
  final String userId;
  @JsonKey(name: 'full_name')
  final String fullName;
  @JsonKey(name: 'created_at')
  final String createdAt;

  UserResponse({required this.userId, required this.fullName, required this.createdAt});
  factory UserResponse.fromJson(Map<String, dynamic> json) => _$UserResponseFromJson(json);
}

// Data layer: DBModel
@HiveType(typeId: 0)
class UserEntity extends HiveObject {
  @HiveField(0) final String id;
  @HiveField(1) final String name;
  @HiveField(2) final DateTime createdAt;
  UserEntity({required this.id, required this.name, required this.createdAt});
}

// Domain layer: DomainModel (pure Dart)
class User {
  final String id;
  final String name;
  final DateTime memberSince;
  const User({required this.id, required this.name, required this.memberSince});
}

// Presentation layer: UI state
class UserDisplayState {
  final String name;
  final String memberSinceText; // "Member since Jan 2024"
  final Color memberBadgeColor; // presentation concern
  const UserDisplayState({required this.name, required this.memberSinceText, required this.memberBadgeColor});
}

// Mappers (Executor classes per SCP)
class UserResponseMapper {
  User toDomain(UserResponse response) => User(
    id: response.userId,
    name: response.fullName,
    memberSince: DateTime.parse(response.createdAt),
  );
}

class UserDisplayMapper {
  UserDisplayState toDisplay(User user) => UserDisplayState(
    name: user.name,
    memberSinceText: 'Member since ${DateFormat.yMMM().format(user.memberSince)}',
    memberBadgeColor: user.memberSince.isBefore(DateTime(2023)) ? Colors.gold : Colors.silver,
  );
}
```

### Startup (Pragmatic)

```
HTTP/Dio Response
    |
    v
JsonModel (fromJson/toJson)
    |
    v
PlainModel (merged domain + presentation)
    |
    v
Model (used directly by Widget)
```

```dart
// Merged model -- pragmatic for small apps
@freezed
class User with _$User {
  const factory User({
    required String id,
    required String name,
    required DateTime memberSince,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => User(
    id: json['user_id'] as String,
    name: json['full_name'] as String,
    memberSince: DateTime.parse(json['created_at'] as String),
  );
}

// Widget formats directly (acceptable for startup MVA)
Text('Member since ${DateFormat.yMMM().format(user.memberSince)}')
```

---

## 10. Flutter-Specific Architectural Considerations

### Cross-Platform Alignment

Flutter is cross-platform by design (iOS, Android, Web, Desktop, Embedded). This aligns perfectly with the book's principle that domain and application layers must be **platform-agnostic**. Flutter enforces this naturally: your domain Dart code runs identically on every platform.

### Platform Channels Belong in the Framework Layer

Platform channels (`MethodChannel`, `EventChannel`) are the boundary between Dart and native code. Per the Dependency Rule, they belong in the outermost layer:

```dart
// Framework layer: platform-specific implementation
class BatteryDataSourceImpl implements BatteryDataSource {
  static const _channel = MethodChannel('com.example/battery');

  @override
  Future<int> getBatteryLevel() async {
    final level = await _channel.invokeMethod<int>('getBatteryLevel');
    return level ?? 0;
  }
}

// Domain layer: knows nothing about platform channels
abstract class BatteryDataSource {
  Future<int> getBatteryLevel();
}
```

### State Restoration and Navigation

State restoration (`RestorationMixin`) and navigation state (`GoRouter`, `Navigator 2.0`) are **framework concerns**. They should not leak into domain or application layers:

```dart
// Framework layer: navigation configuration
GoRouter(
  routes: [
    GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
    GoRoute(path: '/home', builder: (_, __) => const HomePage()),
  ],
)

// Presentation layer: navigation triggered by state, not by framework knowledge
ref.listen(authStateProvider, (_, state) {
  if (state is Authenticated) context.go('/home');
  if (state is Unauthenticated) context.go('/login');
});
```

### Flutter's Declarative/Reactive Nature

The widget tree IS declarative and reactive by design. The book recommends Reactive Programming for all MV* patterns. Flutter's `build()` method is already a pure function from state to UI:

```
State -> build() -> Widget Tree
```

This means Flutter naturally fits the book's RP recommendations. The framework itself is the reactive layer -- you describe **what** the UI should look like for a given state, and Flutter handles **when** and **how** to update.

### freezed + json_serializable for the Model Chain

These two packages together implement the full model transformation chain:

```dart
// Data layer: json_serializable for API models
@JsonSerializable()
class UserResponse { /* fromJson, toJson */ }

// Domain layer: freezed for immutable domain models
@freezed
class User with _$User {
  const factory User({required String id, required String name}) = _User;
}

// Presentation layer: freezed for immutable state
@freezed
class UserState with _$UserState {
  const factory UserState.loading() = _Loading;
  const factory UserState.loaded(User user) = _Loaded;
  const factory UserState.error(String message) = _Error;
}
```

This gives you: immutability (book mandate), `copyWith` (new instances instead of mutation), exhaustive pattern matching (Dart 3 sealed via freezed), and clean separation between API models and domain models (Dependency Rule).

---

## Quick Reference: Book Principle to Flutter Implementation

| Book Principle | Flutter Implementation |
|---|---|
| Dependency Rule | Feature folders with `domain/` having zero imports from `data/` or `presentation/` |
| SCP: 1 ViewModel = 1 Screen | 1 Bloc/Cubit per screen |
| SCP: 1 Repository = 1 Entity | 1 Repository class per domain entity |
| Framework layer is dumb | Widgets only call Bloc methods and render state |
| Immutability mandatory | `@freezed` models, `@immutable` annotations, `const` constructors |
| MVI for complex apps | BLoC with sealed Events + sealed States |
| MVVM for simpler apps | Riverpod `AsyncNotifier` or `ChangeNotifier` |
| DIP via interfaces | Dart abstract classes, bound via get_it or Riverpod providers |
| MVA (Minimum Viable Architecture) | Start with Cubit + manual DI; scale to Bloc + get_it + packages as complexity grows |
| Contain cancer | Isolate third-party libs (Dio, Hive, Firebase) behind abstract classes in the data layer |
| AAA testing pattern | `blocTest` given/when/then or standard `test` with arrange/act/assert |
| Testing pyramid | `flutter_test` (70%) > `testWidgets` (20%) > `integration_test` (10%) |
| Model transformation chain | `json_serializable` -> `freezed` domain models -> UI state classes |
| Select libraries last | Define your architecture first, then pick Bloc vs Riverpod vs Provider |

## Quick Reference: Flutter Library Choices (2025-2026 Community Consensus)

| Concern | Recommended | Alternatives | Notes |
|---|---|---|---|
| State Management | Riverpod (new projects) | BLoC (enterprise/existing), Provider (legacy) | Riverpod is the successor to Provider (same author). BLoC strong in enterprise. |
| DI | Riverpod (doubles as DI) | get_it + injectable | If using Riverpod, get_it is often unnecessary. get_it still standard with BLoC. |
| Networking | dio | http (simpler), chopper | Always behind Repository abstract class |
| Persistence | drift (SQLite) | hive (key-value), shared_preferences | drift entities stay in data layer, never leak to domain |
| Immutable Models | freezed + json_serializable | Dart 3 sealed classes + records | freezed still dominates for complex models. Dart 3 features reduce simpler cases. |
| Testing | mocktail | mockito (requires codegen) | mocktail has overtaken mockito — no codegen needed. |
| BLoC Testing | bloc_test (given/when/then) | - | Maps perfectly to book's AAA pattern |
| Navigation | go_router | auto_route, Navigator 2.0 | go_router is the community standard |
| Code Generation | build_runner | - | Used by freezed, json_serializable, injectable |
