# Networking & Caching: Complete Deep Dive (Noob to Tech Lead)

## 📡 Topic 1: REST API Design

### 🎈 BEGINNER LEVEL - Understanding REST Like a 5-Year-Old

**What is REST?**

Imagine a restaurant:
- You (iOS app) are the customer
- The kitchen (server) prepares food
- The waiter (REST API) takes your order and brings food

REST is like the waiter - it carries requests and responses between your app and the server.

**Basic REST Concepts:**

```swift
// Think of REST as simple requests:

// 1. GET - "Can I see the menu?" (Read data)
GET /users/123

// 2. POST - "I want to order this" (Create new data)
POST /users
Body: { "name": "John", "email": "john@email.com" }

// 3. PUT - "Change my entire order" (Update all data)
PUT /users/123
Body: { "name": "John Updated", "email": "new@email.com" }

// 4. PATCH - "Just add extra cheese" (Update part of data)
PATCH /users/123
Body: { "email": "newemail@email.com" }

// 5. DELETE - "Cancel my order" (Delete data)
DELETE /users/123
```

**Simple REST Client:**

```swift
// Beginner-level implementation
class SimpleAPIClient {
    func getUser(id: String) {
        // 1. Create URL
        let url = URL(string: "https://api.example.com/users/\(id)")!
        
        // 2. Make request
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            // 3. Handle response
            if let error = error {
                print("Error: \(error)")
                return
            }
            
            if let data = data {
                // 4. Parse JSON
                let user = try? JSONDecoder().decode(User.self, from: data)
                print("Got user: \(user?.name ?? "unknown")")
            }
        }
        
        task.resume()
    }
}

struct User: Codable {
    let id: String
    let name: String
    let email: String
}
```

***

### 🚀 INTERMEDIATE LEVEL - Production-Ready REST Client

**Better REST API Design Principles:**

1. **Resource-based URLs** - Use nouns, not verbs
2. **HTTP methods** - Match action semantics
3. **Status codes** - Proper error communication
4. **Versioning** - Handle API changes
5. **Authentication** - Secure requests

**Intermediate Implementation:**

```swift
// MARK: - HTTP Method
enum HTTPMethod: String {
    case get = "GET"
    case post = "POST"
    case put = "PUT"
    case patch = "PATCH"
    case delete = "DELETE"
}

// MARK: - API Error
enum APIError: Error {
    case invalidURL
    case noData
    case decodingError
    case serverError(Int)
    case unauthorized
    case networkError(Error)
    
    var localizedDescription: String {
        switch self {
        case .invalidURL:
            return "Invalid URL"
        case .noData:
            return "No data received"
        case .decodingError:
            return "Failed to decode response"
        case .serverError(let code):
            return "Server error: \(code)"
        case .unauthorized:
            return "Unauthorized access"
        case .networkError(let error):
            return "Network error: \(error.localizedDescription)"
        }
    }
}

// MARK: - Request Builder
struct APIRequest {
    let endpoint: String
    let method: HTTPMethod
    var headers: [String: String] = [:]
    var body: Data?
    var queryParameters: [String: String] = [:]
    
    func buildURLRequest(baseURL: String) throws -> URLRequest {
        // Build URL with query parameters
        var components = URLComponents(string: baseURL + endpoint)!
        
        if !queryParameters.isEmpty {
            components.queryItems = queryParameters.map {
                URLQueryItem(name: $0.key, value: $0.value)
            }
        }
        
        guard let url = components.url else {
            throw APIError.invalidURL
        }
        
        var request = URLRequest(url: url)
        request.httpMethod = method.rawValue
        request.httpBody = body
        
        // Default headers
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        request.setValue("application/json", forHTTPHeaderField: "Accept")
        
        // Custom headers
        headers.forEach { request.setValue($0.value, forHTTPHeaderField: $0.key) }
        
        return request
    }
}

// MARK: - API Client
class APIClient {
    private let baseURL: String
    private let session: URLSession
    private var authToken: String?
    
    init(baseURL: String, configuration: URLSessionConfiguration = .default) {
        self.baseURL = baseURL
        
        // Configure session
        configuration.timeoutIntervalForRequest = 30
        configuration.timeoutIntervalForResource = 60
        self.session = URLSession(configuration: configuration)
    }
    
    // Generic request method
    func request<T: Decodable>(
        _ request: APIRequest,
        responseType: T.Type
    ) async throws -> T {
        // Build URLRequest
        var urlRequest = try request.buildURLRequest(baseURL: baseURL)
        
        // Add authentication
        if let token = authToken {
            urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        // Log request
        logRequest(urlRequest)
        
        // Perform request
        let (data, response) = try await session.data(for: urlRequest)
        
        // Validate response
        try validateResponse(response)
        
        // Log response
        logResponse(data, response)
        
        // Decode
        do {
            let decoder = JSONDecoder()
            decoder.keyDecodingStrategy = .convertFromSnakeCase
            return try decoder.decode(T.self, from: data)
        } catch {
            throw APIError.decodingError
        }
    }
    
    private func validateResponse(_ response: URLResponse) throws {
        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.serverError(0)
        }
        
        switch httpResponse.statusCode {
        case 200...299:
            return
        case 401:
            throw APIError.unauthorized
        case 400...499:
            throw APIError.serverError(httpResponse.statusCode)
        case 500...599:
            throw APIError.serverError(httpResponse.statusCode)
        default:
            throw APIError.serverError(httpResponse.statusCode)
        }
    }
    
    private func logRequest(_ request: URLRequest) {
        print("📤 REQUEST: \(request.httpMethod ?? "") \(request.url?.absoluteString ?? "")")
        if let headers = request.allHTTPHeaderFields {
            print("Headers: \(headers)")
        }
        if let body = request.httpBody, let bodyString = String(data: body, encoding: .utf8) {
            print("Body: \(bodyString)")
        }
    }
    
    private func logResponse(_ data: Data, _ response: URLResponse) {
        if let httpResponse = response as? HTTPURLResponse {
            print("📥 RESPONSE: \(httpResponse.statusCode)")
        }
        if let responseString = String(data: data, encoding: .utf8) {
            print("Data: \(responseString)")
        }
    }
    
    func setAuthToken(_ token: String) {
        self.authToken = token
    }
}

// MARK: - Usage Example
extension APIClient {
    // GET user
    func getUser(id: String) async throws -> User {
        let request = APIRequest(
            endpoint: "/users/\(id)",
            method: .get
        )
        return try await self.request(request, responseType: User.self)
    }
    
    // POST create user
    func createUser(name: String, email: String) async throws -> User {
        let body = ["name": name, "email": email]
        let bodyData = try JSONEncoder().encode(body)
        
        let request = APIRequest(
            endpoint: "/users",
            method: .post,
            body: bodyData
        )
        return try await self.request(request, responseType: User.self)
    }
    
    // PATCH update user
    func updateUserEmail(id: String, newEmail: String) async throws -> User {
        let body = ["email": newEmail]
        let bodyData = try JSONEncoder().encode(body)
        
        let request = APIRequest(
            endpoint: "/users/\(id)",
            method: .patch,
            body: bodyData
        )
        return try await self.request(request, responseType: User.self)
    }
    
    // DELETE user
    func deleteUser(id: String) async throws {
        let request = APIRequest(
            endpoint: "/users/\(id)",
            method: .delete
        )
        let _: EmptyResponse = try await self.request(request, responseType: EmptyResponse.self)
    }
}

struct EmptyResponse: Codable {}
```

***

### 💎 ADVANCED LEVEL - Senior Engineer / Tech Lead

**Enterprise-Grade REST Client with:**
- Request/Response interceptors
- Retry logic with exponential backoff
- Request prioritization
- Response caching
- Request deduplication
- Authentication token refresh
- Mock support for testing

```swift
// MARK: - Advanced API Client Architecture

// MARK: - Interceptor Protocol
protocol RequestInterceptor {
    func intercept(request: URLRequest) async throws -> URLRequest
}

protocol ResponseInterceptor {
    func intercept(data: Data, response: URLResponse) async throws -> Data
}

// MARK: - Authentication Interceptor
class AuthenticationInterceptor: RequestInterceptor {
    private var accessToken: String?
    private var refreshToken: String?
    private let tokenRefreshURL: URL
    
    private var isRefreshing = false
    private var refreshTask: Task<String, Error>?
    
    init(tokenRefreshURL: URL) {
        self.tokenRefreshURL = tokenRefreshURL
    }
    
    func intercept(request: URLRequest) async throws -> URLRequest {
        var modifiedRequest = request
        
        // Add access token
        if let token = accessToken {
            modifiedRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        return modifiedRequest
    }
    
    func handleUnauthorized() async throws -> String {
        // If already refreshing, wait for that task
        if let existingTask = refreshTask {
            return try await existingTask.value
        }
        
        // Create refresh task
        let task = Task<String, Error> {
            defer { 
                self.isRefreshing = false
                self.refreshTask = nil
            }
            
            guard let refreshToken = self.refreshToken else {
                throw APIError.unauthorized
            }
            
            // Refresh token request
            var request = URLRequest(url: self.tokenRefreshURL)
            request.httpMethod = "POST"
            request.setValue("application/json", forHTTPHeaderField: "Content-Type")
            
            let body = ["refresh_token": refreshToken]
            request.httpBody = try JSONEncoder().encode(body)
            
            let (data, _) = try await URLSession.shared.data(for: request)
            
            struct TokenResponse: Codable {
                let accessToken: String
                let refreshToken: String
            }
            
            let response = try JSONDecoder().decode(TokenResponse.self, from: data)
            
            self.accessToken = response.accessToken
            self.refreshToken = response.refreshToken
            
            return response.accessToken
        }
        
        refreshTask = task
        isRefreshing = true
        
        return try await task.value
    }
    
    func setTokens(access: String, refresh: String) {
        self.accessToken = access
        self.refreshToken = refresh
    }
}

// MARK: - Logging Interceptor
class LoggingInterceptor: RequestInterceptor, ResponseInterceptor {
    enum LogLevel {
        case none, basic, headers, body
    }
    
    let logLevel: LogLevel
    
    init(logLevel: LogLevel = .basic) {
        self.logLevel = logLevel
    }
    
    func intercept(request: URLRequest) async throws -> URLRequest {
        guard logLevel != .none else { return request }
        
        print("📤 \(request.httpMethod ?? "GET") \(request.url?.absoluteString ?? "")")
        
        if logLevel == .headers || logLevel == .body {
            if let headers = request.allHTTPHeaderFields {
                print("Headers: \(headers)")
            }
        }
        
        if logLevel == .body {
            if let body = request.httpBody,
               let bodyString = String(data: body, encoding: .utf8) {
                print("Body: \(bodyString)")
            }
        }
        
        return request
    }
    
    func intercept(data: Data, response: URLResponse) async throws -> Data {
        guard logLevel != .none else { return data }
        
        if let httpResponse = response as? HTTPURLResponse {
            let emoji = httpResponse.statusCode < 400 ? "✅" : "❌"
            print("\(emoji) \(httpResponse.statusCode)")
        }
        
        if logLevel == .body {
            if let jsonString = String(data: data, encoding: .utf8) {
                print("Response: \(jsonString)")
            }
        }
        
        return data
    }
}

// MARK: - Retry Interceptor
class RetryInterceptor: ResponseInterceptor {
    let maxRetries: Int
    let retryableStatusCodes: Set<Int>
    
    init(maxRetries: Int = 3, retryableStatusCodes: Set<Int> = [408, 429, 500, 502, 503, 504]) {
        self.maxRetries = maxRetries
        self.retryableStatusCodes = retryableStatusCodes
    }
    
    func intercept(data: Data, response: URLResponse) async throws -> Data {
        // This is called after response, so we just validate
        // Actual retry logic is in AdvancedAPIClient
        return data
    }
    
    func shouldRetry(response: URLResponse?, error: Error?, attempt: Int) -> Bool {
        guard attempt < maxRetries else { return false }
        
        // Retry on network errors
        if error != nil {
            return true
        }
        
        // Retry on specific status codes
        if let httpResponse = response as? HTTPURLResponse,
           retryableStatusCodes.contains(httpResponse.statusCode) {
            return true
        }
        
        return false
    }
    
    func retryDelay(for attempt: Int) -> TimeInterval {
        // Exponential backoff: 1s, 2s, 4s, 8s...
        return pow(2.0, Double(attempt))
    }
}

// MARK: - Advanced API Client
class AdvancedAPIClient {
    private let baseURL: String
    private let session: URLSession
    
    private var requestInterceptors: [RequestInterceptor] = []
    private var responseInterceptors: [ResponseInterceptor] = []
    private let retryInterceptor = RetryInterceptor()
    
    // Request deduplication
    private var ongoingRequests: [String: Task<Data, Error>] = [:]
    private let requestLock = NSLock()
    
    // Cache
    private let cache: URLCache
    
    init(
        baseURL: String,
        configuration: URLSessionConfiguration = .default,
        cacheSize: Int = 50 * 1024 * 1024  // 50MB
    ) {
        self.baseURL = baseURL
        
        // Configure cache
        self.cache = URLCache(
            memoryCapacity: cacheSize / 2,
            diskCapacity: cacheSize,
            diskPath: "api_cache"
        )
        
        configuration.urlCache = cache
        configuration.requestCachePolicy = .returnCacheDataElseLoad
        
        self.session = URLSession(configuration: configuration)
    }
    
    func addRequestInterceptor(_ interceptor: RequestInterceptor) {
        requestInterceptors.append(interceptor)
    }
    
    func addResponseInterceptor(_ interceptor: ResponseInterceptor) {
        responseInterceptors.append(interceptor)
    }
    
    // MARK: - Main Request Method
    func request<T: Decodable>(
        _ apiRequest: APIRequest,
        responseType: T.Type,
        priority: TaskPriority = .medium,
        bypassCache: Bool = false
    ) async throws -> T {
        // Build URLRequest
        var urlRequest = try apiRequest.buildURLRequest(baseURL: baseURL)
        
        if bypassCache {
            urlRequest.cachePolicy = .reloadIgnoringLocalCacheData
        }
        
        // Apply request interceptors
        for interceptor in requestInterceptors {
            urlRequest = try await interceptor.intercept(request: urlRequest)
        }
        
        // Request deduplication
        let requestKey = makeRequestKey(urlRequest)
        
        requestLock.lock()
        if let existingTask = ongoingRequests[requestKey] {
            requestLock.unlock()
            let data = try await existingTask.value
            return try decode(data: data)
        }
        requestLock.unlock()
        
        // Create new request task
        let task = Task(priority: priority) {
            try await self.performRequest(urlRequest)
        }
        
        requestLock.lock()
        ongoingRequests[requestKey] = task
        requestLock.unlock()
        
        do {
            let data = try await task.value
            
            requestLock.lock()
            ongoingRequests.removeValue(forKey: requestKey)
            requestLock.unlock()
            
            return try decode(data: data)
        } catch {
            requestLock.lock()
            ongoingRequests.removeValue(forKey: requestKey)
            requestLock.unlock()
            
            throw error
        }
    }
    
    private func performRequest(_ request: URLRequest) async throws -> Data {
        var lastError: Error?
        
        for attempt in 0..<retryInterceptor.maxRetries {
            do {
                var (data, response) = try await session.data(for: request)
                
                // Apply response interceptors
                for interceptor in responseInterceptors {
                    data = try await interceptor.intercept(data: data, response: response)
                }
                
                // Validate status code
                if let httpResponse = response as? HTTPURLResponse {
                    if httpResponse.statusCode == 401 {
                        // Try to refresh token
                        if let authInterceptor = requestInterceptors.first(where: { $0 is AuthenticationInterceptor }) as? AuthenticationInterceptor {
                            _ = try await authInterceptor.handleUnauthorized()
                            // Retry with new token
                            continue
                        }
                        throw APIError.unauthorized
                    }
                    
                    guard (200...299).contains(httpResponse.statusCode) else {
                        throw APIError.serverError(httpResponse.statusCode)
                    }
                }
                
                return data
                
            } catch {
                lastError = error
                
                // Check if should retry
                if retryInterceptor.shouldRetry(response: nil, error: error, attempt: attempt) {
                    let delay = retryInterceptor.retryDelay(for: attempt)
                    try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                    continue
                } else {
                    throw error
                }
            }
        }
        
        throw lastError ?? APIError.serverError(0)
    }
    
    private func decode<T: Decodable>(data: Data) throws -> T {
        let decoder = JSONDecoder()
        decoder.keyDecodingStrategy = .convertFromSnakeCase
        decoder.dateDecodingStrategy = .iso8601
        
        do {
            return try decoder.decode(T.self, from: data)
        } catch {
            print("❌ Decoding error: \(error)")
            if let jsonString = String(data: data, encoding: .utf8) {
                print("JSON: \(jsonString)")
            }
            throw APIError.decodingError
        }
    }
    
    private func makeRequestKey(_ request: URLRequest) -> String {
        var key = "\(request.httpMethod ?? "GET")_\(request.url?.absoluteString ?? "")"
        
        if let body = request.httpBody,
           let bodyString = String(data: body, encoding: .utf8) {
            key += "_\(bodyString)"
        }
        
        return key
    }
}

// MARK: - Usage Example
class UserService {
    private let client: AdvancedAPIClient
    
    init() {
        self.client = AdvancedAPIClient(baseURL: "https://api.example.com")
        
        // Setup interceptors
        let authInterceptor = AuthenticationInterceptor(
            tokenRefreshURL: URL(string: "https://api.example.com/auth/refresh")!
        )
        client.addRequestInterceptor(authInterceptor)
        
        let loggingInterceptor = LoggingInterceptor(logLevel: .body)
        client.addRequestInterceptor(loggingInterceptor)
        client.addResponseInterceptor(loggingInterceptor)
    }
    
    func getUser(id: String) async throws -> User {
        let request = APIRequest(
            endpoint: "/users/\(id)",
            method: .get
        )
        return try await client.request(request, responseType: User.self)
    }
    
    func searchUsers(query: String, page: Int = 1) async throws -> [User] {
        let request = APIRequest(
            endpoint: "/users/search",
            method: .get,
            queryParameters: [
                "q": query,
                "page": "\(page)",
                "limit": "20"
            ]
        )
        
        struct SearchResponse: Codable {
            let users: [User]
            let totalCount: Int
            let page: Int
        }
        
        let response: SearchResponse = try await client.request(request, responseType: SearchResponse.self)
        return response.users
    }
}
```

***

### 📝 REST API Interview Questions

**Q1: Design a RESTful API for a social media app with posts, comments, and likes. What endpoints would you create?**

**Expected Answer (Tech Lead Level):**

```swift
// MARK: - Resource Hierarchy

// Posts
GET    /posts                      // List all posts (paginated)
GET    /posts/:id                  // Get single post
POST   /posts                      // Create post
PUT    /posts/:id                  // Update entire post
PATCH  /posts/:id                  // Update post fields
DELETE /posts/:id                  // Delete post

// Nested resource: Comments on a post
GET    /posts/:id/comments         // List all comments on a post
POST   /posts/:id/comments         // Create comment on a post
GET    /comments/:id               // Get single comment (alternative)
PUT    /comments/:id               // Update comment
DELETE /comments/:id               // Delete comment

// Likes (as a sub-resource)
POST   /posts/:id/like             // Like a post
DELETE /posts/:id/like             // Unlike a post
GET    /posts/:id/likes            // Get users who liked

// User's posts
GET    /users/:id/posts            // Get all posts by user
GET    /users/:id/feed             // Get user's personalized feed

// Pagination & Filtering
GET    /posts?page=1&limit=20&sort=created_at&order=desc
GET    /posts?author_id=123&tags=tech,ios

// Batch operations
POST   /posts/batch/delete         // Bulk delete
Body: { "post_ids": ["1", "2", "3"] }

// MARK: - Response Format
{
  "data": {
    "id": "123",
    "content": "Hello world",
    "author": {
      "id": "456",
      "name": "John"
    },
    "likes_count": 42,
    "comments_count": 5,
    "created_at": "2026-01-09T10:00:00Z"
  },
  "meta": {
    "page": 1,
    "per_page": 20,
    "total": 100
  },
  "links": {
    "self": "/posts/123",
    "author": "/users/456",
    "comments": "/posts/123/comments"
  }
}
```

**Key Design Principles:**
1. **Nouns for resources, not verbs** (/posts not /getPosts)
2. **Hierarchical relationships** (/posts/:id/comments)
3. **Filtering via query params** (?author_id=123)
4. **Consistent pluralization** (always /posts, not /post)
5. **HATEOAS** (Include links to related resources)
6. **Pagination metadata**
7. **HTTP status codes** (200 OK, 201 Created, 404 Not Found)

***

**Q2: How would you handle API versioning in a production iOS app?**

**Expected Answer:**

```swift
// MARK: - Versioning Strategies

// 1. URL Path Versioning (Recommended for iOS)
GET https://api.example.com/v1/users
GET https://api.example.com/v2/users

// 2. Header Versioning
GET https://api.example.com/users
Header: Accept: application/vnd.company.v2+json

// 3. Query Parameter Versioning
GET https://api.example.com/users?version=2

// MARK: - Implementation

enum APIVersion: String {
    case v1 = "v1"
    case v2 = "v2"
    case v3 = "v3"
}

class VersionedAPIClient {
    let currentVersion: APIVersion
    let fallbackVersion: APIVersion
    
    init(currentVersion: APIVersion = .v2, fallbackVersion: APIVersion = .v1) {
        self.currentVersion = currentVersion
        self.fallbackVersion = fallbackVersion
    }
    
    func buildURL(endpoint: String, version: APIVersion? = nil) -> URL {
        let ver = version ?? currentVersion
        return URL(string: "https://api.example.com/\(ver.rawValue)\(endpoint)")!
    }
    
    // Try current version, fallback if fails
    func requestWithFallback<T: Decodable>(
        endpoint: String,
        responseType: T.Type
    ) async throws -> T {
        do {
            // Try current version
            let url = buildURL(endpoint: endpoint, version: currentVersion)
            return try await performRequest(url: url, responseType: T.self)
        } catch APIError.serverError(404), APIError.serverError(410) {
            // Endpoint doesn't exist in current version, try fallback
            print("⚠️ Falling back to \(fallbackVersion.rawValue)")
            let url = buildURL(endpoint: endpoint, version: fallbackVersion)
            return try await performRequest(url: url, responseType: T.self)
        }
    }
    
    private func performRequest<T: Decodable>(url: URL, responseType: T.Type) async throws -> T {
        // Implementation
        fatalError("Implement request")
    }
}

// MARK: - Version-Specific Response Handling

struct UserV1: Codable {
    let id: String
    let name: String
}

struct UserV2: Codable {
    let id: String
    let firstName: String
    let lastName: String
    let avatarURL: URL
    
    // Compatibility with V1
    var name: String {
        return "\(firstName) \(lastName)"
    }
}

// Adapter pattern
protocol UserProtocol {
    var id: String { get }
    var displayName: String { get }
}

extension UserV1: UserProtocol {
    var displayName: String { name }
}

extension UserV2: UserProtocol {
    var displayName: String { "\(firstName) \(lastName)" }
}
```

**Migration Strategy:**
1. **Deploy v2 alongside v1** (don't break existing apps)
2. **Gradual rollout** (feature flag in app)
3. **Monitor v1 usage** (analytics)
4. **Deprecation notice** (6 months warning)
5. **Sunset v1** (only when < 1% usage)

***

**Q3: How do you handle authentication and token refresh in REST APIs?**

**Expected Answer (Production Implementation):**

```swift
// MARK: - Token Management

actor TokenManager {
    private var accessToken: String?
    private var refreshToken: String?
    private var expiresAt: Date?
    
    private var refreshTask: Task<TokenPair, Error>?
    
    struct TokenPair {
        let accessToken: String
        let refreshToken: String
        let expiresIn: TimeInterval
    }
    
    func setTokens(_ tokens: TokenPair) {
        self.accessToken = tokens.accessToken
        self.refreshToken = tokens.refreshToken
        self.expiresAt = Date().addingTimeInterval(tokens.expiresIn)
    }
    
    func getValidToken() async throws -> String {
        // If token is still valid, return it
        if let token = accessToken,
           let expiresAt = expiresAt,
           Date() < expiresAt.addingTimeInterval(-60) { // Refresh 60s before expiry
            return token
        }
        
        // If refresh is already in progress, wait for it
        if let task = refreshTask {
            let tokens = try await task.value
            return tokens.accessToken
        }
        
        // Start new refresh
        let task = Task<TokenPair, Error> {
            try await self.refreshAccessToken()
        }
        
        self.refreshTask = task
        
        do {
            let tokens = try await task.value
            self.refreshTask = nil
            setTokens(tokens)
            return tokens.accessToken
        } catch {
            self.refreshTask = nil
            throw error
        }
    }
    
    private func refreshAccessToken() async throws -> TokenPair {
        guard let refreshToken = self.refreshToken else {
            throw APIError.unauthorized
        }
        
        var request = URLRequest(url: URL(string: "https://api.example.com/auth/refresh")!)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        let body = ["refresh_token": refreshToken]
        request.httpBody = try JSONEncoder().encode(body)
        
        let (data, response) = try await URLSession.shared.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.unauthorized
        }
        
        struct RefreshResponse: Codable {
            let accessToken: String
            let refreshToken: String
            let expiresIn: TimeInterval
        }
        
        let refreshResponse = try JSONDecoder().decode(RefreshResponse.self, from: data)
        
        return TokenPair(
            accessToken: refreshResponse.accessToken,
            refreshToken: refreshResponse.refreshToken,
            expiresIn: refreshResponse.expiresIn
        )
    }
    
    func clearTokens() {
        accessToken = nil
        refreshToken = nil
        expiresAt = nil
    }
}

// MARK: - Authenticated API Client

class AuthenticatedAPIClient {
    private let tokenManager: TokenManager
    private let baseURL: String
    
    init(baseURL: String, tokenManager: TokenManager) {
        self.baseURL = baseURL
        self.tokenManager = tokenManager
    }
    
    func request<T: Decodable>(
        _ endpoint: String,
        method: HTTPMethod = .get,
        body: Data? = nil
    ) async throws -> T {
        // Get valid token (refreshes if needed)
        let token = try await tokenManager.getValidToken()
        
        // Build request
        var request = URLRequest(url: URL(string: baseURL + endpoint)!)
        request.httpMethod = method.rawValue
        request.httpBody = body
        request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        do {
            let (data, response) = try await URLSession.shared.data(for: request)
            
            guard let httpResponse = response as? HTTPURLResponse else {
                throw APIError.invalidResponse
            }
            
            // Handle 401 - might need another refresh
            if httpResponse.statusCode == 401 {
                // Clear tokens and force re-login
                await tokenManager.clearTokens()
                throw APIError.unauthorized
            }
            
            guard (200...299).contains(httpResponse.statusCode) else {
                throw APIError.serverError(httpResponse.statusCode)
            }
            
            return try JSONDecoder().decode(T.self, from: data)
            
        } catch {
            throw error
        }
    }
}

// MARK: - Usage Example

class AuthService {
    let tokenManager = TokenManager()
    
    func login(email: String, password: String) async throws {
        var request = URLRequest(url: URL(string: "https://api.example.com/auth/login")!)
        request.httpMethod = "POST"
        
        let body = ["email": email, "password": password]
        request.httpBody = try JSONEncoder().encode(body)
        
        let (data, _) = try await URLSession.shared.data(for: request)
        
        struct LoginResponse: Codable {
            let accessToken: String
            let refreshToken: String
            let expiresIn: TimeInterval
        }
        
        let response = try JSONDecoder().decode(LoginResponse.self, from: data)
        
        await tokenManager.setTokens(TokenManager.TokenPair(
            accessToken: response.accessToken,
            refreshToken: response.refreshToken,
            expiresIn: response.expiresIn
        ))
    }
    
    func logout() async {
        await tokenManager.clearTokens()
    }
}
```

**Key Points:**
1. **Proactive refresh** - Refresh before expiry, not after 401
2. **Single refresh** - Use actor to prevent multiple simultaneous refreshes
3. **Secure storage** - Store tokens in Keychain, not UserDefaults
4. **Logout on fail** - Clear tokens if refresh fails
5. **Request queue** - Pause requests during refresh

***

**Q4: How would you implement request/response logging for debugging without impacting production performance?**

**Expected Answer:**

```swift
// MARK: - Logging System

enum LogLevel {
    case off
    case error
    case warning
    case info
    case debug
    case verbose
}

class APILogger {
    static let shared = APILogger()
    
    private var logLevel: LogLevel
    private let logQueue = DispatchQueue(label: "com.app.api.logger", qos: .utility)
    private var logFile: FileHandle?
    
    init() {
        #if DEBUG
        self.logLevel = .verbose
        #else
        self.logLevel = .error
        #endif
        
        setupLogFile()
    }
    
    private func setupLogFile() {
        let paths = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)
        let logURL = paths[0].appendingPathComponent("api_logs.txt")
        
        if !FileManager.default.fileExists(atPath: logURL.path) {
            FileManager.default.createFile(atPath: logURL.path, contents: nil)
        }
        
        logFile = try? FileHandle(forWritingTo: logURL)
        logFile?.seekToEndOfFile()
    }
    
    func logRequest(_ request: URLRequest, level: LogLevel = .debug) {
        guard shouldLog(level) else { return }
        
        logQueue.async {
            var log = "\n📥 REQUEST [\(Date().ISO8601Format())]\n"
            log += "\(request.httpMethod ?? "GET") \(request.url?.absoluteString ?? "unknown")\n"
            
            if level.rawValue >= LogLevel.info.rawValue {
                if let headers = request.allHTTPHeaderFields {
                    log += "Headers: \(self.sanitizeHeaders(headers))\n"
                }
            }
            
            if level.rawValue >= LogLevel.verbose.rawValue {
                if let body = request.httpBody,
                   let bodyString = String(data: body, encoding: .utf8) {
                    log += "Body: \(self.sanitizeBody(bodyString))\n"
                }
            }
            
            self.writeLog(log)
        }
    }
    
    func logResponse(_ data: Data, _ response: URLResponse, level: LogLevel = .debug) {
        guard shouldLog(level) else { return }
        
        logQueue.async {
            var log = "\n📥 RESPONSE [\(Date().ISO8601Format())]\n"
            
            if let httpResponse = response as? HTTPURLResponse {
                let emoji = httpResponse.statusCode < 400 ? "✅" : "❌"
                log += "\(emoji) Status: \(httpResponse.statusCode)\n"
                
                if level.rawValue >= LogLevel.info.rawValue {
                    log += "Headers: \(httpResponse.allHeaderFields)\n"
                }
            }
            
            if level.rawValue >= LogLevel.verbose.rawValue {
                if let jsonString = String(data: data, encoding: .utf8) {
                    log += "Body: \(jsonString)\n"
                }
            }
            
            self.writeLog(log)
        }
    }
    
    func logError(_ error: Error, request: URLRequest?) {
        guard shouldLog(.error) else { return }
        
        logQueue.async {
            var log = "\n❌ ERROR [\(Date().ISO8601Format())]\n"
            if let request = request {
                log += "URL: \(request.url?.absoluteString ?? "unknown")\n"
            }
            log += "Error: \(error.localizedDescription)\n"
            
            self.writeLog(log)
        }
    }
    
    private func shouldLog(_ level: LogLevel) -> Bool {
        return level.rawValue <= logLevel.rawValue
    }
    
    private func sanitizeHeaders(_ headers: [String: String]) -> [String: String] {
        var sanitized = headers
        
        // Remove sensitive headers
        let sensitiveKeys = ["Authorization", "Cookie", "X-API-Key"]
        for key in sensitiveKeys {
            if sanitized[key] != nil {
                sanitized[key] = "[REDACTED]"
            }
        }
        
        return sanitized
    }
    
    private func sanitizeBody(_ body: String) -> String {
        // Redact sensitive fields (password, credit card, etc.)
        var sanitized = body
        
        let sensitivePatterns = [
            "password": "\"password\"\\s*:\\s*\"[^\"]*\"",
            "credit_card": "\"credit_card\"\\s*:\\s*\"[^\"]*\"",
            "ssn": "\"ssn\"\\s*:\\s*\"[^\"]*\""
        ]
        
        for (_, pattern) in sensitivePatterns {
            if let regex = try? NSRegularExpression(pattern: pattern) {
                let range = NSRange(sanitized.startIndex..., in: sanitized)
                sanitized = regex.stringByReplacingMatches(
                    in: sanitized,
                    range: range,
                    withTemplate: "\"password\":\"[REDACTED]\""
                )
            }
        }
        
        return sanitized
    }
    
    private func writeLog(_ log: String) {
        // Write to file
        if let data = log.data(using: .utf8) {
            logFile?.write(data)
        }
        
        // Print to console in debug
        #if DEBUG
        print(log)
        #endif
    }
    
    func exportLogs() -> URL? {
        logFile?.synchronizeFile()
        let paths = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)
        return paths[0].appendingPathComponent("api_logs.txt")
    }
    
    func clearLogs() {
        logQueue.async {
            self.logFile?.truncateFile(atOffset: 0)
        }
    }
}

extension LogLevel {
    var rawValue: Int {
        switch self {
        case .off: return 0
        case .error: return 1
        case .warning: return 2
        case .info: return 3
        case .debug: return 4
        case .verbose: return 5
        }
    }
}
```

**Best Practices:**
1. **Different levels for debug/production**
2. **Async logging** - Don't block main thread
3. **Sanitize sensitive data** - Never log passwords/tokens
4. **File-based logs** - Can be exported for debugging
5. **Size limits** - Rotate logs when too large
6. **Conditional compilation** - Verbose only in DEBUG

***

**Q5: Design an API client that works offline and syncs when network is available**

**Expected Answer:**

```swift
// MARK: - Offline-First API Client

actor OfflineAPIClient {
    private let onlineClient: AdvancedAPIClient
    private let database: LocalDatabase
    private let syncQueue: SyncQueue
    private let reachability: NetworkReachability
    
    init(baseURL: String) {
        self.onlineClient = AdvancedAPIClient(baseURL: baseURL)
        self.database = LocalDatabase()
        self.syncQueue = SyncQueue()
        self.reachability = NetworkReachability()
        
        // Start monitoring network
        Task {
            await self.startNetworkMonitoring()
        }
    }
    
    // MARK: - Read Operations (Cache-First)
    
    func getUser(id: String) async throws -> User {
        // 1. Try local cache first
        if let cachedUser = await database.getUser(id: id) {
            // 2. If online, refresh in background
            if await reachability.isConnected {
                Task(priority: .background) {
                    try? await self.refreshUser(id: id)
                }
            }
            return cachedUser
        }
        
        // 3. No cache, must fetch from network
        guard await reachability.isConnected else {
            throw APIError.noData
        }
        
        let request = APIRequest(endpoint: "/users/\(id)", method: .get)
        let user: User = try await onlineClient.request(request, responseType: User.self)
        
        // 4. Cache the result
        await database.saveUser(user)
        
        return user
    }
    
    // MARK: - Write Operations (Queue for Sync)
    
    func createPost(title: String, content: String) async throws -> Post {
        let tempID = UUID().uuidString
        let post = Post(
            id: tempID,
            title: title,
            content: content,
            status: .pending,
            createdAt: Date()
        )
        
        // 1. Save locally immediately
        await database.savePost(post)
        
        // 2. Queue for sync
        let operation = SyncOperation(
            type: .create,
            endpoint: "/posts",
            method: .post,
            body: try JSONEncoder().encode(post),
            localID: tempID
        )
        await syncQueue.enqueue(operation)
        
        // 3. Try to sync immediately if online
        if await reachability.isConnected {
            await syncNow()
        }
        
        return post
    }
    
    func updatePost(id: String, title: String, content: String) async throws {
        // 1. Update locally
        await database.updatePost(id: id, title: title, content: content)
        
        // 2. Queue for sync
        let operation = SyncOperation(
            type: .update,
            endpoint: "/posts/\(id)",
            method: .patch,
            body: try JSONEncoder().encode(["title": title, "content": content]),
            localID: id
        )
        await syncQueue.enqueue(operation)
        
        // 3. Sync if online
        if await reachability.isConnected {
            await syncNow()
        }
    }
    
    func deletePost(id: String) async throws {
        // 1. Mark as deleted locally
        await database.markPostDeleted(id: id)
        
        // 2. Queue for sync
        let operation = SyncOperation(
            type: .delete,
            endpoint: "/posts/\(id)",
            method: .delete,
            body: nil,
            localID: id
        )
        await syncQueue.enqueue(operation)
        
        // 3. Sync if online
        if await reachability.isConnected {
            await syncNow()
        }
    }
    
    // MARK: - Sync Logic
    
    private func startNetworkMonitoring() async {
        for await isConnected in await reachability.statusStream() {
            if isConnected {
                await syncNow()
            }
        }
    }
    
    func syncNow() async {
        let operations = await syncQueue.pendingOperations()
        
        for operation in operations {
            do {
                try await performSync(operation)
                await syncQueue.markCompleted(operation.id)
            } catch {
                print("Sync failed for operation \(operation.id): \(error)")
                await syncQueue.markFailed(operation.id, error: error)
            }
        }
    }
    
    private func performSync(_ operation: SyncOperation) async throws {
        let request = APIRequest(
            endpoint: operation.endpoint,
            method: operation.method,
            body: operation.body
        )
        
        switch operation.type {
        case .create:
            let response: Post = try await onlineClient.request(request, responseType: Post.self)
            // Update local record with server ID
            await database.updatePostID(from: operation.localID, to: response.id)
            
        case .update:
            let _: Post = try await onlineClient.request(request, responseType: Post.self)
            
        case .delete:
            let _: EmptyResponse = try await onlineClient.request(request, responseType: EmptyResponse.self)
            await database.permanentlyDeletePost(id: operation.localID)
        }
    }
    
    private func refreshUser(id: String) async throws {
        let request = APIRequest(endpoint: "/users/\(id)", method: .get)
        let user: User = try await onlineClient.request(request, responseType: User.self)
        await database.saveUser(user)
    }
}

// MARK: - Supporting Types

actor SyncQueue {
    private var operations: [SyncOperation] = []
    
    func enqueue(_ operation: SyncOperation) {
        operations.append(operation)
    }
    
    func pendingOperations() -> [SyncOperation] {
        return operations.filter { $0.status == .pending }
    }
    
    func markCompleted(_ id: String) {
        if let index = operations.firstIndex(where: { $0.id == id }) {
            operations[index].status = .completed
        }
    }
    
    func markFailed(_ id: String, error: Error) {
        if let index = operations.firstIndex(where: { $0.id == id }) {
            operations[index].status = .failed
            operations[index].retryCount += 1
        }
    }
}

struct SyncOperation {
    let id = UUID().uuidString
    let type: OperationType
    let endpoint: String
    let method: HTTPMethod
    let body: Data?
    let localID: String
    var status: Status = .pending
    var retryCount = 0
    
    enum OperationType {
        case create, update, delete
    }
    
    enum Status {
        case pending, completed, failed
    }
}

struct Post: Codable {
    let id: String
    let title: String
    let content: String
    var status: PostStatus
    let createdAt: Date
    
    enum PostStatus: String, Codable {
        case pending, synced
    }
}

actor LocalDatabase {
    private var users: [String: User] = [:]
    private var posts: [String: Post] = [:]
    
    func getUser(id: String) -> User? {
        return users[id]
    }
    
    func saveUser(_ user: User) {
        users[user.id] = user
    }
    
    func savePost(_ post: Post) {
        posts[post.id] = post
    }
    
    func updatePost(id: String, title: String, content: String) {
        posts[id]?.title = title
        posts[id]?.content = content
    }
    
    func markPostDeleted(id: String) {
        posts[id]?.status = .deleted
    }
    
    func updatePostID(from tempID: String, to serverID: String) {
        if let post = posts[tempID] {
            posts.removeValue(forKey: tempID)
            posts[serverID] = post
        }
    }
    
    func permanentlyDeletePost(id: String) {
        posts.removeValue(forKey: id)
    }
}

actor NetworkReachability {
    private var isConnected = true
    
    func statusStream() -> AsyncStream<Bool> {
        // Simulate network monitoring
        AsyncStream { continuation in
            // In real implementation, use NWPathMonitor
            continuation.yield(isConnected)
        }
    }
}

extension Post {
    enum PostStatus: String, Codable {
        case pending, synced, deleted
    }
    
    var title: String {
        get { _title }
        set { _title = newValue }
    }
    
    var content: String {
        get { _content }
        set { _content = newValue }
    }
    
    private var _title: String
    private var _content: String
}
```

***

**Q6: How would you implement GraphQL on iOS? Compare with REST.**

**Expected Answer:**

```swift
// MARK: - GraphQL Client

class GraphQLClient {
    private let endpoint: URL
    private let session: URLSession
    
    init(endpoint: URL) {
        self.endpoint = endpoint
        self.session = URLSession.shared
    }
    
    func query<T: Decodable>(_ query: String, variables: [String: Any]? = nil) async throws -> T {
        let request = try buildRequest(query: query, variables: variables)
        let (data, _) = try await session.data(for: request)
        
        struct GraphQLResponse<T: Decodable>: Decodable {
            let data: T?
            let errors: [GraphQLError]?
        }
        
        let response = try JSONDecoder().decode(GraphQLResponse<T>.self, from: data)
        
        if let errors = response.errors, !errors.isEmpty {
            throw GraphQLError(message: errors.first!.message)
        }
        
        guard let data = response.data else {
            throw GraphQLError(message: "No data in response")
        }
        
        return data
    }
    
    private func buildRequest(query: String, variables: [String: Any]?) throws -> URLRequest {
        var request = URLRequest(url: endpoint)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        var body: [String: Any] = ["query": query]
        if let variables = variables {
            body["variables"] = variables
        }
        
        request.httpBody = try JSONSerialization.data(withJSONObject: body)
        
        return request
    }
}

struct GraphQLError: Error, Decodable {
    let message: String
}

// MARK: - Usage Example

class UserService {
    private let client: GraphQLClient
    
    init() {
        self.client = GraphQLClient(endpoint: URL(string: "https://api.example.com/graphql")!)
    }
    
    func getUser(id: String) async throws -> User {
        let query = """
        query GetUser($id: ID!) {
            user(id: $id) {
                id
                name
                email
                posts {
                    id
                    title
                }
            }
        }
        """
        
        struct Response: Decodable {
            let user: User
        }
        
        let response: Response = try await client.query(query, variables: ["id": id])
        return response.user
    }
    
    func createPost(title: String, content: String) async throws -> Post {
        let mutation = """
        mutation CreatePost($title: String!, $content: String!) {
            createPost(title: $title, content: $content) {
                id
                title
                content
                createdAt
            }
        }
        """
        
        struct Response: Decodable {
            let createPost: Post
        }
        
        let response: Response = try await client.query(
            mutation,
            variables: ["title": title, "content": content]
        )
        return response.createPost
    }
}

// MARK: - REST vs GraphQL Comparison

/*
REST:
✅ Simple and widely understood
✅ Better caching (HTTP caching)
✅ More tools and documentation
❌ Over-fetching (get unwanted data)
❌ Under-fetching (multiple requests)
❌ Versioning complexity

GET /users/123
Response: {
    "id": "123",
    "name": "John",
    "email": "john@example.com",
    "address": { ... },        // Don't need this
    "preferences": { ... },    // Don't need this
    "created_at": "..."        // Don't need this
}

Then need another request for posts:
GET /users/123/posts

GraphQL:
✅ Single request for everything
✅ Get exactly what you need
✅ No versioning needed
✅ Strongly typed
❌ More complex to implement
❌ Caching is harder
❌ Learning curve

query {
    user(id: "123") {
        id
        name
        posts {
            id
            title
        }
    }
}

Response: Only what you asked for!
{
    "data": {
        "user": {
            "id": "123",
            "name": "John",
            "posts": [
                { "id": "1", "title": "Post 1" },
                { "id": "2", "title": "Post 2" }
            ]
        }
    }
}

When to use REST:
- Simple CRUD operations
- Public APIs
- Heavy caching needs
- Small team

When to use GraphQL:
- Complex data requirements
- Mobile apps (save bandwidth)
- Rapid iteration
- Microservices backend
*/
```

***

## 📡 Topic 2: Caching Strategies

### 🎈 BEGINNER LEVEL - What is Caching?

**Real-World Analogy:**

Imagine you're a student studying for exams:
- **No Cache:** Every time you need a formula, you go to the library, search for the textbook, find the page (takes 10 minutes)
- **With Cache:** You write frequently used formulas on a sticky note on your desk (takes 2 seconds)

That sticky note is your **cache**!

**Why Do We Cache?**

1. **Speed** - Memory access is 1000x faster than network
2. **Offline Support** - App works without internet
3. **Reduced Costs** - Save bandwidth and server load
4. **Better UX** - No loading spinners
5. **Battery Life** - Network requests drain battery

**Cache Hierarchy (Fastest to Slowest):**

```
Level 1: CPU Cache (nanoseconds) - Not accessible to us
Level 2: RAM/Memory Cache (microseconds) - NSCache
Level 3: Disk Cache (milliseconds) - FileManager
Level 4: Network (seconds) - URLSession
```

**Simple Dictionary Cache:**

```swift
// Beginner's first cache - Just a dictionary!
class SimpleCache {
    private var storage: [String: Data] = [:]
    
    func save(_ data: Data, forKey key: String) {
        storage[key] = data
        print("✅ Saved \(key) - Cache size: \(storage.count)")
    }
    
    func get(forKey key: String) -> Data? {
        if let data = storage[key] {
            print("🎯 Cache HIT for \(key)")
            return data
        } else {
            print("❌ Cache MISS for \(key)")
            return nil
        }
    }
    
    func clear() {
        storage.removeAll()
        print("🗑️ Cache cleared")
    }
}

// Usage Example
let cache = SimpleCache()

// Save user data
let userData = "John Doe".data(using: .utf8)!
cache.save(userData, forKey: "user_123")

// Retrieve it
if let cached = cache.get(forKey: "user_123") {
    let name = String(data: cached, encoding: .utf8)
    print("Got name from cache: \(name!)") // Fast! No network call
}

// Try to get something not cached
cache.get(forKey: "user_999") // Cache MISS - need to fetch from network
```

**Problem with Simple Dictionary Cache:**

```swift
// ⚠️ DANGER: Memory explosion!
let imageCache = SimpleCache()

// Download 100 images (each 5MB)
for i in 1...100 {
    let image = downloadLargeImage(id: i) // 5MB each
    let imageData = image.jpegData(compressionQuality: 1.0)!
    imageCache.save(imageData, forKey: "image_\(i)")
}

// Result: 500MB in memory! 💥 App crashes!
```

**We need:**
1. **Size limits** - Don't use all memory
2. **Eviction policy** - Remove old items when full
3. **Thread safety** - Multiple threads accessing cache
4. **Memory warnings** - Clear cache when iOS asks

***

### 🚀 INTERMEDIATE LEVEL - NSCache & Disk Caching

**NSCache - Apple's Built-in Memory Cache**

NSCache solves all the problems of dictionary caching:

```swift
import Foundation

// MARK: - NSCache Introduction

class ImageCacheManager {
    // NSCache is thread-safe and auto-evicts under memory pressure
    private let memoryCache = NSCache<NSString, UIImage>()
    
    init() {
        // Configure cache limits
        memoryCache.countLimit = 100        // Max 100 images
        memoryCache.totalCostLimit = 50 * 1024 * 1024  // Max 50MB
        
        // Listen for memory warnings
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(clearCache),
            name: UIApplication.didReceiveMemoryWarningNotification,
            object: nil
        )
    }
    
    func cacheImage(_ image: UIImage, forKey key: String) {
        // Calculate image size in bytes
        let imageSize = image.jpegData(compressionQuality: 1.0)?.count ?? 0
        
        // Store with cost (helps NSCache make eviction decisions)
        memoryCache.setObject(image, forKey: key as NSString, cost: imageSize)
        
        print("✅ Cached image '\(key)' - Size: \(imageSize / 1024)KB")
    }
    
    func getImage(forKey key: String) -> UIImage? {
        if let cached = memoryCache.object(forKey: key as NSString) {
            print("🎯 Memory cache HIT: \(key)")
            return cached
        }
        print("❌ Memory cache MISS: \(key)")
        return nil
    }
    
    @objc private func clearCache() {
        memoryCache.removeAllObjects()
        print("⚠️ Memory warning - Cache cleared!")
    }
}

// Usage
let cacheManager = ImageCacheManager()

// Cache an image
if let image = UIImage(named: "profile") {
    cacheManager.cacheImage(image, forKey: "user_123_avatar")
}

// Retrieve it instantly
if let cachedImage = cacheManager.getImage(forKey: "user_123_avatar") {
    imageView.image = cachedImage  // Instant! No network call
}
```

**Key NSCache Features:**

1. **Thread-safe** - No need for locks
2. **Auto-eviction** - Removes items when memory is low
3. **Cost-based** - Smart eviction using cost parameter
4. **Count limit** - Maximum number of items
5. **Total cost limit** - Maximum total size

**NSCache Eviction Policy:**

```swift
class NSCacheExplained {
    let cache = NSCache<NSString, NSData>()
    
    func demonstrateEviction() {
        // Set limits
        cache.countLimit = 3  // Only 3 items max
        
        // Add items
        cache.setObject("Data 1".data(using: .utf8)! as NSData, forKey: "1")
        cache.setObject("Data 2".data(using: .utf8)! as NSData, forKey: "2")
        cache.setObject("Data 3".data(using: .utf8)! as NSData, forKey: "3")
        
        print("Cache is full (3 items)")
        
        // Add 4th item - NSCache will evict one automatically
        cache.setObject("Data 4".data(using: .utf8)! as NSData, forKey: "4")
        
        // NSCache decides which one to evict (usually least recently used)
        // You can't control which one it removes!
    }
}
```

***

### **Disk Caching - Persistent Storage**

Memory cache is lost when app terminates. Disk cache persists.

```swift
import Foundation

class DiskCacheManager {
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    
    init() {
        // Use iOS Caches directory (can be purged by system)
        let paths = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)
        cacheDirectory = paths[0].appendingPathComponent("ImageCache")
        
        // Create directory if it doesn't exist
        try? fileManager.createDirectory(at: cacheDirectory, 
                                         withIntermediateDirectories: true)
        
        print("📁 Cache directory: \(cacheDirectory.path)")
    }
    
    // MARK: - Save to Disk
    func saveToDisk(_ data: Data, forKey key: String) {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        
        do {
            try data.write(to: fileURL)
            print("✅ Saved to disk: \(key) (\(data.count / 1024)KB)")
        } catch {
            print("❌ Failed to save: \(error)")
        }
    }
    
    // MARK: - Load from Disk
    func loadFromDisk(forKey key: String) -> Data? {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        
        do {
            let data = try Data(contentsOf: fileURL)
            print("🎯 Loaded from disk: \(key) (\(data.count / 1024)KB)")
            return data
        } catch {
            print("❌ Not found on disk: \(key)")
            return nil
        }
    }
    
    // MARK: - Delete from Disk
    func deleteFromDisk(forKey key: String) {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        try? fileManager.removeItem(at: fileURL)
        print("🗑️ Deleted from disk: \(key)")
    }
    
    // MARK: - Clear All Cache
    func clearDiskCache() {
        guard let files = try? fileManager.contentsOfDirectory(at: cacheDirectory, 
                                                               includingPropertiesForKeys: nil) else {
            return
        }
        
        for file in files {
            try? fileManager.removeItem(at: file)
        }
        
        print("🗑️ Cleared all disk cache (\(files.count) files)")
    }
    
    // MARK: - Get Cache Size
    func getCacheSize() -> Int {
        guard let files = try? fileManager.contentsOfDirectory(at: cacheDirectory, 
                                                               includingPropertiesForKeys: [.fileSizeKey]) else {
            return 0
        }
        
        let totalSize = files.reduce(0) { total, file in
            let size = (try? file.resourceValues(forKeys: [.fileSizeKey]))?.fileSize ?? 0
            return total + size
        }
        
        print("📊 Total cache size: \(totalSize / 1024 / 1024)MB")
        return totalSize
    }
}

// Usage
let diskCache = DiskCacheManager()

// Save image to disk
if let imageData = UIImage(named: "photo")?.jpegData(compressionQuality: 0.8) {
    diskCache.saveToDisk(imageData, forKey: "photo_123.jpg")
}

// Load it back (even after app restart!)
if let data = diskCache.loadFromDisk(forKey: "photo_123.jpg"),
   let image = UIImage(data: data) {
    imageView.image = image
}

// Check cache size
diskCache.getCacheSize()

// Clear old cache
diskCache.clearDiskCache()
```

***

### **Two-Level Cache - Best of Both Worlds**

Combine memory (fast) + disk (persistent):

```swift
class TwoLevelImageCache {
    // Level 1: Fast memory cache
    private let memoryCache = NSCache<NSString, UIImage>()
    
    // Level 2: Persistent disk cache
    private let diskCache = DiskCacheManager()
    
    init() {
        memoryCache.countLimit = 50
        memoryCache.totalCostLimit = 30 * 1024 * 1024  // 30MB
    }
    
    // MARK: - Get Image (Check both levels)
    func getImage(forKey key: String) -> UIImage? {
        // 1. Check memory first (fastest)
        if let memoryImage = memoryCache.object(forKey: key as NSString) {
            print("🎯 L1 Cache HIT (memory): \(key)")
            return memoryImage
        }
        
        // 2. Check disk (slower but still fast)
        if let diskData = diskCache.loadFromDisk(forKey: key),
           let diskImage = UIImage(data: diskData) {
            print("🎯 L2 Cache HIT (disk): \(key)")
            
            // Promote to memory cache for next time
            memoryCache.setObject(diskImage, forKey: key as NSString)
            
            return diskImage
        }
        
        print("❌ Cache MISS (both levels): \(key)")
        return nil
    }
    
    // MARK: - Cache Image (Save to both levels)
    func cacheImage(_ image: UIImage, forKey key: String) {
        // 1. Save to memory (fast access)
        memoryCache.setObject(image, forKey: key as NSString)
        
        // 2. Save to disk (persistent)
        if let imageData = image.jpegData(compressionQuality: 0.8) {
            diskCache.saveToDisk(imageData, forKey: key)
        }
        
        print("✅ Cached in both levels: \(key)")
    }
    
    // MARK: - Remove from Cache
    func removeImage(forKey key: String) {
        memoryCache.removeObject(forKey: key as NSString)
        diskCache.deleteFromDisk(forKey: key)
    }
}

// Usage with UITableView
class ImageCell: UITableViewCell {
    let imageCache = TwoLevelImageCache()
    
    func configure(imageURL: String) {
        // Try cache first
        if let cachedImage = imageCache.getImage(forKey: imageURL) {
            self.imageView?.image = cachedImage
            return  // Done! No network call needed
        }
        
        // Not cached - download it
        downloadImage(from: imageURL) { [weak self] image in
            guard let self = self, let image = image else { return }
            
            // Cache for next time
            self.imageCache.cacheImage(image, forKey: imageURL)
            
            // Update UI
            DispatchQueue.main.async {
                self.imageView?.image = image
            }
        }
    }
    
    func downloadImage(from url: String, completion: @escaping (UIImage?) -> Void) {
        // Network download implementation
    }
}
```

**Performance Comparison:**

```
Network request:     1000ms  (1 second)
Disk cache:          50ms    (20x faster)
Memory cache:        1ms     (1000x faster!)
```


### 💎 ADVANCED LEVEL - LRU Cache Implementation

**What is LRU (Least Recently Used)?**

Think of a music playlist:
- Songs you play often stay in "Recently Played"
- Songs you haven't played in months get removed
- Most recently played song goes to top

**LRU Cache Rules:**
1. **Fixed capacity** - Max N items
2. **Most recently used** - Goes to front
3. **Least recently used** - Gets evicted when full
4. **O(1) operations** - Get and Put must be constant time

**Data Structure:** HashMap + Doubly Linked List

```swift
// MARK: - LRU Cache Node (Doubly Linked List)

class LRUNode<Key: Hashable, Value> {
    var key: Key
    var value: Value
    var prev: LRUNode?
    var next: LRUNode?
    
    init(key: Key, value: Value) {
        self.key = key
        self.value = value
    }
}

// MARK: - Production-Grade LRU Cache

class LRUCache<Key: Hashable, Value> {
    private let capacity: Int
    private var cache: [Key: LRUNode<Key, Value>] = [:]
    
    // Dummy head and tail for easier list manipulation
    private let head = LRUNode<Key, Value>(key: "" as! Key, value: "" as! Value)
    private let tail = LRUNode<Key, Value>(key: "" as! Key, value: "" as! Value)
    
    private let lock = NSLock()  // Thread safety
    
    init(capacity: Int) {
        self.capacity = capacity
        
        // Connect head and tail
        head.next = tail
        tail.prev = head
    }
    
    // MARK: - Get (O(1))
    func get(_ key: Key) -> Value? {
        lock.lock()
        defer { lock.unlock() }
        
        guard let node = cache[key] else {
            return nil
        }
        
        // Move to front (most recently used)
        moveToHead(node)
        
        return node.value
    }
    
    // MARK: - Put (O(1))
    func put(_ key: Key, _ value: Value) {
        lock.lock()
        defer { lock.unlock() }
        
        if let existingNode = cache[key] {
            // Update value and move to front
            existingNode.value = value
            moveToHead(existingNode)
        } else {
            // Create new node
            let newNode = LRUNode(key: key, value: value)
            cache[key] = newNode
            addToHead(newNode)
            
            // Check capacity
            if cache.count > capacity {
                // Remove least recently used (tail.prev)
                if let lruNode = tail.prev, lruNode !== head {
                    removeNode(lruNode)
                    cache.removeValue(forKey: lruNode.key)
                }
            }
        }
    }
    
    // MARK: - Remove
    func remove(_ key: Key) {
        lock.lock()
        defer { lock.unlock() }
        
        guard let node = cache[key] else { return }
        
        removeNode(node)
        cache.removeValue(forKey: key)
    }
    
    // MARK: - Clear
    func clear() {
        lock.lock()
        defer { lock.unlock() }
        
        cache.removeAll()
        head.next = tail
        tail.prev = head
    }
    
    // MARK: - Private Helper Methods
    
    private func addToHead(_ node: LRUNode<Key, Value>) {
        node.prev = head
        node.next = head.next
        head.next?.prev = node
        head.next = node
    }
    
    private func removeNode(_ node: LRUNode<Key, Value>) {
        node.prev?.next = node.next
        node.next?.prev = node.prev
    }
    
    private func moveToHead(_ node: LRUNode<Key, Value>) {
        removeNode(node)
        addToHead(node)
    }
    
    // MARK: - Debug
    func printCache() {
        lock.lock()
        defer { lock.unlock() }
        
        var current = head.next
        var items: [String] = []
        
        while current !== tail {
            items.append("\(current!.key)")
            current = current?.next
        }
        
        print("Cache (MRU → LRU): \(items.joined(separator: " → "))")
    }
}

// MARK: - Usage Example

let cache = LRUCache<String, UIImage>(capacity: 3)

// Add images
cache.put("image1", UIImage(named: "photo1")!)
cache.put("image2", UIImage(named: "photo2")!)
cache.put("image3", UIImage(named: "photo3")!)

cache.printCache()
// Output: Cache (MRU → LRU): image3 → image2 → image1

// Access image1 (becomes most recently used)
_ = cache.get("image1")

cache.printCache()
// Output: Cache (MRU → LRU): image1 → image3 → image2

// Add image4 (cache is full, evicts image2 - least recently used)
cache.put("image4", UIImage(named: "photo4")!)

cache.printCache()
// Output: Cache (MRU → LRU): image4 → image1 → image3

// Try to get image2 (evicted!)
if cache.get("image2") == nil {
    print("image2 was evicted (LRU)")
}
```

**Time Complexity:**
- `get()`: O(1) - HashMap lookup + List update
- `put()`: O(1) - HashMap insert + List update
- `remove()`: O(1) - HashMap delete + List update

**Space Complexity:** O(capacity)

***

### **Advanced: Size-Based LRU Cache**

Instead of count, evict based on total size:

```swift
class SizeLRUCache<Key: Hashable> {
    private let maxSize: Int  // In bytes
    private var currentSize: Int = 0
    
    private var cache: [Key: SizeLRUNode] = [:]
    private let head = SizeLRUNode(key: "" as! Key, data: Data(), size: 0)
    private let tail = SizeLRUNode(key: "" as! Key, data: Data(), size: 0)
    
    private let lock = NSLock()
    
    class SizeLRUNode {
        let key: Key
        var data: Data
        let size: Int
        var prev: SizeLRUNode?
        var next: SizeLRUNode?
        
        init(key: Key, data: Data, size: Int) {
            self.key = key
            self.data = data
            self.size = size
        }
    }
    
    init(maxSize: Int) {
        self.maxSize = maxSize
        head.next = tail
        tail.prev = head
    }
    
    func get(_ key: Key) -> Data? {
        lock.lock()
        defer { lock.unlock() }
        
        guard let node = cache[key] else {
            return nil
        }
        
        moveToHead(node)
        return node.data
    }
    
    func put(_ key: Key, _ data: Data) {
        lock.lock()
        defer { lock.unlock() }
        
        let size = data.count
        
        // Remove existing if updating
        if let existing = cache[key] {
            currentSize -= existing.size
            removeNode(existing)
        }
        
        // Evict until we have space
        while currentSize + size > maxSize && tail.prev !== head {
            if let lru = tail.prev {
                currentSize -= lru.size
                removeNode(lru)
                cache.removeValue(forKey: lru.key)
                print("⚠️ Evicted \(lru.key) (\(lru.size / 1024)KB) - Cache full")
            }
        }
        
        // Add new node
        let newNode = SizeLRUNode(key: key, data: data, size: size)
        cache[key] = newNode
        addToHead(newNode)
        currentSize += size
        
        print("✅ Cached \(key) (\(size / 1024)KB) - Total: \(currentSize / 1024)KB / \(maxSize / 1024)KB")
    }
    
    private func addToHead(_ node: SizeLRUNode) {
        node.prev = head
        node.next = head.next
        head.next?.prev = node
        head.next = node
    }
    
    private func removeNode(_ node: SizeLRUNode) {
        node.prev?.next = node.next
        node.next?.prev = node.prev
    }
    
    private func moveToHead(_ node: SizeLRUNode) {
        removeNode(node)
        addToHead(node)
    }
}

// Usage
let imageCache = SizeLRUCache<String>(maxSize: 10 * 1024 * 1024)  // 10MB max

// Cache images
let image1 = UIImage(named: "large1")!.jpegData(compressionQuality: 0.8)!  // 3MB
let image2 = UIImage(named: "large2")!.jpegData(compressionQuality: 0.8)!  // 4MB
let image3 = UIImage(named: "large3")!.jpegData(compressionQuality: 0.8)!  // 5MB

imageCache.put("img1", image1)  // 3MB cached
imageCache.put("img2", image2)  // 7MB total
imageCache.put("img3", image3)  // 12MB total - evicts img1 (LRU)

// img1 is evicted, img2 and img3 remain (9MB total < 10MB limit)
```


### **Production-Ready: Complete Caching System**

Combining everything for FAANG-level implementation:

```swift
// MARK: - Complete Image Caching System

actor ImageCachingSystem {
    // Three-level cache hierarchy
    private let memoryCache: NSCache<NSString, UIImage>
    private let diskCache: DiskCache
    private let lruCache: LRUCache<String, CacheMetadata>
    
    // Track ongoing downloads (avoid duplicates)
    private var ongoingDownloads: [String: Task<UIImage?, Error>] = [:]
    
    // Configuration
    private let memoryCacheSize: Int
    private let diskCacheSize: Int
    private let maxConcurrentDownloads = 3
    
    struct CacheMetadata {
        let key: String
        let size: Int
        let timestamp: Date
        let accessCount: Int
    }
    
    init(
        memoryCacheSize: Int = 50 * 1024 * 1024,  // 50MB
        diskCacheSize: Int = 200 * 1024 * 1024     // 200MB
    ) {
        self.memoryCacheSize = memoryCacheSize
        self.diskCacheSize = diskCacheSize
        
        // Setup memory cache
        self.memoryCache = NSCache<NSString, UIImage>()
        memoryCache.totalCostLimit = memoryCacheSize
        memoryCache.countLimit = 100
        
        // Setup disk cache
        self.diskCache = DiskCache(maxSize: diskCacheSize)
        
        // Setup LRU tracking
        self.lruCache = LRUCache<String, CacheMetadata>(capacity: 200)
        
        // Setup memory warning observer
        Task { @MainActor in
            NotificationCenter.default.addObserver(
                forName: UIApplication.didReceiveMemoryWarningNotification,
                object: nil,
                queue: .main
            ) { [weak self] _ in
                Task {
                    await self?.handleMemoryWarning()
                }
            }
        }
    }
    
    // MARK: - Main Get Method
    
    func getImage(from url: URL, priority: TaskPriority = .medium) async throws -> UIImage {
        let key = url.absoluteString
        
        // Level 1: Memory cache (fastest)
        if let memoryImage = memoryCache.object(forKey: key as NSString) {
            print("🎯 L1 HIT (memory): \(key)")
            updateAccessMetadata(key: key)
            return memoryImage
        }
        
        // Level 2: Disk cache (fast)
        if let diskData = await diskCache.load(key: key),
           let diskImage = UIImage(data: diskData) {
            print("🎯 L2 HIT (disk): \(key)")
            
            // Promote to memory
            let imageSize = diskData.count
            memoryCache.setObject(diskImage, forKey: key as NSString, cost: imageSize)
            
            updateAccessMetadata(key: key)
            return diskImage
        }
        
        // Level 3: Network (slow) - check for ongoing download first
        if let existingTask = ongoingDownloads[key] {
            print("⏳ Waiting for ongoing download: \(key)")
            return try await existingTask.value ?? UIImage()
        }
        
        // Start new download
        print("📡 Downloading: \(key)")
        let downloadTask = Task(priority: priority) {
            try await self.downloadAndCache(url: url, key: key)
        }
        
        ongoingDownloads[key] = downloadTask
        
        do {
            let image = try await downloadTask.value
            ongoingDownloads.removeValue(forKey: key)
            return image ?? UIImage()
        } catch {
            ongoingDownloads.removeValue(forKey: key)
            throw error
        }
    }
    
    // MARK: - Download & Cache
    
    private func downloadAndCache(url: URL, key: String) async throws -> UIImage? {
        let (data, _) = try await URLSession.shared.data(from: url)
        
        guard let image = UIImage(data: data) else {
            throw CacheError.invalidImage
        }
        
        // Optimize image size
        let optimizedData = optimizeImage(image)
        let imageSize = optimizedData.count
        
        // Cache in memory
        memoryCache.setObject(image, forKey: key as NSString, cost: imageSize)
        
        // Cache on disk
        await diskCache.save(optimizedData, key: key)
        
        // Track in LRU
        let metadata = CacheMetadata(
            key: key,
            size: imageSize,
            timestamp: Date(),
            accessCount: 1
        )
        lruCache.put(key, metadata)
        
        print("✅ Cached: \(key) (\(imageSize / 1024)KB)")
        
        return image
    }
    
    // MARK: - Helper Methods
    
    private func optimizeImage(_ image: UIImage) -> Data {
        // Resize if too large
        let maxDimension: CGFloat = 2048
        var optimized = image
        
        if image.size.width > maxDimension || image.size.height > maxDimension {
            let scale = maxDimension / max(image.size.width, image.size.height)
            let newSize = CGSize(
                width: image.size.width * scale,
                height: image.size.height * scale
            )
            
            UIGraphicsBeginImageContextWithOptions(newSize, false, 0)
            image.draw(in: CGRect(origin: .zero, size: newSize))
            optimized = UIGraphicsGetImageFromCurrentImageContext() ?? image
            UIGraphicsEndImageContext()
        }
        
        // Compress
        return optimized.jpegData(compressionQuality: 0.75) ?? Data()
    }
    
    private func updateAccessMetadata(key: String) {
        if let metadata = lruCache.get(key) {
            let updated = CacheMetadata(
                key: key,
                size: metadata.size,
                timestamp: Date(),
                accessCount: metadata.accessCount + 1
            )
            lruCache.put(key, updated)
        }
    }
    
    private func handleMemoryWarning() {
        print("⚠️ Memory warning - Clearing memory cache")
        memoryCache.removeAllObjects()
    }
    
    // MARK: - Cache Management
    
    func clearMemoryCache() {
        memoryCache.removeAllObjects()
        print("🗑️ Memory cache cleared")
    }
    
    func clearDiskCache() async {
        await diskCache.clear()
        print("🗑️ Disk cache cleared")
    }
    
    func clearAll() async {
        clearMemoryCache()
        await clearDiskCache()
        lruCache.clear()
        print("🗑️ All caches cleared")
    }
    
    func getCacheSize() async -> (memory: Int, disk: Int) {
        let diskSize = await diskCache.getSize()
        return (memory: memoryCacheSize, disk: diskSize)
    }
}

// MARK: - Disk Cache Actor

actor DiskCache {
    private let maxSize: Int
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    
    init(maxSize: Int) {
        self.maxSize = maxSize
        
        let paths = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)
        self.cacheDirectory = paths[0].appendingPathComponent("AdvancedImageCache")
        
        try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
    }
    
    func save(_ data: Data, key: String) {
        let fileURL = cacheDirectory.appendingPathComponent(key.md5Hash)
        try? data.write(to: fileURL)
    }
    
    func load(key: String) -> Data? {
        let fileURL = cacheDirectory.appendingPathComponent(key.md5Hash)
        return try? Data(contentsOf: fileURL)
    }
    
    func clear() {
        guard let files = try? fileManager.contentsOfDirectory(at: cacheDirectory, includingPropertiesForKeys: nil) else {
            return
        }
        
        for file in files {
            try? fileManager.removeItem(at: file)
        }
    }
    
    func getSize() -> Int {
        guard let files = try? fileManager.contentsOfDirectory(at: cacheDirectory, includingPropertiesForKeys: [.fileSizeKey]) else {
            return 0
        }
        
        return files.reduce(0) { total, file in
            let size = (try? file.resourceValues(forKeys: [.fileSizeKey]))?.fileSize ?? 0
            return total + size
        }
    }
}

enum CacheError: Error {
    case invalidImage
}

extension String {
    var md5Hash: String {
        // Simple hash for demo - use actual MD5 in production
        return String(self.hashValue)
    }
}
```

***

### **UITableView Integration Example:**

```swift
class ImageFeedViewController: UITableViewController {
    let imageCache = ImageCachingSystem()
    var imageURLs: [URL] = []
    
    override func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "ImageCell", for: indexPath) as! ImageCell
        let url = imageURLs[indexPath.row]
        
        cell.configure(with: url, cache: imageCache)
        
        return cell
    }
}

class ImageCell: UITableViewCell {
    @IBOutlet weak var photoImageView: UIImageView!
    private var loadTask: Task<Void, Never>?
    
    func configure(with url: URL, cache: ImageCachingSystem) {
        // Cancel previous load
        loadTask?.cancel()
        
        // Show placeholder
        photoImageView.image = UIImage(named: "placeholder")
        
        // Load image
        loadTask = Task {
            do {
                let image = try await cache.getImage(from: url, priority: .userInitiated)
                
                guard !Task.isCancelled else { return }
                
                await MainActor.run {
                    UIView.transition(
                        with: self.photoImageView,
                        duration: 0.3,
                        options: .transitionCrossDissolve
                    ) {
                        self.photoImageView.image = image
                    }
                }
            } catch {
                print("Failed to load image: \(error)")
            }
        }
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        loadTask?.cancel()
        photoImageView.image = nil
    }
}
```

***
<!-- INTERVIEW_QUESTIONS_START -->

## 📝 CACHING STRATEGIES - INTERVIEW QUESTIONS

### **Q1: Implement an LRU Cache with O(1) get and put operations**

**What They're Testing:** Data structures knowledge, algorithm optimization

**Expected Answer (Tech Lead Level):**

```swift
// Complete implementation shown above in Advanced section
// Key points to mention:

/*
1. DATA STRUCTURES USED:
   - HashMap: O(1) lookup
   - Doubly Linked List: O(1) insertion/deletion
   
2. WHY DOUBLY LINKED LIST?
   - Need to move nodes to front (most recently used)
   - Need to remove nodes from middle
   - Singly linked list can't remove from middle in O(1)
   
3. DUMMY HEAD & TAIL:
   - Simplifies edge cases
   - No need to check for null
   - Cleaner code
   
4. OPERATIONS:
   get(key):
   - Check HashMap -> O(1)
   - If found, move to head -> O(1)
   - Return value
   
   put(key, value):
   - Check if exists -> O(1)
   - If yes: update value, move to head -> O(1)
   - If no: add to head, check capacity
   - If over capacity: remove tail.prev (LRU) -> O(1)
   
5. THREAD SAFETY:
   - Add NSLock for concurrent access
   - Lock before any operation
   - Use defer to ensure unlock
*/

// Walk through example:
let cache = LRUCache<String, Int>(capacity: 2)

cache.put("A", 1)  // Cache: A
cache.put("B", 2)  // Cache: B → A (B most recent)
cache.get("A")     // Cache: A → B (A accessed, now most recent)
cache.put("C", 3)  // Cache: C → A (B evicted as LRU)
cache.get("B")     // nil (B was evicted)
```

**Follow-up Questions:**
1. "How would you make it thread-safe?" → Add NSLock
2. "How would you add expiration time?" → Add timestamp to node, check on get
3. "What if you want to evict by size instead of count?" → Track total size, evict until under limit

***

### **Q2: Design a multi-level caching system for an Instagram-like feed. Explain your eviction strategy.**

**What They're Testing:** System design, real-world problem solving, trade-off analysis

**Expected Answer:**

```swift
/*
REQUIREMENT ANALYSIS:
- Feed has images (large files)
- Users scroll fast (need instant loading)
- Limited device storage
- Works offline
- Handles 1000s of images

SOLUTION: Three-Level Cache Hierarchy

LEVEL 1: MEMORY CACHE (NSCache)
  - Capacity: 50MB
  - Stores: Most recent ~50 images
  - Eviction: Automatic by iOS
  - Why: Instant access, no I/O
  - Limitation: Lost on app restart

LEVEL 2: DISK CACHE (FileManager)
  - Capacity: 200MB
  - Stores: ~200-500 images
  - Eviction: LRU with timestamp
  - Why: Persists across app restarts
  - Limitation: Slower than memory

LEVEL 3: NETWORK (URLSession)
  - Capacity: Unlimited
  - Stores: Everything
  - Eviction: N/A
  - Why: Source of truth
  - Limitation: Slow, requires internet

EVICTION STRATEGY:

1. Memory Cache:
   - Let NSCache handle automatically
   - Set totalCostLimit based on device RAM
   - Evicts LRU when memory pressure
   
2. Disk Cache:
   - Custom LRU implementation
   - Track last access time
   - Evict oldest when exceeding 200MB
   - Exception: Keep "favorited" images
   
3. Smart Prefetching:
   - Prefetch next 10 images in scroll direction
   - Lower priority tasks
   - Cancel if user scrolls away
*/

// Implementation shown in production caching system above

/*
KEY DESIGN DECISIONS:

Q: "Why three levels?"
A: Speed vs Capacity trade-off. Memory is fastest but limited. Disk balances speed and capacity. Network is source of truth.

Q: "Why LRU for disk?"
A: Popular images stay cached. Users often scroll back to recent posts.

Q: "Why NSCache for memory?"
A: Handles memory pressure automatically. Don't want to crash on low-memory devices.

Q: "Expiration policy?"
A: 7 days for disk. Balance between freshness and bandwidth savings.

Q: "Prefetching strategy?"
A: Predict user behavior (scrolling down), prefetch at low priority to save bandwidth.
*/
```

***

### **Q3: How would you handle cache invalidation when data changes on the server?**

**What They're Testing:** Cache coherency, real-world problems, API design

**Expected Answer:**

```swift
/*
CACHE INVALIDATION STRATEGIES:

1. TIME-BASED EXPIRATION (TTL - Time To Live)
   - Each cache entry has expiration timestamp
   - Simple but may show stale data
   
2. VERSION-BASED INVALIDATION
   - Server sends version/etag header
   - Client checks version before using cache
   - Efficient but requires server support
   
3. EVENT-BASED INVALIDATION
   - Server pushes invalidation events
   - WebSocket or Push Notifications
   - Real-time but complex
   
4. ON-DEMAND INVALIDATION
   - User triggers refresh (pull-to-refresh)
   - Simple but relies on user action
*/

// Implementation:

actor SmartCache<Key: Hashable, Value: Codable> {
    private var cache: [Key: CacheEntry] = [:]
    
    struct CacheEntry {
        let value: Value
        let timestamp: Date
        let version: String?
        let ttl: TimeInterval
        
        var isExpired: Bool {
            Date().timeIntervalSince(timestamp) > ttl
        }
    }
    
    // STRATEGY 1: TTL-Based
    func get(_ key: Key, ttl: TimeInterval = 300) -> Value? {
        guard let entry = cache[key] else { return nil }
        
        // Check if expired
        if Date().timeIntervalSince(entry.timestamp) > ttl {
            cache.removeValue(forKey: key)
            return nil  // Force refresh
        }
        
        return entry.value
    }
    
    // STRATEGY 2: Version-Based (ETag)
    func getWithVersionCheck(
        _ key: Key,
        currentVersion: String?,
        fetch: () async throws -> (Value, String?)
    ) async throws -> Value {
        // Check cache
        if let entry = cache[key],
           !entry.isExpired {
            
            // Compare versions
            if let cachedVersion = entry.version,
               let serverVersion = currentVersion,
               cachedVersion == serverVersion {
                // Cache is fresh
                return entry.value
            }
        }
        
        // Cache miss or stale - fetch fresh data
        let (value, version) = try await fetch()
        
        // Update cache
        cache[key] = CacheEntry(
            value: value,
            timestamp: Date(),
            version: version,
            ttl: 300
        )
        
        return value
    }
    
    // STRATEGY 3: Event-Based Invalidation
    func invalidate(_ key: Key) {
        cache.removeValue(forKey: key)
        print("��️ Invalidated cache for: \(key)")
    }
    
    func invalidateAll() {
        cache.removeAll()
        print("🗑️ Invalidated entire cache")
    }
}

/*
RECOMMENDATION FOR FAANG INTERVIEW:

"I would use a COMBINATION approach:

1. TTL for baseline (5 min for most data)
2. ETags/Conditional Requests for efficiency
3. WebSocket events for real-time updates
4. Pull-to-refresh for user control

Example:
- User profile: 30 min TTL + ETag
- Feed posts: 5 min TTL + WebSocket events
- Static content: 24 hour TTL
- User-generated content: ETag only

This balances:
✅ Performance (cache hits)
✅ Freshness (not too stale)
✅ Bandwidth (304 responses)
✅ User control (manual refresh)
"
*/
```

***

### **Q4: Explain the difference between NSCache and a Dictionary. When would you use each?**

**What They're Testing:** iOS SDK knowledge, understanding of trade-offs

**Expected Answer:**

```swift
/*
NSCACHE vs DICTIONARY COMPARISON:

┌─────────────────┬──────────────────┬─────────────────┐
│ Feature         │ NSCache          │ Dictionary      │
├─────────────────┼──────────────────┼─────────────────┤
│ Thread Safety   │ ✅ Built-in       │ ❌ Manual needed │
│ Auto Eviction   │ ✅ Memory pressure│ ❌ Never         │
│ Memory Limit    │ ✅ Can set        │ ❌ Grows forever │
│ Cost Tracking   │ ✅ Per-item cost  │ ❌ No concept    │
│ Codable         │ ❌ No             │ ✅ Yes           │
│ Persistence     │ ❌ Memory only    │ ❌ Memory only   │
│ Subscripting    │ ❌ No [key]       │ ✅ Yes [key]     │
│ Performance     │ Fast             │ Fastest         │
└─────────────────┴──────────────────┴─────────────────┘

WHEN TO USE NSCACHE:
✅ Caching large objects (images, data)
✅ Caching expensive computations
✅ Multi-threaded access
✅ Don't want to manually manage memory
✅ System should auto-clear on low memory
✅ OK with items disappearing

WHEN TO USE DICTIONARY:
✅ Small, critical data (settings, configs)
✅ Need subscript syntax
✅ Need to serialize (Codable)
✅ Single-threaded access
✅ Data must never disappear
✅ Need dictionary operations (map, filter)
*/

// EXAMPLE 1: NSCache for Images
class ImageCache {
    private let cache = NSCache<NSString, UIImage>()
    
    init() {
        // Configure limits
        cache.countLimit = 100
        cache.totalCostLimit = 50 * 1024 * 1024  // 50MB
        
        // Will auto-evict under memory pressure
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak cache] _ in
            cache?.removeAllObjects()
        }
    }
}

// EXAMPLE 2: Dictionary for Settings
class SettingsManager {
    // Small, critical data - must never disappear
    private var settings: [String: Any] = [
        "theme": "dark",
        "notifications": true,
        "language": "en"
    ]
    
    // Thread safety if needed
    private let queue = DispatchQueue(label: "settings")
    
    func getSetting<T>(_ key: String) -> T? {
        return queue.sync {
            return settings[key] as? T
        }
    }
}

/*
INTERVIEW TIP:
Always mention: "NSCache is designed for DISPOSABLE cached data that can be regenerated.
If data is critical and can't be regenerated (auth tokens, user data), use Dictionary
or better yet, persistent storage like Keychain or Core Data."
*/
```

***

### **Q5: Design an offline-first caching strategy for a ride-hailing app like Uber**

**What They're Testing:** Real-world mobile app design, offline handling

**Expected Answer:**

```swift
/*
REQUIREMENTS:
1. Show last known driver location even offline
2. Cache recent trip history
3. Save favorite locations
4. Handle requests made offline
5. Sync when online again

ARCHITECTURE:

┌─────────────────────────────────────────┐
│         OFFLINE-FIRST LAYERS            │
├─────────────────────────────────────────┤
│ UI Layer (Always shows data)            │
├─────────────────────────────────────────┤
│ Cache Layer (Local truth)               │
│  - Memory: Active trip data             │
│  - Disk: Recent trips, favorites        │
│  - Database: All persistent data        │
├─────────────────────────────────────────┤
│ Sync Layer (Background sync)            │
│  - Queue pending requests               │
│  - Sync when online                     │
│  - Conflict resolution                  │
├─────────────────────────────────────────┤
│ Network Layer (Source of truth)         │
└─────────────────────────────────────────┘

CACHING STRATEGY:

1. DRIVER LOCATION (Real-time, Memory Cache)
   - Cache last 100 driver locations
   - TTL: 30 seconds
   - Update via WebSocket
   - Fallback to cached when offline
   
2. TRIP HISTORY (Persistent, Disk Cache)
   - Cache last 50 trips
   - No TTL (permanent until cleared)
   - Refresh on app launch
   
3. FAVORITE LOCATIONS (Critical, Database)
   - Store in CoreData
   - Never expire
   - Sync changes when online
   
4. PENDING REQUESTS (Queue, Database)
   - Trip cancellations
   - Ratings
   - Payment updates
   - Replay when online

IMPLEMENTATION:
*/

// Data Layer
actor RideDataManager {
    // Memory: Active trip
    private var activeTrip: Trip?
    
    // Disk: Recent trips (LRU)
    private let tripCache = LRUCache<String, Trip>(capacity: 50)
    
    // Database: Favorites
    private let database: CoreDataStack
    
    // Sync queue
    private let syncQueue: SyncQueue
    
    // Network monitor
    private let networkMonitor: NetworkMonitor
    
    init() {
        self.database = CoreDataStack()
        self.syncQueue = SyncQueue()
        self.networkMonitor = NetworkMonitor()
        
        // Monitor network changes
        Task {
            for await isConnected in networkMonitor.statusStream {
                if isConnected {
                    await syncPendingChanges()
                }
            }
        }
    }
    
    // MARK: - Read Operations (Offline-first)
    
    func getTripHistory() async -> [Trip] {
        // 1. Return cached immediately
        var trips: [Trip] = []
        
        // Get from cache
        // (implementation details)
        
        // 2. Refresh in background if online
        if await networkMonitor.isConnected {
            Task {
                let freshTrips = try? await fetchTripsFromServer()
                // Update cache
            }
        }
        
        return trips
    }
    
    func getFavoriteLocations() async -> [Location] {
        // Always return from database (critical data)
        return await database.fetchFavorites()
    }
    
    // MARK: - Write Operations (Queue for sync)
    
    func cancelTrip(id: String) async {
        // 1. Update local state immediately (optimistic update)
        activeTrip?.status = .cancelled
        
        // 2. Queue for server sync
        let operation = SyncOperation(
            type: .cancelTrip,
            payload: ["trip_id": id],
            timestamp: Date()
        )
        await syncQueue.enqueue(operation)
        
        // 3. Attempt sync if online
        if await networkMonitor.isConnected {
            await syncPendingChanges()
        }
    }
    
    func addFavoriteLocation(_ location: Location) async {
        // 1. Save to database immediately
        await database.saveFavorite(location)
        
        // 2. Queue for sync
        let operation = SyncOperation(
            type: .addFavorite,
            payload: location.toDictionary(),
            timestamp: Date()
        )
        await syncQueue.enqueue(operation)
        
        // 3. Sync when online
        if await networkMonitor.isConnected {
            await syncPendingChanges()
        }
    }
    
    // MARK: - Sync Logic
    
    private func syncPendingChanges() async {
        let operations = await syncQueue.getPending()
        
        for operation in operations {
            do {
                try await performSync(operation)
                await syncQueue.markCompleted(operation.id)
            } catch {
                // Keep in queue for retry
                print("Sync failed: \(error)")
            }
        }
    }
}

/*
KEY POINTS TO MENTION:

1. OPTIMISTIC UPDATES:
   - Update UI immediately
   - Queue for server sync
   - Rollback if sync fails

2. CACHE LAYERS:
   - Memory: Temporary, fast
   - Disk: Recent data
   - Database: Critical data

3. SYNC STRATEGY:
   - Queue all writes
   - Sync when online
   - Handle conflicts

4. USER EXPERIENCE:
   - App always works
   - No loading spinners for cached data
   - Show sync status indicator

5. TRADE-OFFS:
   - Stale data vs always working
   - Storage space vs offline capability
   - Complexity vs user experience
*/
```

***

## 🎯 Key Takeaways for FAANG Interviews

**For NSCache:**
- Mention automatic eviction under memory pressure
- Discuss `totalCostLimit` and `countLimit`
- Explain thread safety benefits
- Know when NOT to use it (critical data)

**For LRU Cache:**
- Explain HashMap + Doubly Linked List combo
- Emphasize O(1) operations
- Discuss thread safety with locks
- Mention real-world uses (Redis, database buffers)

**For Multi-Level Caching:**
- Memory (fast) → Disk (persistent) → Network (source of truth)
- Discuss eviction policies for each level
- Mention cache invalidation strategies
- Explain performance trade-offs

**Common Follow-ups:**
1. "How do you handle race conditions?" → Use actors or locks
2. "What about cache stampede?" → Use request deduplication
3. "How do you measure cache hit rate?" → Track hits vs misses
4. "When would you NOT cache?" → Real-time data, user-specific data

This comprehensive guide covers everything from beginner to tech lead level, with production-ready code suitable for FAANG interviews! 🚀

