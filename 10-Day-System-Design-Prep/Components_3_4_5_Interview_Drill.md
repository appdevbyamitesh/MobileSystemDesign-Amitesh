## 3. Thread-Safe Data Store

### 3.1 Concept Explanation

**Simple Analogy**: Think of a thread-safe data store like a library:
- **Readers** = Multiple people can read different books simultaneously
- **Writers** = Only one person can update the catalog at a time
- **Librarian (Actor)** = Coordinates access to prevent chaos

**Problem It Solves**: iOS apps have multiple features accessing shared state:
- User session data (auth token, user ID, preferences)
- Feature flags (A/B test variants)
- Cached feed state (scroll position, loaded posts)
- Without thread safety → Data races, crashes, corrupted state

**Real App Scenario**: Uber storing rider location, Spotify caching playlists, Banking app with session state

### 3.2 Requirements & Constraints

**Functional Requirements**:
- Store key-value pairs
- Support multiple concurrent readers
- Serialize writes to prevent races
- Snapshot reads (consistent view of data)
- Type-safe API
- Atomic updates

**Non-Functional Requirements**:
| Constraint | Target | Reasoning |
|------------|--------|-----------|
| Read latency | <1ms | High-frequency access |
| Write latency | <5ms | Less frequent, can be slower |
| Memory | Minimal overhead | In-memory only |
| Thread safety | 100% | Zero data races |

### 3.3 Architecture

```
┌──────────────────────────────────────┐
│        DataStore (Actor)             │
│  • Serial execution                  │
│  • Dictionary<Key, Value>            │
│  • Snapshot reads                    │
│  • Atomic updates                    │
└──────────────────────────────────────┘
          │
          ├─ get(key) → Value?
          ├─ set(key, value)
          ├─ delete(key)
          ├─ snapshot() → [Key: Value]
          └─ update(key, transform)
```

### 3.4 Swift Implementation

#### 3.4.1 Actor-Based Data Store

```swift
actor DataStore<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    
    // Read operations
    func get(_ key: Key) -> Value? {
        return storage[key]
    }
    
    func getAll() -> [Key: Value] {
        return storage  // Returns copy
    }
    
    func contains(_ key: Key) -> Bool {
        return storage[key] != nil
    }
    
    // Write operations
    func set(_ key: Key, value: Value) {
        storage[key] = value
    }
    
    func setAll(_ values: [Key: Value]) {
        storage = values
    }
    
    func delete(_ key: Key) {
        storage.removeValue(forKey: key)
    }
    
    func deleteAll() {
        storage.removeAll()
    }
    
    // Atomic update
    func update(_ key: Key, transform: (Value?) -> Value?) {
        let currentValue = storage[key]
        let newValue = transform(currentValue)
        
        if let newValue = newValue {
            storage[key] = newValue
        } else {
            storage.removeValue(forKey: key)
        }
    }
}

// Usage
let sessionStore = DataStore<String, Any>()

Task {
    await sessionStore.set("userID", value: "12345")
    await sessionStore.set("isLoggedIn", value: true)
    
    if let userID = await sessionStore.get("userID") as? String {
        print("User ID: \(userID)")
    }
}
```

#### 3.4.2 Serial Queue Alternative (Pre-Concurrency)

```swift
class ThreadSafeDataStore<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    private let queue = DispatchQueue(label: "com.app.datastore", attributes: .concurrent)
    
    // Read (concurrent)
    func get(_ key: Key) -> Value? {
        return queue.sync {
            storage[key]
        }
    }
    
    // Write (exclusive)
    func set(_ key: Key, value: Value) {
        queue.async(flags: .barrier) {
            self.storage[key] = value
        }
    }
    
    // Atomic update
    func update(_ key: Key, transform: @escaping (Value?) -> Value?) {
        queue.async(flags: .barrier) {
            let currentValue = self.storage[key]
            let newValue = transform(currentValue)
            
            if let newValue = newValue {
                self.storage[key] = newValue
            } else {
                self.storage.removeValue(forKey: key)
            }
        }
    }
    
    // Snapshot (consistent view)
    func snapshot() -> [Key: Value] {
        return queue.sync {
            storage
        }
    }
}
```

**Actor vs Queue Comparison**:
| Aspect | Actor | Serial Queue |
|---------|-------|--------------|
| Syntax | `await store.get("key")` | `store.get("key")` |
| Concurrency | Integrated with async/await | Manual async/sync |
| Compiler support | Yes (isolation checking) | No |
| Performance | ~Same | ~Same |
| Modern iOS | ✅ Preferred | Legacy |

#### 3.4.3 Type-Safe Session Store Example

```swift
struct UserSession: Codable {
    var userID: String?
    var accessToken: String?
    var refreshToken: String?
    var isLoggedIn: Bool = false
    var preferences: [String: String] = [:]
}

actor SessionStore {
    private var session = UserSession()
    
    func login(userID: String, accessToken: String, refreshToken: String) {
        session.userID = userID
        session.accessToken = accessToken
        session.refreshToken = refreshToken
        session.isLoggedIn = true
    }
    
    func logout() {
        session = UserSession()
    }
    
    func getAccessToken() -> String? {
        return session.accessToken
    }
    
    func updateToken(_ newToken: String) {
        session.accessToken = newToken
    }
    
    func isLoggedIn() -> Bool {
        return session.isLoggedIn
    }
    
    func snapshot() -> UserSession {
        return session
    }
}
```

### 3.5 Thread Safety Strategy

**Actor Guarantees**:
- ✅ All methods execute serially (one at a time)
- ✅ No data races possible
- ✅ Compiler enforces `await` for async calls
- ✅ Sendable checking for values crossing isolation

**Common Pitfalls**:

| Pitfall | Code | Solution |
|---------|------|----------|
| Holding reference | `let val = await store.get("key"); use(val)` | OK - value is copied |
| Multiple reads not atomic | `let a = await store.get("a"); let b = await store.get("b")` | Use `snapshot()` |
| Deadlock | Actor calling itself synchronously | Use async properly |

### 3.6 Testing Strategy

```swift
import XCTest

final class DataStoreTests: XCTestCase {
    var store: DataStore<String, Int>!
    
    override func setUp() async throws {
        store = DataStore<String, Int>()
    }
    
    func testBasicGetSet() async throws {
        await store.set("count", value: 42)
        let value = await store.get("count")
        XCTAssertEqual(value, 42)
    }
    
    func testConcurrentReads() async throws {
        await store.set("value", value: 100)
        
        // 100 concurrent reads
        let values = await withTaskGroup(of: Int?.self) { group in
            for _ in 0..<100 {
                group.addTask {
                    await self.store.get("value")
                }
            }
            
            var results: [Int?] = []
            for await value in group {
                results.append(value)
            }
            return results
        }
        
        XCTAssertEqual(values.count, 100)
        XCTAssertTrue(values.allSatisfy { $0 == 100 })
    }
    
    func testAtomicUpdate() async throws {
        await store.set("counter", value: 0)
        
        // 100 concurrent increments
        await withTaskGroup(of: Void.self) { group in
            for _ in 0..<100 {
                group.addTask {
                    await self.store.update("counter") { current in
                        (current ?? 0) + 1
                    }
                }
            }
        }
        
        let final = await store.get("counter")
        XCTAssertEqual(final, 100, "All increments should be atomic")
    }
}
```

### 3.7 Interview Q&A

**Q: Why use an actor instead of a class with locks?**

**A**: Actors provide:
1. **Compiler enforcement**: Can't forget `await`
2. **No deadlocks**: No manual lock management
3. **Async-first**: Plays well with Swift Concurrency
4. **Cleaner code**: Less boilerplate

Compare:
```swift
// Actor: Clean, safe
actor Store {
    var data

: Int = 0
}

// Lock: Verbose, error-prone
class Store {
    private var data: Int = 0
    private let lock = NSLock()
    
    func getData() -> Int {
        lock.lock()
        defer { lock.unlock() }
        return data
    }
}
```

---

## 4. Download Manager With Priority

### 4.1 Concept Explanation

**Simple Analogy**: Like a restaurant kitchen with order priorities:
- **High priority** = VIP table orders (go first)
- **Normal priority** = Regular orders
- **Low priority** = Prep work (when idle)

**Problem It Solves**: Apps need to download multiple files:
- User taps PDF → Download immediately (high priority)
- App prefetches articles → Download when idle (low priority)
- Limited bandwidth → Can't download everything at once

### 4.2 Requirements & Constraints

**Functional Requirements**:
- Priority levels (high, normal, low)
- Concurrency limits (max 3 simultaneous downloads)
- Progress reporting
- Pause/resume
- Cancellation
- De-duplication

**Non-Functional Requirements**:
| Constraint | Target |
|------------|--------|
| Max concurrent | 3 downloads |
| Priority starvation | Prevent low-priority blocking |
| Memory | Stream to disk (don't buffer) |

### 4.3 Swift Implementation

#### 4.3.1 OperationQueue Approach

```swift
import Foundation

enum DownloadPriority {
    case high, normal, low
    
    var operationPriority: Operation.QueuePriority {
        switch self {
        case .high: return .veryHigh
        case .normal: return .normal
        case .low: return .veryLow
        }
    }
}

class DownloadOperation: Operation {
    let url: URL
    let destinationURL: URL
    let priority: DownloadPriority
    
    private(set) var downloadedData: Data?
    private(set) var error: Error?
    
    init(url: URL, destinationURL: URL, priority: DownloadPriority) {
        self.url = url
        self.destinationURL = destinationURL
        self.priority = priority
        super.init()
        self.queuePriority = priority.operationPriority
    }
    
    override func main() {
        guard !isCancelled else { return }
        
        let semaphore = DispatchSemaphore(value: 0)
        
        let task = URLSession.shared.downloadTask(with: url) { [weak self] tempURL, response, error in
            defer { semaphore.signal() }
            
            guard let self = self, !self.isCancelled else { return }
            
            if let error = error {
                self.error = error
                return
            }
            
            guard let tempURL = tempURL else { return }
            
            do {
                try FileManager.default.moveItem(at: tempURL, to: self.destinationURL)
            } catch {
                self.error = error
            }
        }
        
        task.resume()
        semaphore.wait()
    }
}

class DownloadManager {
    private let operationQueue: OperationQueue
    
    init(maxConcurrent: Int = 3) {
        operationQueue = OperationQueue()
        operationQueue.maxConcurrentOperationCount = maxConcurrent
    }
    
    func download(
        from url: URL,
        to destinationURL: URL,
        priority: DownloadPriority = .normal,
        completion: @escaping (Result<URL, Error>) -> Void
    ) {
        let operation = DownloadOperation(
            url: url,
            destinationURL: destinationURL,
            priority: priority
        )
        
        operation.completionBlock = {
            if let error = operation.error {
                completion(.failure(error))
            } else {
                completion(.success(destinationURL))
            }
        }
        
        operationQueue.addOperation(operation)
    }
}
```

#### 4.3.2 Async/Await TaskGroup Approach

```swift
actor DownloadManager {
    private var activeDownloads: [URL: Task<URL, Error>] = [:]
    private let maxConcurrent = 3
    
    func download(from url: URL, priority: TaskPriority = .medium) async throws -> URL {
        // De-duplication
        if let existingTask = activeDownloads[url] {
            return try await existingTask.value
        }
        
        let task = Task(priority: priority) {
            try await self.performDownload(url: url)
        }
        
        activeDownloads[url] = task
        
        defer {
            activeDownloads.removeValue(forKey: url)
        }
        
        return try await task.value
    }
    
    private func performDownload(url: URL) async throws -> URL {
        let (tempURL, _) = try await URLSession.shared.download(from: url)
        // Move to permanent location
        return tempURL
    }
}
```

### 4.4 Interview Q&A

**Q: How do you prevent priority inversion (low priority blocking high)?**

**A**: Use priority inheritance or separate queues per priority level.

---

## 5. Observable Pattern Implementation

### 5.1 Concept Explanation

**Simple Analogy**: Like subscribing to a newsletter:
- **Publisher** = Newsletter company
- **Subscriber** = Email recipient
- **Event** = New newsletter issue

**Problem It Solves**: UI needs to react to data changes:
- User logs in → Update profile screen
- New message arrives → Show notification
- Network status changes → Show/hide offline banner

### 5.2 NotificationCenter

```swift
// Post notification
NotificationCenter.default.post(
    name: .userDidLogin,
    object: nil,
    userInfo: ["userID": "123"]
)

// Observe
NotificationCenter.default.addObserver(
    forName: .userDidLogin,
    object: nil,
    queue: .main
) { notification in
    if let userID = notification.userInfo?["userID"] as? String {
        print("User logged in: \(userID)")
    }
}

extension Notification.Name {
    static let userDidLogin = Notification.Name("userDidLogin")
}
```

**Pros**: Simple, system-wide, no dependencies
**Cons**: Stringly-typed, hard to debug, no backpressure

### 5.3 Combine

```swift
import Combine

class AuthService {
    @Published var isLoggedIn: Bool = false
    
    func login() {
        isLoggedIn = true
    }
}

// Subscribe
let auth = AuthService()
let cancellable = auth.$isLoggedIn
    .sink { isLoggedIn in
        print("Login state: \(isLoggedIn)")
    }
```

### 5.4 Custom Lightweight Observable

```swift
class Observable<Value> {
    typealias Observer = (Value) -> Void
    
    private var observers: [UUID: Observer] = [:]
    private let queue = DispatchQueue(label: "observable")
    
    var value: Value {
        didSet {
            notifyObservers()
        }
    }
    
    init(_ value: Value) {
        self.value = value
    }
    
    func observe(_ observer: @escaping Observer) -> ObservationToken {
        let id = UUID()
        queue.async {
            self.observers[id] = observer
            observer(self.value)  // Immediate callback
        }
        
        return ObservationToken { [weak self] in
            self?.queue.async {
                self?.observers.removeValue(forKey: id)
            }
        }
    }
    
    private func notifyObservers() {
        queue.async {
            for observer in self.observers.values {
                observer(self.value)
            }
        }
    }
}

class ObservationToken {
    private let cancellationClosure: () -> Void
    
    init(cancellationClosure: @escaping () -> Void) {
        self.cancellationClosure = cancellationClosure
    }
    
    deinit {
        cancellationClosure()
    }
}
```

---

## 6. Mobile System Design Interview Drill

### Design an iOS Feed with Caching, Downloads, Thread-Safety, and Observability

**Scenario**: Design Instagram-like feed that loads efficiently with minimal network usage.

### 6.1 Requirements

**Functional**:
- Display posts with images
- Infinite scroll pagination
- Offline support
- Pull-to-refresh
- Like/comment actions

**Non-Functional**:
- Load first screen <2s
- 60fps scrolling
- <100MB memory
- <500MB disk cache
- Network usage <50MB per session

### 6.2 API Assumptions

```swift
// GET /feed?cursor={cursor}&limit=20
struct FeedResponse: Codable {
    let posts: [Post]
    let nextCursor: String?
}

struct Post: Codable {
    let id: String
    let imageURL: URL
    let caption: String
    let likes: Int
}
```

### 6.3 Architecture

```
┌─────────────────────────────────────────┐
│           FeedViewModel                 │
│  • Manages state                        │
│  • Coordinates dependencies             │
└───────┬─────────────────────────────────┘
        │
        ├─→ NetworkManager (fetch posts)
        ├─→ ImageCache (load images)
        ├─→ DataStore (cache feed state)
        └─→ DownloadManager (prefetch)
```

### 6.4 Implementation

```swift
@MainActor
class FeedViewModel: ObservableObject {
    @Published var posts: [Post] = []
    @Published var isLoading = false
    
    private let networkManager: NetworkManager
    private let imageCache: ImageCacheManager
    private let feedStore: DataStore<String, [Post]>
    private var nextCursor: String?
    
    init(
        networkManager: NetworkManager,
        imageCache: ImageCacheManager,
        feedStore: DataStore<String, [Post]>
    ) {
        self.networkManager = networkManager
        self.imageCache = imageCache
        self.feedStore = feedStore
        
        Task {
            await loadCachedFeed()
        }
    }
    
    func loadNextPage() async {
        guard !isLoading else { return }
        isLoading = true
        defer { isLoading = false }
        
        do {
            let request = GetFeedRequest(cursor: nextCursor, limit: 20)
            let response = try await networkManager.request(request)
            
            posts.append(contentsOf: response.posts)
            nextCursor = response.nextCursor
            
            // Cache to disk
            await feedStore.set("cachedFeed", value: posts)
            
            // Prefetch images
            prefetchImages(for: response.posts)
        } catch {
            print("Error: \(error)")
        }
    }
    
    private func loadCachedFeed() async {
        if let cached = await feedStore.get("cachedFeed") {
            posts = cached
        }
    }
    
    private func prefetchImages(for posts: [Post]) {
        let urls = posts.map { $0.imageURL }
        imageCache.prefetch(urls)
    }
}
```

### 6.5 Caching Strategy

**Memory**: Recent 50 images (~50MB)
**Disk**: Last 7 days (~500MB)
**Network**: Paginated fetch (20 posts at a time)

### 6.6 Failure Scenarios

1. **No network**: Show cached feed, disable refresh
2. **Image load fails**: Show placeholder
3. **Partial page load**: Show successful posts
4. **Memory warning**: Clear memory cache

### 6.7 Follow-Up Questions

**Q: How would you optimize for slow networks?**
**A**: Progressive image loading, lower quality images on cellular

**Q: How to handle real-time updates (new posts)?**
**A**: WebSocket connection, poll every 30s, or push notifications

**Q: What if user has 10,000 posts cached?**
**A**: Lazy loading, virtual scrolling, paginate disk reads

---

**This completes all five components and the interview drill!**

