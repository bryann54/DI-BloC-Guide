# 🔄 Complete Feature Flow

## The Hotels Feature: End-to-End

Let's follow a complete request from button tap to UI update!

## 🎬 The Journey of a Search Request

### Step-by-Step Breakdown

```
USER TAPS "SEARCH HOTELS"
         ↓
[1] Widget → BLoC
         ↓
[2] BLoC → UseCase
         ↓
[3] UseCase → Repository
         ↓
[4] Repository → DataSource
         ↓
[5] DataSource → ClientProvider
         ↓
[6] ClientProvider → Dio → API
         ↓
[7] Response flows back up
         ↓
[8] Widget rebuilds with data
```

### Detailed Flow with Code

## [1] Widget Sends Event

```dart
// hotels_page.dart
class HotelsPage extends StatelessWidget {
  void _searchHotels(BuildContext context) {
    // Get the BLoC
    final bloc = context.read<HotelsBloc>();
    
    // Create search parameters
    final params = GetHotelsParams(
      q: 'Bali Hotels',
      checkInDate: '2024-03-01',
      checkOutDate: '2024-03-05',
      currency: 'USD',
    );
    
    // Send event to BLoC
    bloc.add(ListHotelsEvent(params: params));
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: Text('Hotels')),
      body: Column(
        children: [
          ElevatedButton(
            onPressed: () => _searchHotels(context),
            child: Text('Search Hotels'),
          ),
          Expanded(
            child: BlocBuilder<HotelsBloc, HotelsState>(
              builder: (context, state) {
                // We'll see this part in step [8]
                if (state is HotelsLoadingState) {
                  return Center(child: CircularProgressIndicator());
                }
                
                if (state is ListHotelsSuccess) {
                  return ListView.builder(
                    itemCount: state.hotels.length,
                    itemBuilder: (context, index) {
                      return HotelCard(hotel: state.hotels[index]);
                    },
                  );
                }
                
                if (state is ListHotelsError) {
                  return Center(child: Text('Error: ${state.error}'));
                }
                
                return Center(child: Text('Search for hotels'));
              },
            ),
          ),
        ],
      ),
    );
  }
}
```

**What happened:**
✅ User tapped button  
✅ Created `GetHotelsParams` with search criteria  
✅ Sent `ListHotelsEvent` to BLoC  

---

## [2] BLoC Processes Event

```dart
// hotels_bloc.dart
@injectable
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  final SharedPreferencesManager _preferencesManager;
  final ListHotelsUsecase _listHotelsUsecase;
  final List<PropertyModel> hotels = [];

  HotelsBloc(this._listHotelsUsecase, this._preferencesManager)
      : super(HotelsInitial()) {
    // Register event handler
    on<ListHotelsEvent>(_listHotels);
  }

  FutureOr<void> _listHotels(
      ListHotelsEvent event, Emitter<HotelsState> emit) async {
    
    // Step 2.1: Emit loading state
    emit(HotelsLoadingState());
    
    // Step 2.2: Add user preferences to params
    final params = event.params.copyWith(
        hl: _preferencesManager.getString(
            SharedPreferencesManager.language) ?? 'en');
    
    // Step 2.3: Call use case
    final response = await _listHotelsUsecase.call(params);
    
    // Step 2.4: Handle response and emit appropriate state
    emit(response.fold(
      // Error case
      (error) => ListHotelsError(error: error.toString()),
      
      // Success case
      (response) {
        hotels.clear();
        hotels.addAll(response.properties);
        pagination = response.pagination;
        return ListHotelsSuccess(hotels: hotels);
      },
    ));
  }
}
```

**What happened:**
✅ BLoC received `ListHotelsEvent`  
✅ Emitted `HotelsLoadingState` (UI shows spinner)  
✅ Added user's language preference  
✅ Called UseCase with enhanced params  

---

## [3] UseCase Executes Business Logic

```dart
// list_hotels_usecase.dart
@lazySingleton
class ListHotelsUsecase implements UseCase<SearchResponse, GetHotelsParams> {
  final HotelsRepository _repo;

  ListHotelsUsecase(this._repo);

  @override
  Future<Either<Failure, SearchResponse>> call(GetHotelsParams params) async {
    // Business logic: Set default if query is empty
    final query = params.q.isEmpty ? 'Bali Hotels' : params.q;
    
    // Convert params to API model
    final queryModel = QueryHotelModel(
      engine: params.engine,
      q: query,
      gl: params.gl,
      hl: params.hl,
      currency: params.currency,
      checkInDate: params.checkInDate,
      checkOutDate: params.checkOutDate,
      nextPageToken: params.nextPageToken,
    );
    
    // Call repository
    return await _repo.listHotels(queryModel);
  }
}
```

**What happened:**
✅ Applied business rule (default query)  
✅ Converted domain params to data model  
✅ Called repository interface  

---

## [4] Repository Coordinates Data

```dart
// hotels_repository_impl.dart
@LazySingleton(as: HotelsRepository)
class HotelsRepositoryImpl implements HotelsRepository {
  final HotelsRemoteDatasource _remoteDatasource;

  HotelsRepositoryImpl(this._remoteDatasource);

  @override
  Future<Either<Failure, SearchResponse>> listHotels(
      QueryHotelModel query) async {
    try {
      // Step 4.1: Get data from datasource
      final result = await _remoteDatasource.listHotels(query);
      
      // Step 4.2: Check for server errors
      if (result is ServerError) {
        return Left(ServerFailure(badResponse: result));
      }
      
      // Step 4.3: Convert to domain entity
      final searchResponse = SearchResponse.fromJson(result);
      
      // Step 4.4: Return success
      return Right(searchResponse);
      
    } on ServerException {
      return const Left(ServerFailure());
    }
  }
}
```

**What happened:**
✅ Called remote datasource  
✅ Checked for errors  
✅ Converted JSON to domain entity  
✅ Wrapped result in Either  

---

## [5] DataSource Makes API Call

```dart
// hotels_remote_datasource.dart
@lazySingleton
class HotelsRemoteDatasource {
  final ClientProvider _clientProvider;

  HotelsRemoteDatasource(this._clientProvider);

  Future<dynamic> listHotels(QueryHotelModel queryHotelModel) async {
    try {
      // Convert model to JSON and make API call
      return await _clientProvider.get(
          query: queryHotelModel.toJson()
      );
    } catch (e) {
      debugPrint('listHotels response: $e');
      rethrow;
    }
  }
}

// query_hotel_model.dart
@JsonSerializable()
class QueryHotelModel extends Equatable {
  final String engine;
  final String q;
  @JsonKey(name: 'check_in_date')
  final String checkInDate;
  @JsonKey(name: 'check_out_date')
  final String checkOutDate;

  Map<String, dynamic> toJson() => _$QueryHotelModelToJson(this);
  // Converts to:
  // {
  //   "engine": "google_hotels",
  //   "q": "Bali Hotels",
  //   "check_in_date": "2024-03-01",
  //   "check_out_date": "2024-03-05",
  //   "api_key": "xxx" ← Added by ApiKeyInterceptor
  // }
}
```

**What happened:**
✅ Converted model to JSON  
✅ Called ClientProvider  

---

## [6] ClientProvider → Dio → Network

```dart
// client_provider.dart
@lazySingleton
class ClientProvider {
  final DioClient _dioClient;

  ClientProvider(this._dioClient);

  Future<dynamic> get({String? url, Map<String, dynamic>? query}) async {
    try {
      // Make GET request
      final response = await _dioClient.dio.get(
        url ?? '',
        queryParameters: query,
        options: _dioClient.open,
      );
      
      // Return data on success
      return response.data;
      
    } on DioException catch (error) {
      // Return ServerError on failure
      return ServerError.withError(error: error);
    }
  }
}
```

**Actual HTTP Request:**
```http
GET https://serpapi.com/search?engine=google_hotels&q=Bali+Hotels&check_in_date=2024-03-01&check_out_date=2024-03-05&gl=us&hl=en&currency=USD&api_key=your_key_here
```

**Interceptors in Action:**

```
Request Flow:

Original Request
     ↓
[LoggingInterceptor.onRequest]
→ Prints: "--> GET https://serpapi.com/search"
→ Prints: "queryParameters: {engine: google_hotels, q: Bali Hotels, ...}"
     ↓
[ApiKeyInterceptor.onRequest]
→ Adds: queryParameters['api_key'] = 'your_key'
     ↓
HTTP Call to Server
     ↓
Response Received
     ↓
[LoggingInterceptor.onResponse]
→ Prints: "<-- 200 https://serpapi.com/search"
→ Prints: "Response: {hotels: [...], pagination: {...}}"
     ↓
Return to ClientProvider
```

**What happened:**
✅ Request logged  
✅ API key added  
✅ HTTP call made  
✅ Response logged  
✅ Data returned  

---

## [7] Response Flows Back

### JSON Response
```json
{
  "search_metadata": { ... },
  "search_parameters": { ... },
  "properties": [
    {
      "type": "hotel",
      "name": "Bali Beach Resort",
      "description": "Beachfront resort with pool",
      "rate_per_night": {
        "lowest": "$150",
        "extracted_lowest": 150
      },
      "images": [ ... ],
      "link": "https://..."
    },
    // ... more hotels
  ],
  "pagination": {
    "current_from": 1,
    "current_to": 20,
    "next_page_token": "abc123"
  }
}
```

### DataSource → Repository
```dart
// Raw JSON comes back to datasource
final result = {...}; // JSON above

// Returns to repository
return result;
```

### Repository → UseCase
```dart
// Repository converts JSON to entity
final searchResponse = SearchResponse.fromJson(result);

// Returns wrapped in Either
return Right(searchResponse);
```

### UseCase → BLoC
```dart
// Either<Failure, SearchResponse> comes back
final response = await _listHotelsUsecase.call(params);

// BLoC handles with fold
response.fold(
  (failure) => ListHotelsError(error: failure.toString()),
  (success) => ListHotelsSuccess(hotels: success.properties),
);
```

**What happened:**
✅ JSON converted to `SearchResponse`  
✅ Wrapped in `Either`  
✅ Returned through all layers  

---

## [8] Widget Rebuilds

```dart
BlocBuilder<HotelsBloc, HotelsState>(
  builder: (context, state) {
    // State changes from LoadingState → Success
    
    if (state is ListHotelsSuccess) {
      // Build list of hotels
      return ListView.builder(
        itemCount: state.hotels.length,
        itemBuilder: (context, index) {
          final hotel = state.hotels[index];
          return HotelCard(
            name: hotel.name,
            price: hotel.ratePerNight?.extractedLowest ?? 0,
            image: hotel.thumbnail,
            onTap: () {
              // Navigate to details
            },
          );
        },
      );
    }
    
    return SizedBox.shrink();
  },
)
```

**What happened:**
✅ BlocBuilder detected state change  
✅ Rebuilt widget tree  
✅ Displayed hotel list  

---

## 🎨 Visual Timeline

```
T=0ms
┌────────────────┐
│  User taps     │
│  Search button │
└────────┬───────┘
         │
T=10ms   ▼
┌─────────────────────┐
│ ListHotelsEvent     │ ← Event created
│ sent to BLoC        │
└────────┬────────────┘
         │
T=15ms   ▼
┌─────────────────────┐
│ HotelsLoadingState  │ ← State emitted
│ Widget shows spinner│
└────────┬────────────┘
         │
T=20ms   ▼
┌─────────────────────┐
│ UseCase called      │ ← Business logic
└────────┬────────────┘
         │
T=25ms   ▼
┌─────────────────────┐
│ Repository called   │ ← Data coordination
└────────┬────────────┘
         │
T=30ms   ▼
┌─────────────────────┐
│ DataSource called   │ ← API call prep
└────────┬────────────┘
         │
T=35ms   ▼
┌─────────────────────┐
│ HTTP Request sent   │ ← Network call
└────────┬────────────┘
         │
         ⋮ (Network delay)
         │
T=500ms  ▼
┌─────────────────────┐
│ Response received   │ ← JSON data
└────────┬────────────┘
         │
T=510ms  ▼
┌─────────────────────┐
│ JSON → Entity       │ ← Conversion
└────────┬────────────┘
         │
T=520ms  ▼
┌─────────────────────┐
│ ListHotelsSuccess   │ ← State emitted
│ Widget rebuilds     │
└─────────────────────┘
```

## 🔄 State Transition Diagram

```
┌──────────────────┐
│  HotelsInitial   │ ← App starts
└────────┬─────────┘
         │ User taps search
         ▼
┌──────────────────────┐
│ HotelsLoadingState   │ ← Shows spinner
└────────┬─────────────┘
         │
         ├─────────────────────┐
         │                     │
    If Success            If Error
         │                     │
         ▼                     ▼
┌───────────────────┐   ┌──────────────────┐
│ListHotelsSuccess  │   │ ListHotelsError  │
│(shows hotel list) │   │ (shows error msg)│
└───────┬───────────┘   └──────────────────┘
        │
        │ User scrolls to bottom
        ▼
┌──────────────────┐
│   LoadingMore    │ ← Shows bottom loader
└────────┬─────────┘
         │
         ▼
┌───────────────────┐
│ListHotelsSuccess  │ ← Updated with more hotels
│(appended list)    │
└───────────────────┘
```

## Pagination Flow

When user scrolls to bottom:

```dart
// In widget
scrollController.addListener(() {
  if (scrollController.position.pixels ==
      scrollController.position.maxScrollExtent) {
    // Reached bottom - load more!
    context.read<HotelsBloc>().add(
      LoadMoreHotelsEvent(params: currentParams)
    );
  }
});

// In BLoC
FutureOr<void> _loadMore(
    LoadMoreHotelsEvent event, Emitter<HotelsState> emit) async {
  
  // Don't show full loading, just indicator at bottom
  emit(LoadingMore());
  
  // Use next page token from previous response
  final params = event.params.copyWith(
      nextPageToken: pagination.nextPageToken);
  
  final response = await _listHotelsUsecase.call(params);
  
  emit(response.fold(
    // On error, keep showing current hotels
    (error) => ListHotelsSuccess(hotels: hotels),
    
    // On success, append new hotels
    (more) {
      hotels.addAll(more.properties); // Add to existing list
      pagination.nextPageToken = more.pagination.nextPageToken;
      return ListHotelsSuccess(hotels: hotels);
    },
  ));
}
```

## Error Handling Flow

```
API Call Fails
     ↓
DioException thrown
     ↓
ClientProvider catches
     ↓
Returns ServerError
     ↓
Repository checks: result is ServerError
     ↓
Returns Left(ServerFailure)
     ↓
UseCase returns Either to BLoC
     ↓
BLoC.fold receives Left (failure)
     ↓
Emits ListHotelsError(error message)
     ↓
BlocBuilder receives error state
     ↓
Shows error message to user
```

## Complete Dependency Graph

```
User Action (Tap Search)
         ↓
┌────────────────────┐
│    HotelsPage      │ ← Created by routing
└────────┬───────────┘
         │
         ▼
┌────────────────────┐
│    HotelsBloc      │ ← Injected by getIt
└────────┬───────────┘
         │
         ├────────────────────┬───────────────────────┐
         ▼                    ▼                       ▼
┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐
│ListHotelsUsecase │  │SharedPreferences │  │(Internal State)│
└────────┬─────────┘  └──────────────────┘  └────────────────┘
         │
         ▼
┌──────────────────┐
│HotelsRepository  │ ← Interface
└────────┬─────────┘
         │
         ▼
┌────────────────────────┐
│HotelsRepositoryImpl    │ ← Implementation
└────────┬───────────────┘
         │
         ▼
┌──────────────────────────┐
│HotelsRemoteDatasource    │
└────────┬─────────────────┘
         │
         ▼
┌──────────────────────┐
│   ClientProvider     │
└────────┬─────────────┘
         │
         ▼
┌──────────────────────┐
│     DioClient        │
└────────┬─────────────┘
         │
         ├────────────┬───────────────┐
         ▼            ▼               ▼
┌──────────┐  ┌────────────┐  ┌────────────┐
│   Dio    │  │  API Key   │  │  Base URL  │
└──────────┘  └────────────┘  └────────────┘
```

## 🎓 Understanding the Flow

### Key Observations

1. **One-way data flow**: Event → Processing → State → UI
2. **Layers are independent**: Each can be tested alone
3. **Error handling at every level**: Each layer catches and transforms errors
4. **Immutable states**: States are value objects, not mutable
5. **Single source of truth**: BLoC holds the state

### Common Questions

**Q: Why so many layers?**
<details>
<summary>Answer</summary>
Each layer has ONE job. This makes code easier to understand, test, and change. If API changes, you only touch Data layer. If business rules change, only Domain layer.
</details>

**Q: Why can't Widget talk directly to API?**
<details>
<summary>Answer</summary>
It could, but then:
- Hard to test UI without real API
- Business logic mixed with UI code
- Can't swap data sources easily
- Changes to API break UI directly
</details>

**Q: What if I just need to display data without business logic?**
<details>
<summary>Answer</summary>
Still use layers! UseCase might be thin (just pass-through), but structure stays consistent. Makes it easy to add logic later.
</details>

## Next Steps

You've seen the complete flow! Now let's look at best practices and common patterns.

👉 [Continue to Best Practices](./07-best-practices.md)

---

## 📚 Key Takeaways

✅ Events flow down, States flow up  
✅ Each layer transforms data appropriately  
✅ Errors are caught and handled at every level  
✅ Dependencies are injected, making everything testable  
✅ BLoC orchestrates but doesn't contain business logic  
✅ Pagination is handled by updating state with appended data  
