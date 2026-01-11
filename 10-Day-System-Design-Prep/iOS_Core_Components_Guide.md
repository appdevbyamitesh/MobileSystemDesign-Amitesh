# iOS Core Components - Mobile System Design Guide

**Staff-Level iOS Engineer Interview Preparation**

This guide implements five critical iOS components with beginner → intermediate → advanced progression. Each component includes production-grade Swift code, thread safety, testing strategies, and interview preparation.

---

## Table of Contents

1. [Image Caching System](#1-image-caching-system)
2. [Network Request Manager](#2-network-request-manager)
3. [Thread-Safe Data Store](#3-thread-safe-data-store)
4. [Download Manager With Priority](#4-download-manager-with-priority)
5. [Observable Pattern Implementation](#5-observable-pattern-implementation)
6. [Mobile System Design Interview Drill](#6-mobile-system-design-interview-drill)

---

## 1. Image Caching System

### 1.1 Concept Explanation

**Simple Analogy**: Imagine a restaurant kitchen with three storage areas:
- **Memory Cache** = Counter space (fastest access, limited space, cleared at closing)
- **Disk Cache** = Refrigerator (slower but larger, persists overnight)
- **Network** = Grocery store (slowest, but infinite variety)

**Problem It Solves**: Loading images from the network is slow (100-500ms) and expensive (bandwidth, battery). Users scroll through feeds with hundreds of images. Without caching, the app would:
- Waste bandwidth downloading the same image repeatedly
- Drain battery with excessive network calls
- Provide poor UX with constant loading spinners

**Real App Scenario**: Instagram feed, WhatsApp profile pictures, Pinterest boards

### 1.2 Requirements & Constraints

**Functional Requirements**:
- Store images in memory for instant access
- Persist to disk for offline availability
- Evict old entries when limits reached
- Handle concurrent requests for same image (de-duplication)
- Support cancellation when cells scroll off-screen

**Non-Functional Requirements**:
| Constraint | Target | Reasoning |
|------------|--------|-----------|
| Memory usage | < 50MB | iOS memory limits, background termination |
| Disk usage | < 500MB | User storage, App Store guidelines |
| Cache hit latency | < 5ms | 60fps scrolling (16.7ms frame budget) |
| Disk read latency | < 50ms | Perceived as instant by users |
| Network fallback | < 500ms | Acceptable loading time |
| Memory eviction | LRU | Recent images likely to be viewed again |

**Mobile-Specific Constraints**:
- Memory pressure: iOS sends warnings, must respond
- Background limits: 30s background execution time
- Battery: Prefer disk reads over network calls
- Thread safety: UIImage creation on background threads

### 1.3 Architecture

```
┌─────────────────────────────────────────────────────┐
│                  ImageLoader                        │
│  (Public API: async load, cancel, prefetch)         │
└─────────────────┬───────────────────────────────────┘
                  │
                  ▼
┌─────────────────────────────────────────────────────┐
│             ImageCacheManager                       │
│  • Check memory cache                               │
│  • Check disk cache                                 │
│  • Fetch from network (if needed)                   │
│  • De-duplicate requests                            │
└──────┬──────────────────┬────────────────┬──────────┘
       │                  │                │
       ▼                  ▼                ▼
┌─────────────┐  ┌───────────────┐  ┌──────────────┐
│ MemoryCache │  │   DiskCache   │  │ NetworkClient│
│  (NSCache)  │  │(FileManager)  │  │ (URLSession) │
│             │  │               │  │              │
│ • LRU evict │  │ • Hash keys   │  │ • Download   │
│ • Cost      │  │ • TTL check   │  │ • Decode     │
│   limits    │  │ • Cleanup     │  │              │
└─────────────┘  └───────────────┘  └──────────────┘
```

**Module Responsibilities**:

1. **ImageLoader**: Public-facing API
   - Async image loading with URL
   - Cancellation support
   - Completion handlers / async/await

2. **ImageCacheManager**: Orchestration layer
   - Cache policy enforcement (cache-first, network-first)
   - Request de-duplication
   - Memory pressure handling
   - Metrics/logging

3. **MemoryCache**: Fast in-memory storage
   - LRU eviction via NSCache
   - Cost-based limits (image dimensions)
   - Automatic eviction on memory warnings

4. **DiskCache**: Persistent storage
   - File-based caching
   - MD5 hashing for filenames
   - TTL/expiry logic
   - Background cleanup

5. **NetworkClient**: HTTP layer
   - URLSession data tasks
   - Response validation
   - Image decoding off main thread

### 1.4 Algorithms & Data Structures

**Memory Cache - LRU with NSCache**:
- **Structure**: NSCache (built-in LRU)
- **Key**: String (URL)
- **Value**: UIImage
- **Cost**: width × height (pixel count)
- **Limit**: 50MB total cost
- **Eviction**: Automatic LRU when limit reached
- **Time Complexity**:
  - Get: O(1)
  - Set: O(1)
  - Evict: O(1) amortized

**Disk Cache - Hash Map + File System**:
- **Structure**: FileManager + in-memory metadata dictionary
- **Key**: MD5(URL) for filename
- **Value**: Image data (PNG/JPEG)
- **Metadata**: Last accessed timestamp, size, TTL
- **Cleanup Policy**: 
  - Remove files older than 7 days
  - Remove oldest files if total > 500MB
- **Time Complexity**:
  - Get: O(1) lookup + disk I/O (~5-50ms)
  - Set: O(1) write + disk I/O
  - Cleanup: O(n) where n = file count

**Request De-duplication**:
- **Structure**: Dictionary<URL, Task>
- **Logic**: If URL already has in-flight request, await same Task
- **Benefit**: 100 cells requesting same image = 1 network call

**Cache Policy Decision Tree**:
```
Request Image(URL)
  ├─ In memory? → Return immediately (5ms)
  ├─ On disk?
  │   ├─ TTL valid? → Load from disk (50ms)
  │   └─ TTL expired?
  │       ├─ Stale-while-revalidate? → Return stale + fetch new
  │       └─ Network-only? → Fetch from network
  └─ Not cached → Fetch from network (500ms)
```

### 1.5 Swift Implementation

#### 1.5.1 Memory Cache

```swift
import UIKit

/// In-memory LRU cache using NSCache with cost-based eviction
final class MemoryImageCache {
    private let cache = NSCache<NSString, UIImage>()
    private let costLimit: Int
    private let countLimit: Int
    
    init(costLimit: Int = 50_000_000, countLimit: Int = 100) {
        self.costLimit = costLimit
        self.countLimit = countLimit
        
        cache.totalCostLimit = costLimit  // 50MB in pixels
        cache.countLimit = countLimit     // Max 100 images
        
        // Respond to memory warnings
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleMemoryWarning),
            name: UIApplication.didReceiveMemoryWarningNotification,
            object: nil
        )
    }
    
    func get(_ key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
    
    func set(_ image: UIImage, forKey key: String) {
        // Cost = pixel count (more accurate than byte size)
        let cost = Int(image.size.width * image.size.height * image.scale * image.scale)
        cache.setObject(image, forKey: key as NSString, cost: cost)
    }
    
    func remove(_ key: String) {
        cache.removeObject(forKey: key as NSString)
    }
    
    func removeAll() {
        cache.removeAllObjects()
    }
    
    @objc private func handleMemoryWarning() {
        // Clear cache on memory pressure
        removeAll()
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
    }
}
```

**Why This Design?**:
- NSCache provides built-in LRU eviction (no manual implementation needed)
- Cost calculation uses pixel count (more accurate than file size for memory pressure)
- Automatic eviction on memory warnings prevents app termination
- Thread-safe by default (NSCache is thread-safe)

#### 1.5.2 Disk Cache

```swift
import Foundation
import CryptoKit

/// File-based disk cache with TTL and cleanup policies
actor DiskImageCache {
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    private let maxDiskSize: Int
    private let defaultTTL: TimeInterval
    
    // Metadata for fast lookups
    private var metadata: [String: CacheMetadata] = [:]
    
    struct CacheMetadata: Codable {
        let key: String
        let fileSize: Int
        let createdAt: Date
        let lastAccessed: Date
        var isExpired: Bool {
            Date().timeIntervalSince(createdAt) > 7 * 24 * 60 * 60 // 7 days
        }
    }
    
    init(
        cacheDirectory: URL? = nil,
        maxDiskSize: Int = 500_000_000,  // 500MB
        defaultTTL: TimeInterval = 7 * 24 * 60 * 60  // 7 days
    ) throws {
        if let directory = cacheDirectory {
            self.cacheDirectory = directory
        } else {
            // Use default cache directory
            let cachesURL = try fileManager.url(
                for: .cachesDirectory,
                in: .userDomainMask,
                appropriateFor: nil,
                create: true
            )
            self.cacheDirectory = cachesURL.appendingPathComponent("ImageCache")
        }
        
        self.maxDiskSize = maxDiskSize
        self.defaultTTL = defaultTTL
        
        // Create directory if needed
        try fileManager.createDirectory(
            at: self.cacheDirectory,
            withIntermediateDirectories: true
        )
        
        // Load metadata
        try loadMetadata()
    }
    
    func get(_ key: String) async throws -> Data? {
        let filename = hash(key)
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        
        guard let meta = metadata[filename] else {
            return nil
        }
        
        // Check TTL
        if meta.isExpired {
            try await remove(key)
            return nil
        }
        
        // Update access time
        var updatedMeta = meta
        updatedMeta = CacheMetadata(
            key: key,
            fileSize: meta.fileSize,
            createdAt: meta.createdAt,
            lastAccessed: Date()
        )
        metadata[filename] = updatedMeta
        
        return try Data(contentsOf: fileURL)
    }
    
    func set(_ data: Data, forKey key: String) async throws {
        let filename = hash(key)
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        
        try data.write(to: fileURL)
        
        let meta = CacheMetadata(
            key: key,
            fileSize: data.count,
            createdAt: Date(),
            lastAccessed: Date()
        )
        metadata[filename] = meta
        
        // Cleanup if over limit
        try await cleanupIfNeeded()
        try saveMetadata()
    }
    
    func remove(_ key: String) async throws {
        let filename = hash(key)
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        
        try? fileManager.removeItem(at: fileURL)
        metadata.removeValue(forKey: filename)
        try saveMetadata()
    }
    
    func removeAll() async throws {
        let contents = try fileManager.contentsOfDirectory(
            at: cacheDirectory,
            includingPropertiesForKeys: nil
        )
        
        for fileURL in contents {
            try? fileManager.removeItem(at: fileURL)
        }
        
        metadata.removeAll()
        try saveMetadata()
    }
    
    // MARK: - Private Helpers
    
    private func hash(_ string: String) -> String {
        let data = Data(string.utf8)
        let hash = Insecure.MD5.hash(data: data)
        return hash.map { String(format: "%02x", $0) }.joined()
    }
    
    private func cleanupIfNeeded() async throws {
        let totalSize = metadata.values.reduce(0) { $0 + $1.fileSize }
        
        guard totalSize > maxDiskSize else { return }
        
        // Sort by last accessed (LRU)
        let sortedFiles = metadata.values.sorted { $0.lastAccessed < $1.lastAccessed }
        
        var currentSize = totalSize
        for meta in sortedFiles {
            guard currentSize > maxDiskSize else { break }
            
            try await remove(meta.key)
            currentSize -= meta.fileSize
        }
    }
    
    private func loadMetadata() throws {
        let metadataURL = cacheDirectory.appendingPathComponent("metadata.json")
        
        guard fileManager.fileExists(atPath: metadataURL.path) else {
            return
        }
        
        let data = try Data(contentsOf: metadataURL)
        let decoder = JSONDecoder()
        metadata = try decoder.decode([String: CacheMetadata].self, from: data)
    }
    
    private func saveMetadata() throws {
        let metadataURL = cacheDirectory.appendingPathComponent("metadata.json")
        let encoder = JSONEncoder()
        let data = try encoder.encode(metadata)
        try data.write(to: metadataURL)
    }
}
```

**Why Actor?**:
- Prevents data races on `metadata` dictionary
- File I/O is already async, actor fits naturally
- Serializes all disk operations (prevents corruption)

#### 1.5.3 Request De-duplication

```swift
/// Manages in-flight requests to prevent duplicate network calls
actor RequestCoordinator {
    private var inFlightTasks: [String: Task<UIImage, Error>] = [:]
    
    func getOrCreate(
        forKey key: String,
        operation: @escaping () async throws -> UIImage
    ) async throws -> UIImage {
        // Check if request already in flight
        if let existingTask = inFlightTasks[key] {
            return try await existingTask.value
        }
        
        // Create new task
        let task = Task {
            try await operation()
        }
        
        inFlightTasks[key] = task
        
        defer {
            inFlightTasks.removeValue(forKey: key)
        }
        
        return try await task.value
    }
    
    func cancel(_ key: String) {
        inFlightTasks[key]?.cancel()
        inFlightTasks.removeValue(forKey: key)
    }
    
    func cancelAll() {
        for task in inFlightTasks.values {
            task.cancel()
        }
        inFlightTasks.removeAll()
    }
}
```

#### 1.5.4 Main Image Cache Manager

```swift
import UIKit

enum CachePolicy {
    case cacheFirst      // Memory → Disk → Network
    case networkFirst    // Network → Cache (for fresh data)
    case cacheOnly       // Memory → Disk (no network)
    case networkOnly     // Network (bypass cache)
}

enum ImageCacheError: Error {
    case invalidURL
    case invalidImageData
    case networkError(Error)
    case cancelled
}

/// Main orchestrator for image caching
final class ImageCacheManager {
    static let shared = ImageCacheManager()
    
    private let memoryCache: MemoryImageCache
    private let diskCache: DiskImageCache
    private let coordinator = RequestCoordinator()
    private let session: URLSession
    
    init(
        memoryCache: MemoryImageCache = MemoryImageCache(),
        diskCache: DiskImageCache = try! DiskImageCache(),
        session: URLSession = .shared
    ) {
        self.memoryCache = memoryCache
        self.diskCache = diskCache
        self.session = session
    }
    
    // MARK: - Public API
    
    func loadImage(
        from url: URL,
        policy: CachePolicy = .cacheFirst
    ) async throws -> UIImage {
        let key = url.absoluteString
        
        // Use coordinator to de-duplicate requests
        return try await coordinator.getOrCreate(forKey: key) {
            try await self.fetchImage(from: url, key: key, policy: policy)
        }
    }
    
    func cancelLoad(for url: URL) async {
        await coordinator.cancel(url.absoluteString)
    }
    
    func prefetch(_ urls: [URL]) {
        Task {
            await withTaskGroup(of: Void.self) { group in
                for url in urls {
                    group.addTask {
                        _ = try? await self.loadImage(from: url)
                    }
                }
            }
        }
    }
    
    func clearMemoryCache() {
        memoryCache.removeAll()
    }
    
    func clearDiskCache() async throws {
        try await diskCache.removeAll()
    }
    
    // MARK: - Private Implementation
    
    private func fetchImage(
        from url: URL,
        key: String,
        policy: CachePolicy
    ) async throws -> UIImage {
        switch policy {
        case .cacheFirst:
            return try await cacheFirstFetch(url: url, key: key)
        case .networkFirst:
            return try await networkFirstFetch(url: url, key: key)
        case .cacheOnly:
            return try await cacheOnlyFetch(key: key)
        case .networkOnly:
            return try await networkOnlyFetch(url: url)
        }
    }
    
    private func cacheFirstFetch(url: URL, key: String) async throws -> UIImage {
        // 1. Check memory
        if let image = memoryCache.get(key) {
            return image
        }
        
        // 2. Check disk
        if let data = try await diskCache.get(key),
           let image = await decodeImage(from: data) {
            // Promote to memory cache
            memoryCache.set(image, forKey: key)
            return image
        }
        
        // 3. Fetch from network
        return try await networkFetch(url: url, key: key)
    }
    
    private func networkFirstFetch(url: URL, key: String) async throws -> UIImage {
        // Always fetch from network for fresh data
        return try await networkFetch(url: url, key: key)
    }
    
    private func cacheOnlyFetch(key: String) async throws -> UIImage {
        // Check memory
        if let image = memoryCache.get(key) {
            return image
        }
        
        // Check disk
        if let data = try await diskCache.get(key),
           let image = await decodeImage(from: data) {
            memoryCache.set(image, forKey: key)
            return image
        }
        
        throw ImageCacheError.invalidImageData
    }
    
    private func networkOnlyFetch(url: URL) async throws -> UIImage {
        let (data, _) = try await session.data(from: url)
        
        guard let image = await decodeImage(from: data) else {
            throw ImageCacheError.invalidImageData
        }
        
        return image
    }
    
    private func networkFetch(url: URL, key: String) async throws -> UIImage {
        let (data, _) = try await session.data(from: url)
        
        guard let image = await decodeImage(from: data) else {
            throw ImageCacheError.invalidImageData
        }
        
        // Save to both caches
        memoryCache.set(image, forKey: key)
        try await diskCache.set(data, forKey: key)
        
        return image
    }
    
    private func decodeImage(from data: Data) async -> UIImage? {
        // Decode on background thread to avoid blocking
        return await Task.detached(priority: .userInitiated) {
            UIImage(data: data)
        }.value
    }
}
```

### 1.6 iOS Integration Examples

#### UIImageView Extension

```swift
import UIKit

private var taskKey: UInt8 = 0

extension UIImageView {
    private var loadTask: Task<Void, Never>? {
        get {
            objc_getAssociatedObject(self, &taskKey) as? Task<Void, Never>
        }
        set {
            objc_setAssociatedObject(self, &taskKey, newValue, .OBJC_ASSOCIATION_RETAIN)
        }
    }
    
    func loadImage(
        from url: URL,
        placeholder: UIImage? = nil,
        transition: UIView.AnimationOptions = .transitionCrossDissolve
    ) {
        // Cancel previous load
        loadTask?.cancel()
        
        // Set placeholder
        self.image = placeholder
        
        loadTask = Task { @MainActor in
            do {
                let image = try await ImageCacheManager.shared.loadImage(from: url)
                
                guard !Task.isCancelled else { return }
                
                // Animate transition
                UIView.transition(
                    with: self,
                    duration: 0.3,
                    options: transition,
                    animations: { self.image = image }
                )
            } catch {
                // Handle error (could set error placeholder)
                print("Failed to load image: \(error)")
            }
        }
    }
    
    func cancelImageLoad() {
        loadTask?.cancel()
        loadTask = nil
    }
}

// Usage in UITableViewCell
class FeedCell: UITableViewCell {
    @IBOutlet weak var postImageView: UIImageView!
    
    func configure(with post: Post) {
        postImageView.loadImage(
            from: post.imageURL,
            placeholder: UIImage(named: "placeholder")
        )
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        postImageView.cancelImageLoad()
        postImageView.image = nil
    }
}
```

#### SwiftUI Integration

```swift
import SwiftUI

@MainActor
class ImageLoader: ObservableObject {
    @Published var image: UIImage?
    @Published var isLoading = false
    @Published var error: Error?
    
    private var task: Task<Void, Never>?
    
    func load(from url: URL) {
        task?.cancel()
        
        isLoading = true
        error = nil
        
        task = Task {
            do {
                let loadedImage = try await ImageCacheManager.shared.loadImage(from: url)
                
                guard !Task.isCancelled else { return }
                
                self.image = loadedImage
                self.isLoading = false
            } catch {
                guard !Task.isCancelled else { return }
                
                self.error = error
                self.isLoading = false
            }
        }
    }
    
    func cancel() {
        task?.cancel()
        task = nil
    }
}

struct CachedAsyncImage: View {
    let url: URL
    let placeholder: Image
    
    @StateObject private var loader = ImageLoader()
    
    var body: some View {
        Group {
            if let image = loader.image {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } else if loader.isLoading {
                ProgressView()
            } else {
                placeholder
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            }
        }
        .onAppear {
            loader.load(from: url)
        }
        .onDisappear {
            loader.cancel()
        }
    }
}

// Usage
struct FeedView: View {
    let posts: [Post]
    
    var body: some View {
        ScrollView {
            LazyVStack {
                ForEach(posts) { post in
                    CachedAsyncImage(
                        url: post.imageURL,
                        placeholder: Image(systemName: "photo")
                    )
                    .frame(height: 300)
                    .clipped()
                }
            }
        }
    }
}
```

### 1.7 Thread Safety Strategy

**Memory Cache (NSCache)**:
- ✅ Thread-safe by default
- No additional synchronization needed
- Can be accessed from any thread

**Disk Cache (Actor)**:
- ✅ Actor provides serial execution
- All file operations serialized automatically
- Prevents race conditions on metadata dictionary
- FileManager operations are inherently thread-unsafe, actor protects them

**Request Coordinator (Actor)**:
- ✅ Prevents race on `inFlightTasks` dictionary
- Multiple threads can request same image safely
- Only one network call per unique URL

**UIImage Decoding**:
- Performed on background thread via `Task.detached`
- UIKit operations (setting imageView.image) on main thread with `@MainActor`

**Common Pitfalls & Solutions**:

| Pitfall | Solution |
|---------|----------|
| Updating UI from background thread | Use `@MainActor` or `DispatchQueue.main.async` |
| Data race on shared state | Use actors or serial queues |
| Retain cycles in closures | Use `[weak self]` or structured concurrency |
| Block main thread during decode | Decode on background thread |
| Memory warnings not handled | Observe notifications, clear cache |

### 1.8 Testing Strategy

#### Unit Tests

```swift
import XCTest
@testable import ImageCache

final class MemoryCacheTests: XCTestCase {
    var cache: MemoryImageCache!
    
    override func setUp() {
        super.setUp()
        cache = MemoryImageCache(costLimit: 1000, countLimit: 10)
    }
    
    func testBasicGetSet() {
        let image = UIImage(systemName: "star.fill")!
        cache.set(image, forKey: "test")
        
        let retrieved = cache.get("test")
        XCTAssertNotNil(retrieved)
    }
    
    func testEvictionOnMemoryWarning() {
        let image = UIImage(systemName: "star.fill")!
        cache.set(image, forKey: "test")
        
        // Simulate memory warning
        NotificationCenter.default.post(
            name: UIApplication.didReceiveMemoryWarningNotification,
            object: nil
        )
        
        let retrieved = cache.get("test")
        XCTAssertNil(retrieved, "Cache should be cleared on memory warning")
    }
}

final class DiskCacheTests: XCTestCase {
    var diskCache: DiskImageCache!
    var tempDirectory: URL!
    
    override func setUp() async throws {
        try await super.setUp()
        
        tempDirectory = FileManager.default.temporaryDirectory
            .appendingPathComponent(UUID().uuidString)
        
        diskCache = try await DiskImageCache(
            cacheDirectory: tempDirectory,
            maxDiskSize: 1_000_000
        )
    }
    
    override func tearDown() async throws {
        try? FileManager.default.removeItem(at: tempDirectory)
        try await super.tearDown()
    }
    
    func testWriteAndRead() async throws {
        let testData = "Hello, World!".data(using: .utf8)!
        
        try await diskCache.set(testData, forKey: "test")
        let retrieved = try await diskCache.get("test")
        
        XCTAssertEqual(retrieved, testData)
    }
    
    func testTTLExpiration() async throws {
        // This would require mocking Date or using a test-only TTL
        // Production code should support dependency injection of clock
    }
}
```

#### Concurrency Tests

```swift
final class RequestCoordinatorTests: XCTestCase {
    func testDeduplication() async throws {
        let coordinator = RequestCoordinator()
        var callCount = 0
        
        // Simulate 100 concurrent requests for same resource
        let operation = {
            callCount += 1
            try await Task.sleep(nanoseconds: 100_000_000) // 100ms
            return UIImage(systemName: "star")!
        }
        
        let results = await withTaskGroup(of: UIImage?.self) { group in
            for _ in 0..<100 {
                group.addTask {
                    try? await coordinator.getOrCreate(
                        forKey: "test",
                        operation: operation
                    )
                }
            }
            
            var images: [UIImage?] = []
            for await result in group {
                images.append(result)
            }
            return images
        }
        
        XCTAssertEqual(results.count, 100)
        XCTAssertEqual(callCount, 1, "Operation should only execute once")
    }
}
```

#### Edge Case Tests

```swift
final class ImageCacheManagerTests: XCTestCase {
    func testCancellation() async throws {
        let manager = ImageCacheManager()
        let url = URL(string: "https://example.com/large-image.jpg")!
        
        let task = Task {
            try await manager.loadImage(from: url)
        }
        
        // Cancel immediately
        task.cancel()
        
        do {
            _ = try await task.value
            XCTFail("Should throw cancellation error")
        } catch {
            // Expected
        }
    }
    
    func testInvalidImageData() async throws {
        // Mock URLSession to return invalid data
        // Test that error is properly propagated
    }
    
    func testDiskFull() async throws {
        // Test behavior when disk write fails
    }
}
```

### 1.9 Failure Handling

**Network Failures**:
```swift
// Retry logic with exponential backoff
private func networkFetchWithRetry(
    url: URL,
    key: String,
    retryCount: Int = 3
) async throws -> UIImage {
    var lastError: Error?
    
    for attempt in 0..<retryCount {
        do {
            return try await networkFetch(url: url, key: key)
        } catch {
            lastError = error
            
            // Don't retry on cancellation or 404
            if Task.isCancelled || (error as? URLError)?.code == .fileDoesNotExist {
                throw error
            }
            
            // Exponential backoff: 1s, 2s, 4s
            let delay = UInt64(pow(2.0, Double(attempt)) * 1_000_000_000)
            try await Task.sleep(nanoseconds: delay)
        }
    }
    
    throw lastError ?? ImageCacheError.networkError(URLError(.unknown))
}
```

**Disk Corruption**:
```swift
// Graceful degradation if metadata is corrupt
private func loadMetadata() throws {
    do {
        // Try to load metadata
        let data = try Data(contentsOf: metadataURL)
        metadata = try JSONDecoder().decode([String: CacheMetadata].self, from: data)
    } catch {
        // If corrupt, start fresh (don't crash)
        print("Metadata corrupted, rebuilding from disk")
        metadata = [:]
        try rebuildMetadataFromDisk()
    }
}
```

**Partial Failures**:
```swift
// Prefetch doesn't fail if one image fails
func prefetch(_ urls: [URL]) {
    Task {
        await withTaskGroup(of: Result<UIImage, Error>.self) { group in
            for url in urls {
                group.addTask {
                    do {
                        let image = try await self.loadImage(from: url)
                        return .success(image)
                    } catch {
                        return .failure(error)
                    }
                }
            }
            
            var successCount = 0
            var failureCount = 0
            
            for await result in group {
                switch result {
                case .success:
                    successCount += 1
                case .failure:
                    failureCount += 1
                }
            }
            
            print("Prefetch complete: \(successCount) succeeded, \(failureCount) failed")
        }
    }
}
```

### 1.10 Instrumentation

#### Logging

```swift
import OSLog

extension ImageCacheManager {
    private static let logger = Logger(
        subsystem: "com.app.imagecache",
        category: "ImageCache"
    )
    
    private func logCacheHit(_ key: String, source: String) {
        Self.logger.debug("Cache HIT [\(source)]: \(key, privacy: .private)")
    }
    
    private func logCacheMiss(_ key: String) {
        Self.logger.debug("Cache MISS: \(key, privacy: .private)")
    }
    
    private func logNetworkFetch(_ url: URL, duration: TimeInterval) {
        Self.logger.info("Network fetch: \(url, privacy: .public) took \(duration)s")
    }
}
```

#### Metrics

```swift
final class ImageCacheMetrics {
    private(set) var memoryHits = 0
    private(set) var diskHits = 0
    private(set) var networkFetches = 0
    private(set) var totalRequests = 0
    
    func recordMemoryHit() {
        memoryHits += 1
        totalRequests += 1
    }
    
    func recordDiskHit() {
        diskHits += 1
        totalRequests += 1
    }
    
    func recordNetworkFetch() {
        networkFetches += 1
        totalRequests += 1
    }
    
    var hitRate: Double {
        guard totalRequests > 0 else { return 0 }
        return Double(memoryHits + diskHits) / Double(totalRequests)
    }
    
    func report() -> String {
        """
        Image Cache Metrics:
        - Total Requests: \(totalRequests)
        - Memory Hits: \(memoryHits) (\(percentage(memoryHits))%)
        - Disk Hits: \(diskHits) (\(percentage(diskHits))%)
        - Network Fetches: \(networkFetches) (\(percentage(networkFetches))%)
        - Hit Rate: \(String(format: "%.2f", hitRate * 100))%
        """
    }
    
    private func percentage(_ value: Int) -> String {
        guard totalRequests > 0 else { return "0" }
        return String(format: "%.1f", Double(value) / Double(totalRequests) * 100)
    }
}
```

#### Debugging Tips

1. **Check cache hit rate**: Should be > 70% for good performance
2. **Monitor memory usage**: Use Instruments > Allocations
3. **Profile disk I/O**: Use Instruments > File Activity
4. **Check for leaks**: Instruments > Leaks (look for retained images)
5. **Measure scroll performance**: FPS should stay at 60 during scrolling

**Debug Tools**:
```swift
#if DEBUG
extension ImageCacheManager {
    func debugCacheState() -> String {
        """
        Memory Cache: Objects cached (estimate)
        Disk Cache: \(diskCache.metadata.count) files
        Total Disk Size: \(diskCache.totalSize / 1_000_000)MB
        """
    }
}
#endif
```

### 1.11 Interview Q&A

**Q1: Why use both memory and disk caching instead of just one?**

**Answer**: 
Memory and disk caches serve different purposes with different trade-offs:

**Memory Cache (NSCache)**:
- **Speed**: ~5ms access (direct RAM access)
- **Capacity**: Limited (50MB typical, cleared on memory pressure)
- **Persistence**: Cleared when app terminates
- **Use case**: Recently viewed images in active scroll

**Disk Cache**:
- **Speed**: ~50ms access (file I/O overhead)
- **Capacity**: Larger (500MB typical)
- **Persistence**: Survives app restarts
- **Use case**: Offline support, reducing network calls

**Real scenario**: Instagram feed
- First scroll: Memory cache empty → Disk cache provides images from previous session → No loading spinners
- Continue scrolling: Memory cache serves visible cells instantly → 60fps smooth scrolling
- background: App terminated → Memory cache cleared
- Relaunch: Disk cache provides instant feed → Good cold start UX

**Trade-off**: Complexity vs performance. Could use only disk, but scroll would be janky (50ms per cell load). Could use only memory, but poor offline experience.

---

**Q2: How do you handle memory warnings in iOS?**

**Answer**:
Memory warnings occur when iOS detects memory pressure. Failure to respond can lead to app termination.

**Strategy**:
1. **Observe notifications**:
```swift
NotificationCenter.default.addObserver(
    forName: UIApplication.didReceiveMemoryWarningNotification,
    object: nil,
    queue: .main
) { [weak self] _ in
    self?.handleMemoryWarning()
}
```

2. **Clear memory cache** (not disk):
```swift
func handleMemoryWarning() {
    memoryCache.removeAll()
    // DO NOT clear disk cache - it's not using active memory
}
```

3. **Reduce cache limits** (proactive):
```swift
// In low-memory devices (e.g., iPhone SE)
let deviceMemory = ProcessInfo.processInfo.physicalMemory
let cacheLimit = deviceMemory > 2_000_000_000 ? 50_000_000 : 25_000_000
```

**Advanced**: Use `os_proc_available_memory()` API to proactively reduce cache before warning.

---

**Q3: Explain your request de-duplication strategy. Why is it important?**

**Answer**:
**Problem**: In a feed with 100 cells, if 50 cells show the same profile picture, without de-duplication we'd make 50 network requests for the same image.

**Solution**: Request Coordinator with in-flight task tracking
```swift
// First request: Creates Task, stores in dictionary
let task = Task { try await download(url) }
inFlightTasks[url] = task

// Subsequent requests: Await same Task
return try await inFlightTasks[url]!.value
```

**Benefits**:
- Bandwidth: 50 requests → 1 request (50x reduction)
- Server load: Less API calls
- Battery: Network radio stays on longer when used more
- Latency: Subsequent requests complete immediately when first completes

**Real scenario**: WhatsApp group chat
- 20 messages from same person
- Without de-dup: 20 network requests for profile picture
- With de-dup: 1 network request, 19 instant cache hits

**Trade-off**: Slight memory overhead for task tracking vs massive bandwidth savings.

---

**Q4: How would you implement stale-while-revalidate?**

**Answer**:
Stale-while-revalidate returns cached content immediately while fetching fresh content in background.

**Implementation**:
```swift
private func staleWhileRevalidate(url: URL, key: String) async throws -> UIImage {
    // 1. Check cache (memory or disk)
    if let image = memoryCache.get(key) ?? await diskCachedImage(key) {
        // Return stale content immediately
        Task {
            // Background refresh (fire-and-forget)
            try? await networkFetch(url: url, key: key)
        }
        return image
    }
    
    // 2. If no cache, fetch from network (blocking)
    return try await networkFetch(url: url, key: key)
}
```

**Benefits**:
- **UX**: Instant display (no loading spinner)
- **Freshness**: Content updates in background
- **Perceived performance**: App feels faster

**When to use**:
- ✅ Feed images (stale is acceptable, freshness desired)
- ✅ Profile pictures (change infrequently)
- ❌ Sensitive content (bank balances, medical data)
- ❌ Legal content (terms of service, contracts)

---

**Q5: What's the Big-O complexity of your cache operations?**

**Answer**:

| Operation | Memory (NSCache) | Disk (FileManager + Dict) |
|-----------|------------------|---------------------------|
| Get | O(1) | O(1) dict lookup + O(1) file read* |
| Set | O(1) | O(1) dict insert + O(1) file write* |
| Evict | O(1) amortized | O(1) |
| Cleanup | N/A (automatic) | O(n log n) for sorting by LRU |

\* File I/O is technically O(size of file), but constant for our use case (images ~100KB-1MB)

**Memory Cache**:
- NSCache uses hash table internally → O(1) lookups
- LRU eviction is handled by system → O(1) amortized

**Disk Cache**:
- Metadata dictionary: O(1) hash table
- File reads: O(file size), typically 5-50ms for images
- Cleanup: O(n log n) where n = number of files (sort by timestamp)

**Optimization**: For 10,000 images, cleanup could be expensive. Solution:
- Lazy cleanup: Only clean on app launch, not every write
- Background cleanup: Run on low-priority queue
- Incremental cleanup: Remove 10 oldest files at a time

---

**Q6: How would you test this system for race conditions?**

**Answer**:
Race conditions are hard to reproduce deterministically. Use Thread Sanitizer (TSan) + targeted stress tests.

**1. Enable Thread Sanitizer**:
- Xcode → Edit Scheme → Run → Diagnostics → Thread Sanitizer

**2. Stress Test**:
```swift
func testConcurrentAccess() async {
    let cache = ImageCacheManager()
    let url = URL(string: "https://example.com/test.jpg")!
    
    // Hammer the cache from 100 concurrent threads
    await withTaskGroup(of: Void.self) { group in
        for _ in 0..<100 {
            group.addTask {
                for _ in 0..<100 {
                    _ = try? await cache.loadImage(from: url)
                }
            }
        }
    }
    
    // TSan will catch any data races
}
```

**3. Test Scenarios**:
- Concurrent reads from same cache entry
- Concurrent writes to same key
- Read during eviction
- Disk I/O during metadata update
- Memory warning during fetch

**4. Actor Isolation Verification**:
```swift
// This should trigger compiler error if actor isolation broken
let diskCache = DiskImageCache()
let metadata = diskCache.metadata  // ERROR: actor-isolated property
```

**Industry approach**: Instagram uses similar testing + months of dogfooding before release.

---

**Q7: How would you optimize this for a low-end device (e.g., iPhone SE)?**

**Answer**:
Low-end devices have constraints:
- **RAM**: 2GB vs 6GB (high-end)
- **Storage**: Often nearly full
- **CPU**: Slower image decode
- **Network**: Sometimes limited data plans

**Optimizations**:

**1. Reduce cache sizes**:
```swift
let memory = ProcessInfo.processInfo.physicalMemory
let memoryCacheLimit = memory > 3_000_000_000 ? 50MB : 20MB
let diskCacheLimit = memory > 3_000_000_000 ? 500MB : 200MB
```

**2. Downscale images**:
```swift
func downsample(imageAt url: URL, to size: CGSize) -> UIImage? {
    let options: [CFString: Any] = [
        kCGImageSourceCreateThumbnailFromImageIfAbsent: true,
        kCGImageSourceThumbnailMaxPixelSize: max(size.width, size.height),
        kCGImageSourceShouldCacheImmediately: true
    ]
    
    guard let source = CGImageSourceCreateWithURL(url as CFURL, nil),
          let image = CGImageSourceCreateThumbnailAtIndex(source, 0, options as CFDictionary) else {
        return nil
    }
    
    return UIImage(cgImage: image)
}
```

**3. Aggressive eviction**:
```swift
// More aggressive memory cache eviction
cache.countLimit = 30  // vs 100 on high-end
```

**4. Prioritize Wi-Fi over cellular**:
```swift
if reachability.connection == .cellular && !allowsCellular {
    return try await cacheOnlyFetch(key: key)  // Use cache, skip network
}
```

**5. Reduce image quality**:
```swift
// Request smaller image variants from API
let size = UIScreen.main.bounds.width * UIScreen.main.scale
let imageURL = buildURL(baseURL, size: size)  // e.g., /image?w=750 vs /image?w=1125
```

**Measurement**: Test on actual device, measure:
- Memory usage (Instruments → Allocations)
- Time to first image (should be < 100ms from cache)
- Scroll FPS (should stay 60fps)

---

**Q8: How would you add analytics to track cache performance in production?**

**Answer**:
Need to balance data collection with user privacy and performance.

**Metrics to collect**:
1. Cache hit rate (memory/disk/network %)
2. Average load time (cache vs network)
3. Cache size (MB)
4. Number of evictions
5. Network failures

**Implementation**:
```swift
final class ImageCacheAnalytics {
    static let shared = ImageCacheAnalytics()
    
    func logCacheAccess(source: CacheSource, duration: TimeInterval) {
        // Send to analytics service (e.g., Firebase, Datadog)
        Analytics.record(event: "image_cache_access", parameters: [
            "source": source.rawValue,
            "duration_ms": Int(duration * 1000),
            "cache_hit": source != .network
        ])
    }
    
    func logCacheEviction(reason: EvictionReason) {
        Analytics.record(event: "cache_eviction", parameters: [
            "reason": reason.rawValue
        ])
    }
}
```

**Aggregation** (to reduce events):
```swift
// Instead of logging every access, batch them
private var batchedMetrics: [String: Int] = [:]

func incrementCounter(_ key: String) {
    batchedMetrics[key, default: 0] += 1
}

// Flush every 5 minutes
func flushMetrics() {
    Analytics.record(event: "image_cache_batch", parameters: batchedMetrics)
    batchedMetrics.removeAll()
}
```

**Privacy considerations**:
- Don't log URLs (PII)
- Aggregate data (hit % not individual hits)
- Sample (log 1% of accesses, not 100%)

**Dashboard**:
- P50/P95/P99 load times
- Cache hit rate trend over time
- Memory pressure events
- Network failures by region

---

**Q9: What would change if you had to support video caching?**

**Answer**:
Videos introduce new challenges vs images:
- **Size**: Videos are 100x-1000x larger (10MB-100MB vs 100KB)
- **Streaming**: Can't load entire video into memory
- **Playback**: Need partial/range requests
- **Format**: More complex (containers, codecs)

**Architecture changes**:

**1. Remove memory cache** (too large):
```swift
// Images: Full file in memory
// Videos: Only metadata in memory

struct VideoMetadata {
    let url: URL
    let localPath: URL
    let duration: TimeInterval
    let size: Int64
}
```

**2. Streaming support**:
```swift
// Use AVAssetResourceLoaderDelegate for custom loading
class VideoCache: NSObject, AVAssetResourceLoaderDelegate {
    func resourceLoader(
        _ resourceLoader: AVAssetResourceLoader,
        shouldWaitForLoadingOfRequestedResource loadingRequest: AVAssetResourceLoadingRequest
    ) -> Bool {
        // Check disk cache for requested byte range
        // If available, serve from disk
        // Otherwise, download from network
    }
}
```

**3. Partial downloads**:
```swift
// Support range requests (HTTP 206)
let rangeHeader = "bytes=\(start)-\(end)"
request.setValue(rangeHeader, forHTTPHeaderField: "Range")
```

**4. Prefetch strategy**:
```swift
// For images: Prefetch entire file
// For videos: Prefetch first 5 seconds only
func prefetchVideo(url: URL, duration: TimeInterval = 5.0) async {
    let estimatedSize = 5_000_000  // ~5MB for 5s at typical bitrate
    // Download first N bytes only
}
```

**5. Eviction policy**:
```swift
// Much more aggressive (videos consume GB quickly)
let maxDiskSize = 1_000_000_000  // 1GB vs 500MB for images
let maxVideoCount = 50  // vs 10,000 images
```

**Different cache manager** entirely:
- `ImageCacheManager` for images (memory + disk + full file)
- `VideoCacheManager` for videos (disk only + streaming + partial)

---

**Q10: Walk me through what happens when a user scrolls through a feed quickly.**

**Answer**:
Let's trace a typical scenario: User scrolls fast through 100 cells in 10 seconds.

**Step-by-step flow**:

**1. Cell appears** (cell 0):
```swift
// UITableViewCell.cellForRowAt
imageView.loadImage(from: post.imageURL)
```

**2. Cache check** (5ms):
```swift
// Memory cache: MISS (first load)
// Disk cache: HIT (from previous session)
// Load from disk (50ms)
```

**3. Image decode** (30ms on background):
```swift
Task.detached {
    UIImage(data: diskData)  // Decoding happens here
}
```

**4. UI update** (on main thread):
```swift
@MainActor
imageView.image = decodedImage  // 16.7ms frame budget
```

**5. Cell scrolls off-screen** (100ms later):
```swift
// prepareForReuse called
imageView.cancelImageLoad()  // Cancel if still loading
imageView.image = nil
```

**6. Next 10 cells** (cells 1-10):
- All trigger same flow
- All hit disk cache (fast)
- Decoding happens in parallel (background threads)
- UI updates as they complete

**7. Fast scroll** (cells 50-60 scroll through in 100ms):
```swift
// Cell 50: Starts loading
// Cell 51: Starts loading
// Cell 50: Scrolls off → CANCELLED
// Cell 52: Starts loading
// Cell 51: Scrolls off → CANCELLED
// ...
// Only cells 58-60 (currently visible) complete loading
```

**Optimizations in play**:
- **Cancellation**: Prevents wasted work on invisible cells
- **Background decode**: Main thread stays at 60fps
- **Memory cache promotion**: Disk-loaded images go to memory for later
- **Request de-dup**: Multiple cells with same image = 1 disk read

**Performance metrics**:
- First cell: 50ms (disk) + 30ms (decode) = 80ms total
- Subsequent cells with same image: <5ms (memory cache)
- Scroll FPS: 60fps (frame time 16.7ms, non-blocking)

**What breaks scrolling**:
- ❌ Decoding on main thread: Drops to 30fps
- ❌ Synchronous disk I/O on main: Stutters
- ❌ No cancellation: Waste CPU on invisible cells
- ❌ No memory cache: Every scroll = 50ms disk reads

**Production apps** (Instagram, Pinterest):
- Prefetch next 20 cells during scroll
- Downsample images to display size
- Use progressive JPEG for faster perceived load
- Monitor frame rate, drop quality if <60fps

---

This completes the Image Caching System component. The implementation provides production-grade caching with:
- ✅ Memory + Disk caching with LRU eviction
- ✅ Swift Concurrency (async/await, actors)
- ✅ Request de-duplication
- ✅ Memory pressure handling
- ✅ Comprehensive testing strategy
- ✅ Interview-ready explanations

---

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

