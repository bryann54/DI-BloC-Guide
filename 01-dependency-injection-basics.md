# 🎯 Understanding Dependency Injection

## What is Dependency Injection?

Think of it like this: Instead of a class creating its own tools, we **hand it the tools it needs**.

### 🏠 Real-World Analogy

Imagine you're a chef:

**Without DI (Bad way):**
```dart
class Chef {
  final Knife knife = Knife(); // Chef makes own knife
  final Pan pan = Pan();       // Chef makes own pan
  
  void cook() {
    knife.cut();
    pan.heat();
  }
}
```

**With DI (Good way):**
```dart
class Chef {
  final Knife knife;  // Someone gives the chef a knife
  final Pan pan;      // Someone gives the chef a pan
  
  Chef(this.knife, this.pan); // Tools are "injected"
  
  void cook() {
    knife.cut();
    pan.heat();
  }
}
```

## Why Do We Need This?

### 1. **Easy Testing** 🧪
```dart
// Without DI - Hard to test!
class UserService {
  final ApiClient api = ApiClient(); // Always uses real API
  
  Future<User> getUser() => api.fetchUser();
}

// With DI - Easy to test!
class UserService {
  final ApiClient api;
  
  UserService(this.api); // We can inject a fake API for testing!
  
  Future<User> getUser() => api.fetchUser();
}

// In tests:
final fakeApi = FakeApiClient();
final service = UserService(fakeApi); // Inject fake for testing
```

### 2. **Flexibility** 🔄
```dart
// Switch implementations easily
final prodService = UserService(RealApiClient());
final testService = UserService(MockApiClient());
```

### 3. **Single Responsibility** 📦
```dart
// Each class focuses on ONE job
// Not responsible for creating its dependencies
```

## Types of Dependency Injection

### 1. Constructor Injection (Most Common)
```dart
class HotelsRepository {
  final HotelsRemoteDatasource _datasource;
  
  // Dependencies injected through constructor
  HotelsRepository(this._datasource);
}
```

### 2. Property Injection (Less Common)
```dart
class SomeService {
  late ApiClient api; // Set after creation
}
```

### 3. Method Injection (Rare)
```dart
class Processor {
  void process(Database db) { // Dependency passed to method
    db.save();
  }
}
```

## The DI Container Pattern

Instead of manually creating every object, we use a **container** that knows how to create and provide everything:

```dart
// Manual way (NO! Too much work!)
final dio = Dio();
final client = DioClient(dio, apiKey);
final provider = ClientProvider(client);
final datasource = HotelsRemoteDatasource(provider);
final repo = HotelsRepositoryImpl(datasource);
final usecase = ListHotelsUsecase(repo);
final bloc = HotelsBloc(usecase, prefs);

// DI Container way (YES! So easy!)
final bloc = getIt<HotelsBloc>(); // Container creates everything!
```

## 🎨 Visual Flow

```
┌─────────────────────────────────────┐
│     Your App Starts                 │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  Configure DI Container (GetIt)     │
│  - Register all dependencies        │
│  - Define how to create each object │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│   When You Need Something...        │
│   getIt<HotelsBloc>()              │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  GetIt Creates Everything You Need: │
│  1. Creates Dio                     │
│  2. Creates DioClient (needs Dio)   │
│  3. Creates Datasource (needs Client)│
│  4. Creates Repository (needs DS)   │
│  5. Creates UseCase (needs Repo)    │
│  6. Creates Bloc (needs UseCase)    │
│  7. Returns Bloc to you!            │
└─────────────────────────────────────┘
```

## Key Principles

### 1. Depend on Abstractions, Not Concrete Classes
```dart
// Bad
class HotelsBloc {
  final ListHotelsUsecaseImpl usecase; // Depends on concrete class
}

// Good
class HotelsBloc {
  final ListHotelsUsecase usecase; // Depends on interface/abstract
}
```

### 2. Single Instance vs New Instance

**Singleton** - One instance for entire app:
```dart
@lazySingleton  // Only one instance ever created
class ApiClient { }
```

**Factory** - New instance every time:
```dart
@injectable  // New instance each time you ask
class HotelsBloc { }
```

### 3. Lifecycle Management
```dart
// App starts → Create once → Use everywhere → App ends
@lazySingleton
class SharedPreferencesManager { }

// Screen opens → Create → Use → Screen closes → Destroy
@injectable
class HotelsBloc { }
```

## 🎯 Quiz Yourself!

**Question 1:** Why don't we do `final api = ApiClient()` inside a class?
<details>
<summary>Answer</summary>
Because then we can't test it easily! We can't replace it with a fake version. Also, the class becomes responsible for creating its dependencies, which violates single responsibility.
</details>

**Question 2:** What's the difference between `@lazySingleton` and `@injectable`?
<details>
<summary>Answer</summary>
- `@lazySingleton`: Creates ONE instance that lives for the entire app lifetime
- `@injectable`: Creates a NEW instance every time you ask for it
</details>

**Question 3:** What problem does a DI container solve?
<details>
<summary>Answer</summary>
It automatically creates objects and their dependencies, so you don't have to manually create a long chain of objects. Just ask for what you need, and it figures out how to build it!
</details>

## Next Steps

Now that you understand the "why" behind DI, let's see how **GetIt** and **Injectable** implement this in Flutter!

👉 [Continue to GetIt & Injectable Setup](./02-getit-injectable-setup.md)

---

## 📚 Key Takeaways

✅ DI means giving a class its dependencies instead of creating them inside  
✅ Makes testing easier because we can swap real objects with fake ones  
✅ DI containers (like GetIt) manage object creation for us  
✅ Use `@lazySingleton` for app-wide objects, `@injectable` for temporary ones  
