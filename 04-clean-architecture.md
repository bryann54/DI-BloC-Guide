# 🏗️ Clean Architecture

## What is Clean Architecture?

Clean Architecture is a way to organize code so that:
- Each part has ONE job
- Parts don't know too much about each other
- Easy to test each part separately
- Easy to change one part without breaking others

Think of it like a well-organized office building! 🏢

## 🎨 The Layer Cake

```
┌─────────────────────────────────────────┐
│         PRESENTATION LAYER              │  ← What users see
│  (BLoCs, Widgets, Pages, Screens)       │  
└──────────────┬──────────────────────────┘
               │ talks to
┌──────────────▼──────────────────────────┐
│          DOMAIN LAYER                   │  ← Business rules
│  (UseCases, Entities, Repositories)     │  
└──────────────┬──────────────────────────┘
               │ talks to
┌──────────────▼──────────────────────────┐
│           DATA LAYER                    │  ← Gets/Saves data
│  (Repositories Impl, Data Sources,      │
│   Models, API Clients)                  │
└─────────────────────────────────────────┘
```

## The Dependency Rule 📜

**Inner layers don't know about outer layers!**

```
Presentation → Can use Domain
Domain → CANNOT use Presentation

Domain → Defines contracts (interfaces)
Data → Implements contracts

✅ Good: Presentation depends on Domain
✅ Good: Data depends on Domain
❌ Bad: Domain depends on Data
❌ Bad: Domain depends on Presentation
```

## Layer 1: Data Layer

**Job:** Get data from somewhere (API, Database, Cache)

### Structure in Your Project

```
lib/features/hotels/data/
├── datasources/
│   └── hotels_remote_datasource.dart    ← Talks to API
├── models/
│   ├── query_hotel_model.dart           ← API request model
│   └── search_response.dart             ← API response model
└── repositories/
    └── hotels_repository_impl.dart      ← Implements domain contract
```

### Remote Data Source

```dart
@lazySingleton
class HotelsRemoteDatasource {
  final ClientProvider _clientProvider;

  HotelsRemoteDatasource(this._clientProvider);

  Future<dynamic> listHotels(QueryHotelModel queryHotelModel) async {
    try {
      // Make API call with query parameters
      return await _clientProvider.get(query: queryHotelModel.toJson());
    } catch (e) {
      debugPrint('listHotels response: $e');
      rethrow;
    }
  }
}
```

**What it does:**
- Takes a model (request)
- Calls API through ClientProvider
- Returns raw response or throws error

### Models

Models are for converting between JSON and Dart objects:

```dart
@JsonSerializable()
class QueryHotelModel extends Equatable {
  final String engine;
  final String q;
  @JsonKey(name: 'check_in_date')
  final String checkInDate;
  @JsonKey(name: 'check_out_date')
  final String checkOutDate;
  
  const QueryHotelModel({
    required this.engine,
    required this.q,
    required this.checkInDate,
    required this.checkOutDate,
  });

  // Convert to JSON for API
  Map<String, dynamic> toJson() => _$QueryHotelModelToJson(this);
  
  // Create from JSON response
  factory QueryHotelModel.fromJson(Map<String, dynamic> json) =>
      _$QueryHotelModelFromJson(json);

  @override
  List<Object?> get props => [engine, q, checkInDate, checkOutDate];
}
```

### Repository Implementation

```dart
@LazySingleton(as: HotelsRepository)
class HotelsRepositoryImpl implements HotelsRepository {
  final HotelsRemoteDatasource _remoteDatasource;

  HotelsRepositoryImpl(this._remoteDatasource);

  @override
  Future<Either<Failure, SearchResponse>> listHotels(
      QueryHotelModel query) async {
    try {
      // Get raw data from datasource
      final result = await _remoteDatasource.listHotels(query);
      
      // Handle server errors
      if (result is ServerError) {
        return Left(ServerFailure(badResponse: result));
      }
      
      // Convert to domain entity and return
      return Right(SearchResponse.fromJson(result));
      
    } on ServerException {
      return const Left(ServerFailure());
    }
  }
}
```

**Key Points:**
- Implements the interface from Domain layer
- Converts data source responses to domain entities
- Handles errors and wraps in Either<Failure, Success>

## Layer 2: Domain Layer

**Job:** Contains business rules and defines contracts

### Structure

```
lib/features/hotels/domain/
├── entities/          ← Pure business objects (usually in models)
├── repositories/      ← Contracts (interfaces)
│   └── hotels_repository.dart
└── usecases/          ← Business logic operations
    └── list_hotels_usecase.dart
```

### Repository Interface

```dart
abstract class HotelsRepository {
  Future<Either<Failure, SearchResponse>> listHotels(QueryHotelModel query);
}
```

**Why abstract?**
- Domain defines WHAT should happen
- Data layer defines HOW it happens
- Presentation doesn't care about the HOW

### UseCase

```dart
@lazySingleton
class ListHotelsUsecase implements UseCase<SearchResponse, GetHotelsParams> {
  final HotelsRepository _repo;

  ListHotelsUsecase(this._repo);

  @override
  Future<Either<Failure, SearchResponse>> call(GetHotelsParams params) async {
    // Business logic: Build query from params
    return await _repo.listHotels(QueryHotelModel(
      engine: params.engine,
      q: params.q.isEmpty ? 'Bali Hotels' : params.q, // Default value
      gl: params.gl,
      hl: params.hl,
      currency: params.currency,
      checkInDate: params.checkInDate,
      checkOutDate: params.checkOutDate,
      nextPageToken: params.nextPageToken,
    ));
  }
}
```

**UseCase Pattern:**
- One class = One business operation
- Called like a function: `usecase.call(params)`
- Contains business rules (like default values)
- Returns Either<Failure, Success>

### Base UseCase

```dart
abstract class UseCase<Type, Params> {
  Future<Either<Failure, Type?>> call(Params params);
}

// No params needed
class NoParams extends Equatable {
  @override
  List<Object?> get props => [];
}

// Specific params for this use case
class GetHotelsParams extends Equatable {
  final String engine;
  final String q;
  final String checkInDate;
  final String checkOutDate;
  final String? nextPageToken;

  const GetHotelsParams({
    this.engine = 'google_hotels',
    this.q = 'Bali Resorts',
    required this.checkInDate,
    required this.checkOutDate,
    this.nextPageToken,
  });

  GetHotelsParams copyWith({
    String? q,
    String? nextPageToken,
    // ... other fields
  }) {
    return GetHotelsParams(
      q: q ?? this.q,
      nextPageToken: nextPageToken ?? this.nextPageToken,
      checkInDate: this.checkInDate,
      checkOutDate: this.checkOutDate,
    );
  }

  @override
  List<Object?> get props => [engine, q, checkInDate, checkOutDate];
}
```

## Layer 3: Presentation Layer

**Job:** Show UI and handle user interactions

### Structure

```
lib/features/hotels/presentation/
├── bloc/
│   ├── hotels_bloc.dart       ← State management
│   ├── hotels_event.dart      ← User actions
│   └── hotels_state.dart      ← UI states
├── pages/
│   └── hotels_page.dart       ← Full screens
└── widgets/
    └── hotel_card.dart        ← Reusable components
```

### The BLoC (Already covered!)

```dart
@injectable
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  final ListHotelsUsecase _listHotelsUsecase;
  final SharedPreferencesManager _preferencesManager;

  HotelsBloc(this._listHotelsUsecase, this._preferencesManager)
      : super(HotelsInitial()) {
    on<ListHotelsEvent>(_listHotels);
  }

  FutureOr<void> _listHotels(
      ListHotelsEvent event, Emitter<HotelsState> emit) async {
    emit(HotelsLoadingState());
    final params = event.params.copyWith(
        hl: _preferencesManager.getString(
            SharedPreferencesManager.language));
    final response = await _listHotelsUsecase.call(params);
    emit(response.fold(
      (error) => ListHotelsError(error: error.toString()),
      (data) => ListHotelsSuccess(hotels: data.properties),
    ));
  }
}
```

## 🔄 Complete Data Flow

Let's trace a user searching for hotels:

```
1. USER TAPS "SEARCH"
   ↓
2. Widget sends event to BLoC
   context.read<HotelsBloc>().add(
     ListHotelsEvent(params: searchParams)
   )
   ↓
3. BLoC receives event
   HotelsBloc._listHotels() is called
   ↓
4. BLoC emits loading state
   emit(HotelsLoadingState())
   Widget shows loading spinner
   ↓
5. BLoC calls UseCase
   _listHotelsUsecase.call(params)
   ↓
6. UseCase calls Repository
   _repo.listHotels(queryModel)
   ↓
7. Repository calls Data Source
   _remoteDatasource.listHotels(query)
   ↓
8. Data Source calls API
   _clientProvider.get(query: query.toJson())
   ↓
9. API returns data
   ↓
10. Data Source returns raw response
    ↓
11. Repository converts to domain entity
    SearchResponse.fromJson(result)
    ↓
12. Repository wraps in Either
    Right(searchResponse)
    ↓
13. UseCase returns Either to BLoC
    ↓
14. BLoC emits success or error state
    emit(ListHotelsSuccess(hotels: data))
    ↓
15. Widget rebuilds with new state
    BlocBuilder shows hotel list
```

## 🎨 Visual Architecture

```
┌──────────────────────────────────────────────────────┐
│                 PRESENTATION                         │
│                                                      │
│  ┌────────────┐         ┌─────────────┐            │
│  │  Widget    │────────▶│ HotelsBloc  │            │
│  │ (UI Code)  │◀────────│ (Manager)   │            │
│  └────────────┘         └──────┬──────┘            │
│                                 │                    │
└─────────────────────────────────┼────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────┐
│                 DOMAIN          │                    │
│                                 ▼                    │
│                      ┌────────────────────┐         │
│                      │ ListHotelsUsecase  │         │
│                      │ (Business Logic)   │         │
│                      └──────────┬─────────┘         │
│                                 │                    │
│                                 ▼                    │
│                      ┌────────────────────┐         │
│                      │ HotelsRepository   │         │
│                      │   (Interface)      │         │
│                      └────────────────────┘         │
│                                                      │
└─────────────────────────────────┬────────────────────┘
                                  │
┌─────────────────────────────────┼────────────────────┐
│                  DATA           │                    │
│                                 ▼                    │
│                   ┌──────────────────────────┐      │
│                   │ HotelsRepositoryImpl     │      │
│                   │ (Implementation)         │      │
│                   └───────────┬──────────────┘      │
│                               │                      │
│                               ▼                      │
│                   ┌──────────────────────────┐      │
│                   │ HotelsRemoteDatasource   │      │
│                   │ (API Calls)              │      │
│                   └───────────┬──────────────┘      │
│                               │                      │
│                               ▼                      │
│                   ┌──────────────────────────┐      │
│                   │    ClientProvider        │      │
│                   │    (HTTP Client)         │      │
│                   └──────────────────────────┘      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

## Why This Architecture?

### 1. Testability 🧪

Each layer can be tested independently:

```dart
// Test UseCase without API
test('ListHotelsUsecase returns hotels', () async {
  // Arrange
  final mockRepo = MockHotelsRepository();
  final usecase = ListHotelsUsecase(mockRepo);
  
  when(mockRepo.listHotels(any))
      .thenAnswer((_) async => Right(fakeHotels));
  
  // Act
  final result = await usecase.call(params);
  
  // Assert
  expect(result.isRight(), true);
});

// Test BLoC without network
test('HotelsBloc emits success state', () {
  final mockUsecase = MockListHotelsUsecase();
  final bloc = HotelsBloc(mockUsecase, mockPrefs);
  
  when(mockUsecase.call(any))
      .thenAnswer((_) async => Right(fakeResponse));
  
  bloc.add(ListHotelsEvent(params: params));
  
  expect(
    bloc.stream,
    emitsInOrder([
      HotelsLoadingState(),
      ListHotelsSuccess(hotels: fakeHotels),
    ]),
  );
});
```

### 2. Flexibility 🔄

Easy to swap implementations:

```dart
// Switch from API to local database
@LazySingleton(as: HotelsRepository)
class HotelsLocalRepositoryImpl implements HotelsRepository {
  final HotelsLocalDatasource _localDatasource;
  
  // Same interface, different implementation!
  @override
  Future<Either<Failure, SearchResponse>> listHotels(query) async {
    return await _localDatasource.getHotelsFromDb(query);
  }
}
```

### 3. Separation of Concerns 📦

Each layer has clear responsibilities:

| Layer | Responsibility | Knows About |
|-------|---------------|-------------|
| Presentation | UI, User interactions | Domain only |
| Domain | Business rules | Nothing (pure) |
| Data | Data fetching/storage | Domain contracts |

### 4. Maintainability 🔧

Changes are isolated:

```
API changes? 
→ Only update Data layer

Business rules change?
→ Only update Domain layer

UI redesign?
→ Only update Presentation layer
```

## Common Patterns

### Pattern 1: Either for Error Handling

```dart
// Instead of try-catch everywhere
Future<SearchResponse> listHotels(); // Can throw!

// Use Either
Future<Either<Failure, SearchResponse>> listHotels();

// Handle gracefully
final result = await repository.listHotels();
result.fold(
  (failure) => handleError(failure),
  (success) => showData(success),
);
```

### Pattern 2: Entities vs Models

```dart
// Model (Data layer) - matches API
class HotelModel {
  final String hotelId;
  final String name;
  @JsonKey(name: 'check_in')
  final String checkInTime;
  
  factory HotelModel.fromJson(Map<String, dynamic> json);
}

// Entity (Domain layer) - business focused
class Hotel {
  final String id;
  final String name;
  final DateTime checkIn;
  
  // May have business methods
  bool isAvailable() => checkIn.isAfter(DateTime.now());
}
```

### Pattern 3: Dependency Inversion

```dart
// ❌ Bad: Concrete dependency
class HotelsBloc {
  final HotelsRepositoryImpl repo; // Depends on implementation
}

// ✅ Good: Abstract dependency
class HotelsBloc {
  final HotelsRepository repo; // Depends on interface
}
```

## 🎓 Quick Quiz

**Q1:** Which layer should never import widgets?
<details>
<summary>Answer</summary>
Domain and Data layers. Only Presentation layer can import widgets.
</details>

**Q2:** Where do you put business logic?
<details>
<summary>Answer</summary>
In the Domain layer, specifically in UseCases. BLoCs should be thin and just orchestrate.
</details>

**Q3:** Can the Domain layer import Dio or HTTP packages?
<details>
<summary>Answer</summary>
No! Domain should be pure Dart with no external dependencies. Only Data layer can import these.
</details>

## Next Steps

Now you understand the architecture! Let's see how the API client is set up.

👉 [Continue to API Client Setup](./05-api-client-setup.md)

---

## 📚 Key Takeaways

✅ Three layers: Presentation → Domain ← Data  
✅ Inner layers don't know about outer layers  
✅ Domain defines contracts, Data implements them  
✅ Each layer can be tested independently  
✅ Use Either for elegant error handling  
✅ UseCases contain business logic  
