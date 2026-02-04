# 🎭 BLoC State Management

## What is BLoC?

**BLoC** = **B**usiness **Lo**gic **C**omponent

Think of it as a manager that:
1. Receives **Events** (user actions)
2. Processes them (business logic)
3. Outputs **States** (UI updates)

## 🎨 The BLoC Flow

```
┌──────────────┐
│    Widget    │
│   (The UI)   │
└──────┬───────┘
       │ 1. User taps button
       │    (sends Event)
       ▼
┌──────────────────┐
│      BLoC        │
│  (The Manager)   │
│                  │
│  Event → Logic   │
│  Logic → State   │
└──────┬───────────┘
       │ 2. Sends new State
       │
       ▼
┌──────────────┐
│    Widget    │
│  (Rebuilds)  │
└──────────────┘
```

## Real-World Analogy 🏪

Imagine a coffee shop:

- **Widget** = Customer (asks for coffee)
- **Event** = Order ("I want a latte")
- **BLoC** = Barista (makes the coffee)
- **State** = Order status ("making coffee", "coffee ready")

```dart
// Customer (Widget) places order (Event)
context.read<CoffeeBloc>().add(OrderCoffeeEvent(type: 'latte'));

// Barista (BLoC) processes and updates status (State)
emit(MakingCoffeeState());
// ... make coffee ...
emit(CoffeeReadyState(coffee));

// Customer (Widget) gets notification (Listens to State)
BlocBuilder<CoffeeBloc, CoffeeState>(
  builder: (context, state) {
    if (state is CoffeeReadyState) {
      return Text('Your ${state.coffee} is ready!');
    }
    return Text('Making your coffee...');
  }
)
```

## The Three Parts of BLoC

### 1. Events (What Happens)

Events are user actions or system events:

```dart
// hotels_event.dart
part of 'hotels_bloc.dart';

abstract class HotelsEvent extends Equatable {
  const HotelsEvent();

  @override
  List<Object> get props => [];
}

// Event: User wants to see hotels
class ListHotelsEvent extends HotelsEvent {
  final GetHotelsParams params;

  const ListHotelsEvent({required this.params});
}

// Event: User scrolls to bottom (load more)
class LoadMoreHotelsEvent extends HotelsEvent {
  final GetHotelsParams params;

  const LoadMoreHotelsEvent({required this.params});
}
```

**Why Equatable?**
```dart
// Without Equatable
final event1 = ListHotelsEvent(params: myParams);
final event2 = ListHotelsEvent(params: myParams);
print(event1 == event2); // false (different objects)

// With Equatable
final event1 = ListHotelsEvent(params: myParams);
final event2 = ListHotelsEvent(params: myParams);
print(event1 == event2); // true (same values)
```

### 2. States (How Things Are)

States represent the current situation:

```dart
// hotels_state.dart
part of 'hotels_bloc.dart';

abstract class HotelsState extends Equatable {
  const HotelsState();

  @override
  List<Object> get props => [];
}

// Initial state (nothing happened yet)
class HotelsInitial extends HotelsState {}

// Loading state (fetching data)
class HotelsLoadingState extends HotelsState {}

// Loading more state (pagination)
class LoadingMore extends HotelsState {}

// Success state (data received)
class ListHotelsSuccess extends HotelsState {
  final List<PropertyModel> hotels;

  const ListHotelsSuccess({required this.hotels});
  
  @override
  List<Object> get props => [hotels]; // For comparison
}

// Error state (something went wrong)
class ListHotelsError extends HotelsState {
  final String error;

  const ListHotelsError({required this.error});
  
  @override
  List<Object> get props => [error];
}
```

**State Lifecycle:**
```
HotelsInitial → HotelsLoadingState → ListHotelsSuccess
                                   ↘ ListHotelsError (if fails)
```

### 3. BLoC (The Logic)

The BLoC processes events and emits states:

```dart
// hotels_bloc.dart
@injectable
class HotelsBloc extends Bloc<HotelsEvent, HotelsState> {
  final SharedPreferencesManager _preferencesManager;
  final ListHotelsUsecase _listHotelsUsecase;
  final List<PropertyModel> hotels = []; // Internal data
  late SerpApiPagination pagination;

  // Constructor: Set initial state and register event handlers
  HotelsBloc(this._listHotelsUsecase, this._preferencesManager)
      : super(HotelsInitial()) {
    // When ListHotelsEvent comes in, call _listHotels
    on<ListHotelsEvent>(_listHotels);
    // When LoadMoreHotelsEvent comes in, call _loadMore
    on<LoadMoreHotelsEvent>(_loadMore);
  }

  // Handle ListHotelsEvent
  FutureOr<void> _listHotels(
      ListHotelsEvent event, Emitter<HotelsState> emit) async {
    
    // 1. Emit loading state (UI shows loading indicator)
    emit(HotelsLoadingState());
    
    // 2. Add user's language preference to params
    final params = event.params.copyWith(
        hl: _preferencesManager.getString(
            SharedPreferencesManager.language));
    
    // 3. Call the use case (business logic)
    final response = await _listHotelsUsecase.call(params);
    
    // 4. Handle the response
    emit(response.fold(
      // If error, emit error state
      (error) => ListHotelsError(error: error.toString()),
      
      // If success, emit success state
      (response) {
        hotels.clear();
        hotels.addAll(response.properties);
        pagination = response.pagination;
        return ListHotelsSuccess(hotels: hotels);
      },
    ));
  }

  // Handle LoadMoreHotelsEvent (pagination)
  FutureOr<void> _loadMore(
      LoadMoreHotelsEvent event, Emitter<HotelsState> emit) async {
    
    // 1. Emit loading more state (show loading at bottom)
    emit(LoadingMore());
    
    // 2. Use the next page token from previous response
    final params = event.params.copyWith(
        nextPageToken: pagination.nextPageToken);
    
    // 3. Fetch more data
    final response = await _listHotelsUsecase.call(params);
    
    // 4. Handle response
    emit(response.fold(
      // If error, keep showing current hotels
      (error) => ListHotelsSuccess(hotels: hotels),
      
      // If success, add new hotels to list
      (more) {
        hotels.addAll(more.properties);
        pagination.nextPageToken = more.pagination.nextPageToken;
        pagination.currentFrom = more.pagination.currentFrom;
        pagination.currentTo = more.pagination.currentTo;
        return ListHotelsSuccess(hotels: hotels);
      },
    ));
  }
}
```

## 🔄 Complete Flow Diagram

```
Widget                  BLoC                     UseCase           Repository
  │                       │                         │                  │
  │──ListHotelsEvent──────│                         │                  │
  │                       │                         │                  │
  │                       ├─emit(Loading)───────────┤                  │
  │◄──Loading State───────┤                         │                  │
  │                       │                         │                  │
  │                       ├─call(params)────────────│                  │
  │                       │                         │                  │
  │                       │                         ├──listHotels()────│
  │                       │                         │                  │
  │                       │                         │                  ├─API Call
  │                       │                         │                  │
  │                       │                         │◄─────response────┤
  │                       │                         │                  │
  │                       │◄────Either<Failure,─────┤                  │
  │                       │      SearchResponse>    │                  │
  │                       │                         │                  │
  │                       ├─emit(Success/Error)─────┤                  │
  │◄──Success State───────┤   with hotels           │                  │
  │   (rebuilds UI)       │                         │                  │
```

## Using BLoC in Widgets

### 1. Providing the BLoC

```dart
@RoutePage()
class MainScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MultiBlocProvider(
      providers: [
        // GetIt creates HotelsBloc with all dependencies
        BlocProvider(create: (context) => getIt<HotelsBloc>()),
        BlocProvider(create: (context) => getIt<FavouritesBloc>()),
      ],
      child: // ... your screen
    );
  }
}
```

### 2. Sending Events

```dart
// Method 1: Using context.read (doesn't rebuild)
context.read<HotelsBloc>().add(
  ListHotelsEvent(params: GetHotelsParams(
    checkInDate: '2024-01-01',
    checkOutDate: '2024-01-05',
  ))
);

// Method 2: Using BlocProvider.of
BlocProvider.of<HotelsBloc>(context).add(
  LoadMoreHotelsEvent(params: currentParams)
);
```

### 3. Listening to States

**Option A: BlocBuilder (Rebuild on State Change)**
```dart
BlocBuilder<HotelsBloc, HotelsState>(
  builder: (context, state) {
    // Loading state
    if (state is HotelsLoadingState) {
      return Center(child: CircularProgressIndicator());
    }
    
    // Success state
    if (state is ListHotelsSuccess) {
      return ListView.builder(
        itemCount: state.hotels.length,
        itemBuilder: (context, index) {
          return HotelCard(hotel: state.hotels[index]);
        },
      );
    }
    
    // Error state
    if (state is ListHotelsError) {
      return Center(child: Text('Error: ${state.error}'));
    }
    
    // Initial state
    return Center(child: Text('Search for hotels'));
  },
)
```

**Option B: BlocListener (Side Effects, No Rebuild)**
```dart
BlocListener<HotelsBloc, HotelsState>(
  listener: (context, state) {
    // Show snackbar on error
    if (state is ListHotelsError) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(state.error)),
      );
    }
    
    // Navigate on success
    if (state is ListHotelsSuccess && state.hotels.isEmpty) {
      // Maybe show empty state screen
    }
  },
  child: // ... your widget
)
```

**Option C: BlocConsumer (Both!)**
```dart
BlocConsumer<HotelsBloc, HotelsState>(
  // For side effects
  listener: (context, state) {
    if (state is ListHotelsError) {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text(state.error)),
      );
    }
  },
  // For rebuilding UI
  builder: (context, state) {
    if (state is HotelsLoadingState) {
      return CircularProgressIndicator();
    }
    
    if (state is ListHotelsSuccess) {
      return HotelsList(hotels: state.hotels);
    }
    
    return SizedBox.shrink();
  },
)
```

## 🎯 Best Practices

### 1. Keep BLoC Pure
```dart
// ❌ Bad: UI logic in BLoC
class HotelsBloc {
  void showDialog() {
    // Don't do this!
  }
}

// ✅ Good: Only business logic
class HotelsBloc {
  void _listHotels(event, emit) async {
    emit(HotelsLoadingState());
    final result = await _usecase.call(params);
    emit(result.fold(
      (error) => ListHotelsError(error: error.toString()),
      (data) => ListHotelsSuccess(hotels: data),
    ));
  }
}
```

### 2. Use Meaningful State Names
```dart
// ❌ Bad
class State1 extends HotelsState {}
class State2 extends HotelsState {}

// ✅ Good
class HotelsLoadingState extends HotelsState {}
class ListHotelsSuccess extends HotelsState {}
```

### 3. Handle All States
```dart
// ✅ Always handle all possible states
BlocBuilder<HotelsBloc, HotelsState>(
  builder: (context, state) {
    if (state is HotelsLoadingState) { /* ... */ }
    if (state is ListHotelsSuccess) { /* ... */ }
    if (state is ListHotelsError) { /* ... */ }
    if (state is LoadingMore) { /* ... */ }
    
    // Default case
    return SizedBox.shrink();
  },
)
```

### 4. Don't Emit States in Constructors
```dart
// ❌ Bad
HotelsBloc() : super(HotelsInitial()) {
  emit(HotelsLoadingState()); // Don't emit here!
}

// ✅ Good
HotelsBloc() : super(HotelsInitial()) {
  // Wait for events
}
```

## 📊 State Management Patterns

### Pattern 1: Loading → Success/Error
```dart
emit(LoadingState());
final result = await fetchData();
emit(result.fold(
  (error) => ErrorState(error),
  (data) => SuccessState(data),
));
```

### Pattern 2: Loading More (Pagination)
```dart
// Keep existing data visible
emit(LoadingMore()); // Not full loading

final more = await fetchMore();
more.fold(
  (error) => emit(SuccessState(currentData)), // Keep current on error
  (newData) {
    currentData.addAll(newData);
    emit(SuccessState(currentData));
  },
);
```

### Pattern 3: Optimistic Update
```dart
// Update UI immediately
final updatedList = [...hotels, newHotel];
emit(SuccessState(updatedList));

// Then sync with server
final result = await saveToServer(newHotel);
result.fold(
  (error) {
    // Revert on error
    emit(SuccessState(hotels));
    emit(ErrorState(error));
  },
  (saved) => emit(SuccessState([...hotels, saved])),
);
```

## 🔍 Debugging BLoCs

### BLoC Observer

```dart
// global_bloc_observer.dart
class AppGlobalBlocObserver extends BlocObserver {
  @override
  void onCreate(BlocBase bloc) {
    super.onCreate(bloc);
    print('BLoC Created: ${bloc.runtimeType}');
  }

  @override
  void onEvent(Bloc bloc, Object? event) {
    super.onEvent(bloc, event);
    print('Event: ${event.runtimeType} in ${bloc.runtimeType}');
  }

  @override
  void onChange(BlocBase bloc, Change change) {
    super.onChange(bloc, change);
    print('State Changed: ${change.currentState.runtimeType} → ${change.nextState.runtimeType}');
  }

  @override
  void onError(BlocBase bloc, Object error, StackTrace stackTrace) {
    super.onError(bloc, error, stackTrace);
    print('Error in ${bloc.runtimeType}: $error');
  }
}

// In main.dart
void main() {
  Bloc.observer = AppGlobalBlocObserver();
  runApp(MyApp());
}
```

## 🎓 Quiz Time!

**Q1:** What's the difference between BlocBuilder and BlocListener?
<details>
<summary>Answer</summary>

- **BlocBuilder**: Rebuilds the widget when state changes (for UI updates)
- **BlocListener**: Executes code when state changes but doesn't rebuild (for side effects like showing dialogs, navigation)
</details>

**Q2:** Why do we use `Equatable` for Events and States?
<details>
<summary>Answer</summary>
So BLoC can compare events and states by their values, not by object reference. This prevents unnecessary rebuilds when the same state is emitted.
</details>

**Q3:** When should you use `context.read()` vs `context.watch()`?
<details>
<summary>Answer</summary>

- `context.read<Bloc>()`: When you just want to add an event (doesn't listen to changes)
- `context.watch<Bloc>()`: When you want to rebuild on state changes (usually in build method)
</details>

## Next Steps

Now you understand BLoC! Let's see how it fits into the complete Clean Architecture pattern.

👉 [Continue to Clean Architecture](./04-clean-architecture.md)

---

## 📚 Key Takeaways

✅ BLoC receives Events, processes logic, emits States  
✅ Use BlocBuilder for UI updates, BlocListener for side effects  
✅ Always handle all possible states in your UI  
✅ Keep BLoC pure - no UI code inside BLoC  
✅ Use Equatable to compare events and states by value  
