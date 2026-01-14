# LLD Problem 7: LRU Cache

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Data Structures**: HashMap + Doubly Linked List
- **Memory Management**: iOS memory constraints
- **Thread Safety**: Concurrent cache access
- **Real-world Application**: Image caching

---

# R — Requirements

## Functional Requirements

```markdown
1. Cache Operations
   - Get value by key (O(1))
   - Put key-value pair (O(1))
   - Automatic eviction when capacity reached
   - LRU eviction policy

2. Memory Management
   - Configurable capacity
   - Memory warning handling
   - Optional disk persistence
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Get/Put | O(1) | Performance critical |
| Thread safety | Required | Concurrent access |
| Memory limit | Configurable | Device constraints |

---

# E — Entities

```swift
// MARK: - Node for Doubly Linked List

class CacheNode<Key: Hashable, Value> {
    let key: Key
    var value: Value
    var prev: CacheNode?
    var next: CacheNode?
    var size: Int  // For memory-based eviction
    
    init(key: Key, value: Value, size: Int = 1) {
        self.key = key
        self.value = value
        self.size = size
    }
}

// MARK: - Doubly Linked List

class DoublyLinkedList<Key: Hashable, Value> {
    private var head: CacheNode<Key, Value>?
    private var tail: CacheNode<Key, Value>?
    private(set) var count: Int = 0
    
    func addToFront(_ node: CacheNode<Key, Value>) {
        node.next = head
        node.prev = nil
        
        if let head = head {
            head.prev = node
        }
        head = node
        
        if tail == nil {
            tail = node
        }
        count += 1
    }
    
    func remove(_ node: CacheNode<Key, Value>) {
        if node === head {
            head = node.next
        }
        if node === tail {
            tail = node.prev
        }
        
        node.prev?.next = node.next
        node.next?.prev = node.prev
        node.prev = nil
        node.next = nil
        count -= 1
    }
    
    func moveToFront(_ node: CacheNode<Key, Value>) {
        remove(node)
        addToFront(node)
    }
    
    func removeLast() -> CacheNode<Key, Value>? {
        guard let tail = tail else { return nil }
        remove(tail)
        return tail
    }
    
    func removeAll() {
        head = nil
        tail = nil
        count = 0
    }
}
```

---

# H — Handling Concurrency

```swift
actor LRUCache<Key: Hashable, Value> {
    private var map: [Key: CacheNode<Key, Value>] = [:]
    private var list = DoublyLinkedList<Key, Value>()
    private let capacity: Int
    private var currentSize: Int = 0
    
    init(capacity: Int) {
        self.capacity = capacity
    }
    
    func get(_ key: Key) -> Value? {
        guard let node = map[key] else {
            return nil
        }
        
        // Move to front (most recently used)
        list.moveToFront(node)
        return node.value
    }
    
    func put(_ key: Key, value: Value, size: Int = 1) {
        if let existingNode = map[key] {
            // Update existing
            existingNode.value = value
            currentSize = currentSize - existingNode.size + size
            existingNode.size = size
            list.moveToFront(existingNode)
        } else {
            // Add new
            let newNode = CacheNode(key: key, value: value, size: size)
            map[key] = newNode
            list.addToFront(newNode)
            currentSize += size
            
            // Evict if over capacity
            while currentSize > capacity, let evicted = list.removeLast() {
                map.removeValue(forKey: evicted.key)
                currentSize -= evicted.size
            }
        }
    }
    
    func remove(_ key: Key) {
        guard let node = map.removeValue(forKey: key) else { return }
        list.remove(node)
        currentSize -= node.size
    }
    
    func clear() {
        map.removeAll()
        list.removeAll()
        currentSize = 0
    }
    
    var count: Int { map.count }
}
```

---

# A — Architecture & Patterns

## Why NSCache Alone is Insufficient

```swift
// NSCache limitations:
// 1. No access to eviction order
// 2. Can't enumerate contents
// 3. No size tracking control
// 4. No guaranteed LRU behavior

class NSCacheWrapper {
    let cache = NSCache<NSString, AnyObject>()
    
    init() {
        cache.countLimit = 100
        cache.totalCostLimit = 50 * 1024 * 1024  // 50MB
        
        // ❌ Can't know WHICH items will be evicted
        // ❌ Can't iterate over cached items
        // ❌ No access order guarantee
    }
}

// ✅ Our LRU Cache:
// - Guaranteed LRU eviction
// - Can enumerate all items
// - Precise size control
// - Predictable behavior
```

### When NSCache IS Sufficient

| Use NSCache When | Use Custom LRU When |
|------------------|---------------------|
| Simple caching | Need eviction control |
| Memory pressure handling automatic | Need to know what's cached |
| No iteration needed | Need to persist cache |
| Apple's optimization sufficient | Need order guarantees |

---

## Complete Image Cache Implementation

```swift
actor ImageCache {
    static let shared = ImageCache()
    
    private let memoryCache: LRUCache<String, UIImage>
    private let diskCache: DiskCache?
    private let maxMemoryBytes: Int
    private var currentMemoryBytes: Int = 0
    
    private init(maxMemoryMB: Int = 100, maxDiskMB: Int = 500) {
        self.maxMemoryBytes = maxMemoryMB * 1024 * 1024
        self.memoryCache = LRUCache(capacity: maxMemoryBytes)
        self.diskCache = try? DiskCache(maxSizeMB: maxDiskMB)
        
        // Handle memory warnings
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            Task { await self?.handleMemoryWarning() }
        }
    }
    
    func image(for url: URL) async -> UIImage? {
        let key = url.absoluteString
        
        // 1. Check memory cache
        if let memoryImage = await memoryCache.get(key) {
            return memoryImage
        }
        
        // 2. Check disk cache
        if let diskData = await diskCache?.data(for: key),
           let diskImage = UIImage(data: diskData) {
            // Promote to memory cache
            let size = estimateSize(of: diskImage)
            await memoryCache.put(key, value: diskImage, size: size)
            return diskImage
        }
        
        // 3. Fetch from network
        do {
            let (data, _) = try await URLSession.shared.data(from: url)
            guard let image = UIImage(data: data) else { return nil }
            
            // Store in both caches
            let size = estimateSize(of: image)
            await memoryCache.put(key, value: image, size: size)
            await diskCache?.store(data, for: key)
            
            return image
        } catch {
            return nil
        }
    }
    
    func handleMemoryWarning() async {
        // Clear half the memory cache
        await memoryCache.evict(fraction: 0.5)
    }
    
    private func estimateSize(of image: UIImage) -> Int {
        guard let cgImage = image.cgImage else { return 0 }
        return cgImage.bytesPerRow * cgImage.height
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Get/Put/Eviction

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│ Caller │ │  LRUCache  │ │   HashMap  │ │   DLL      │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │
    │ get(key)   │              │              │
    │───────────▶│              │              │
    │            │              │              │
    │            │ lookup       │              │
    │            │─────────────▶│              │
    │            │              │              │
    │            │ node         │              │
    │            │◀─────────────│              │
    │            │              │              │
    │            │ moveToFront(node)           │
    │            │────────────────────────────▶│
    │            │              │              │
    │ value      │              │              │
    │◀───────────│              │              │
    │            │              │              │
─────────────────────────────────────────────────────
    │            │              │              │
    │ put(k, v)  │              │              │
    │───────────▶│              │              │
    │            │              │              │
    │            │ insert       │              │
    │            │─────────────▶│              │
    │            │              │              │
    │            │ addToFront   │              │
    │            │────────────────────────────▶│
    │            │              │              │
    │            │ check capacity              │
    │            │──┐           │              │
    │            │◀─┘           │              │
    │            │              │              │
    │            │ evict (if needed)           │
    │            │────────────────────────────▶│ removeLast
    │            │              │              │
    │            │ remove evicted key          │
    │            │─────────────▶│              │
    │            │              │              │
```

---

# E — Edge Cases

## Edge Case 1: Memory Warning

```swift
extension ImageCache {
    func handleMemoryWarning() async {
        // Strategy: Evict based on priority
        // 1. First, evict low-quality placeholders
        // 2. Then, evict older accessed items
        
        let currentCount = await memoryCache.count
        let toEvict = currentCount / 2  // Evict 50%
        
        for _ in 0..<toEvict {
            await memoryCache.evictLRU()
        }
        
        // Log for monitoring
        Analytics.log("cache_eviction", params: ["count": toEvict])
    }
}
```

## Edge Case 2: Concurrent Access to Same Key

```swift
// Actor handles this automatically
actor LRUCache<Key: Hashable, Value> {
    // All methods are isolated - no race conditions
    
    // Multiple callers accessing same key:
    // - Each call is serialized
    // - All see consistent state
}

// If using non-actor:
class ThreadSafeLRUCache<Key: Hashable, Value> {
    private let lock = NSLock()
    
    func get(_ key: Key) -> Value? {
        lock.lock()
        defer { lock.unlock() }
        // ... implementation
    }
}
```

## Edge Case 3: Value Updated After Retrieval

```swift
// For reference types, cache holds reference
actor LRUCache<Key: Hashable, Value> {
    func get(_ key: Key) -> Value? {
        // If Value is class, caller can mutate
        // Consider returning copy for safety
    }
}

// For image cache: UIImage is reference type but immutable
// So no issue

// For mutable objects:
actor SafeLRUCache<Key: Hashable, Value: Copying> {
    func get(_ key: Key) -> Value? {
        return map[key]?.value.copy()  // Return copy
    }
}
```

---

# D — Design Trade-offs

| Approach | Pros | Cons |
|----------|------|------|
| HashMap + DLL | O(1) all ops | More memory |
| Array + linear search | Simple | O(n) eviction |
| NSCache | Apple optimized | Less control |

**Decision:** HashMap + DLL for predictable O(1) performance

---

# iOS Implementation

## Complete Usage Example

```swift
// MARK: - SwiftUI Image Loading

struct CachedAsyncImage: View {
    let url: URL
    @State private var image: UIImage?
    @State private var isLoading = false
    
    var body: some View {
        Group {
            if let image = image {
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fill)
            } else if isLoading {
                ProgressView()
            } else {
                Color.gray
            }
        }
        .task {
            isLoading = true
            image = await ImageCache.shared.image(for: url)
            isLoading = false
        }
    }
}

// MARK: - Prefetching for TableView

class ImagePrefetcher {
    private var prefetchTasks: [URL: Task<Void, Never>] = [:]
    
    func prefetch(urls: [URL]) {
        for url in urls {
            guard prefetchTasks[url] == nil else { continue }
            
            prefetchTasks[url] = Task {
                _ = await ImageCache.shared.image(for: url)
                prefetchTasks.removeValue(forKey: url)
            }
        }
    }
    
    func cancelPrefetch(urls: [URL]) {
        for url in urls {
            prefetchTasks[url]?.cancel()
            prefetchTasks.removeValue(forKey: url)
        }
    }
}
```

---

# Interview Tips

## What to Say

```markdown
1. "HashMap gives O(1) lookup, DLL gives O(1) order updates..."
2. "NSCache is insufficient because we need eviction control..."
3. "Actor provides thread safety without manual locking..."
4. "For iOS, I'd add memory warning handling..."
```

## Red Flags to Avoid

```markdown
❌ "I'll use an array and search linearly"
   → O(n) operations

❌ "NSCache does everything we need"
   → Shows incomplete understanding

❌ Forgetting thread safety
   → Race conditions

❌ No memory warning handling
   → iOS-specific oversight
```

---

*This is how an Amazon iOS LLD interview expects you to approach the LRU Cache problem!*
