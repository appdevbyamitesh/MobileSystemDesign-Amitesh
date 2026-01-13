# iOS API Design Best Practices - Client & Server Perspective

> [!IMPORTANT]
> This guide covers API design from both perspectives: what iOS engineers should know about designing APIs (server-side) and how to consume them effectively (client-side).

---

## Why API Design Matters for iOS Interviews

**At Uber L4 level, you're expected to:**
- Design RESTful APIs for mobile features
- Discuss trade-offs (REST vs GraphQL vs gRPC)
- Handle pagination, versioning, error responses
- Optimize for mobile constraints (bandwidth, battery)
- Understand backend implications of client requests

---

## Part 1: RESTful API Design Fundamentals

### HTTP Methods & Semantics

```
GET     - Retrieve resources (idempotent, cacheable)
POST    - Create new resources
PUT     - Replace entire resource (idempotent)
PATCH   - Partial update
DELETE  - Remove resource (idempotent)
```

**Interview Example:**

```swift
// BAD: Using POST for everything
POST /getTrips      // Wrong! GET should be used
POST /updateTrip    // Wrong! PUT/PATCH should be used

// GOOD: Proper HTTP semantics
GET    /trips                    // List trips
GET    /trips/{id}               // Get single trip
POST   /trips                    // Create new trip
PUT    /trips/{id}               // Update entire trip
PATCH  /trips/{id}               // Partial update (e.g., only status)
DELETE /trips/{id}               // Cancel/delete trip
```

### URL Design Best Practices

| Pattern | Good ✅ | Bad ❌ |
|---------|---------|---------|
| **Nouns, not verbs** | `/users`, `/trips` | `/getUsers`, `/createTrip` |
| **Plural resources** | `/users/123/trips` | `/user/123/trip` |
| **Hierarchical** | `/users/123/trips/456` | `/trip?userId=123&id=456` |
| **Lowercase** | `/user-profiles` | `/UserProfiles` |
| **Hyphens** | `/user-preferences` | `/user_preferences` |

### Request/Response Format

**Request Example:**
```json
POST /trips HTTP/1.1
Host: api.uber.com
Authorization: Bearer eyJhbGc...
Content-Type: application/json
Accept: application/json
X-Request-ID: uuid-12345

{
  "pickupLocation": {
    "latitude": 37.7749,
    "longitude": -122.4194,
    "address": "123 Main St, SF"
  },
  "dropoffLocation": {
    "latitude": 37.8044,
    "longitude": -122.2712,
    "address": "456 Oak Ave, Oakland"
  },
  "rideType": "uberX"
}
```

**Response Example:**
```json
HTTP/1.1 201 Created
Content-Type: application/json
X-Request-ID: uuid-12345
X-RateLimit-Remaining: 99

{
  "data": {
    "id": "trip_abc123",
    "status": "pending",
    "pickupLocation": { ... },
    "dropoffLocation": { ... },
    "estimatedFare": {
      "amount": 25.50,
      "currency": "USD"
    },
    "createdAt": "2026-01-13T18:08:00Z"
  },
  "meta": {
    "requestId": "uuid-12345",
    "timestamp": "2026-01-13T18:08:00Z"
  }
}
```

---

## Part 2: Status Codes & Error Handling

### HTTP Status Code Usage

| Code | Meaning | When to Use |
|------|---------|-------------|
| **200 OK** | Success | GET, PUT, PATCH successful |
| **201 Created** | Resource created | POST successful |
| **204 No Content** | Success, no body | DELETE successful |
| **304 Not Modified** | Cached version valid | Conditional GET |
| **400 Bad Request** | Invalid input | Validation errors |
| **401 Unauthorized** | Auth required | Missing/invalid token |
| **403 Forbidden** | Insufficient permissions | Valid token, wrong role |
| **404 Not Found** | Resource missing | ID doesn't exist |
| **409 Conflict** | State conflict | Duplicate, version mismatch |
| **422 Unprocessable** | Semantic error | Valid JSON, invalid logic |
| **429 Too Many Requests** | Rate limited | Exceeded quota |
| **500 Internal Error** | Server failure | Unexpected error |
| **503 Service Unavailable** | Temporary down | Maintenance, overload |

### Error Response Format

**Standard Error Structure:**
```json
{
  "error": {
    "code": "INVALID_PICKUP_LOCATION",
    "message": "Pickup location is outside service area",
    "details": [
      {
        "field": "pickupLocation.latitude",
        "issue": "Coordinates outside supported region"
      }
    ],
    "requestId": "uuid-12345",
    "timestamp": "2026-01-13T18:08:00Z",
    "retryable": false
  }
}
```

**iOS Client Handling:**
```swift
struct APIError: Codable, LocalizedError {
    let code: String
    let message: String
    let details: [ErrorDetail]?
    let requestId: String
    let retryable: Bool
    
    var errorDescription: String? { message }
    
    var shouldRetry: Bool {
        retryable || [500, 502, 503, 504].contains(httpStatusCode)
    }
}

// Client usage
do {
    let trip = try await networkService.createTrip(request)
} catch let apiError as APIError {
    if apiError.code == "INVALID_PICKUP_LOCATION" {
        showLocationError(apiError.message)
    } else if apiError.shouldRetry {
        retryWithExponentialBackoff()
    } else {
        showGenericError()
    }
}
```

---

## Part 3: Pagination Strategies

### Cursor-Based Pagination (Recommended for Feeds)

**API Design:**
```
GET /trips?cursor=eyJpZCI6MTIzfQ&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "nextCursor": "eyJpZCI6MTQzfQ",
    "prevCursor": "eyJpZCI6MTAzfQ",
    "hasMore": true
  }
}
```

**Why Cursor-Based:**
- ✅ Consistent results (no duplicates if new items added)
- ✅ Performs well with large datasets
- ✅ Works with distributed databases
- ❌ Can't jump to arbitrary page
- ❌ More complex server implementation

**iOS Client:**
```swift
func fetchTrips(cursor: String?) async throws -> TripResponse {
    var components = URLComponents(string: "\(baseURL)/trips")!
    var queryItems: [URLQueryItem] = [
        URLQueryItem(name: "limit", value: "20")
    ]
    
    if let cursor = cursor {
        queryItems.append(URLQueryItem(name: "cursor", value: cursor))
    }
    
    components.queryItems = queryItems
    let (data, _) = try await session.data(from: components.url!)
    return try JSONDecoder().decode(TripResponse.self, from: data)
}
```

### Offset-Based Pagination (For Search Results)

**API Design:**
```
GET /trips?offset=40&limit=20

Response:
{
  "data": [...],
  "pagination": {
    "offset": 40,
    "limit": 20,
    "total": 150,
    "hasMore": true
  }
}
```

**Why Offset-Based:**
- ✅ Can jump to specific page
- ✅ Calculate total pages easily
- ❌ Duplicates possible if items added
- ❌ Poor performance for large offsets (DB does `SKIP 10000`)

### Keyset Pagination (High Performance)

**API Design:**
```
GET /trips?sinceId=123&limit=20

Server query:
SELECT * FROM trips WHERE id > 123 ORDER BY id LIMIT 20
```

**Why Keyset:**
- ✅ Best performance (uses index)
- ✅ No duplicates
- ❌ Requires sortable unique field
- ❌ Can't jump to arbitrary page

### Interview Question: "Which pagination should we use for Uber trip list?"

**Strong Answer:**
> "I'd use **cursor-based pagination** because:
> 1. Trip list is chronological (newest first) - fits cursor model
> 2. Users rarely jump to specific page (unlike search)
> 3. New trips added frequently - cursor prevents duplicates
> 4. At Uber's scale, offset-based would be slow (millions of trips)
> 
> **Trade-off:** Can't implement page numbers UI, but infinite scroll is better UX for mobile anyway.
> 
> **Alternative:** For search results with filters, I'd use offset-based so users can navigate pages."

---

## Part 4: API Versioning

### Versioning Strategies

| Strategy | Example | Pros | Cons |
|----------|---------|------|------|
| **URL Path** | `/v1/trips`, `/v2/trips` | Clear, easy to route | URL duplication |
| **Query Param** | `/trips?version=2` | Same URL | Easy to miss |
| **Header** | `Accept: application/vnd.uber.v2+json` | Clean URLs | Less visible |
| **Content Negotiation** | `Accept: application/json; version=2` | RESTful | Complex |

**Recommended for Mobile: URL Path**

```swift
class NetworkService {
    let baseURL = "https://api.uber.com/v1"  // Version in base URL
    
    // When v2 needed
    let baseURLv2 = "https://api.uber.com/v2"
}
```

**Breaking Changes That Require New Version:**
- Removing fields
- Changing field types (`string` → `int`)
- Changing response structure
- Changing authentication

**Non-Breaking Changes (Same Version):**
- Adding optional fields
- Adding new endpoints
- Deprecating (but keeping) fields

### Deprecation Strategy

```json
{
  "data": {
    "oldField": "value",  // Deprecated
    "newField": "value"
  },
  "meta": {
    "deprecations": [
      {
        "field": "oldField",
        "message": "Use newField instead",
        "sunsetDate": "2026-06-01"
      }
    ]
  }
}
```

**iOS Client Handling:**
```swift
struct TripResponse: Codable {
    @available(*, deprecated, message: "Use newField instead")
    let oldField: String?
    let newField: String?
    
    var fieldValue: String {
        newField ?? oldField ?? ""
    }
}
```

---

## Part 5: Client-Side Best Practices

### Request Management

**1. Deduplication (Prevent duplicate requests)**

```swift
actor RequestManager {
    private var activeRequests: [String: Task<Data, Error>] = [:]
    
    func fetch(url: URL) async throws -> Data {
        let key = url.absoluteString
        
        // Return existing request if in-flight
        if let existingTask = activeRequests[key] {
            return try await existingTask.value
        }
        
        // Create new request
        let task = Task {
            defer { activeRequests[key] = nil }
            let (data, _) = try await URLSession.shared.data(from: url)
            return data
        }
        
        activeRequests[key] = task
        return try await task.value
    }
}
```

**2. Request Cancellation**

```swift
class FeedViewModel {
    private var loadTask: Task<Void, Never>?
    
    func loadFeed() {
        // Cancel previous request
        loadTask?.cancel()
        
        loadTask = Task {
            do {
                let feed = try await repository.fetchFeed()
                guard !Task.isCancelled else { return }
                await MainActor.run {
                    self.posts = feed.posts
                }
            } catch {
                // Handle error
            }
        }
    }
}
```

**3. Retry with Exponential Backoff**

```swift
func fetchWithRetry<T>(
    maxRetries: Int = 3,
    operation: @escaping () async throws -> T
) async throws -> T {
    var lastError: Error?
    
    for attempt in 0..<maxRetries {
        do {
            return try await operation()
        } catch let error as URLError where error.code == .networkConnectionLost {
            lastError = error
            
            // Exponential backoff: 1s, 2s, 4s
            let delay = pow(2.0, Double(attempt))
            try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
        } catch {
            throw error  // Non-retryable error
        }
    }
    
    throw lastError ?? NetworkError.unknown
}

// Usage
let trips = try await fetchWithRetry {
    try await networkService.fetchTrips()
}
```

**4. Timeout Configuration**

```swift
let config = URLSessionConfiguration.default
config.timeoutIntervalForRequest = 10.0  // Individual request timeout
config.timeoutIntervalForResource = 60.0  // Total time including retries

let session = URLSession(configuration: config)
```

### Caching Headers

**Server sends:**
```
HTTP/1.1 200 OK
Cache-Control: public, max-age=3600
ETag: "abc123"
Last-Modified: Mon, 13 Jan 2026 10:00:00 GMT
```

**Client conditional request:**
```
GET /trips/123 HTTP/1.1
If-None-Match: "abc123"
If-Modified-Since: Mon, 13 Jan 2026 10:00:00 GMT
```

**Server response if unchanged:**
```
HTTP/1.1 304 Not Modified
```

**iOS Implementation:**
```swift
var request = URLRequest(url: url)
request.cachePolicy = .returnCacheDataElseLoad

// URLCache automatically handles ETag/Last-Modified
let (data, response) = try await session.data(for: request)

if let httpResponse = response as? HTTPURLResponse,
   httpResponse.statusCode == 304 {
    // Use cached data
}
```

---

## Part 6: Rate Limiting & Throttling

### Server-Side Rate Limiting

**Response Headers:**
```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1673640000
Retry-After: 60

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Try again in 60 seconds",
    "retryAfter": 60
  }
}
```

**iOS Client Handling:**
```swift
func handleRateLimitResponse(_ response: HTTPURLResponse) async throws {
    if response.statusCode == 429 {
        let retryAfter = response.value(forHTTPHeaderField: "Retry-After")
        let delay = TimeInterval(retryAfter ?? "60") ?? 60
        
        // Wait and retry
        try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
        
        // Retry original request
        return try await retryRequest()
    }
}
```

### Client-Side Throttling

**Debounce (only call after user stops typing):**
```swift
class SearchViewModel {
    private var searchTask: Task<Void, Never>?
    
    func search(query: String) {
        searchTask?.cancel()
        
        searchTask = Task {
            // Wait 300ms
            try? await Task.sleep(nanoseconds: 300_000_000)
            
            guard !Task.isCancelled else { return }
            
            // Make API call
            let results = try? await api.search(query)
            // ...
        }
    }
}
```

**Throttle (limit to 1 call per second):**
```swift
actor ThrottledAPI {
    private var lastCallTime: Date?
    private let minInterval: TimeInterval = 1.0
    
    func call<T>(operation: () async throws -> T) async throws -> T {
        if let lastCall = lastCallTime {
            let elapsed = Date().timeIntervalSince(lastCall)
            if elapsed < minInterval {
                try await Task.sleep(nanoseconds: UInt64((minInterval - elapsed) * 1_000_000_000))
            }
        }
        
        lastCallTime = Date()
        return try await operation()
    }
}
```

---

## Part 7: Authentication & Security

### Token-Based Authentication

**1. Access Token + Refresh Token Pattern**

```swift
struct AuthTokens: Codable {
    let accessToken: String
    let refreshToken: String
    let expiresIn: Int      // seconds
    let tokenType: String   // "Bearer"
}

class AuthManager {
    private var tokens: AuthTokens?
    private let keychain = KeychainService()
    
    func authenticate(email: String, password: String) async throws {
        let request = LoginRequest(email: email, password: password)
        let response = try await api.login(request)
        
        tokens = response.tokens
        keychain.save(tokens)  // Secure storage
    }
    
    func getValidAccessToken() async throws -> String {
        guard let tokens = tokens else {
            throw AuthError.notAuthenticated
        }
        
        // Check if expired
        if isTokenExpired(tokens.accessToken) {
            // Refresh token
            let newTokens = try await refreshAccessToken(tokens.refreshToken)
            self.tokens = newTokens
            keychain.save(newTokens)
            return newTokens.accessToken
        }
        
        return tokens.accessToken
    }
    
    private func refreshAccessToken(_ refreshToken: String) async throws -> AuthTokens {
        // POST /auth/refresh with refresh token
        // ...
    }
}
```

**2. Automatic Token Injection**

```swift
class AuthenticatedNetworkService {
    private let authManager: AuthManager
    
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        var urlRequest = endpoint.urlRequest
        
        // Inject token
        let token = try await authManager.getValidAccessToken()
        urlRequest.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        
        let (data, response) = try await session.data(for: urlRequest)
        
        // Handle 401 (token expired/invalid)
        if let httpResponse = response as? HTTPURLResponse,
           httpResponse.statusCode == 401 {
            // Token refresh failed, logout user
            await authManager.logout()
            throw AuthError.sessionExpired
        }
        
        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

### Security Best Practices

**1. Certificate Pinning**
```swift
class SecureNetworkService: NSObject, URLSessionDelegate {
    func urlSession(
        _ session: URLSession,
        didReceive challenge: URLAuthenticationChallenge,
        completionHandler: @escaping (URLSession.AuthChallengeDisposition, URLCredential?) -> Void
    ) {
        guard let serverTrust = challenge.protectionSpace.serverTrust else {
            completionHandler(.cancelAuthenticationChallenge, nil)
            return
        }
        
        // Pin certificate
        let certificate = SecCertificateCreateWithData(nil, pinnedCertData as CFData)!
        if SecTrustEvaluateWithError(serverTrust, nil) {
            completionHandler(.useCredential, URLCredential(trust: serverTrust))
        } else {
            completionHandler(.cancelAuthenticationChallenge, nil)
        }
    }
}
```

**2. Request Signing**
```swift
func signRequest(_ request: URLRequest) -> URLRequest {
    var signed = request
    let timestamp = "\(Int(Date().timeIntervalSince1970))"
    let nonce = UUID().uuidString
    
    // Create signature: HMAC-SHA256(method + path + timestamp + nonce + body)
    let signatureBody = "\(request.httpMethod ?? "")\(request.url?.path ?? "")\(timestamp)\(nonce)"
    let signature = HMACSHA256(signatureBody, secret: apiSecret)
    
    signed.setValue(timestamp, forHTTPHeaderField: "X-Timestamp")
    signed.setValue(nonce, forHTTPHeaderField: "X-Nonce")
    signed.setValue(signature, forHTTPHeaderField: "X-Signature")
    
    return signed
}
```

---

## Part 8: REST vs GraphQL vs gRPC

### Comparison Table

| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| **Data Fetching** | Multiple endpoints | Single endpoint, flexible query | Strongly typed |
| **Over-fetching** | Common (returns all fields) | None (request exact fields) | Minimal |
| **Under-fetching** | Requires multiple requests | Single request | N/A |
| **Caching** | HTTP caching | Complex | Custom |
| **Mobile Size** | Moderate | Large (runtime) | Small (compiled) |
| **Learning Curve** | Low | Medium | High |
| **Tooling** | Excellent | Good (Apollo) | Good (protobuf) |
| **Best For** | CRUD, simple apps | Complex queries, flexibility | Low latency, streaming |

### When to Use Each (Interview Answer)

**REST:**
> "I'd use REST for Uber's trip list because:
> - Simple CRUD operations
> - Standard HTTP caching works well
> - Lightweight client code
> - Industry-standard, easy onboarding"

**GraphQL:**
> "GraphQL makes sense for a dashboard with many data sources:
> - Flexible queries (trip + driver + payment in one request)
> - Prevents over-fetching (mobile bandwidth savings)
> - Trade-off: Larger client bundle, no HTTP caching"

**gRPC:**
> "gRPC is ideal for real-time location tracking:
> - Bidirectional streaming (server pushes location updates)
> - Compact binary format (saves bandwidth)
> - Low latency
> - Trade-off: Harder to debug, not browser-friendly"

---

## Part 9: Mobile-Specific Optimizations

### Bandwidth Optimization

**1. Response Compression**
```swift
let config = URLSessionConfiguration.default
config.httpAdditionalHeaders = [
    "Accept-Encoding": "gzip, deflate, br"  // Request compression
]
```

**2. Minimal Payloads**
```
// BAD: Return entire trip object
{
  "id": "123",
  "driver": { /* 50 fields */ },
  "vehicle": { /* 30 fields */ },
  "route": { /* large polyline */ }
}

// GOOD: Return only what's needed for list
{
  "id": "123",
  "driverName": "John",
  "vehicleType": "uberX",
  "pickupAddress": "123 Main St",
  "fare": 25.50,
  "thumbnailUrl": "..."
}
```

**3. Field Selection (GraphQL-style)**
```
GET /trips?fields=id,driverName,fare,status

Response:
{
  "data": [
    {"id": "123", "driverName": "John", "fare": 25.50, "status": "completed"}
  ]
}
```

### Battery Optimization

**1. Batch Requests**
```swift
// BAD: 3 separate requests
let trips = await fetchTrips()
let driver = await fetchDriver(trips.first.driverId)
let payment = await fetchPayment(trips.first.id)

// GOOD: Single batch request
POST /batch
[
  {"method": "GET", "path": "/trips"},
  {"method": "GET", "path": "/drivers/123"},
  {"method": "GET", "path": "/payments/123"}
]
```

**2. Conditional Requests (ETags)**
```swift
// Only download if changed
request.setValue(lastETag, forHTTPHeaderField: "If-None-Match")

if response.statusCode == 304 {
    // Use cache, no download
}
```

**3. Background Fetch Wisely**
```swift
// Don't: Poll every minute
Timer.scheduledTimer(withTimeInterval: 60, repeats: true) { ... }

// Do: Use push notifications to trigger fetch
func application(_ app: UIApplication, didReceiveRemoteNotification: ...) {
    Task {
        await refreshData()
    }
}
```

---

## Part 10: Interview Questions & Answers

### Q1: "Design the API for Uber's trip booking flow"

**Answer:**
```
POST /rides/estimate
Request: { pickupLocation, dropoffLocation, rideType }
Response: { estimatedFare, estimatedTime, driverCount }

POST /rides/request
Request: { pickupLocation, dropoffLocation, rideType, paymentMethod }
Response: { rideId, status: "searching", estimatedWait }

GET /rides/{rideId}
Response: { status, driver: {...}, eta, location }

PATCH /rides/{rideId}/cancel
Response: { status: "cancelled", cancellationFee }

Webhook (server → client):
POST {webhookUrl}
{ event: "driver_assigned", rideId, driver: {...} }
```

**Key Points:**
- Estimation before committing (saves server resources)
- Polling GET /rides/{id} for status updates (or WebSocket)
- PATCH for cancel (not DELETE, ride record remains)
- Webhook for real-time updates (battery-efficient)

### Q2: "How would you handle API versioning for a breaking change?"

**Answer:**
> "For Uber's payment API change (adding required `cardToken` field):
>
> **Plan:**
> 1. Deploy `/v2/payments` with new field required
> 2. Keep `/v1/payments` for 6 months (deprecation period)
> 3. Update iOS app to use v2
> 4. Server logs v1 usage, send push notifications to update
> 5. After 6 months, return 410 Gone from v1
>
> **Client handling:**
> ```swift
> let apiVersion = appVersion >= "5.0" ? "v2" : "v1"
> let baseURL = "https://api.uber.com/\(apiVersion)"
> ```
>
> **Trade-off:** Maintaining two versions costs server resources, but prevents breaking existing apps."

### Q3: "How would you optimize API calls for a poor network (2G)?"

**Answer:**
> "Multiple strategies:
> 1. **Request prioritization** - Critical first (ride status), defer analytics
> 2. **Smaller payloads** - Field selection, compression
> 3. **Longer timeouts** - 30s instead of 10s
> 4. **Adaptive quality** - Low-res images on slow network
> 5. **Batch requests** - Combine multiple into one
> 6. **Prefetch critical data** - When on WiFi, cache aggressively
> 7. **Offline-first** - Show cached trip list, sync when connected
>
> **Implementation:**
> ```swift
> if NetworkMonitor.is2G {
>     request.cachePolicy = .returnCacheDataElseLoad
>     request.timeoutInterval = 30
>     imageQuality = .low
> }
> ```"

---

## Part 11: API Design Interview Checklist

When designing an API in an interview, cover:

✅ **HTTP Method** - Correct semantic usage (GET, POST, PUT, PATCH, DELETE)
✅ **URL Structure** - RESTful, hierarchical, plural nouns
✅ **Request/Response Format** - JSON structure, field naming (camelCase)
✅ **Status Codes** - Appropriate codes (201 vs 200, 404 vs 422)
✅ **Error Handling** - Structured error responses with `code`, `message`, `details`
✅ **Pagination** - Cursor/offset/keyset, explain choice
✅ **Versioning** - How to handle breaking changes
✅ **Authentication** - Token injection, refresh flow
✅ **Rate Limiting** - Client handling of 429
✅ **Caching** - ETag, Cache-Control headers
✅ **Mobile Optimization** - Compression, minimal payloads, batching
✅ **Trade-offs** - Why this approach over alternatives

---

## Quick Reference

### Common Headers

```
// Request
Authorization: Bearer {token}
Content-Type: application/json
Accept: application/json
Accept-Encoding: gzip
If-None-Match: "etag123"
X-Request-ID: uuid

// Response
Cache-Control: public, max-age=3600
ETag: "etag123"
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-Request-ID: uuid
```

### Swift URLSession Template

```swift
func apiCall<T: Decodable>(endpoint: String, method: String = "GET", body: Encodable? = nil) async throws -> T {
    var request = URLRequest(url: URL(string: "\(baseURL)\(endpoint)")!)
    request.httpMethod = method
    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    
    if let body = body {
        request.httpBody = try JSONEncoder().encode(body)
    }
    
    let (data, response) = try await session.data(for: request)
    
    guard let httpResponse = response as? HTTPURLResponse else {
        throw NetworkError.invalidResponse
    }
    
    switch httpResponse.statusCode {
    case 200...299:
        return try JSONDecoder().decode(T.self, from: data)
    case 401:
        throw NetworkError.unauthorized
    case 429:
        throw NetworkError.rateLimited
    default:
        let error = try? JSONDecoder().decode(APIError.self, from: data)
        throw error ?? NetworkError.unknown
    }
}
```

---

> [!TIP]
> In interviews, always mention mobile-specific considerations: bandwidth, battery, intermittent connectivity!
