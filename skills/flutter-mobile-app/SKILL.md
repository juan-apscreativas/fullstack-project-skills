---
name: flutter-mobile-app
description: >
  Use when working in a Flutter mobile project — setting up project structure,
  choosing state management, configuring navigation, implementing API services,
  designing offline-first architecture, or following Flutter architecture
  conventions for production apps.
---

# Flutter Mobile App Architecture

## Overview

Defines architecture conventions for production Flutter apps. Organizes by **layer** (not by feature), with a hybrid BLoC + Provider state management approach, centralized networking, and offline-first support.

<HARD-GATE>
**Dependencies flow downward only.** `pages → providers/blocs → services → models → core`. Never import upward or create circular dependencies. A page may use a provider; a provider may use a service; a service may use a model. Never the reverse.
</HARD-GATE>

## Anti-Pattern: "Logic in Widgets"

Putting API calls, business rules, or conditional branching inside widgets or pages. Pages have one job: compose widgets, consume state, and navigate. Business logic lives in providers/blocs and services. Data access lives in services.

## Project Structure

```
lib/
├── core/                   # App-wide utilities and constants
│   ├── colors.dart         # Color palette
│   ├── text_styles.dart    # Typography definitions
│   ├── constants.dart      # Static values (timeouts, keys, enums)
│   ├── global.dart         # Singleton: SharedPreferences, navigator key
│   └── extensions.dart     # Dart extension methods
├── models/                 # Data classes (fromJson/toJson)
├── services/               # Business logic & external communication
│   ├── api_service.dart    # HTTP client wrapper
│   ├── error_handler.dart  # Centralized error handling
│   ├── storage_service.dart # SharedPreferences abstraction
│   └── ...
├── blocs/                  # BLoC classes (cross-cutting concerns)
│   └── auth/
│       ├── auth_bloc.dart
│       ├── auth_event.dart
│       └── auth_state.dart
├── providers/              # ChangeNotifier classes (screen-level state)
├── pages/                  # Full-screen widgets (one per route)
│   └── sections/           # Sub-views within a page
├── widgets/                # Reusable UI components
└── main.dart               # Entry point, DI setup, route config
```

### Principles

- **One file, one purpose.** A model file holds a model. A page file holds a page.
- **No circular imports.** Dependencies flow downward only.
- **Keep `main.dart` lean.** Initialize services, configure DI, define routes, run the app. No business logic.

## State Management

### When to Use What

| Approach | Best For | Complexity |
|----------|----------|------------|
| **Provider** | Feature-level state, form state, simple apps | Low |
| **BLoC** | Cross-cutting concerns, event-driven state, complex flows | Medium |
| **Riverpod** | Type-safe DI, auto-dispose, testability | Medium |
| **setState** | Local widget state (animations, toggles) | Minimal |

### Hybrid BLoC + Provider (Recommended)

- **BLoC**: Auth, connectivity, permissions, location, background tasks
- **Provider**: Forms, lists, page-specific data, UI toggles

This avoids over-engineering simple screens with BLoC boilerplate while keeping cross-cutting state predictable.

### Provider Pattern

```dart
class AuthProvider extends ChangeNotifier {
  bool _loading = false;
  String? _token;
  String? _error;

  bool get loading => _loading;
  String? get token => _token;
  String? get error => _error;
  bool get isAuthenticated => _token != null;

  Future<void> login(String email, String password) async {
    _loading = true;
    _error = null;
    notifyListeners();

    try {
      final response = await ApiService.post('/auth/login', body: {
        'email': email,
        'password': password,
      });
      _token = response['token'];
      await StorageService.set('token', _token!);
    } catch (e) {
      _error = ErrorHandler.message(e);
    } finally {
      _loading = false;
      notifyListeners();
    }
  }
}
```

### BLoC Pattern

```dart
// Events
abstract class AuthEvent extends Equatable {
  @override
  List<Object?> get props => [];
}
class AuthLogin extends AuthEvent {
  final String token;
  AuthLogin(this.token);
  @override
  List<Object?> get props => [token];
}
class AuthLogout extends AuthEvent {}
class AuthCheckSession extends AuthEvent {}

// States
abstract class AuthState extends Equatable {
  @override
  List<Object?> get props => [];
}
class AuthInitial extends AuthState {}
class AuthAuthenticated extends AuthState {
  final String token;
  AuthAuthenticated(this.token);
  @override
  List<Object?> get props => [token];
}
class AuthUnauthenticated extends AuthState {}

// Bloc
class AuthBloc extends Bloc<AuthEvent, AuthState> {
  AuthBloc() : super(AuthInitial()) {
    on<AuthLogin>((event, emit) => emit(AuthAuthenticated(event.token)));
    on<AuthLogout>((event, emit) async {
      await StorageService.remove('token');
      emit(AuthUnauthenticated());
    });
    on<AuthCheckSession>((event, emit) async {
      final token = StorageService.get('token');
      emit(token != null ? AuthAuthenticated(token) : AuthUnauthenticated());
    });
  }
}
```

## Navigation

### Named Routes (Simple Apps)

```dart
MaterialApp(
  navigatorKey: Global.navigatorKey,
  initialRoute: '/',
  routes: {
    '/': (_) => SplashPage(),
    '/login': (_) => LoginPage(),
    '/home': (_) => HomePage(),
  },
);
```

### GoRouter (Medium-Large Apps)

Use `go_router` when you need deep links, redirects, guards, or nested navigation.

```dart
final router = GoRouter(
  navigatorKey: Global.navigatorKey,
  initialLocation: '/',
  redirect: (context, state) {
    final auth = context.read<AuthProvider>();
    final isLogin = state.matchedLocation == '/login';
    if (!auth.isAuthenticated && !isLogin) return '/login';
    if (auth.isAuthenticated && isLogin) return '/home';
    return null;
  },
  routes: [
    GoRoute(path: '/', builder: (_, __) => SplashPage()),
    GoRoute(path: '/login', builder: (_, __) => LoginPage()),
    GoRoute(path: '/home', builder: (_, __) => HomePage()),
    GoRoute(path: '/detail/:id', builder: (_, state) =>
      DetailPage(id: state.pathParameters['id']!)),
  ],
);
```

## Networking — ApiService

Centralize all HTTP calls. Never call `http.get()` from a widget or provider directly.

```dart
class ApiService {
  static String _baseUrl = '';
  static String? _token;

  static void configure({required String baseUrl, String? token}) {
    _baseUrl = baseUrl;
    _token = token;
  }

  static void setToken(String? token) => _token = token;

  static Map<String, String> get _headers => {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
    if (_token != null) 'Authorization': 'Bearer $_token',
  };

  static Future<Map<String, dynamic>> get(String path, {Map<String, dynamic>? params}) async {
    final uri = Uri.https(_baseUrl, path, params?.map((k, v) => MapEntry(k, v.toString())));
    final response = await http.get(uri, headers: _headers).timeout(const Duration(seconds: 15));
    return _handleResponse(response);
  }

  static Future<Map<String, dynamic>> post(String path, {Map<String, dynamic>? body}) async {
    final uri = Uri.https(_baseUrl, path);
    final response = await http.post(uri, headers: _headers, body: jsonEncode(body))
        .timeout(const Duration(seconds: 15));
    return _handleResponse(response);
  }

  static Map<String, dynamic> _handleResponse(http.Response response) {
    final body = jsonDecode(response.body);
    if (response.statusCode >= 200 && response.statusCode < 300) return body;
    throw ApiException(
      statusCode: response.statusCode,
      message: body['message'] ?? 'Unknown error',
      errors: body['errors'],
    );
  }
}

class ApiException implements Exception {
  final int statusCode;
  final String message;
  final dynamic errors;
  ApiException({required this.statusCode, required this.message, this.errors});
  bool get isUnauthorized => statusCode == 401;
  bool get isValidation => statusCode == 422;
  bool get isServerError => statusCode >= 500;
}
```

## Local Storage — StorageService

Wrap SharedPreferences behind a service so storage can be swapped or mocked.

```dart
class StorageService {
  static late SharedPreferences _prefs;
  static Future<void> init() async => _prefs = await SharedPreferences.getInstance();

  static Future<void> set(String key, String value) => _prefs.setString(key, value);
  static String? get(String key) => _prefs.getString(key);
  static Future<void> remove(String key) => _prefs.remove(key);

  static Future<void> setJson(String key, Map<String, dynamic> value) =>
      _prefs.setString(key, jsonEncode(value));
  static Map<String, dynamic>? getJson(String key) {
    final raw = _prefs.getString(key);
    return raw != null ? jsonDecode(raw) : null;
  }

  static Future<void> clear() => _prefs.clear();
}
```

### Storage Decision Guide

| Storage | Use Case |
|---------|----------|
| **SharedPreferences** | Tokens, settings, small cached JSON, user preferences |
| **Hive** | Larger structured data, offline collections, fast reads |
| **SQLite (sqflite/drift)** | Relational data, complex queries, large datasets |
| **Secure Storage** | Sensitive credentials, encryption keys, biometric-protected data |
| **File system** | Images, documents, large binary data |

## Error Handling

```dart
class ErrorHandler {
  static String message(dynamic error) {
    if (error is ApiException) {
      if (error.isUnauthorized) return 'Session expired. Please log in again.';
      if (error.isValidation) return _validationMessage(error.errors);
      if (error.isServerError) return 'Server error. Try again later.';
      return error.message;
    }
    if (error is TimeoutException) return 'Request timed out. Check your connection.';
    if (error is SocketException) return 'No internet connection.';
    return 'An unexpected error occurred.';
  }

  static String _validationMessage(dynamic errors) {
    if (errors is Map) {
      final first = errors.values.first;
      if (first is List && first.isNotEmpty) return first.first.toString();
    }
    return 'Invalid data. Please check the form.';
  }

  static void report(dynamic error, StackTrace? stack) {
    debugPrint('ERROR: $error');
    if (stack != null) debugPrint(stack.toString());
  }
}
```

### Error Handling Mixin for Providers

```dart
mixin ProviderErrorMixin on ChangeNotifier {
  String? _error;
  bool _loading = false;

  String? get error => _error;
  bool get loading => _loading;

  Future<T?> runSafe<T>(Future<T> Function() action) async {
    _error = null;
    _loading = true;
    notifyListeners();
    try {
      return await action();
    } catch (e, stack) {
      _error = ErrorHandler.message(e);
      ErrorHandler.report(e, stack);
      return null;
    } finally {
      _loading = false;
      notifyListeners();
    }
  }
}
```

## Offline-First Architecture

### Offline Queue

```dart
class OfflineQueue {
  static const _key = 'offline_queue';

  static Future<void> enqueue(Map<String, dynamic> action) async {
    final queue = StorageService.getJsonList(_key);
    action['queued_at'] = DateTime.now().toIso8601String();
    queue.add(action);
    await StorageService.setJsonList(_key, queue);
  }

  static Future<void> sync() async {
    final queue = StorageService.getJsonList(_key);
    if (queue.isEmpty) return;
    final failed = <Map<String, dynamic>>[];
    for (final action in queue) {
      try {
        await ApiService.post(action['endpoint'], body: action['data']);
      } catch (_) {
        failed.add(action);
      }
    }
    await StorageService.setJsonList(_key, failed);
  }

  static int get pendingCount => StorageService.getJsonList(_key).length;
}
```

### Connectivity Monitoring

Use `connectivity_plus` with a BLoC to trigger sync on reconnect:

```dart
class ConnectionBloc extends Bloc<ConnectionEvent, ConnectionState> {
  late StreamSubscription _subscription;

  ConnectionBloc() : super(ConnectionInitial()) {
    _subscription = Connectivity().onConnectivityChanged.listen((results) {
      final connected = results.any((r) => r != ConnectivityResult.none);
      if (connected) {
        add(ConnectionRestored());
        OfflineQueue.sync();
      } else {
        add(ConnectionLost());
      }
    });
    on<ConnectionRestored>((_, emit) => emit(ConnectionOnline()));
    on<ConnectionLost>((_, emit) => emit(ConnectionOffline()));
  }

  @override
  Future<void> close() { _subscription.cancel(); return super.close(); }
}
```

## Authentication — JWT Flow

```
Login Page → POST /auth/login → { token, user }
  → Store token (StorageService)
  → Set token (ApiService.setToken)
  → Navigate to /home

App Start → SplashPage
  → Read token from StorageService
  → If token: GET /me → success → /home, 401 → clear token → /login
  → If no token: → /login
```

## Models & Serialization

| Models Count | Recommendation |
|-------------|----------------|
| < 10 | Manual `fromJson`/`toJson` |
| 10-20 | Manual, consider codegen if models change often |
| 20+ | Use `json_serializable` or `freezed` |

```dart
class UserModel {
  final int id;
  final String name;
  final String email;
  final String? avatar;

  const UserModel({required this.id, required this.name, required this.email, this.avatar});

  factory UserModel.fromJson(Map<String, dynamic> json) => UserModel(
    id: json['id'], name: json['name'], email: json['email'], avatar: json['avatar'],
  );

  Map<String, dynamic> toJson() => {'id': id, 'name': name, 'email': email, 'avatar': avatar};

  UserModel copyWith({String? name, String? email, String? avatar}) => UserModel(
    id: id, name: name ?? this.name, email: email ?? this.email, avatar: avatar ?? this.avatar,
  );
}
```

## Theming & Design System

```dart
// core/colors.dart
class AppColors {
  static const Color primary = Color(0xFFFF8654);
  static const Color secondary = Color(0xFF92C4B0);
  static const Color success = Color(0xFF22C55E);
  static const Color warning = Color(0xFFEAB308);
  static const Color error = Color(0xFFEF4444);
  static const Color background = Color(0xFFF8F8FA);
  static const Color surface = Color(0xFFFFFFFF);
  static const Color textPrimary = Color(0xFF181B1F);
  static const Color textSecondary = Color(0xFF64748B);
  static const Color border = Color(0xFFE2E8F0);
}
```

Bundle fonts locally instead of fetching at runtime — add to `pubspec.yaml` under `flutter.fonts`.

## Environment & Flavors

```dart
enum Environment { dev, staging, production }

class AppConfig {
  static late Environment env;
  static late String baseUrl;
  static late String appName;

  static void init(Environment environment) {
    env = environment;
    switch (environment) {
      case Environment.dev:
        baseUrl = 'dev-api.example.com'; appName = 'MyApp DEV';
      case Environment.staging:
        baseUrl = 'staging-api.example.com'; appName = 'MyApp STG';
      case Environment.production:
        baseUrl = 'api.example.com'; appName = 'MyApp';
    }
  }
}
```

Run with: `flutter run -t lib/main_dev.dart`

For separate bundle IDs per environment, use Flutter Flavors in `android/app/build.gradle`.

## Starter Dependencies

```yaml
dependencies:
  provider: ^6.0.3
  http: ^1.1.0
  shared_preferences: ^2.0.15
  google_fonts: ^6.2.1
  intl: ^0.20.2
  connectivity_plus: ^6.1.4
  # flutter_bloc: ^9.1.1       # Add if using BLoC
  # equatable: ^2.0.3          # Add if using BLoC
  # go_router: ^14.0.0         # Add for complex navigation

dev_dependencies:
  flutter_test:
    sdk: flutter
  mocktail: ^1.0.4
  flutter_lints: ^6.0.0
  # bloc_test: ^9.1.0          # Add if using BLoC
```

## Anti-Patterns

### Logic in Widgets
```dart
// WRONG — API call in a widget
onPressed: () async {
  final response = await http.get(Uri.parse('https://api.example.com/items'));
  setState(() => items = jsonDecode(response.body));
}

// CORRECT — delegate to provider
onPressed: () => context.read<ItemProvider>().loadItems()
```

### Direct SharedPreferences Access
```dart
// WRONG — accessing SharedPreferences directly
final prefs = await SharedPreferences.getInstance();
final token = prefs.getString('token');

// CORRECT — use StorageService
final token = StorageService.get('token');
```

### Fat Provider / Fat BLoC
```dart
// WRONG — provider doing HTTP, caching, validation, formatting
class OrderProvider extends ChangeNotifier {
  Future<void> loadOrders() async {
    final response = await http.get(...);  // Should be in ApiService
    final cached = prefs.getString(...);   // Should be in StorageService
    // 50 lines of logic...
  }
}

// CORRECT — provider orchestrates services
class OrderProvider extends ChangeNotifier with ProviderErrorMixin {
  Future<void> loadOrders() async {
    await runSafe(() async {
      final response = await ApiService.get('/orders');
      orders = (response['data'] as List).map((e) => Order.fromJson(e)).toList();
    });
  }
}
```

### Running Commands Outside Flutter

```bash
# WRONG — running Dart directly
dart run build_runner build

# CORRECT — use Flutter wrapper
flutter pub run build_runner build --delete-conflicting-outputs
```

## Related Skills

Works well with: `project-orchestration`, `mobile-feature-dev`, `api-contract-sync`

## When to Use

This skill applies when working in a Flutter mobile project — setting up project structure, choosing state management, configuring navigation, implementing services, designing offline-first architecture, or establishing theming conventions. For web frontend work, use `nextjs-feature-based`. For backend work, use `laravel-modular-monolith`.

## Coexistence with Superpowers

This skill activates when working in a Flutter mobile project.
Superpowers has no equivalent — it is technology-agnostic and does not define Flutter architecture conventions.
This skill complements any Superpowers phase by applying Flutter mobile architecture conventions.

If Superpowers is not installed, this skill works identically.

## Coexistence with frontend-design

The `frontend-design` skill handles **visual aesthetics**: typography, color, motion, and spatial composition.
This skill handles **architecture and technical standards**: project structure, state management, navigation, services, and testing.

When both are active, they complement each other — frontend-design guides how things look,
this skill guides how things are structured.
