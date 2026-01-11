# 🎯 Advanced iOS Interview Cheat Sheet
## For Uber, Amazon, Google & Top-Tier Tech Companies

> **Focus:** What senior interviewers ACTUALLY ask — beyond the basics. This guide covers the non-obvious, advanced patterns and deep knowledge that separates candidates who "pass" from those who get "strong hire."

---

## Table of Contents

1. [System Design: HLD Must-Know Patterns](#-hld-must-know-patterns)
2. [Low-Level Design: Implementation Deep Dives](#-lld-implementation-deep-dives)
3. [Concurrency: What They Actually Ask](#-concurrency-what-they-actually-ask)
4. [Memory & Performance: The Silent Killers](#-memory--performance-the-silent-killers)
5. [Networking: Production-Grade Patterns](#-networking-production-grade-patterns)
6. [Swift Deep: Language Mastery](#-swift-deep-language-mastery)
7. [Company-Specific Focus Areas](#-company-specific-focus-areas)
8. [Quick Reference Tables](#-quick-reference-tables)

---

## 🏗️ HLD Must-Know Patterns

### Pattern 1: Single Source of Truth (SSOT)

**This comes up in 90% of system design interviews!**

```
❌ WRONG: Multiple sources of truth
┌─────────┐    ┌─────────┐    ┌─────────┐
│   UI    │ ←→ │ Network │ ←→ │  Cache  │
└─────────┘    └─────────┘    └─────────┘
     ↓              ↓              ↓
   Which is correct? 💥 DATA INCONSISTENCY

✅ CORRECT: Single source of truth
┌─────────┐    ┌─────────────────────┐    ┌─────────┐
│   UI    │ ←─ │ Repository (SSOT)   │ ─→ │ Network │
└─────────┘    │  ↕ Local Database   │    └─────────┘
               └─────────────────────┘
                  ↑ All reads here
```

**iOS Implementation Pattern:**
```swift
// Repository Pattern with Observable Store
final class UserRepository {
    // SSOT: The database is the truth
    private let database: CoreDataStack
    private let api: UserAPIService
    
    // Combine-based observable
    var usersPublisher: AnyPublisher<[User], Never> {
        database.observeUsers()
    }
    
    func refreshUsers() async throws {
        let remote = try await api.fetchUsers()
        await database.upsert(remote) // Database is updated
        // UI automatically updates through publisher!
    }
}
```

**Key Interview Points:**
- Repository owns ALL data access (read + write)
- UI observes database, never directly reads from network
- Network changes flow through database to UI
- Enables offline-first by design

---

### Pattern 2: Pagination Strategies (Critical for Feeds!)

| Strategy | Use Case | Pros | Cons |
|----------|----------|------|------|
| **Offset-based** | Static content | Simple | Duplicates on insert |
| **Cursor-based** | Real-time feeds | No duplicates | More complex |
| **Keyset** | Large datasets | Fast, consistent | Requires sorting |

**Cursor Pagination Implementation:**
```swift
struct PaginatedResponse<T: Codable>: Codable {
    let items: [T]
    let nextCursor: String? // nil = no more pages
    let hasMore: Bool
}

class FeedPaginator {
    private var currentCursor: String?
    private var isLoading = false
    private var hasMore = true
    
    func loadNextPage() async throws -> [Post] {
        guard !isLoading, hasMore else { return [] }
        isLoading = true
        defer { isLoading = false }
        
        let response = try await api.getFeed(cursor: currentCursor)
        currentCursor = response.nextCursor
        hasMore = response.hasMore
        return response.items
    }
    
    func reset() {
        currentCursor = nil
        hasMore = true
    }
}
```

**Infinite Scroll with Prefetch:**
```swift
class FeedViewController: UIViewController, UICollectionViewDataSourcePrefetching {
    func collectionView(_ collectionView: UICollectionView, 
                       prefetchItemsAt indexPaths: [IndexPath]) {
        let threshold = posts.count - 5
        if indexPaths.contains(where: { $0.item >= threshold }) {
            Task { try? await paginator.loadNextPage() }
        }
    }
}
```

---

### Pattern 3: Conflict Resolution Strategies

**This is asked in EVERY chat/collaborative app design!**

| Strategy | When To Use | Example |
|----------|-------------|---------|
| **Last-Write-Wins (LWW)** | Simple fields, user profiles | User bio update |
| **Server-Authoritative** | Money, inventory | Payment, stock |
| **Client-Wins** | Offline-first priority | Draft messages |
| **CRDTs** | Real-time collab | Google Docs, Figma |
| **Merge** | Non-conflicting data | List additions |

**Implementation Example - Field-level LWW:**
```swift
struct SyncableField<T: Codable>: Codable {
    var value: T
    var updatedAt: Date
    var deviceId: String
}

struct UserProfile: Codable {
    var name: SyncableField<String>
    var avatar: SyncableField<URL?>
    var bio: SyncableField<String>
}

func merge(local: UserProfile, server: UserProfile) -> UserProfile {
    var result = UserProfile()
    
    // Field-by-field merge with timestamp comparison
    result.name = local.name.updatedAt > server.name.updatedAt 
        ? local.name : server.name
    result.avatar = local.avatar.updatedAt > server.avatar.updatedAt 
        ? local.avatar : server.avatar
    result.bio = local.bio.updatedAt > server.bio.updatedAt 
        ? local.bio : server.bio
    
    return result
}
```

---

### Pattern 4: Rate Limiting & Debouncing (Asked at Uber!)

**Debounce vs Throttle:**
```
Debounce: │───x───x───x─────────│→ fires after quiet period
Throttle: │x───────x───────x────│→ fires at fixed intervals
```

**Production-Ready Debouncer:**
```swift
final class Debouncer {
    private let queue: DispatchQueue
    private var workItem: DispatchWorkItem?
    private let delay: TimeInterval
    
    init(delay: TimeInterval, queue: DispatchQueue = .main) {
        self.delay = delay
        self.queue = queue
    }
    
    func debounce(_ action: @escaping () -> Void) {
        workItem?.cancel()
        let item = DispatchWorkItem(block: action)
        workItem = item
        queue.asyncAfter(deadline: .now() + delay, execute: item)
    }
}

// Usage in search
class SearchViewModel {
    private let debouncer = Debouncer(delay: 0.3)
    
    func onSearchTextChange(_ query: String) {
        debouncer.debounce { [weak self] in
            Task { await self?.executeSearch(query) }
        }
    }
}
```

**Throttle for Scroll Events:**
```swift
final class Throttler {
    private var lastFire: Date = .distantPast
    private let interval: TimeInterval
    private let queue: DispatchQueue
    
    init(interval: TimeInterval, queue: DispatchQueue = .main) {
        self.interval = interval
        self.queue = queue
    }
    
    func throttle(_ action: @escaping () -> Void) {
        let now = Date()
        guard now.timeIntervalSince(lastFire) >= interval else { return }
        lastFire = now
        queue.async(execute: action)
    }
}
```

---

## 🔧 LLD Implementation Deep Dives

### LRU Cache - Write This From Scratch!

**This is asked at Google, Amazon, Uber frequently!**

```swift
final class LRUCache<Key: Hashable, Value> {
    private class Node {
        let key: Key
        var value: Value
        var prev: Node?
        var next: Node?
        
        init(key: Key, value: Value) {
            self.key = key
            self.value = value
        }
    }
    
    private var cache: [Key: Node] = [:]
    private var head: Node? // Most recently used
    private var tail: Node? // Least recently used
    private let capacity: Int
    private let lock = NSLock()
    
    init(capacity: Int) {
        self.capacity = max(1, capacity)
    }
    
    func get(_ key: Key) -> Value? {
        lock.lock()
        defer { lock.unlock() }
        
        guard let node = cache[key] else { return nil }
        moveToHead(node)
        return node.value
    }
    
    func set(_ key: Key, value: Value) {
        lock.lock()
        defer { lock.unlock() }
        
        if let node = cache[key] {
            node.value = value
            moveToHead(node)
        } else {
            let newNode = Node(key: key, value: value)
            cache[key] = newNode
            addToHead(newNode)
            
            if cache.count > capacity {
                removeLRU()
            }
        }
    }
    
    private func moveToHead(_ node: Node) {
        guard node !== head else { return }
        removeNode(node)
        addToHead(node)
    }
    
    private func addToHead(_ node: Node) {
        node.next = head
        node.prev = nil
        head?.prev = node
        head = node
        if tail == nil { tail = node }
    }
    
    private func removeNode(_ node: Node) {
        node.prev?.next = node.next
        node.next?.prev = node.prev
        if node === head { head = node.next }
        if node === tail { tail = node.prev }
    }
    
    private func removeLRU() {
        guard let lru = tail else { return }
        removeNode(lru)
        cache.removeValue(forKey: lru.key)
    }
}
```

**Time Complexity:**
- Get: O(1)
- Set: O(1)
- Space: O(n)

---

### Thread-Safe Data Structures

**Reader-Writer Lock Pattern:**
```swift
final class ThreadSafeStore<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    private let queue = DispatchQueue(label: "store.queue", 
                                       attributes: .concurrent)
    
    // Multiple concurrent reads
    func get(_ key: Key) -> Value? {
        queue.sync { storage[key] }
    }
    
    // Exclusive write access
    func set(_ key: Key, value: Value) {
        queue.async(flags: .barrier) { [weak self] in
            self?.storage[key] = value
        }
    }
    
    // Batch operations
    func batchUpdate(_ updates: [Key: Value]) {
        queue.async(flags: .barrier) { [weak self] in
            for (key, value) in updates {
                self?.storage[key] = value
            }
        }
    }
}
```

**Actor-Based (Modern Swift):**
```swift
actor ThreadSafeStore<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    
    func get(_ key: Key) -> Value? {
        storage[key]
    }
    
    func set(_ key: Key, value: Value) {
        storage[key] = value
    }
    
    // Nonisolated read for performance-critical path
    nonisolated func unsafeGet(_ key: Key) -> Value? {
        // ⚠️ Use only if you understand implications
        Task { await self.get(key) }
        return nil
    }
}
```

---

### Image Loading System Design

**Production Architecture:**
```
┌─────────────────────────────────────────────────────────┐
│                    ImageLoader                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────┐ │
│  │MemoryCache  │→ │ DiskCache   │→ │URLSession       │ │
│  │(NSCache)    │  │(FileManager)│  │(Background)     │ │
│  └─────────────┘  └─────────────┘  └─────────────────┘ │
│         ↑               ↑                   ↓          │
│         └───────────────┴───────────────────┘          │
│                    Write Back                           │
└─────────────────────────────────────────────────────────┘
```

**Core Implementation:**
```swift
final class ImageLoader {
    static let shared = ImageLoader()
    
    private let memoryCache = NSCache<NSString, UIImage>()
    private let fileManager = FileManager.default
    private let downloadQueue = OperationQueue()
    private var ongoingTasks: [URL: DownloadTask] = [:]
    private let lock = NSLock()
    
    private lazy var cacheDirectory: URL = {
        let paths = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)
        return paths[0].appendingPathComponent("ImageCache")
    }()
    
    init() {
        downloadQueue.maxConcurrentOperationCount = 4
        memoryCache.totalCostLimit = 100 * 1024 * 1024 // 100MB
        try? fileManager.createDirectory(at: cacheDirectory, 
                                         withIntermediateDirectories: true)
    }
    
    @MainActor
    func load(url: URL, into imageView: UIImageView) async {
        let key = url.absoluteString as NSString
        
        // 1. Check memory cache
        if let cached = memoryCache.object(forKey: key) {
            imageView.image = cached
            return
        }
        
        // 2. Check disk cache
        if let diskImage = loadFromDisk(url: url) {
            memoryCache.setObject(diskImage, forKey: key)
            imageView.image = diskImage
            return
        }
        
        // 3. Download
        do {
            let image = try await download(url: url)
            memoryCache.setObject(image, forKey: key)
            saveToDisk(image: image, url: url)
            imageView.image = image
        } catch {
            // Handle error or show placeholder
        }
    }
    
    private func download(url: URL) async throws -> UIImage {
        let (data, _) = try await URLSession.shared.data(from: url)
        guard let image = UIImage(data: data) else {
            throw ImageError.invalidData
        }
        return image
    }
    
    private func loadFromDisk(url: URL) -> UIImage? {
        let path = diskPath(for: url)
        return UIImage(contentsOfFile: path.path)
    }
    
    private func saveToDisk(image: UIImage, url: URL) {
        guard let data = image.jpegData(compressionQuality: 0.8) else { return }
        let path = diskPath(for: url)
        try? data.write(to: path)
    }
    
    private func diskPath(for url: URL) -> URL {
        let hash = url.absoluteString.sha256()
        return cacheDirectory.appendingPathComponent(hash)
    }
    
    func cancelLoad(for url: URL) {
        lock.lock()
        ongoingTasks[url]?.cancel()
        ongoingTasks.removeValue(forKey: url)
        lock.unlock()
    }
}

enum ImageError: Error {
    case invalidData
    case downloadFailed
}
```

---

## ⚡ Concurrency: What They Actually Ask

### 1. Race Condition Detection

**Question: "Find the bug in this code"**
```swift
// ❌ BUG: Race condition!
class Counter {
    var count = 0
    
    func increment() {
        DispatchQueue.global().async {
            self.count += 1 // Multiple threads reading/writing!
        }
    }
}

// ✅ FIX 1: Serial queue
class Counter {
    private var count = 0
    private let queue = DispatchQueue(label: "counter.queue")
    
    func increment() {
        queue.async { self.count += 1 }
    }
    
    var value: Int {
        queue.sync { count }
    }
}

// ✅ FIX 2: Actor
actor Counter {
    var count = 0
    func increment() { count += 1 }
}
```

### 2. Deadlock Scenarios

**Know these patterns!**

```swift
// ❌ DEADLOCK 1: sync on current queue
let queue = DispatchQueue(label: "deadlock")
queue.sync {
    queue.sync { // 💥 Deadlock!
        print("Never reached")
    }
}

// ❌ DEADLOCK 2: Main queue sync from main
DispatchQueue.main.sync { // 💥 Deadlock on main thread!
    print("Never reached")
}

// ❌ DEADLOCK 3: Lock ordering
let lock1 = NSLock()
let lock2 = NSLock()

// Thread A              // Thread B
lock1.lock()             lock2.lock()
lock2.lock() // waits    lock1.lock() // waits
// 💥 Deadlock!
```

### 3. Priority Inversion

**Uber/Apple loves this question!**

```
Problem:
┌───────────┐ holds │L│ waiting for CPU
│  Low      │───────────────────────→
└───────────┘
┌───────────┐ consumes all CPU
│  Medium   │████████████████████████
└───────────┘
┌───────────┐ waiting for │L│
│  High     │─────────────────────X
└───────────┘
                         ↑ Inverted: High waits for Low!

Solution: Priority Inheritance
When High waits for Low's lock, temporarily boost Low to High priority
```

**Swift Solution:**
```swift
// Use dispatch queues with proper QoS
let queue = DispatchQueue(label: "critical", 
                          qos: .userInteractive, // High priority
                          attributes: [])

// Or use os_unfair_lock instead of NSLock
import os
var unfairLock = os_unfair_lock()
os_unfair_lock_lock(&unfairLock)
// critical section
os_unfair_lock_unlock(&unfairLock)
```

### 4. Task Cancellation

```swift
class SearchService {
    private var currentTask: Task<[Result], Error>?
    
    func search(_ query: String) async throws -> [Result] {
        // Cancel previous search
        currentTask?.cancel()
        
        let task = Task {
            // Check cancellation before expensive work
            try Task.checkCancellation()
            
            let results = try await api.search(query)
            
            // Check again after network call
            try Task.checkCancellation()
            
            return results
        }
        
        currentTask = task
        return try await task.value
    }
}
```

---

## 💾 Memory & Performance: The Silent Killers

### Retain Cycle Patterns

```swift
// ❌ BUG 1: Closure capturing self
class ViewController: UIViewController {
    var completionHandler: (() -> Void)?
    
    override func viewDidLoad() {
        completionHandler = {
            self.doSomething() // Strong reference to self
        }
    }
}

// ✅ FIX: Capture list
completionHandler = { [weak self] in
    self?.doSomething()
}

// ❌ BUG 2: Timer retain cycle
class ViewController: UIViewController {
    var timer: Timer?
    
    override func viewDidLoad() {
        timer = Timer.scheduledTimer(timeInterval: 1, 
                                     target: self, // Strong ref!
                                     selector: #selector(tick),
                                     userInfo: nil, 
                                     repeats: true)
    }
}

// ✅ FIX: Weak target wrapper or block-based timer
timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
    self?.tick()
}

// ❌ BUG 3: Delegate strong reference
class Parent {
    let child = Child()
    
    init() {
        child.delegate = self // If delegate is strong = cycle!
    }
}

class Child {
    var delegate: ParentDelegate? // Should be weak!
}

// ✅ FIX
weak var delegate: ParentDelegate?
```

### Memory Leak Detection

```swift
// Add to debug builds
class MemoryLeakTracker {
    static var instances: [String: Int] = [:]
    
    static func track(_ object: AnyObject) {
        let type = String(describing: type(of: object))
        instances[type, default: 0] += 1
        
        // Print on each alloc
        print("📈 \(type): \(instances[type]!) instances")
    }
    
    static func untrack(_ object: AnyObject) {
        let type = String(describing: type(of: object))
        instances[type, default: 0] -= 1
        
        print("📉 \(type): \(instances[type]!) instances")
    }
}

// Usage in ViewControllers
class MyViewController: UIViewController {
    init() {
        super.init(nibName: nil, bundle: nil)
        #if DEBUG
        MemoryLeakTracker.track(self)
        #endif
    }
    
    deinit {
        #if DEBUG
        MemoryLeakTracker.untrack(self)
        #endif
    }
}
```

### Autoreleasepool for Batch Operations

```swift
// ❌ Memory spike
func processLargeDataset() {
    for i in 0..<1_000_000 {
        let data = createLargeObject(i) // Accumulates in memory
        process(data)
    }
}

// ✅ Controlled memory usage
func processLargeDataset() {
    for i in 0..<1_000_000 {
        autoreleasepool {
            let data = createLargeObject(i)
            process(data)
            // Memory released here
        }
    }
}
```

---

## 🌐 Networking: Production-Grade Patterns

### Retry with Exponential Backoff

```swift
struct RetryPolicy {
    let maxRetries: Int
    let baseDelay: TimeInterval
    let maxDelay: TimeInterval
    let jitter: Bool
    
    func delay(for attempt: Int) -> TimeInterval {
        let exponential = baseDelay * pow(2.0, Double(attempt))
        let capped = min(exponential, maxDelay)
        
        if jitter {
            return capped * Double.random(in: 0.5...1.5)
        }
        return capped
    }
}

func fetchWithRetry<T: Decodable>(
    _ request: URLRequest,
    policy: RetryPolicy = RetryPolicy(maxRetries: 3, 
                                       baseDelay: 1, 
                                       maxDelay: 30, 
                                       jitter: true)
) async throws -> T {
    var lastError: Error?
    
    for attempt in 0..<policy.maxRetries {
        do {
            let (data, response) = try await URLSession.shared.data(for: request)
            
            guard let httpResponse = response as? HTTPURLResponse else {
                throw NetworkError.invalidResponse
            }
            
            // Success
            if 200..<300 ~= httpResponse.statusCode {
                return try JSONDecoder().decode(T.self, from: data)
            }
            
            // Don't retry 4xx errors (client errors)
            if 400..<500 ~= httpResponse.statusCode {
                throw NetworkError.clientError(httpResponse.statusCode)
            }
            
            // 5xx errors - retry
            throw NetworkError.serverError(httpResponse.statusCode)
            
        } catch {
            lastError = error
            
            // Check if retryable
            guard isRetryable(error) else { throw error }
            
            // Wait before retry
            let delay = policy.delay(for: attempt)
            try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
        }
    }
    
    throw lastError ?? NetworkError.unknown
}

private func isRetryable(_ error: Error) -> Bool {
    if let urlError = error as? URLError {
        switch urlError.code {
        case .timedOut, .networkConnectionLost, .notConnectedToInternet:
            return true
        default:
            return false
        }
    }
    if let networkError = error as? NetworkError {
        switch networkError {
        case .serverError: return true
        default: return false
        }
    }
    return false
}
```

### Request Interceptor Pattern

```swift
protocol RequestInterceptor {
    func intercept(_ request: URLRequest) async throws -> URLRequest
    func intercept(_ response: HTTPURLResponse, data: Data) async throws -> Data
}

class AuthInterceptor: RequestInterceptor {
    private let tokenStore: TokenStore
    
    func intercept(_ request: URLRequest) async throws -> URLRequest {
        var request = request
        
        // Add auth header
        if let token = tokenStore.accessToken {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        return request
    }
    
    func intercept(_ response: HTTPURLResponse, data: Data) async throws -> Data {
        // Handle 401 - refresh token
        if response.statusCode == 401 {
            try await tokenStore.refreshTokens()
            throw NetworkError.retryRequired
        }
        return data
    }
}

class LoggingInterceptor: RequestInterceptor {
    func intercept(_ request: URLRequest) async throws -> URLRequest {
        print("🌐 Request: \(request.httpMethod ?? "") \(request.url?.absoluteString ?? "")")
        return request
    }
    
    func intercept(_ response: HTTPURLResponse, data: Data) async throws -> Data {
        print("📥 Response: \(response.statusCode)")
        return data
    }
}
```

### Request Coalescing (Deduplication)

```swift
actor RequestCoalescer {
    private var pendingRequests: [URL: Task<Data, Error>] = [:]
    
    func fetch(url: URL) async throws -> Data {
        // Check if request already in flight
        if let existing = pendingRequests[url] {
            return try await existing.value
        }
        
        // Create new request
        let task = Task {
            defer { pendingRequests.removeValue(forKey: url) }
            let (data, _) = try await URLSession.shared.data(from: url)
            return data
        }
        
        pendingRequests[url] = task
        return try await task.value
    }
}
```

---

## 🧬 Swift Deep: Language Mastery

### Type Erasure (Google Loves This!)

**Problem: Protocol with associated type can't be used as type**
```swift
protocol DataFetcher {
    associatedtype DataType
    func fetch() async throws -> DataType
}

// ❌ Can't do this:
// var fetchers: [DataFetcher] = [] // Error!

// ✅ Solution: Type Erasure
struct AnyDataFetcher<T>: DataFetcher {
    private let _fetch: () async throws -> T
    
    init<F: DataFetcher>(_ fetcher: F) where F.DataType == T {
        _fetch = fetcher.fetch
    }
    
    func fetch() async throws -> T {
        try await _fetch()
    }
}

// Now you can:
var fetchers: [AnyDataFetcher<User>] = []
```

### Property Wrappers

```swift
@propertyWrapper
struct UserDefault<T> {
    let key: String
    let defaultValue: T
    
    var wrappedValue: T {
        get { UserDefaults.standard.object(forKey: key) as? T ?? defaultValue }
        set { UserDefaults.standard.set(newValue, forKey: key) }
    }
}

@propertyWrapper
struct Atomic<Value> {
    private var value: Value
    private let lock = NSLock()
    
    init(wrappedValue: Value) {
        self.value = wrappedValue
    }
    
    var wrappedValue: Value {
        get {
            lock.lock()
            defer { lock.unlock() }
            return value
        }
        set {
            lock.lock()
            defer { lock.unlock() }
            value = newValue
        }
    }
}

// Usage
class Settings {
    @UserDefault(key: "theme", defaultValue: "light")
    var theme: String
    
    @Atomic
    var counter: Int = 0
}
```

### ResultBuilder (SwiftUI-style DSL)

```swift
@resultBuilder
struct ArrayBuilder<Element> {
    static func buildBlock(_ components: Element...) -> [Element] {
        components
    }
    
    static func buildOptional(_ component: [Element]?) -> [Element] {
        component ?? []
    }
    
    static func buildEither(first component: [Element]) -> [Element] {
        component
    }
    
    static func buildEither(second component: [Element]) -> [Element] {
        component
    }
}

// Usage
func buildValidators(@ArrayBuilder<Validator> _ content: () -> [Validator]) -> [Validator] {
    content()
}

let validators = buildValidators {
    EmailValidator()
    PasswordValidator()
    if requiresPhone {
        PhoneValidator()
    }
}
```

---

## 🏢 Company-Specific Focus Areas

### Amazon 🟠

**Technical Focus:**
- Scalability & performance optimization
- Offline-first architecture
- Error handling & edge cases
- Clean code principles

**Leadership Principles to Prepare:**
| Principle | Sample Story Topic |
|-----------|-------------------|
| Customer Obsession | Feature you built based on user feedback |
| Ownership | Bug you fixed without being asked |
| Invent and Simplify | Complex process you simplified |
| Learn and Be Curious | New technology you learned |
| Dive Deep | Root cause analysis story |
| Deliver Results | Tight deadline delivery |

---

### Google 🔵

**Technical Focus:**
- Algorithm efficiency (Big-O analysis)
- Data structure choices
- System design scalability
- Code quality & testability

**Key Interview Themes:**
- "How would you test this?"
- "What's the time/space complexity?"
- "How would this scale to 1M users?"
- "What are alternatives?"

**Must Practice:**
- Binary search variations
- Graph algorithms (BFS/DFS)
- Dynamic programming
- Tree traversals

---

### Uber 🟢

**Technical Focus:**
- Real-time systems design
- Location-based services
- Offline handling
- Map/location streaming

**Specific Topics:**
- WebSocket vs polling
- Location tracking architecture
- Geohashing for nearby search
- State machine for ride states
- Battery optimization for background location

---

## 📋 Quick Reference Tables

### When to Use What Cache

| Cache Type | TTL | Use Case | Example |
|------------|-----|----------|---------|
| Memory (NSCache) | Session | Hot data, images | Current feed images |
| Disk (FileManager) | Days-Weeks | Expensive to fetch | User avatar |
| UserDefaults | Forever | Small key-value | User preferences |
| Core Data | App-controlled | Complex relational | Offline messages |
| Keychain | Forever | Sensitive | Auth tokens |

### Architecture Pattern Selection

| App Type | Recommended Pattern |
|----------|-------------------|
| Simple (few screens) | MVC |
| Medium complexity | MVVM + Coordinator |
| Large team | VIPER / Clean Architecture |
| SwiftUI | MVVM + Repository |
| Modularized | µServices + Dependency Injection |

### Concurrency API Selection

| Scenario | Use |
|----------|-----|
| Simple background work | `Task { }` |
| Cancellable search | `Task` with cancellation |
| Thread-safe state | `actor` |
| Shared mutable state | `actor` or dispatch barrier |
| Dependencies between tasks | `TaskGroup` |
| Limit concurrency | `TaskGroup` or `OperationQueue` |
| Legacy/Objective-C | GCD queues |

### Memory Management Cheat Sheet

| Reference Type | When to Use | nilable? | Crash if nil? |
|----------------|-------------|----------|---------------|
| **strong** (default) | Normal ownership | N/A | N/A |
| **weak** | Delegates, parent refs | Yes (optional) | No |
| **unowned** | Guaranteed same lifetime | No | YES |
| **unowned(unsafe)** | C interop only | No | YES |

---

## 🎯 10-Day Focus Areas

| Day | Morning (3h) | Afternoon (3h) |
|-----|--------------|----------------|
| 1 | HLD patterns (SSOT, Pagination) | Practice: Design Twitter Feed |
| 2 | HLD patterns (Caching, Sync) | Practice: Design WhatsApp |
| 3 | LLD: LRU Cache implementation | LLD: Thread-safe structures |
| 4 | LLD: Image loader, Network layer | Practice: Build both from scratch |
| 5 | Concurrency deep dive | Practice: Find race conditions |
| 6 | Memory management | Debugging retain cycles |
| 7 | Swift advanced (Type erasure, POP) | Algorithm practice |
| 8 | Mock HLD interview | Mock LLD interview |
| 9 | Behavioral prep (STAR stories) | Company research |
| 10 | Quick review all cheat sheets | Rest & mental prep |

---

## ✅ Pre-Interview Checklist

**Technical:**
- [ ] Can implement LRU cache from scratch in 15 min
- [ ] Can design Instagram feed end-to-end
- [ ] Know all retain cycle patterns
- [ ] Understand race conditions/deadlocks
- [ ] Can write thread-safe code
- [ ] Know when to use each architecture pattern

**Behavioral:**
- [ ] 2-3 stories per leadership principle
- [ ] All stories in STAR format
- [ ] Quantifiable results in each story
- [ ] Questions to ask interviewer prepared

**Logistics:**
- [ ] Xcode/IDE ready
- [ ] Whiteboard/paper ready
- [ ] Environment tested (if remote)
- [ ] 8 hours sleep

---

> **Remember:** Top-tier companies hire builders who can ship quality code under pressure. Show them you think about edge cases, write clean code, and understand tradeoffs. Good luck! 🚀
