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

### 🎈 BEGINNER LEVEL

**What is Caching?**

Think of caching like your brain's memory:
- You remember your friend's phone number (cache)
- Instead of looking it up in contacts every time (network request)
- You just recall it from memory (much faster!)

**Why Cache?**
1. **Speed** - Instant access
2. **Offline** - Works without internet
3. **Cost** - Save bandwidth and battery
4. **UX** - No loading spinners

**Simple Cache:**

```swift
// Simplest cache - Dictionary
class SimpleCache {
    private var storage: [String: String] = [:]
    
    func save(_ value: String, forKey key: String) {
        storage[key] = value
    }
    
    func get(forKey key: String) -> String? {
        return storage[key]
    }
    
    func remove(forKey key: String) {
        storage.removeValue(forKey: key)
    }
    
    func clear() {
        storage.removeAll()
    }
}

// Usage
let cache = SimpleCache()
cache.save("John Doe", forKey: "user_name")
let name = cache.get(forKey: "user_name")  // "John Doe"
```
