# API Client Setup

## The HTTP Stack

Your app uses **Dio** (a powerful HTTP client for Flutter) with a layered approach:

```
┌─────────────────────────────────────┐
│    Feature (e.g., HotelsBloc)       │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│        ClientProvider               │  ← High-level API methods
│  (get, post, put, delete)           │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│         DioClient                   │  ← Dio instance + setup
│  (interceptors, config)             │
└────────────────┬────────────────────┘
                 │
┌────────────────▼────────────────────┐
│            Dio                      │  ← Raw HTTP client
│  (actual network calls)             │
└─────────────────────────────────────┘
```

## Step 1: Environment Configuration

### .env File Setup

```
# env/.dev.env
BASE_URL=https://serpapi.com/search
API_KEY=your_api_key_here
```

### Loading Environment Variables

```dart
// In main.dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  
  // Load environment variables
  if (kReleaseMode) {
    await dotenv.load(fileName: "env/.env");
  } else {
    await dotenv.load(fileName: "env/.dev.env");
  }
  
  await configureDependencies();
  runApp(MyApp());
}
```

**Why?**
- Keep secrets out of code
- Different configs for dev/prod
- Easy to change without rebuilding

## Step 2: Register with GetIt

```dart
// lib/core/di/register_modules.dart
@module
abstract class RegisterModules {
  // Provide the base URL from environment
  @Named('BaseUrl')
  String get baseUrl => dotenv.env['BASE_URL']!;

  // Provide the API key from environment
  @Named('ApiKey')
  String get apiKey => dotenv.env['API_KEY']!;

  // Create Dio instance with base URL
  @lazySingleton
  Dio dio(@Named('BaseUrl') String url) => Dio(
      BaseOptions(
        baseUrl: url,
        connectTimeout: const Duration(seconds: 10),
      ));
}
```

**Flow:**
1. GetIt asks for Dio
2. Needs `baseUrl` parameter
3. Gets it from `@Named('BaseUrl')`
4. Creates Dio with that URL
5. Returns configured Dio

## Step 3: DioClient Setup

### The DioClient Class

```dart
@lazySingleton
class DioClient {
  final Options open = Options(
      headers: {'Content-Type': 'application/json'}
  );
  
  final Dio dio;
  final String _apiKey;

  DioClient(this.dio, @Named('ApiKey') this._apiKey) {
    // Add logging interceptor (see network requests in console)
    dio.interceptors.add(DioLogInterceptors(printBody: kDebugMode));
    
    // Add API key to all requests
    dio.interceptors.add(ApiKeyInterceptor(_apiKey));
  }

  DioClient getInstance() => this;
}
```

**What are Interceptors?**

Think of them as checkpoints that every request passes through:

```
Your Request
     ↓
┌────────────────┐
│ Interceptor 1  │  ← Add headers
└────────┬───────┘
         ↓
┌────────────────┐
│ Interceptor 2  │  ← Add API key
└────────┬───────┘
         ↓
┌────────────────┐
│ Interceptor 3  │  ← Log request
└────────┬───────┘
         ↓
   Network Call
         ↓
   Response
         ↓
┌────────────────┐
│ Interceptor 3  │  ← Log response
└────────┬───────┘
         ↓
┌────────────────┐
│ Interceptor 2  │  ← Handle errors
└────────┬───────┘
         ↓
┌────────────────┐
│ Interceptor 1  │  ← Transform data
└────────┬───────┘
         ↓
  Your Response
```

## Step 4: Interceptors

### API Key Interceptor

```dart
class ApiKeyInterceptor extends Interceptor {
  final String apiKey;

  ApiKeyInterceptor(this.apiKey);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    // Add API key to every request automatically
    options.queryParameters['api_key'] = apiKey;
    
    // Continue with the modified request
    super.onRequest(options, handler);
  }
}
```

**What it does:**
```dart
// You write:
dio.get('/search', queryParameters: {'q': 'hotels'});

// API Key Interceptor adds:
// /search?q=hotels&api_key=your_key_here
```

### Logging Interceptor

```dart
class DioLogInterceptors extends Interceptor {
  bool? printBody;

  DioLogInterceptors({this.printBody});

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    debugPrint("--> ${options.method.toUpperCase()} ${options.baseUrl}${options.path}");
    debugPrint("Headers:");
    options.headers.forEach((k, v) => debugPrint('$k: $v'));
    debugPrint("queryParameters:");
    options.queryParameters.forEach((k, v) => debugPrint('$k: $v'));
    
    if (options.data != null && (printBody ?? false)) {
      debugPrint("Body: ${options.data}");
    }
    
    debugPrint("--> END ${options.method.toUpperCase()}");
    return super.onRequest(options, handler);
  }

  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    debugPrint("<-- ${response.statusCode} ${response.requestOptions.baseUrl}${response.requestOptions.path}");
    
    if (printBody ?? false) {
      debugPrint('Response: ${response.data}');
    }
    
    debugPrint("<-- END HTTP");
    return super.onResponse(response, handler);
  }

  @override
  Future onError(DioException err, ErrorInterceptorHandler handler) async {
    debugPrint("<-- ERROR ${err.message}");
    debugPrint("${err.response?.data ?? 'Unknown Error'}");
    debugPrint("<-- End error");
    return super.onError(err, handler);
  }
}
```

**Console Output Example:**
```
--> GET https://serpapi.com/search
Headers:
Content-Type: application/json
queryParameters:
q: Bali Hotels
api_key: abc123
--> END GET

<-- 200 https://serpapi.com/search
Response: { "hotels": [...] }
<-- END HTTP
```

## Step 5: ClientProvider

The high-level API wrapper:

```dart
@lazySingleton
class ClientProvider {
  final DioClient _dioClient;

  ClientProvider(this._dioClient);

  Future<dynamic> get({String? url, Map<String, dynamic>? query}) async {
    try {
      final response = await _dioClient.dio.get(
        url ?? '',
        queryParameters: query,
        options: _dioClient.open,
      );
      return response.data;
    } on DioException catch (error) {
      return ServerError.withError(error: error);
    }
  }

  Future<dynamic> post({String? url, Map<String, dynamic>? payload}) async {
    try {
      final response = await _dioClient.dio.post(
        url ?? '', 
        options: _dioClient.open, 
        data: payload
      );
      return response.data;
    } on DioException catch (error) {
      return ServerError.withError(error: error);
    }
  }

  Future<dynamic> put({String? url, Map<String, dynamic>? payload}) async {
    try {
      final response = await _dioClient.dio.put(
        url ?? '', 
        options: _dioClient.open, 
        data: payload
      );
      return response.data;
    } on DioException catch (error) {
      return ServerError.withError(error: error);
    }
  }
}
```

**Why wrap Dio?**
- Consistent error handling
- Can return either data or ServerError
- Easy to add common logic (auth, retries, etc.)
- Can mock for testing

## Step 6: Error Handling

### ServerError Model

```dart
@JsonSerializable()
class ServerError implements Exception {
  int? _errorCode;
  String _errorMessage = "";

  ServerError();

  ServerError.withError({required DioException error}) {
    _handleError(error);
  }

  getErrorCode() => _errorCode;
  getErrorMessage() => _errorMessage;

  _handleError(DioException error) {
    switch (error.type) {
      case DioExceptionType.cancel:
        _errorMessage = "Request was cancelled";
        break;
      
      case DioExceptionType.connectionTimeout:
        _errorMessage = "Connection timeout";
        break;
      
      case DioExceptionType.receiveTimeout:
        _errorMessage = "Server Error. Please try again later...";
        break;
      
      case DioExceptionType.badResponse:
        _errorMessage = '${error.response?.data}';
        break;
      
      case DioExceptionType.sendTimeout:
        _errorMessage = "Please check your internet connection";
        break;
      
      default:
        _errorMessage = "Connection failed due to internet connection";
    }
    return _errorMessage;
  }
}
```

### Error Flow

```dart
// In ClientProvider
Future<dynamic> get({query}) async {
  try {
    final response = await _dioClient.dio.get(...);
    return response.data;  // Success: return data
  } on DioException catch (error) {
    return ServerError.withError(error: error);  // Error: return ServerError
  }
}

// In Repository
Future<Either<Failure, SearchResponse>> listHotels(query) async {
  try {
    final result = await _remoteDatasource.listHotels(query);
    
    // Check if result is an error
    if (result is ServerError) {
      return Left(ServerFailure(badResponse: result));
    }
    
    // Success: convert and return
    return Right(SearchResponse.fromJson(result));
    
  } on ServerException {
    return const Left(ServerFailure());
  }
}
```

## Complete Request Flow

Let's trace a GET request:

```
1. Feature calls ClientProvider
   _clientProvider.get(query: {'q': 'hotels'})
   ↓
2. ClientProvider calls Dio
   _dioClient.dio.get('', queryParameters: query)
   ↓
3. Request goes through interceptors
   
   LoggingInterceptor:
   → Prints request details
   
   ApiKeyInterceptor:
   → Adds api_key to query params
   
   ↓
4. Dio makes HTTP call
   GET https://serpapi.com/search?q=hotels&api_key=abc123
   ↓
5. Response comes back
   ↓
6. Goes through interceptors (reverse order)
   
   LoggingInterceptor:
   → Prints response details
   
   ↓
7. ClientProvider receives response
   
   If success:
   → Returns response.data
   
   If error:
   → Returns ServerError
   
   ↓
8. Repository handles result
   
   If ServerError:
   → Returns Left(ServerFailure)
   
   If data:
   → Converts to entity
   → Returns Right(entity)
```

## Visual Architecture

```
┌──────────────────────────────────────────┐
│         Environment Variables            │
│  BASE_URL, API_KEY (from .env)          │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│         RegisterModules                  │
│  @Named('BaseUrl') String baseUrl        │
│  @Named('ApiKey') String apiKey          │
│  @lazySingleton Dio dio(baseUrl)         │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│           DioClient                      │
│  - Holds Dio instance                    │
│  - Adds interceptors:                    │
│    * Logging                             │
│    * API Key                             │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│        ClientProvider                    │
│  - get(url, query)                       │
│  - post(url, payload)                    │
│  - put(url, payload)                     │
│  - Handles DioExceptions                 │
│  - Returns data or ServerError           │
└────────────────┬─────────────────────────┘
                 │
                 ▼
┌──────────────────────────────────────────┐
│     HotelsRemoteDatasource               │
│  Uses ClientProvider to make API calls   │
└──────────────────────────────────────────┘
```

## Configuration Options

### Timeouts

```dart
Dio(BaseOptions(
  baseUrl: url,
  connectTimeout: const Duration(seconds: 10),  // Connection timeout
  receiveTimeout: const Duration(seconds: 10),  // Response timeout
  sendTimeout: const Duration(seconds: 10),     // Send timeout
))
```

### Headers

```dart
// Global headers (for all requests)
Dio(BaseOptions(
  headers: {
    'Content-Type': 'application/json',
    'Accept': 'application/json',
  }
))

// Per-request headers
dio.get(
  '/search',
  options: Options(
    headers: {'Authorization': 'Bearer $token'}
  )
)
```

### Response Type

```dart
dio.get(
  '/download',
  options: Options(
    responseType: ResponseType.bytes, // For downloads
  )
)

dio.get(
  '/data',
  options: Options(
    responseType: ResponseType.json, // Default
  )
)
```

## Advanced Patterns

### Pattern 1: Retry Logic

```dart
class RetryInterceptor extends Interceptor {
  final int maxRetries;
  
  RetryInterceptor({this.maxRetries = 3});

  @override
  Future onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.type == DioExceptionType.connectionTimeout) {
      if (err.requestOptions.extra['retryCount'] ?? 0 < maxRetries) {
        err.requestOptions.extra['retryCount'] = 
            (err.requestOptions.extra['retryCount'] ?? 0) + 1;
        
        return handler.resolve(await dio.fetch(err.requestOptions));
      }
    }
    return super.onError(err, handler);
  }
}
```

### Pattern 2: Auth Token

```dart
class AuthInterceptor extends Interceptor {
  final TokenManager tokenManager;
  
  AuthInterceptor(this.tokenManager);

  @override
  void onRequest(RequestOptions options, RequestInterceptorHandler handler) {
    final token = tokenManager.getToken();
    if (token != null) {
      options.headers['Authorization'] = 'Bearer $token';
    }
    super.onRequest(options, handler);
  }

  @override
  Future onError(DioException err, ErrorInterceptorHandler handler) async {
    if (err.response?.statusCode == 401) {
      // Refresh token
      final newToken = await tokenManager.refreshToken();
      
      if (newToken != null) {
        // Retry with new token
        err.requestOptions.headers['Authorization'] = 'Bearer $newToken';
        return handler.resolve(await dio.fetch(err.requestOptions));
      }
    }
    return super.onError(err, handler);
  }
}
```

### Pattern 3: Response Transformation

```dart
class ResponseTransformInterceptor extends Interceptor {
  @override
  void onResponse(Response response, ResponseInterceptorHandler handler) {
    // Unwrap nested data
    if (response.data is Map && response.data['data'] != null) {
      response.data = response.data['data'];
    }
    super.onResponse(response, handler);
  }
}
```

## Testing the API Client

```dart
// Mock ClientProvider
class MockClientProvider extends Mock implements ClientProvider {}

test('fetches hotels successfully', () async {
  final mockClient = MockClientProvider();
  final datasource = HotelsRemoteDatasource(mockClient);
  
  when(mockClient.get(query: any))
      .thenAnswer((_) async => {'hotels': []});
  
  final result = await datasource.listHotels(queryModel);
  
  expect(result, isA<Map>());
  verify(mockClient.get(query: any)).called(1);
});
```

## Quiz Time!

**Q1:** Why use interceptors instead of adding logic directly in each request?
<details>
<summary>Answer</summary>
Interceptors apply to ALL requests automatically. You write the logic once (like adding API key) and it works everywhere. Makes code DRY (Don't Repeat Yourself).
</details>

**Q2:** What's the advantage of returning `ServerError` instead of throwing?
<details>
<summary>Answer</summary>
It lets the Repository handle errors gracefully with Either<Failure, Success>. The caller can pattern match and handle different error types elegantly without try-catch blocks.
</details>

**Q3:** Why separate ClientProvider from Dio?
<details>
<summary>Answer</summary>
- Easier to test (mock ClientProvider)
- Consistent error handling
- Can add common logic without modifying Dio
- Abstraction layer if you want to switch HTTP libraries
</details>

## Next Steps

Now you understand the full stack! Let's see everything working together in a complete feature.

 [Continue to Complete Feature Flow](./06-complete-feature-flow.md)

---

## Key Takeaways

✅ Environment variables keep secrets safe  
✅ Interceptors add functionality to all requests  
✅ ClientProvider wraps Dio for better error handling  
✅ Logging interceptor helps debug API calls  
✅ ServerError provides structured error information  
✅ Dependency injection makes the whole stack testable  
