### 2.7 Thread Safety Strategy

**NetworkManager (Actor)**:
- ✅ Actor ensures all requests are serialized
- ✅ `activeTasks` dictionary is protected from races
- ✅ Thread-safe task cancellation

**AuthenticationManager (Actor)**:
- ✅ Token refresh is atomic (only one refresh at a time)
- ✅ Multiple concurrent requests don't cause duplicate refreshes
- ✅ If refresh in progress, subsequent requests await it

**RetryHandler (Actor)**:
- ✅ Retry state is isolated per request
- ✅ No shared mutable state

**URLSession**:
- ✅ Thread-safe by design
- ✅ Can be called from any thread

**Common Pitfalls**:
| Pitfall | Solution |
|---------|----------|
| Multiple token refreshes | Use actor with refresh task tracking |
| Race on retry counter | Make retry handler actor-isolated |
| Cancellation not propagated | Store tasks and cancel explicitly |
| Background thread UI updates | Use `@MainActor` for completion handlers |

###  2.8 Testing Strategy

#### Unit Tests

```swift
import XCTest
@testable import App

final class NetworkManagerTests: XCTestCase {
    var networkManager: NetworkManager!
    var mockSession: MockURLSession!
    var mockAuthManager: AuthenticationManager!
    
    override func setUpWithError() throws {
        mockSession = MockURLSession()
        mockAuthManager = AuthenticationManager { "test_token" }
        
        networkManager = NetworkManager(
            baseURL: URL(string: "https://api.test.com")!,
            session: mockSession,
            authManager: mockAuthManager
        )
    }
    
    func testSuccessfulRequest() async throws {
        // Arrange
        let expectedUser = User(id: "1", name: "Test", email: "test@example.com")
        mockSession.mockResponse = (
            try! JSONEncoder().encode(expectedUser),
            HTTPURLResponse(
                url: URL(string: "https://api.test.com")!,
                statusCode: 200,
                httpVersion: nil,
                headerFields: nil
            )!
        )
        
        // Act
        let request = GetUserRequest(userID: "1")
        let user = try await networkManager.request(request)
        
        // Assert
        XCTAssertEqual(user.id, expectedUser.id)
        XCTAssertEqual(user.name, expectedUser.name)
    }
    
    func test404Error() async throws {
        // Arrange
        mockSession.mockResponse = (
            Data(),
            HTTPURLResponse(
                url: URL(string: "https://api.test.com")!,
                statusCode: 404,
                httpVersion: nil,
                headerFields: nil
            )!
        )
        
        // Act & Assert
        do {
            let request = GetUserRequest(userID: "1")
            _ = try await networkManager.request(request)
            XCTFail("Should throw error")
        } catch let error as NetworkError {
            if case .httpError(let code, _) = error {
                XCTAssertEqual(code, 404)
            } else {
                XCTFail("Wrong error type")
            }
        }
    }
}

// Mock URLSession
class MockURLSession: URLSessionProtocol {
    var mockResponse: (Data, URLResponse)?
    var mockError: Error?
    
    func data(for request: URLRequest) async throws -> (Data, URLResponse) {
        if let error = mockError {
            throw error
        }
        
        guard let response = mockResponse else {
            fatalError("Mock response not set")
        }
        
        return response
    }
}
```

#### Retry Logic Tests

```swift
func testRetryWithBackoff() async throws {
    var attemptCount = 0
    
    let mockSession = MockURLSession()
    mockSession.mockError = URLError(.timedOut)
    
    let networkManager = NetworkManager(
        baseURL: URL(string: "https://api.test.com")!,
        session: mockSession,
        authManager: mockAuthManager,
        retryConfiguration: RetryConfiguration(
            maxAttempts: 3,
            baseDelay: 0.1,  // Fast for testing
            maxDelay: 1.0,
            jitter: 0.0,  // No jitter for predictable tests
            retryableHTTPCodes: [408, 500]
        )
    )
    
    mockSession.dataHandler = { _ in
        attemptCount += 1
        if attemptCount < 3 {
            throw URLError(.timedOut)
        } else {
            let user = User(id: "1", name: "Success", email: "test@test.com")
            let data = try! JSONEncoder().encode(user)
            let response = HTTPURLResponse(
                url: URL(string: "https://api.test.com")!,
                statusCode: 200,
                httpVersion: nil,
                headerFields: nil
            )!
            return (data, response)
        }
    }
    
    let startTime = Date()
    let request = GetUserRequest(userID: "1")
    let user = try await networkManager.request(request)
    let duration = Date().timeIntervalSince(startTime)
    
    XCTAssertEqual(attemptCount, 3)
    XCTAssertEqual(user.name, "Success")
    XCTAssertGreaterThan(duration, 0.3)  // At least 0.1 + 0.2 delay
}
```

#### Auth Manager Tests

```swift
func testTokenRefreshOnUnauthorized() async throws {
    var refreshCallCount = 0
    
    let authManager = AuthenticationManager {
        refreshCallCount += 1
        return "new_token_\(refreshCallCount)"
    }
    
    // First call should trigger token fetch
    let token1 = try await authManager.getToken()
    XCTAssertEqual(refreshCallCount, 1)
    XCTAssertEqual(token1, "new_token_1")
    
    // Subsequent call should use cached token
    let token2 = try await authManager.getToken()
    XCTAssertEqual(refreshCallCount, 1)  // No new call
    XCTAssertEqual(token2, "new_token_1")
    
    // Clear token and fetch again
    await authManager.clearToken()
    let token3 = try await authManager.getToken()
    XCTAssertEqual(refreshCallCount, 2)
    XCTAssertEqual(token3, "new_token_2")
}

func testConcurrentTokenRefresh() async throws {
    var refreshCallCount = 0
    
    let authManager = AuthenticationManager {
        refreshCallCount += 1
        try await Task.sleep(nanoseconds: 100_000_000)  // 100ms
        return "token_\(refreshCallCount)"
    }
    
    // 10 concurrent requests should only trigger 1 refresh
    let tokens = await withTaskGroup(of: String.self) { group in
        for _ in 0..<10 {
            group.addTask {
                try! await authManager.getToken()
            }
        }
        
        var results: [String] = []
        for await token in group {
            results.append(token)
        }
        return results
    }
    
    XCTAssertEqual(refreshCallCount, 1, "Should only refresh once")
    XCTAssertEqual(Set(tokens).count, 1, "All tokens should be identical")
}
```

### 2.9 Failure Handling

**Network Timeout**:
```swift
// Set request timeout
var urlRequest = URLRequest(url: url)
urlRequest.timeoutInterval = 30

// Catch and handle
catch let error as URLError where error.code == .timedOut {
    // Retry or show user-friendly message
    throw NetworkError.timeout
}
```

**No Network Connection**:
```swift
import Network

class NetworkMonitor {
    let monitor = NWPathMonitor()
    private(set) var isConnected = false
    
    func start() {
        monitor.pathUpdateHandler = { [weak self] path in
            self?.isConnected = (path.status == .satisfied)
        }
        monitor.start(queue: DispatchQueue.global())
    }
}

// In NetworkManager
if !networkMonitor.isConnected {
    throw NetworkError.noNetwork
}
```

**Partial JSON Decoding**:
```swift
// Use optional properties for non-critical fields
struct User: Codable {
    let id: String  // Required
    let name: String  // Required
    let bio: String?  // Optional - won't fail if missing
    let avatarURL: URL?  // Optional
}

// Or use custom decoding
struct UserResponse: Codable {
    let id: String
    let name: String
    let metadata: [String: AnyCodable]?  // Catch-all for unknown fields
    
    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: CodingKeys.self)
        id = try container.decode(String.self, forKey: .id)
        name = try container.decode(String.self, forKey: .name)
        metadata = try? container.decode([String: AnyCodable].self, forKey: .metadata)
    }
}
```

**Request Cancellation**:
```swift
// Store task ID for cancellation
class FeedViewModel {
    private var loadTaskID: UUID?
    
    func loadFeed() async {
        let taskID = UUID()
        loadTaskID = taskID
        
        do {
            let posts = try await networkManager.request(GetFeedRequest())
            // Update UI
        } catch is CancellationError {
            // Ignore cancellation
        }
    }
    
    func cancelLoad() async {
        if let taskID = loadTaskID {
            await networkManager.cancel(taskID: taskID)
        }
    }
}
```

### 2.10 Instrumentation

#### Metrics

```swift
struct NetworkMetrics {
    var totalRequests = 0
    var successfulRequests = 0
    var failedRequests = 0
    var retryCount = 0
    var totalDuration: TimeInterval = 0
    
    var successRate: Double {
        guard totalRequests > 0 else { return 0 }
        return Double(successfulRequests) / Double(totalRequests)
    }
    
    var averageDuration: TimeInterval {
        guard totalRequests > 0 else { return 0 }
        return totalDuration / Double(totalRequests)
    }
}

actor MetricsCollector {
    private var metrics = NetworkMetrics()
    
    func recordRequest(success: Bool, duration: TimeInterval, didRetry: Bool) {
        metrics.totalRequests += 1
        metrics.totalDuration += duration
        
        if success {
            metrics.successfulRequests += 1
        } else {
            metrics.failedRequests += 1
        }
        
        if didRetry {
            metrics.retryCount += 1
        }
    }
    
    func getMetrics() -> NetworkMetrics {
        return metrics
    }
    
    func reset() {
        metrics = NetworkMetrics()
    }
}
```

#### Distributed Tracing

```swift
import OSLog

extension NetworkManager {
    private func executeRequestWith Tracing<T: NetworkRequest>(
        _ request: T,
        requiresAuth: Bool
    ) async throws -> T.Response {
        // Create trace span
        let signposter = OSSignposter(subsystem: "com.app.network", category: "Request")
        let signpostID = signposter.makeSignpostID()
        let state = signposter.beginInterval("network_request", id: signpostID)
        
        defer {
            signposter.endInterval("network_request", state)
        }
        
        // Execute request (existing code)
        return try await executeRequest(request, requiresAuth: requiresAuth)
    }
}

// View in Instruments > os_signpost
// Shows request duration, nesting, etc.
```

### 2.11 Interview Q&A

**Q1: Why use exponential backoff with jitter for retries?**

**Answer**:
**Exponential Backoff**: Increasing delay between retries prevents overwhelming the server.
- Attempt 1: Wait 1s
- Attempt 2: Wait 2s  
- Attempt 3: Wait 4s

**Why not fixed delay** (e.g., always 1s)?
- Server is likely still overloaded after 1s
- Wastes battery/bandwidth with rapid retries

**Jitter** (random component): Prevents thundering herd problem.

**Problem**: If 10,000 clients all fail at same time and retry after exact 1s, they'll hit server simultaneously again.

**Solution**: Add randomness
```swift
delay = 2^attempt + random(0, 0.5)
// Spreads retries across 1.0-1.5s, 2.0-2.5s, 4.0-4.5s
```

**Real scenario**: Instagram Stories server outage
- 1M users try to load stories simultaneously
- Server returns 503
- Without jitter: All retry after 1s → Same overload
- With jitter: Retries spread over 500ms window → Gradual recovery

**Trade-off**: Complexity vs fairness and system stability.

---

**Q2: How do you handle token refresh without duplicating requests?**

**Answer**:
**Problem**: Multiple requests hitting 401 simultaneously would each try to refresh token → N refresh calls for 1 expired token.

**Solution**: Actor-based token management with refresh task tracking

```swift
actor AuthenticationManager {
    private var refreshTask: Task<String, Error>?
    
    func getToken() async throws -> String {
        // If refresh in progress, await it
        if let task = refreshTask {
            return try await task.value
        }
        
        // Start new refresh
        let task = Task {
            try await callRefreshAPI()
        }
        refreshTask = task
        
        defer { refreshTask = nil }
        
        return try await task.value
    }
}
```

**Flow**:
1. Request A gets 401 → calls `getToken()` → starts refresh task
2. Request B gets 401 → calls `getToken()` → awaits existing refresh task
3. Request C gets 401 → calls `getToken()` → awaits existing refresh task
4. Refresh completes → all three requests get new token

**Benefits**:
- Only 1 network call for token refresh
- All pending requests wait for same result
- Thread-safe (actor isolation)

**Production consideration**: Queue failed requests and replay with new token (don't fail them).

---

**Q3: Why make NetworkManager an actor instead of using DispatchQueue?**

**Answer**:
**Actors (modern approach)**:
```swift
actor NetworkManager {
    private var activeTasks: [UUID: Task<Any, Error>] = [:]
    
    func cancel(taskID: UUID) {
        activeTasks[taskID]?.cancel()
    }
}
```

**DispatchQueue (legacy approach)**:
```swift
class NetworkManager {
    private var activeTasks: [UUID: Task<Any, Error>] = [:]
    private let queue = DispatchQueue(label: "network")
    
    func cancel(taskID: UUID) {
        queue.async {
            self.activeTasks[taskID]?.cancel()
        }
    }
}
```

**Actor advantages**:
1. **Compile-time safety**: Compiler prevents data races
2. **Async/await native**: Works seamlessly with Swift Concurrency
3. **Less boilerplate**: No manual queue management
4. **Reentrancy protection**: Prevents subtle bugs

**Example of actor preventing bug**:
```swift
// With actor: Won't compile (error: actor-isolated property)
let manager = NetworkManager()
let tasks = manager.activeTasks  // ERROR

// With DispatchQueue: Compiles but has data race
let manager = NetworkManager()
let tasks = manager.activeTasks  // DATA RACE if queue hasn't executed yet
```

**When to use DispatchQueue**:
- Legacy codebases (pre-Swift 5.5)
- Need precise queue QoS control
- Interop with Objective-C

**Modern best practice**: Use actors for new Swift code.

---

**Q4: How would you test network code that uses async/await?**

**Answer**:
**Strategy**: Protocol-based dependency injection + mock implementations

**1. Define protocol**:
```swift
protocol URLSessionProtocol {
    func data(for request: URLRequest) async throws -> (Data, URLResponse)
}

extension URLSession: URLSessionProtocol { ... }
```

**2. Inject dependency**:
```swift
actor NetworkManager {
    private let session: URLSessionProtocol  // Not URLSession directly
    
    init(session: URLSessionProtocol = URLSession.shared) {
        self.session = session
    }
}
```

**3. Mock in tests**:
```swift
class MockURLSession: URLSessionProtocol {
    var mockResponses: [URLRequest: (Data, URLResponse)] = [:]
    
    func data(for request: URLRequest) async throws -> (Data, URLResponse) {
        guard let response = mockResponses[request] else {
            throw URLError(.badServerResponse)
        }
        return response
    }
}
```

**4. Write test**:
```swift
func testGetUser() async throws {
    let mock = MockURLSession()
    mock.mockResponses[userRequest] = (userData, successResponse)
    
    let manager = NetworkManager(session: mock)
    let user = try await manager.request(GetUserRequest(userID: "1"))
    
    XCTAssertEqual(user.name, "Test User")
}
```

**Benefits**:
- No network calls in tests (fast, reliable)
- Full control over responses (test error cases)
- Can test timing (delays, timeouts)

**Alternative**: Use URLProtocol subclass to intercept URLSession requests (more complex but no protocol needed).

---

**Q5: How do you handle pagination in a feed with this architecture?**

**Answer**:
**Pattern**: Cursor-based or offset-based pagination

**1. Define paginated request**:
```swift
struct GetFeedRequest: NetworkRequest {
    typealias Response = FeedPage
    
    let cursor: String?  // nil for first page
    let limit: Int
    
    var method: HTTPMethod { .get }
    var path: String { "/feed" }
    var queryParameters: [String: String]? {
        var params = ["limit": "\(limit)"]
        if let cursor = cursor {
            params["cursor"] = cursor
        }
        return params
    }
}

struct FeedPage: Codable {
    let posts: [Post]
    let nextCursor: String?  // nil if last page
    let hasMore: Bool
}
```

**2. ViewModel with pagination state**:
```swift
@MainActor
class FeedViewModel: ObservableObject {
    @Published var posts: [Post] = []
    @Published var isLoading = false
    @Published var hasMore = true
    
    private var nextCursor: String?
    private let networkManager: NetworkManager
    
    func loadNextPage() async {
        guard !isLoading, hasMore else { return }
        
        isLoading = true
        defer { isLoading = false }
        
        do {
            let request = GetFeedRequest(cursor: nextCursor, limit: 20)
            let page = try await networkManager.request(request)
            
            posts.append(contentsOf: page.posts)
            nextCursor = page.nextCursor
            hasMore = page.hasMore
        } catch {
            // Handle error
        }
    }
    
    func refresh() async {
        posts = []
        nextCursor = nil
        hasMore = true
        await loadNextPage()
    }
}
```

**3. SwiftUI view with infinite scroll**:
```swift
struct FeedView: View {
    @StateObject var viewModel = FeedViewModel()
    
    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(viewModel.posts) { post in
                    PostRow(post: post)
                        .onAppear {
                            if post == viewModel.posts.last {
                                Task {
                                    await viewModel.loadNextPage()
                                }
                            }
                        }
                }
                
                if viewModel.hasMore {
                    ProgressView()
                        .onAppear {
                            Task {
                                await viewModel.loadNextPage()
                            }
                        }
                }
            }
        }
        .refreshable {
            await viewModel.refresh()
        }
    }
}
```

**Cursor vs Offset**:
- **Offset**: `?offset=40&limit=20` → Can skip items if new inserted
- **Cursor**: `?cursor=post_id_40&limit=20` → Stable even with inserts

**Best practice**: Use cursor-based for real-time feeds (Twitter, Instagram).

---

**Q6: How would you implement request prioritization?**

**Answer**:
**Use case**: User taps into a post (high priority) while feed images load in background (low priority).

**Implementation**:

**1. Add priority to requests**:
```swift
protocol NetworkRequest {
    var priority: TaskPriority { get }
}

extension NetworkRequest {
    var priority: TaskPriority { .medium }  // Default
}

struct GetPostDetailsRequest: NetworkRequest {
    var priority: TaskPriority { .userInitiated }  // High priority
}

struct PrefetchImageRequest: NetworkRequest {
    var priority: TaskPriority { .utility }  // Low priority
}
```

**2. Create Task with priority**:
```swift
func request<T: NetworkRequest>(_ request: T) async throws -> T.Response {
    let task = Task(priority: request.priority) {
        try await self.executeRequest(request)
    }
    
    return try await task.value
}
```

**3. URLSession QoS mapping**:
```swift
let configuration = URLSessionConfiguration.default
switch request.priority {
case .high, .userInitiated:
    configuration.networkServiceType = .responsiveData
case .utility, .background:
    configuration.networkServiceType = .background
default:
    configuration.networkServiceType = .default
}
```

**iOS Scheduling**:
- High priority tasks get CPU/network first
- Low priority tasks can be delayed/throttled
- System manages based on battery, thermal state

**Real scenario**: Instagram
- Tap on post → Load comments (high priority)
- Scroll feed → Prefetch off-screen images (low priority)
- If device hot/low battery → Pause low priority, keep high priority

---

This completes Component 2: Network Request Manager with production-grade implementation, retry logic, auth handling, and comprehensive interview preparation.

---

