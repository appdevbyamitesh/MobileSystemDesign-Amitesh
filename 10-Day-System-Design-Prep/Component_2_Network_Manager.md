## 2. Network Request Manager

### 2.1 Concept Explanation

**Simple Analogy**: Think of a network manager like a personal assistant who makes phone calls for you:
- **Request Building** = Dialing the right number with the right information
- **Retry Logic** = Calling again if the line is busy or no one answers
- **Authentication** = Providing valid credentials to get through
- **Cancellation** = Hanging up if you change your mind

**Problem It Solves**: Making network requests in iOS apps is complex:
- Need to handle failures gracefully (network drops, server errors)
- Must retry transient failures without retrying destructive operations
- Authentication tokens expire and need refresh
- Requests can be cancelled when user navigates away
- Need testability (mock responses, inject dependencies)

**Real App Scenario**: Uber app fetching ride status, Spotify loading playlists, Banking app checking balance

### 2.2 Requirements & Constraints

**Functional Requirements**:
- Build type-safe HTTP requests (GET, POST, PUT, DELETE)
- Parse JSON responses with Codable
- Inject authentication tokens automatically
- Refresh expired tokens without user intervention
- Retry failed requests with backoff strategy
- Cancel in-flight requests
- Log requests/responses for debugging (with PII redaction)

**Non-Functional Requirements**:
| Constraint | Target | Reasoning |
|------------|--------|-----------|
| Request timeout | 30s | Balance UX vs reliability |
| Retry attempts | 3 max | Avoid infinite loops |
| Backoff delay | 1s, 2s, 4s (exponential) | Reduce server load |
| Token refresh | Atomic | Prevent multiple refresh calls |
| Memory | Minimal | Don't cache all responses |
| Testability | 100% mockable | Unit/integration tests |

**Mobile-Specific Constraints**:
- Network can switch (Wi-Fi → Cellular → Offline)
- Background time limits (30s for task completion)
- Battery efficiency (batch requests, avoid polling)
- User data limits (Wi-Fi preferred for large downloads)

### 2.3 Architecture

```
┌─────────────────────────────────────────────────┐
│              NetworkManager                     │
│  (Public API: request, upload, download)        │
└─────────┬──────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────┐
│           RequestBuilder                        │
│  • Base URL                                     │
│  • Path + query params                          │
│  • Headers + body                               │
└─────────┬───────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────┐
│         InterceptorChain                        │
│  1. AuthInterceptor (inject token)              │
│  2. LoggingInterceptor (log request/response)   │
│  3. RetryInterceptor (handle failures)          │
└─────────┬───────────────────────────────────────┘
          │
          ▼
┌─────────────────────────────────────────────────┐
│          URLSessionClient                       │
│  • Execute HTTP request                         │
│  • Parse response                               │
│  • Handle errors                                │
└─────────────────────────────────────────────────┘
```

**Module Responsibilities**:

1. **NetworkManager**: High-level API
   - Generic request method with Codable
   - Task cancellation tracking
   - Dependency injection point

2. **RequestBuilder**: Type-safe request construction
   - Fluent API for building requests
   - Validates required parameters
   - Encodes request body

3. **Interceptor Chain**: Request/response middleware
   - Auth injection
   - Logging (with redaction)
   - Retry logic

4. **URLSessionClient**: Protocol wrapper around URLSession
   - Enables mocking for tests
   - Handles low-level HTTP details

### 2.4 Algorithms & Data Structures

**Exponential Backoff with Jitter**:
```
delay = min(maxDelay, baseDelay * 2^attempt) + random(0, jitter)

Example:
Attempt 0: 1s + random(0, 0.5s) = 1.0-1.5s
Attempt 1: 2s + random(0, 0.5s) = 2.0-2.5s
Attempt 2: 4s + random(0, 0.5s) = 4.0-4.5s
```

**Why Jitter?**: Prevents thundering herd when many clients retry simultaneously.

**Retry Decision Matrix**:
| Error Type | Retry? | Idempotent Only? |
|------------|--------|------------------|
| Network timeout | Yes | Yes |
| 5xx server error | Yes | Yes |
| 429 rate limit | Yes | No (use Retry-After header) |
| 401 unauthorized | Try refresh token | No |
| 400 bad request | No | No |
| No network | No (fail fast) | N/A |

**Time Complexity**:
- Request execution: O(1) (single HTTP call)
- Retry with backoff: O(n) where n = retry count (max 3)
- Token refresh: O(1) with mutex (only one refresh at a time)

### 2.5 Swift Implementation

#### 2.5.1 HTTP Method & Request Protocol

```swift
import Foundation

enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case delete = "DELETE"
    case patch = "PATCH"
}

protocol NetworkRequest {
    associatedtype Response: Decodable
    
    var method: HTTPMethod { get }
    var path: String { get }
    var queryParameters: [String: String]? { get }
    var headers: [String: String]? { get }
    var body: Encodable? { get }
}

// Default implementations
extension NetworkRequest {
    var queryParameters: [String: String]? { nil }
    var headers: [String: String]? { nil }
    var body: Encodable? { nil }
}
```

#### 2.5.2 URL Session Protocol (for testing)

```swift
protocol URLSessionProtocol {
    func data(for request: URLRequest) async throws -> (Data, URLResponse)
}

extension URLSession: URLSessionProtocol {
    func data(for request: URLRequest) async throws -> (Data, URLResponse) {
        return try await data(for: request, delegate: nil)
    }
}
```

#### 2.5.3 Network Errors

```swift
enum NetworkError: Error, LocalizedError {
    case invalidURL
    case invalidResponse
    case httpError(statusCode: Int, data: Data?)
    case decodingError(Error)
    case unauthorized
    case noNetwork
    case timeout
    case cancelled
    case unknown(Error)
    
    var errorDescription: String? {
        switch self {
        case .invalidURL:
            return "Invalid URL provided"
        case .invalidResponse:
            return "Server returned invalid response"
        case .httpError(let code, _):
            return "HTTP error: \(code)"
        case .decodingError:
            return "Failed to decode response"
        case .unauthorized:
            return "Unauthorized - please log in"
        case .noNetwork:
            return "No network connection"
        case .timeout:
            return "Request timed out"
        case .cancelled:
            return "Request was cancelled"
        case .unknown(let error):
            return "Unknown error: \(error.localizedDescription)"
        }
    }
    
    var isRetryable: Bool {
        switch self {
        case .timeout, .httpError(let code, _):
            return code >= 500 || code == 429
        case .noNetwork:
            return false  // Fail fast, don't retry
        case .unauthorized, .invalidURL, .invalidResponse, .decodingError, .cancelled:
            return false
        case .unknown:
            return true  // Default to retry for unknown errors
        }
    }
    
    var requiresAuthentication: Bool {
        switch self {
        case .unauthorized, .httpError(401, _):
            return true
        default:
            return false
        }
    }
}
```

#### 2.5.4 Request Builder

```swift
struct RequestBuilder {
    private let baseURL: URL
    
    init(baseURL: URL) {
        self.baseURL = baseURL
    }
    
    func build<T: NetworkRequest>(_ request: T) throws -> URLRequest {
        // 1. Build URL with path
        var components = URLComponents(url: baseURL, resolvingAgainstBaseURL: true)!
        components.path = request.path
        
        // 2. Add query parameters
        if let queryParams = request.queryParameters, !queryParams.isEmpty {
            components.queryItems = queryParams.map {
                URLQueryItem(name: $0.key, value: $0.value)
            }
        }
        
        guard let url = components.url else {
            throw NetworkError.invalidURL
        }
        
        // 3. Create URLRequest
        var urlRequest = URLRequest(url: url)
        urlRequest.httpMethod = request.method.rawValue
        urlRequest.timeoutInterval = 30
        
        // 4. Add headers
        urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
        urlRequest.setValue("application/json", forHTTPHeaderField: "Accept")
        
        if let headers = request.headers {
            for (key, value) in headers {
                urlRequest.setValue(value, forHTTPHeaderField: key)
            }
        }
        
        // 5. Encode body
        if let body = request.body {
            do {
                let encoder = JSONEncoder()
                encoder.dateEncodingStrategy = .iso8601
                urlRequest.httpBody = try encoder.encode(AnyEncodable(body))
            } catch {
                throw NetworkError.decodingError(error)
            }
        }
        
        return urlRequest
    }
}

// Helper for type-erased encoding
private struct AnyEncodable: Encodable {
    private let _encode: (Encoder) throws -> Void
    
    init<T: Encodable>(_ wrapped: T) {
        _encode = wrapped.encode
    }
    
    func encode(to encoder: Encoder) throws {
        try _encode(encoder)
    }
}
```

#### 2.5.5 Authentication Manager

```swift
actor AuthenticationManager {
    private var currentToken: String?
    private var refreshTask: Task<String, Error>?
    
    private let tokenProvider: () async throws -> String
    
    init(tokenProvider: @escaping () async throws -> String) {
        self.tokenProvider = tokenProvider
    }
    
    func getToken() async throws -> String {
        // If already refreshing, await that task
        if let refreshTask = refreshTask {
            return try await refreshTask.value
        }
        
        // If we have a fresh token, return it
        if let token = currentToken {
            return token
        }
        
        // Start refresh
        let task = Task {
            let newToken = try await tokenProvider()
            currentToken = newToken
            return newToken
        }
        
        refreshTask = task
        
        defer {
            refreshTask = nil
        }
        
        return try await task.value
    }
    
    func setToken(_ token: String) {
        currentToken = token
    }
    
    func clearToken() {
        currentToken = nil
        refreshTask?.cancel()
        refreshTask = nil
    }
}
```

#### 2.5.6 Retry Logic with Exponential Backoff

```swift
struct RetryConfiguration {
    let maxAttempts: Int
    let baseDelay: TimeInterval
    let maxDelay: TimeInterval
    let jitter: TimeInterval
    let retryableHTTPCodes: Set<Int>
    
    static let `default` = RetryConfiguration(
        maxAttempts: 3,
        baseDelay: 1.0,
        maxDelay: 10.0,
        jitter: 0.5,
        retryableHTTPCodes: [408, 429, 500, 502, 503, 504]
    )
    
    func delay(for attempt: Int) -> TimeInterval {
        let exponentialDelay = min(maxDelay, baseDelay * pow(2.0, Double(attempt)))
        let jitterValue = Double.random(in: 0...jitter)
        return exponentialDelay + jitterValue
    }
    
    func shouldRetry(_ error: NetworkError, attempt: Int, method: HTTPMethod) -> Bool {
        guard attempt < maxAttempts else { return false }
        
        // Only retry idempotent methods (GET, PUT, DELETE)
        // Don't retry POST (not idempotent by default)
        guard method != .post else { return false }
        
        return error.isRetryable
    }
}

actor RetryHandler {
    private let configuration: RetryConfiguration
    
    init(configuration: RetryConfiguration = .default) {
        self.configuration = configuration
    }
    
    func execute<T>(
        method: HTTPMethod,
        operation: () async throws -> T
    ) async throws -> T {
        var lastError: Error?
        
        for attempt in 0..<configuration.maxAttempts {
            do {
                return try await operation()
            } catch let error as NetworkError {
                lastError = error
                
                // Check if should retry
                if !configuration.shouldRetry(error, attempt: attempt, method: method) {
                    throw error
                }
                
                // Wait before retry
                let delay = configuration.delay(for: attempt)
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                
            } catch {
                throw error
            }
        }
        
        throw lastError ?? NetworkError.unknown(NSError(domain: "Retry", code: -1))
    }
}
```

#### 2.5.7 Main Network Manager

```swift
import Foundation
import OSLog

actor NetworkManager {
    private let baseURL: URL
    private let session: URLSessionProtocol
    private let requestBuilder: RequestBuilder
    private let authManager: AuthenticationManager
    private let retryHandler: RetryHandler
    private let logger = Logger(subsystem: "com.app.network", category: "NetworkManager")
    
    // Track active tasks for cancellation
    private var activeTasks: [UUID: Task<Any, Error>] = [:]
    
    init(
        baseURL: URL,
        session: URLSessionProtocol = URLSession.shared,
        authManager: AuthenticationManager,
        retryConfiguration: RetryConfiguration = .default
    ) {
        self.baseURL = baseURL
        self.session = session
        self.requestBuilder = RequestBuilder(baseURL: baseURL)
        self.authManager = authManager
        self.retryHandler = RetryHandler(configuration: retryConfiguration)
    }
    
    // MARK: - Public API
    
    func request<T: NetworkRequest>(
        _ request: T,
        requiresAuth: Bool = true
    ) async throws -> T.Response {
        let taskID = UUID()
        
        let task: Task<T.Response, Error> = Task {
            try await retryHandler.execute(method: request.method) {
                try await self.executeRequest(request, requiresAuth: requiresAuth)
            }
        }
        
        activeTasks[taskID] = task as! Task<Any, Error>
        
        defer {
            activeTasks.removeValue(forKey: taskID)
        }
        
        return try await task.value
    }
    
    func cancel(taskID: UUID) {
        activeTasks[taskID]?.cancel()
        activeTasks.removeValue(forKey: taskID)
    }
    
    func cancelAll() {
        for task in activeTasks.values {
            task.cancel()
        }
        activeTasks.removeAll()
    }
    
    // MARK: - Private Implementation
    
    private func executeRequest<T: NetworkRequest>(
        _ request: T,
        requiresAuth: Bool
    ) async throws -> T.Response {
        // 1. Build URLRequest
        var urlRequest = try requestBuilder.build(request)
        
        // 2. Inject auth token if needed
        if requiresAuth {
            let token = try await authManager.getToken()
            urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        // 3. Log request
        logRequest(urlRequest)
        
        // 4. Execute request
        let startTime = Date()
        let (data, response) = try await session.data(for: urlRequest)
        let duration = Date().timeIntervalSince(startTime)
        
        // 5. Log response
        logResponse(response, data: data, duration: duration)
        
        // 6. Validate response
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        
        // 7. Handle errors
        guard (200...299).contains(httpResponse.statusCode) else {
            let error = NetworkError.httpError(statusCode: httpResponse.statusCode, data: data)
            
            // Try to refresh token if unauthorized
            if error.requiresAuthentication {
                // Clear cached token
                await authManager.clearToken()
            }
            
            throw error
        }
        
        // 8. Decode response
        do {
            let decoder = JSONDecoder()
            decoder.dateDecodingStrategy = .iso8601
            return try decoder.decode(T.Response.self, from: data)
        } catch {
            logger.error("Decoding failed: \(error.localizedDescription)")
            logger.debug("Raw response: \(String(data: data, encoding: .utf8) ?? "invalid")")
            throw NetworkError.decodingError(error)
        }
    }
    
    // MARK: - Logging
    
    private func logRequest(_ request: URLRequest) {
        var logMessage = "→ \(request.httpMethod ?? "?") \(request.url?.absoluteString ?? "?")"
        
        if let headers = request.allHTTPHeaderFields {
            let redactedHeaders = redactSensitiveHeaders(headers)
            logMessage += "\nHeaders: \(redactedHeaders)"
        }
        
        if let body = request.httpBody,
           let bodyString = String(data: body, encoding: .utf8) {
            logMessage += "\nBody: \(redactBody(bodyString))"
        }
        
        logger.debug("\(logMessage)")
    }
    
    private func logResponse(_ response: URLResponse, data: Data, duration: TimeInterval) {
        guard let httpResponse = response as? HTTPURLResponse else { return }
        
        let statusIcon = (200...299).contains(httpResponse.statusCode) ? "✓" : "✗"
        var logMessage = "← \(statusIcon) \(httpResponse.statusCode) "
        logMessage += "(\(String(format: "%.2f", duration))s)"
        
        if let bodyString = String(data: data, encoding: .utf8) {
            logMessage += "\nBody: \(redactBody(bodyString))"
        }
        
        logger.debug("\(logMessage)")
    }
    
    private func redactSensitiveHeaders(_ headers: [String: String]) -> [String: String] {
        var redacted = headers
        let sensitiveKeys = ["Authorization", "Cookie", "Set-Cookie", "X-API-Key"]
        
        for key in sensitiveKeys {
            if redacted[key] != nil {
                redacted[key] = "***REDACTED***"
            }
        }
        
        return redacted
    }
    
    private func redactBody(_ body: String) -> String {
        // In production, redact PII fields like email, phone, password
        var redacted = body
        
        // Simple regex-based redaction (production would be more sophisticated)
        let patterns = [
            ("\"password\"\\s*:\\s*\"[^\"]+\"", "\"password\":\"***\""),
            ("\"email\"\\s*:\\s*\"[^\"]+\"", "\"email\":\"***@***\""),
            ("\"token\"\\s*:\\s*\"[^\"]+\"", "\"token\":\"***\"")
        ]
        
        for (pattern, replacement) in patterns {
            if let regex = try? NSRegularExpression(pattern: pattern) {
                let range = NSRange(redacted.startIndex..., in: redacted)
                redacted = regex.stringByReplacingMatches(
                    in: redacted,
                    range: range,
                    withTemplate: replacement
                )
            }
        }
        
        return redacted
    }
}
```

#### 2.5.8 Concrete Request Examples

```swift
// 1. GET Request
struct GetUserRequest: NetworkRequest {
    typealias Response = User
    
    let userID: String
    
    var method: HTTPMethod { .get }
    var path: String { "/users/\(userID)" }
}

// 2. POST Request with body
struct CreatePostRequest: NetworkRequest {
    typealias Response = Post
    
    struct Body: Encodable {
        let title: String
        let content: String
        let tags: [String]
    }
    
    let postBody: Body
    
    var method: HTTPMethod { .post }
    var path: String { "/posts" }
    var body: Encodable? { postBody }
}

// 3. GET Request with query parameters
struct SearchRequest: NetworkRequest {
    typealias Response = SearchResults
    
    let query: String
    let page: Int
    let limit: Int
    
    var method: HTTPMethod { .get }
    var path: String { "/search" }
    var queryParameters: [String : String]? {
        [
            "q": query,
            "page": "\(page)",
            "limit": "\(limit)"
        ]
    }
}

// Models
struct User: Codable {
    let id: String
    let name: String
    let email: String
}

struct Post: Codable {
    let id: String
    let title: String
    let content: String
    let createdAt: Date
}

struct SearchResults: Codable {
    let results: [Post]
    let totalCount: Int
    let page: Int
}
```

### 2.6 Usage Examples

#### Basic Request

```swift
// Setup
let baseURL = URL(string: "https://api.example.com/v1")!
let authManager = AuthenticationManager {
    // Token provider (could fetch from keychain or API)
    return "user_access_token_12345"
}

let networkManager = NetworkManager(
    baseURL: baseURL,
    authManager: authManager
)

// Make request
Task {
    do {
        let request = GetUserRequest(userID: "123")
        let user = try await networkManager.request(request)
        print("User: \(user.name)")
    } catch {
        print("Error: \(error)")
    }
}
```

#### With Token Refresh

```swift
class TokenManager {
    private var accessToken: String?
    private var refreshToken: String?
    
    func getAccessToken() async throws -> String {
        if let token = accessToken, !isExpired(token) {
            return token
        }
        
        // Refresh token
        return try await refreshAccessToken()
    }
    
    private func refreshAccessToken() async throws -> String {
        guard let refreshToken = refreshToken else {
            throw NetworkError.unauthorized
        }
        
        // Call /auth/refresh endpoint
        let url = URL(string: "https://api.example.com/auth/refresh")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("Bearer \(refreshToken)", forHTTPHeaderField: "Authorization")
        
        let (data, _) = try await URLSession.shared.data(for: request)
        let response = try JSONDecoder().decode(TokenResponse.self, from: data)
        
        self.accessToken = response.accessToken
        return response.accessToken
    }
    
    private func isExpired(_ token: String) -> Bool {
        // JWT expiry check (decode and check exp claim)
        // Simplified for example
        return false
    }
}

struct TokenResponse: Codable {
    let accessToken: String
    let refreshToken: String
}
```

### (Continued in next message due to length...)

