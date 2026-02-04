# 🔧 GetIt & Injectable Setup

## What Are These Tools?

### GetIt
A **service locator** - think of it as a smart box where you store all your app's objects. When you need something, you ask the box!

### Injectable
A **code generator** - it automatically writes the boring setup code for GetIt, so you don't have to!

## 🎨 The Big Picture

```
┌────────────────────────────────────────────────┐
│  You Write Annotations                         │
│  @injectable, @lazySingleton, @module          │
└──────────────┬─────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────┐
│  Injectable Code Generator Runs                │
│  (flutter pub run build_runner build)          │
└──────────────┬─────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────┐
│  Generated Code: injector.config.dart          │
│  - Contains all registration logic             │
│  - Knows how to create everything              │
└──────────────┬─────────────────────────────────┘
               │
               ▼
┌────────────────────────────────────────────────┐
│  GetIt Container                               │
│  - Stores and provides instances               │
│  - getIt<WhatYouNeed>()                       │
└────────────────────────────────────────────────┘
```

## Step 1: Setup Files

### The Injector File (`lib/core/di/injector.dart`)

```dart
import 'package:get_it/get_it.dart';
import 'package:injectable/injectable.dart';
import 'package:buenro_hotels/core/di/injector.config.dart';

// This is our magic box! 📦
final getIt = GetIt.instance;

@InjectableInit(
  initializerName: 'init',
  preferRelativeImports: true,
  asExtension: true,
)
Future<GetIt> configureDependencies() async => getIt.init();
```

**What's happening here?**
- `GetIt.instance` creates the global container
- `@InjectableInit` tells Injectable to generate setup code
- `configureDependencies()` will be called at app startup

### The Modules File (`lib/core/di/register_modules.dart`)

Modules are for things you can't easily annotate (like third-party packages):

```dart
import 'package:dio/dio.dart';
import 'package:flutter_dotenv/flutter_dotenv.dart';
import 'package:injectable/injectable.dart';
import 'package:shared_preferences/shared_preferences.dart';

@module
abstract class RegisterModules {
  // Async registration (requires waiting)
  @preResolve
  Future<SharedPreferences> prefs() async =>
      await SharedPreferences.getInstance();

  // Named strings (for configuration)
  @Named('BaseUrl')
  String get baseUrl => dotenv.env['BASE_URL']!;

  @Named('ApiKey')
  String get apiKey => dotenv.env['API_KEY']!;

  // Factory with dependencies
  @lazySingleton
  Dio dio(@Named('BaseUrl') String url) => Dio(
      BaseOptions(
        baseUrl: url,
        connectTimeout: const Duration(seconds: 10)
      ));
}
```

**Breaking it down:**

1. **`@module`** - Marks this as a module class
2. **`@preResolve`** - For async operations that must complete before anything else
3. **`@Named('...')`** - Give a name to string/primitive values so GetIt knows which one to use
4. **`@lazySingleton`** - Creates one instance when first needed

## Step 2: Annotation Types

### @injectable (Factory - New Every Time)

Use for things that should be created fresh each time:

```dart
@injectable
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  final SharedPreferencesManager _preferencesManager;
  final ListHotelsUsecase _listHotelsUsecase;

  HotelsBloc(this._listHotelsUsecase, this._preferencesManager)
      : super(HotelsInitial());
}
```

**When to use:**
- BLoCs (new for each screen)
- Controllers
- View-specific services

### @lazySingleton (One Instance for App)

Use for things that should exist only once:

```dart
@lazySingleton
class HotelsRemoteDatasource {
  final ClientProvider _clientProvider;

  HotelsRemoteDatasource(this._clientProvider);
  
  Future<dynamic> listHotels(QueryHotelModel query) async {
    return await _clientProvider.get(query: query.toJson());
  }
}
```

**When to use:**
- API clients
- Databases
- Storage managers
- Repositories

### @LazySingleton(as: Interface)

Use when implementing an interface:

```dart
// The interface
abstract class HotelsRepository {
  Future<Either<Failure, SearchResponse>> listHotels(QueryHotelModel query);
}

// The implementation
@LazySingleton(as: HotelsRepository)
class HotelsRepositoryImpl implements HotelsRepository {
  final HotelsRemoteDatasource _remoteDatasource;

  HotelsRepositoryImpl(this._remoteDatasource);

  @override
  Future<Either<Failure, SearchResponse>> listHotels(
      QueryHotelModel query) async {
    // Implementation
  }
}
```

**Why do this?**
- You ask for `HotelsRepository` (the interface)
- GetIt gives you `HotelsRepositoryImpl` (the concrete class)
- In tests, you can register a fake implementation instead!

## Step 3: The Dependency Chain

Let's see how everything connects in your Hotels feature:

```dart
// Level 1: External Dependencies (Module)
@module
class RegisterModules {
  @Named('BaseUrl')
  String get baseUrl => dotenv.env['BASE_URL']!;
  
  @Named('ApiKey')
  String get apiKey => dotenv.env['API_KEY']!;
  
  @lazySingleton
  Dio dio(@Named('BaseUrl') String url) => Dio(...);
}

// Level 2: Core Infrastructure
@lazySingleton
class DioClient {
  final Dio dio;
  final String _apiKey;
  
  DioClient(this.dio, @Named('ApiKey') this._apiKey) {
    // Setup interceptors
  }
}

@lazySingleton
class ClientProvider {
  final DioClient _dioClient;
  
  ClientProvider(this._dioClient);
  
  Future<dynamic> get({Map<String, dynamic>? query}) async {
    // Make API call
  }
}

// Level 3: Data Layer
@lazySingleton
class HotelsRemoteDatasource {
  final ClientProvider _clientProvider;
  
  HotelsRemoteDatasource(this._clientProvider);
  
  Future<dynamic> listHotels(QueryHotelModel query) async {
    return await _clientProvider.get(query: query.toJson());
  }
}

@LazySingleton(as: HotelsRepository)
class HotelsRepositoryImpl implements HotelsRepository {
  final HotelsRemoteDatasource _remoteDatasource;
  
  HotelsRepositoryImpl(this._remoteDatasource);
}

// Level 4: Domain Layer
@lazySingleton
class ListHotelsUsecase {
  final HotelsRepository _repo;
  
  ListHotelsUsecase(this._repo);
}

// Level 5: Presentation Layer
@injectable
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  final ListHotelsUsecase _listHotelsUsecase;
  final SharedPreferencesManager _preferencesManager;
  
  HotelsBloc(this._listHotelsUsecase, this._preferencesManager)
      : super(HotelsInitial());
}
```

## 🔄 Visual Dependency Graph

```
                    App Starts
                        │
                        ▼
              ┌─────────────────┐
              │  GetIt Container│
              └────────┬────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
        ▼              ▼              ▼
    ┌──────┐    ┌──────────┐   ┌────────────┐
    │ Dio  │    │ BaseUrl  │   │  ApiKey    │
    └──┬───┘    └─────┬────┘   └─────┬──────┘
       │              │              │
       └──────────────┼──────────────┘
                      ▼
              ┌───────────────┐
              │   DioClient   │
              └───────┬───────┘
                      │
                      ▼
              ┌───────────────┐
              │ ClientProvider│
              └───────┬───────┘
                      │
                      ▼
         ┌────────────────────────┐
         │HotelsRemoteDatasource  │
         └────────────┬───────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │ HotelsRepositoryImpl   │
         └────────────┬───────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │  ListHotelsUsecase     │
         └────────────┬───────────┘
                      │
                      ▼
         ┌────────────────────────┐
         │     HotelsBloc         │ ← You request this!
         └────────────────────────┘
```

## Step 4: App Initialization

In your `main.dart`:

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Load environment variables
  await dotenv.load(fileName: "env/.dev.env");
  
  // Configure all dependencies! 🎉
  await configureDependencies();
  
  runApp(MyApp());
}
```

## Step 5: Using Dependencies

### In Widgets (with BlocProvider)

```dart
@RoutePage()
class MainScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        // GetIt creates the HotelsBloc with all dependencies!
        BlocProvider(create: (context) => getIt<HotelsBloc>()),
        BlocProvider(create: (context) => getIt<FavouritesBloc>()),
        BlocProvider(create: (context) => getIt<AccountBloc>())
      ],
      child: // ... your UI
    );
  }
}
```

### In Regular Classes

```dart
class LocaleProvider with ChangeNotifier {
  void loadLocale() {
    // Get SharedPreferencesManager from GetIt
    final String langCode = getIt<SharedPreferencesManager>()
            .getString(SharedPreferencesManager.language) ??
        'en';
    _locale = Locale(langCode);
  }
}
```

## Step 6: Generating the Code

After adding your annotations, run:

```bash
# Generate once
flutter pub run build_runner build

# Or watch for changes (auto-regenerate)
flutter pub run build_runner watch

# Clean and regenerate everything
flutter pub run build_runner build --delete-conflicting-outputs
```

This creates `injector.config.dart` with all the registration code!

## 🎯 Common Patterns in Your Project

### Pattern 1: Named Dependencies
```dart
// Register
@Named('BaseUrl')
String get baseUrl => dotenv.env['BASE_URL']!;

// Use
DioClient(this.dio, @Named('ApiKey') this._apiKey)
```

### Pattern 2: Interface Implementation
```dart
// Interface
abstract class HotelsRepository { }

// Implementation
@LazySingleton(as: HotelsRepository)
class HotelsRepositoryImpl implements HotelsRepository { }

// Usage - GetIt automatically provides the implementation
final repo = getIt<HotelsRepository>(); // Gets HotelsRepositoryImpl
```

### Pattern 3: Async PreResolve
```dart
@preResolve
Future<SharedPreferences> prefs() async =>
    await SharedPreferences.getInstance();
```

## 🐛 Common Issues

### Issue 1: "Type not registered"
```dart
// ❌ Forgot annotation
class MyService { }

// ✅ Add annotation
@lazySingleton
class MyService { }

// Then regenerate: flutter pub run build_runner build
```

### Issue 2: Circular dependency
```dart
// ❌ A needs B, B needs A
@injectable
class ServiceA {
  ServiceA(ServiceB b);
}

@injectable
class ServiceB {
  ServiceB(ServiceA a); // Circular!
}

// ✅ Redesign to break the circle
```

### Issue 3: Missing module registration
```dart
// ❌ No @module annotation
class RegisterModules { }

// ✅ Add annotation
@module
abstract class RegisterModules { }
```

## 📝 Quick Reference

| Annotation | Lifecycle | Use Case |
|------------|-----------|----------|
| `@injectable` | New instance each time | BLoCs, Controllers |
| `@lazySingleton` | One instance (created when needed) | Repositories, API clients |
| `@singleton` | One instance (created immediately) | Rarely used |
| `@module` | Provides external dependencies | Dio, SharedPreferences |
| `@preResolve` | Async registration | Futures that must complete first |

## Next Steps

Now you understand how dependencies are registered and provided. Let's see how **BLoC** uses these dependencies for state management!

👉 [Continue to BLoC State Management](./03-bloc-state-management.md)

---

## 🎓 Quick Quiz

**Q1:** What's the difference between `@injectable` and `@lazySingleton`?
<details>
<summary>Answer</summary>

- `@injectable`: Creates a NEW instance every time you call `getIt<T>()`
- `@lazySingleton`: Creates ONE instance the first time, then reuses it
</details>

**Q2:** Why use `@LazySingleton(as: Interface)`?
<details>
<summary>Answer</summary>
So you can depend on the interface (abstraction) rather than the concrete implementation. Makes testing easier and follows SOLID principles!
</details>

**Q3:** When do you need `@module`?
<details>
<summary>Answer</summary>
For third-party classes you can't annotate (like Dio, SharedPreferences) or for providing configuration values (like API keys, base URLs).
</details>
