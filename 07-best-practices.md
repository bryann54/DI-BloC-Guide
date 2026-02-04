#  Best Practices & Common Patterns

## Project Structure

### Recommended Folder Structure

```
lib/
├── main.dart
├── common/
│   ├── helpers/
│   │   ├── app_router.dart
│   │   ├── base_usecase.dart
│   │   └── date_helpers.dart
│   ├── notifiers/
│   │   └── locale_provider.dart
│   ├── res/
│   │   ├── l10n.dart
│   │   └── theme_helper.dart
│   ├── utils/
│   │   └── debouncer.dart
│   └── widgets/
│       ├── global_bloc_observer.dart
│       └── loading_indicator.dart
│
├── core/
│   ├── api_client/
│   │   ├── client/
│   │   │   ├── api_key_interceptor.dart
│   │   │   ├── dio_client.dart
│   │   │   └── logging_interceptor.dart
│   │   ├── models/
│   │   │   └── server_error.dart
│   │   └── client_provider.dart
│   ├── di/
│   │   ├── injector.dart
│   │   ├── injector.config.dart ← Generated
│   │   └── register_modules.dart
│   ├── errors/
│   │   ├── exceptions.dart
│   │   └── failures.dart
│   └── storage/
│       └── storage_preference_manager.dart
│
└── features/
    └── hotels/
        ├── data/
        │   ├── datasources/
        │   │   └── hotels_remote_datasource.dart
        │   ├── models/
        │   │   ├── query_hotel_model.dart
        │   │   ├── query_hotel_model.g.dart ← Generated
        │   │   └── search_response.dart
        │   └── repositories/
        │       └── hotels_repository_impl.dart
        ├── domain/
        │   ├── repositories/
        │   │   └── hotels_repository.dart
        │   └── usecases/
        │       └── list_hotels_usecase.dart
        └── presentation/
            ├── bloc/
            │   ├── hotels_bloc.dart
            │   ├── hotels_event.dart
            │   └── hotels_state.dart
            ├── pages/
            │   └── hotels_page.dart
            └── widgets/
                ├── hotel_card.dart
                └── hotel_list.dart
```

## Naming Conventions

### Files
```dart
// ✅ Good
hotels_bloc.dart
hotels_repository.dart
list_hotels_usecase.dart

// ❌ Bad
HotelsBloc.dart
hotels-repository.dart
listHotelsUseCase.dart
```

### Classes
```dart
// ✅ Good
class HotelsBloc { }
class ListHotelsUsecase { }
class HotelsRemoteDatasource { }

// ❌ Bad
class hotelsBloc { }
class list_hotels_usecase { }
class Hotels_Remote_Datasource { }
```

### Constants
```dart
// ✅ Good
static const String language = 'language';
const Duration timeout = Duration(seconds: 10);

// ❌ Bad
static const String LANGUAGE = 'language';
const timeout = Duration(seconds: 10);
```

### Private Variables
```dart
// ✅ Good
class HotelsBloc {
  final ListHotelsUsecase _listHotelsUsecase;
  final SharedPreferencesManager _preferencesManager;
}

// ❌ Bad (public when should be private)
class HotelsBloc {
  final ListHotelsUsecase listHotelsUsecase;
  final SharedPreferencesManager preferencesManager;
}
```

## BLoC Best Practices

### 1. Keep BLoCs Thin

```dart
// ❌ Bad - Too much logic in BLoC
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  FutureOr<void> _listHotels(event, emit) async {
    emit(HotelsLoadingState());
    
    // Business logic here - DON'T DO THIS!
    final query = event.params.q.isEmpty ? 'Bali Hotels' : event.params.q;
    final checkIn = DateTime.parse(event.params.checkInDate);
    
    if (checkIn.isBefore(DateTime.now())) {
      emit(ListHotelsError(error: 'Invalid date'));
      return;
    }
    
    // API call logic here - DON'T DO THIS!
    final response = await dio.get('/search', queryParameters: {...});
    
    emit(ListHotelsSuccess(hotels: response.data));
  }
}

// ✅ Good - Delegate to UseCase
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  final ListHotelsUsecase _usecase;
  
  FutureOr<void> _listHotels(event, emit) async {
    emit(HotelsLoadingState());
    
    // Just orchestrate - business logic is in UseCase
    final response = await _usecase.call(event.params);
    
    emit(response.fold(
      (error) => ListHotelsError(error: error.toString()),
      (data) => ListHotelsSuccess(hotels: data.properties),
    ));
  }
}
```

### 2. Always Handle All States

```dart
// ❌ Bad - Missing states
BlocBuilder<HotelsBloc, HotelsState>(
  builder: (context, state) {
    if (state is ListHotelsSuccess) {
      return HotelsList(hotels: state.hotels);
    }
    return SizedBox.shrink(); // What about loading? Error?
  },
)

// ✅ Good - Handle everything
BlocBuilder<HotelsBloc, HotelsState>(
  builder: (context, state) {
    if (state is HotelsLoadingState) {
      return Center(child: CircularProgressIndicator());
    }
    
    if (state is ListHotelsSuccess) {
      return HotelsList(hotels: state.hotels);
    }
    
    if (state is ListHotelsError) {
      return ErrorWidget(message: state.error);
    }
    
    if (state is LoadingMore) {
      return Column(
        children: [
          HotelsList(hotels: state.hotels),
          CircularProgressIndicator(),
        ],
      );
    }
    
    // Initial state
    return EmptyState(message: 'Search for hotels');
  },
)
```

### 3. Use BlocConsumer for Side Effects

```dart
// ✅ Good - Separate concerns
BlocConsumer<HotelsBloc, HotelsState>(
  // Side effects (navigation, snackbars, dialogs)
  listener: (context, state) {
    if (state is ListHotelsError) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(state.error)),
      );
    }
    
    if (state is ListHotelsSuccess && state.hotels.isEmpty) {
      // Maybe navigate to empty state screen
      context.router.push(EmptyStateRoute());
    }
  },
  // UI building
  builder: (context, state) {
    if (state is HotelsLoadingState) {
      return LoadingIndicator();
    }
    
    if (state is ListHotelsSuccess) {
      return HotelsList(hotels: state.hotels);
    }
    
    return SizedBox.shrink();
  },
)
```

### 4. Don't Emit States in Constructors

```dart
// ❌ Bad
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  HotelsBloc(this._usecase) : super(HotelsInitial()) {
    emit(HotelsLoadingState()); // Don't emit here!
    _loadInitialData();
  }
}

// ✅ Good - Use events
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  HotelsBloc(this._usecase) : super(HotelsInitial()) {
    on<InitializeEvent>(_initialize);
  }
}

// In widget
void initState() {
  super.initState();
  context.read<HotelsBloc>().add(InitializeEvent());
}
```

### 5. Close Resources Properly

```dart
// ✅ Good - Clean up
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  StreamSubscription? _subscription;
  
  @override
  Future<void> close() {
    _subscription?.cancel();
    return super.close();
  }
}
```

## Dependency Injection Patterns

### 1. Named Dependencies

```dart
// ✅ When you have multiple instances of same type
@module
abstract class RegisterModules {
  @Named('BaseUrl')
  String get baseUrl => dotenv.env['BASE_URL']!;
  
  @Named('ApiKey')
  String get apiKey => dotenv.env['API_KEY']!;
  
  @lazySingleton
  DioClient client(@Named('BaseUrl') String url, @Named('ApiKey') String key) {
    return DioClient(url, key);
  }
}
```

### 2. Environment-Based Registration

```dart
// ✅ Different implementations for dev/prod
@module
abstract class RegisterModules {
  @dev
  @lazySingleton
  HotelsRepository devRepo(HotelsLocalDatasource local) {
    return HotelsLocalRepositoryImpl(local);
  }
  
  @prod
  @LazySingleton(as: HotelsRepository)
  HotelsRepositoryImpl prodRepo(HotelsRemoteDatasource remote) {
    return HotelsRepositoryImpl(remote);
  }
}

// In main.dart
await configureDependencies(environment: Environment.dev);
```

### 3. PreResolve for Async Dependencies

```dart
// ✅ Good - PreResolve ensures SharedPreferences is ready
@module
abstract class RegisterModules {
  @preResolve
  Future<SharedPreferences> prefs() async {
    return await SharedPreferences.getInstance();
  }
}

// ❌ Bad - Might not be ready when needed
@module
abstract class RegisterModules {
  @lazySingleton
  SharedPreferences prefs() {
    return SharedPreferences.getInstance(); // Returns Future!
  }
}
```

## Error Handling Patterns

### 1. Use Either for Graceful Errors

```dart
// ✅ Good - Explicit error handling
Future<Either<Failure, List<Hotel>>> getHotels() async {
  try {
    final data = await api.fetch();
    return Right(data);
  } catch (e) {
    return Left(ServerFailure());
  }
}

// Usage
final result = await getHotels();
result.fold(
  (error) => showError(error),
  (hotels) => displayHotels(hotels),
);

// ❌ Bad - Exceptions everywhere
Future<List<Hotel>> getHotels() async {
  return await api.fetch(); // What if it throws?
}

// Usage requires try-catch everywhere
try {
  final hotels = await getHotels();
  displayHotels(hotels);
} catch (e) {
  showError(e);
}
```

### 2. Specific Failure Types

```dart
// ✅ Good - Specific failures
abstract class Failure extends Equatable {
  const Failure();
}

class ServerFailure extends Failure {
  final ServerError? badResponse;
  const ServerFailure({this.badResponse});
  
  @override
  List<Object?> get props => [badResponse];
  
  @override
  String toString() => badResponse?.getErrorMessage() ?? 'Server error';
}

class NetworkFailure extends Failure {
  @override
  List<Object?> get props => [];
  
  @override
  String toString() => 'No internet connection';
}

class CacheFailure extends Failure {
  @override
  List<Object?> get props => [];
  
  @override
  String toString() => 'Cache error';
}

// Handle specifically
result.fold(
  (failure) {
    if (failure is NetworkFailure) {
      showNoInternetDialog();
    } else if (failure is ServerFailure) {
      showServerErrorMessage(failure.toString());
    }
  },
  (data) => showData(data),
);
```

### 3. Error State in BLoC

```dart
// ✅ Good - Preserve data on error
class HotelsState extends Equatable {
  final List<Hotel> hotels;
  final String? error;
  final bool isLoading;
  
  const HotelsState({
    this.hotels = const [],
    this.error,
    this.isLoading = false,
  });
  
  HotelsState copyWith({
    List<Hotel>? hotels,
    String? error,
    bool? isLoading,
  }) {
    return HotelsState(
      hotels: hotels ?? this.hotels,
      error: error,
      isLoading: isLoading ?? this.isLoading,
    );
  }
}

// In BLoC
emit(state.copyWith(isLoading: true));

final result = await usecase.call(params);

result.fold(
  (error) => emit(state.copyWith(
    isLoading: false,
    error: error.toString(),
  )),
  (newHotels) => emit(state.copyWith(
    isLoading: false,
    hotels: newHotels,
    error: null,
  )),
);

// ❌ Bad - Lose data on error
emit(LoadingState());
// ... error occurs ...
emit(ErrorState(error)); // User loses their data!
```

## Testing Patterns

### 1. Test BLoC

```dart
void main() {
  late HotelsBloc bloc;
  late MockListHotelsUsecase mockUsecase;
  late MockSharedPreferencesManager mockPrefs;

  setUp(() {
    mockUsecase = MockListHotelsUsecase();
    mockPrefs = MockSharedPreferencesManager();
    bloc = HotelsBloc(mockUsecase, mockPrefs);
  });

  tearDown(() {
    bloc.close();
  });

  group('ListHotelsEvent', () {
    final params = GetHotelsParams(
      checkInDate: '2024-01-01',
      checkOutDate: '2024-01-05',
    );

    test('emits [Loading, Success] when usecase succeeds', () async {
      // Arrange
      when(() => mockPrefs.getString(any())).thenReturn('en');
      when(() => mockUsecase.call(any()))
          .thenAnswer((_) async => Right(fakeSearchResponse));

      // Act
      bloc.add(ListHotelsEvent(params: params));

      // Assert
      await expectLater(
        bloc.stream,
        emitsInOrder([
          HotelsLoadingState(),
          isA<ListHotelsSuccess>()
              .having((s) => s.hotels.length, 'hotels count', 5),
        ]),
      );
    });

    test('emits [Loading, Error] when usecase fails', () async {
      // Arrange
      when(() => mockPrefs.getString(any())).thenReturn('en');
      when(() => mockUsecase.call(any()))
          .thenAnswer((_) async => Left(ServerFailure()));

      // Act
      bloc.add(ListHotelsEvent(params: params));

      // Assert
      await expectLater(
        bloc.stream,
        emitsInOrder([
          HotelsLoadingState(),
          isA<ListHotelsError>()
              .having((s) => s.error, 'error message', isNotEmpty),
        ]),
      );
    });
  });
}
```

### 2. Test UseCase

```dart
void main() {
  late ListHotelsUsecase usecase;
  late MockHotelsRepository mockRepo;

  setUp(() {
    mockRepo = MockHotelsRepository();
    usecase = ListHotelsUsecase(mockRepo);
  });

  test('should get hotels from repository', () async {
    // Arrange
    final params = GetHotelsParams(
      checkInDate: '2024-01-01',
      checkOutDate: '2024-01-05',
    );
    
    when(() => mockRepo.listHotels(any()))
        .thenAnswer((_) async => Right(fakeSearchResponse));

    // Act
    final result = await usecase.call(params);

    // Assert
    expect(result, Right(fakeSearchResponse));
    verify(() => mockRepo.listHotels(any())).called(1);
  });

  test('should use default query when q is empty', () async {
    // Arrange
    final params = GetHotelsParams(
      q: '',
      checkInDate: '2024-01-01',
      checkOutDate: '2024-01-05',
    );

    when(() => mockRepo.listHotels(any()))
        .thenAnswer((_) async => Right(fakeSearchResponse));

    // Act
    await usecase.call(params);

    // Assert
    final captured = verify(() => mockRepo.listHotels(captureAny()))
        .captured
        .first as QueryHotelModel;
    
    expect(captured.q, 'Bali Hotels'); // Default value
  });
}
```

### 3. Test Repository

```dart
void main() {
  late HotelsRepositoryImpl repository;
  late MockHotelsRemoteDatasource mockDatasource;

  setUp(() {
    mockDatasource = MockHotelsRemoteDatasource();
    repository = HotelsRepositoryImpl(mockDatasource);
  });

  test('should return SearchResponse when call is successful', () async {
    // Arrange
    final query = QueryHotelModel(...);
    final jsonResponse = {'properties': [...], 'pagination': {...}};
    
    when(() => mockDatasource.listHotels(query))
        .thenAnswer((_) async => jsonResponse);

    // Act
    final result = await repository.listHotels(query);

    // Assert
    expect(result, isA<Right<Failure, SearchResponse>>());
    result.fold(
      (l) => fail('Should be Right'),
      (r) => expect(r, isA<SearchResponse>()),
    );
  });

  test('should return ServerFailure when call returns ServerError', () async {
    // Arrange
    final query = QueryHotelModel(...);
    final serverError = ServerError()
        .setErrorMessage('Server error');
    
    when(() => mockDatasource.listHotels(query))
        .thenAnswer((_) async => serverError);

    // Act
    final result = await repository.listHotels(query);

    // Assert
    expect(result, isA<Left<Failure, SearchResponse>>());
    result.fold(
      (l) => expect(l, isA<ServerFailure>()),
      (r) => fail('Should be Left'),
    );
  });
}
```

## Performance Patterns

### 1. Use Const Constructors

```dart
// ✅ Good - Reuses instances
class ListHotelsSuccess extends HotelsState {
  final List<PropertyModel> hotels;

  const ListHotelsSuccess({required this.hotels});
}

// Usage
emit(const ListHotelsSuccess(hotels: []));

// ❌ Bad - Creates new instance every time
class ListHotelsSuccess extends HotelsState {
  final List<PropertyModel> hotels;

  ListHotelsSuccess({required this.hotels});
}
```

### 2. Debounce Search Input

```dart
// ✅ Good - Avoid too many API calls
class SearchHotels extends StatefulWidget {
  @override
  State<SearchHotels> createState() => _SearchHotelsState();
}

class _SearchHotelsState extends State<SearchHotels> {
  final _debouncer = Debouncer(milliseconds: 500);
  
  void _onSearchChanged(String query) {
    _debouncer.run(() {
      context.read<HotelsBloc>().add(
        SearchHotelsEvent(query: query),
      );
    });
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      onChanged: _onSearchChanged,
      decoration: InputDecoration(hintText: 'Search hotels'),
    );
  }
}

// Debouncer utility
class Debouncer {
  final int milliseconds;
  Timer? _timer;

  Debouncer({required this.milliseconds});

  void run(VoidCallback action) {
    _timer?.cancel();
    _timer = Timer(Duration(milliseconds: milliseconds), action);
  }
  
  void dispose() {
    _timer?.cancel();
  }
}
```

### 3. Lazy Load Images

```dart
// ✅ Good - Load images as needed
ListView.builder(
  itemCount: hotels.length,
  itemBuilder: (context, index) {
    return HotelCard(
      hotel: hotels[index],
      // Image loads when card is visible
      image: CachedNetworkImage(
        imageUrl: hotels[index].thumbnail,
        placeholder: (context, url) => Shimmer(...),
        errorWidget: (context, url, error) => Icon(Icons.error),
      ),
    );
  },
)
```

## Code Organization Tips

### 1. Group Related Code

```dart
// ✅ Good - Related code together
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  // Dependencies
  final ListHotelsUsecase _listHotelsUsecase;
  final SharedPreferencesManager _preferencesManager;
  
  // State
  final List<PropertyModel> hotels = [];
  late SerpApiPagination pagination;
  
  // Constructor
  HotelsBloc(this._listHotelsUsecase, this._preferencesManager)
      : super(HotelsInitial()) {
    on<ListHotelsEvent>(_listHotels);
    on<LoadMoreHotelsEvent>(_loadMore);
  }
  
  // Event handlers
  FutureOr<void> _listHotels(event, emit) async { }
  FutureOr<void> _loadMore(event, emit) async { }
  
  // Helper methods (if any)
  void _clearCache() { }
}
```

### 2. Use Extensions for Readability

```dart
// ✅ Good - Extensions for common operations
extension DateTimeExtension on DateTime {
  String toApiFormat() => DateFormat('yyyy-MM-dd').format(this);
  
  bool isToday() {
    final now = DateTime.now();
    return year == now.year && month == now.month && day == now.day;
  }
}

extension StringExtension on String {
  bool get isValidEmail {
    return RegExp(r'^[\w-\.]+@([\w-]+\.)+[\w-]{2,4}$').hasMatch(this);
  }
}

// Usage
final date = DateTime.now();
final apiDate = date.toApiFormat(); // Clean!

if (email.isValidEmail) { }
```

### 3. Use Meaningful Variable Names

```dart
// ❌ Bad
final r = await _uc.call(p);
final h = r.fold((e) => [], (d) => d.p);

// ✅ Good
final result = await _listHotelsUsecase.call(params);
final hotels = result.fold(
  (error) => <Hotel>[],
  (response) => response.properties,
);
```

## Common Pitfalls to Avoid

### 1. Don't Mix Concerns

```dart
// ❌ Bad - UI logic in BLoC
class HotelsBloc {
  void showSuccessDialog() {
    Get.dialog(...); // UI code in BLoC!
  }
}

// ✅ Good - BLoC only emits states
class HotelsBloc {
  emit(HotelsSuccess(showDialog: true));
}

// Widget handles UI
BlocListener<HotelsBloc, HotelsState>(
  listener: (context, state) {
    if (state is HotelsSuccess && state.showDialog) {
      showDialog(...);
    }
  },
)
```

### 2. Don't Ignore Dispose

```dart
// ❌ Bad - Memory leak
class MyWidget extends StatefulWidget {
  @override
  State<MyWidget> createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> {
  final _controller = TextEditingController();
  final _scrollController = ScrollController();
  
  // Missing dispose!
}

// ✅ Good - Clean up
class _MyWidgetState extends State<MyWidget> {
  final _controller = TextEditingController();
  final _scrollController = ScrollController();
  
  @override
  void dispose() {
    _controller.dispose();
    _scrollController.dispose();
    super.dispose();
  }
}
```

### 3. Don't Emit Same State

```dart
// ❌ Bad - Unnecessary emissions
emit(ListHotelsSuccess(hotels: currentHotels));
emit(ListHotelsSuccess(hotels: currentHotels)); // Same state!

// ✅ Good - Check before emitting
if (state != newState) {
  emit(newState);
}

// Or use Equatable (automatic)
class ListHotelsSuccess extends Equatable {
  final List<Hotel> hotels;
  
  @override
  List<Object?> get props => [hotels];
}
```

##  Final Checklist

Before pushing code, check:

### Architecture
- [ ] BLoC only contains orchestration logic
- [ ] Business logic is in UseCases
- [ ] Data fetching is in DataSources
- [ ] Domain layer has no dependencies on outer layers

### Dependency Injection
- [ ] All classes are registered with GetIt
- [ ] Using appropriate lifecycle (@injectable vs @lazySingleton)
- [ ] No manual object creation in features
- [ ] Regenerated injector after changes

### State Management
- [ ] All states extend Equatable
- [ ] All events extend Equatable
- [ ] Handled all possible states in UI
- [ ] No state emission in constructors

### Error Handling
- [ ] Using Either<Failure, Success>
- [ ] Specific failure types
- [ ] Errors handled at every layer
- [ ] User-friendly error messages

### Testing
- [ ] Unit tests for BLoCs
- [ ] Unit tests for UseCases
- [ ] Unit tests for Repositories
- [ ] Mocked all dependencies

### Performance
- [ ] Using const constructors
- [ ] Debouncing user input
- [ ] Lazy loading where appropriate
- [ ] Disposed controllers and streams

---

##  Congratulations!

You now understand:
- ✅ Dependency Injection with GetIt & Injectable
- ✅ State Management with BLoC
- ✅ Clean Architecture patterns
- ✅ API client setup
- ✅ Complete feature implementation
- ✅ Best practices and common patterns

Keep practicing these patterns, and soon they'll become second nature!

Happy coding! 
