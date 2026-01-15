# LLD Problem 11: Feed Image Loader

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Core iOS Skill**: Every app needs image loading
- **Performance**: Memory, disk, network optimization
- **Concurrency**: Async loading, cancellation, thread safety
- **Caching Strategy**: Multi-level cache design
- **Real-world Application**: Social feeds, e-commerce

---

# R — Requirements

## Functional Requirements

```markdown
1. Image Loading
   - Load images from URL
   - Support multiple image formats (JPEG, PNG, WebP, GIF)
   - Placeholder while loading
   - Error state with retry

2. Caching
   - Memory cache (L1)
   - Disk cache (L2)
   - Cache eviction policies
   - Cache size limits

3. Performance
   - Async loading without blocking UI
   - Request deduplication
   - Request cancellation on scroll
   - Image resizing/downsampling

4. Prefetching
   - Prefetch upcoming images
   - Cancel prefetch on scroll direction change
   - Priority-based loading
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Load time | < 300ms (cached) | Perceived speed |
| Memory | < 100MB cache | Device limits |
| Disk cache | < 500MB | Storage limits |
| Thread safety | Required | Concurrent cells |
| Scroll FPS | 60 | Smooth experience |

## iOS-Specific Requirements

```markdown
- Memory warning handling
- Background/foreground transitions
- Cell reuse cancellation
- UIKit/SwiftUI support
- Thumbnail generation
```

## Clarifying Questions

1. **Formats**: Which image formats to support? → JPEG, PNG, WebP, GIF
2. **Transformations**: Resize, crop, blur? → Resize to target size
3. **Animated GIF**: Full support? → Basic support
4. **CDN**: Images from CDN? → Yes, with caching headers
5. **Offline**: Work offline with cache? → Yes

---

# E — Entities

## Core Classes

```swift
// MARK: - Image Request

struct ImageRequest: Hashable {
    let url: URL
    let targetSize: CGSize?
    let priority: ImagePriority
    let options: ImageOptions
    
    func hash(into hasher: inout Hasher) {
        hasher.combine(url)
        hasher.combine(targetSize?.width)
        hasher.combine(targetSize?.height)
    }
    
    static func == (lhs: ImageRequest, rhs: ImageRequest) -> Bool {
        lhs.url == rhs.url && lhs.targetSize == rhs.targetSize
    }
}

enum ImagePriority: Int, Comparable {
    case low = 0
    case normal = 1
    case high = 2
    case veryHigh = 3
    
    static func < (lhs: ImagePriority, rhs: ImagePriority) -> Bool {
        lhs.rawValue < rhs.rawValue
    }
}

struct ImageOptions: Hashable {
    var transition: ImageTransition = .fadeIn(0.2)
    var placeholder: UIImage?
    var failureImage: UIImage?
    var shouldDownsample: Bool = true
    var cachePolicy: CachePolicy = .all
}

enum ImageTransition {
    case none
    case fadeIn(TimeInterval)
    case crossDissolve(TimeInterval)
}

enum CachePolicy {
    case all           // Memory + Disk
    case memoryOnly    // Memory only
    case diskOnly      // Disk only
    case none          // No caching
}

// MARK: - Image Result

enum ImageResult {
    case success(UIImage, ImageSource)
    case failure(ImageError)
}

enum ImageSource {
    case memory
    case disk
    case network
}

enum ImageError: Error {
    case invalidURL
    case networkError(Error)
    case decodingFailed
    case cancelled
    case notFound
}

// MARK: - Cache Entry

struct CacheEntry {
    let image: UIImage
    let data: Data?
    let size: Int
    let createdAt: Date
    let url: URL
}
```

## Entity Relationships

```
┌─────────────────┐
│  ImageLoader    │ (Facade)
└────────┬────────┘
         │
    ┌────┴────┬──────────────┐
    │         │              │
    ▼         ▼              ▼
┌────────┐ ┌────────┐ ┌────────────┐
│ Memory │ │  Disk  │ │  Network   │
│ Cache  │ │ Cache  │ │  Fetcher   │
└────────┘ └────────┘ └─────┬──────┘
                            │
                            ▼
                     ┌────────────┐
                     │   Decoder  │
                     └────────────┘
```

---

# S — States

## Image Loading States

```swift
enum ImageLoadingState: Equatable {
    case idle
    case loading(progress: Double)
    case success(UIImage)
    case failure(ImageError)
    
    static func == (lhs: ImageLoadingState, rhs: ImageLoadingState) -> Bool {
        switch (lhs, rhs) {
        case (.idle, .idle):
            return true
        case (.loading(let p1), .loading(let p2)):
            return p1 == p2
        case (.success(let i1), .success(let i2)):
            return i1 === i2
        case (.failure, .failure):
            return true
        default:
            return false
        }
    }
}
```

## State Diagram

```
                    ┌─────────────┐
                    │    Idle     │
                    └──────┬──────┘
                           │ load()
                           ▼
                    ┌─────────────┐
         ┌─────────│   Loading   │─────────┐
         │         └──────┬──────┘         │
         │                │                │
         │ cancel         │ network        │ error
         │                │ complete       │
         ▼                ▼                ▼
  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
  │   Idle      │  │   Success   │  │   Failure   │
  └─────────────┘  └─────────────┘  └──────┬──────┘
                                           │
                                           │ retry
                                           ▼
                                    ┌─────────────┐
                                    │   Loading   │
                                    └─────────────┘
```

---

# H — Handling Concurrency

## Thread-Safe Memory Cache

```swift
actor MemoryCache {
    private var cache: [String: CacheEntry] = [:]
    private var accessOrder: [String] = []  // For LRU
    private let maxSize: Int  // In bytes
    private var currentSize: Int = 0
    
    init(maxSizeMB: Int = 100) {
        self.maxSize = maxSizeMB * 1024 * 1024
    }
    
    func get(_ key: String) -> UIImage? {
        guard let entry = cache[key] else { return nil }
        
        // Update access order for LRU
        if let index = accessOrder.firstIndex(of: key) {
            accessOrder.remove(at: index)
            accessOrder.append(key)
        }
        
        return entry.image
    }
    
    func set(_ key: String, image: UIImage, data: Data?) {
        let size = estimateSize(of: image)
        
        // Evict if necessary
        while currentSize + size > maxSize, let oldest = accessOrder.first {
            evict(oldest)
        }
        
        // Store
        let entry = CacheEntry(
            image: image,
            data: data,
            size: size,
            createdAt: Date(),
            url: URL(string: key)!
        )
        
        cache[key] = entry
        accessOrder.append(key)
        currentSize += size
    }
    
    func remove(_ key: String) {
        evict(key)
    }
    
    func clear() {
        cache.removeAll()
        accessOrder.removeAll()
        currentSize = 0
    }
    
    // Handle memory warning - evict 50%
    func handleMemoryWarning() {
        let targetSize = maxSize / 2
        while currentSize > targetSize, let oldest = accessOrder.first {
            evict(oldest)
        }
    }
    
    private func evict(_ key: String) {
        guard let entry = cache.removeValue(forKey: key) else { return }
        accessOrder.removeAll { $0 == key }
        currentSize -= entry.size
    }
    
    private func estimateSize(of image: UIImage) -> Int {
        guard let cgImage = image.cgImage else { return 0 }
        return cgImage.bytesPerRow * cgImage.height
    }
}
```

## Thread-Safe Disk Cache

```swift
actor DiskCache {
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    private let maxSize: Int
    private var currentSize: Int = 0
    private var fileInfos: [String: FileInfo] = [:]
    
    struct FileInfo {
        let size: Int
        var lastAccess: Date
    }
    
    init(maxSizeMB: Int = 500) throws {
        self.maxSize = maxSizeMB * 1024 * 1024
        
        let caches = fileManager.urls(for: .cachesDirectory, in: .userDomainMask).first!
        cacheDirectory = caches.appendingPathComponent("ImageCache", isDirectory: true)
        
        try fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        
        // Calculate initial size
        Task { await calculateCurrentSize() }
    }
    
    func get(_ key: String) async -> Data? {
        let fileURL = fileURL(for: key)
        
        guard fileManager.fileExists(atPath: fileURL.path) else {
            return nil
        }
        
        // Update access time
        fileInfos[key]?.lastAccess = Date()
        
        return try? Data(contentsOf: fileURL)
    }
    
    func set(_ key: String, data: Data) async {
        let fileURL = fileURL(for: key)
        let size = data.count
        
        // Evict if necessary
        while currentSize + size > maxSize {
            await evictOldest()
        }
        
        do {
            try data.write(to: fileURL)
            fileInfos[key] = FileInfo(size: size, lastAccess: Date())
            currentSize += size
        } catch {
            print("Failed to write to disk cache: \(error)")
        }
    }
    
    func remove(_ key: String) {
        let fileURL = fileURL(for: key)
        
        if let info = fileInfos.removeValue(forKey: key) {
            currentSize -= info.size
        }
        
        try? fileManager.removeItem(at: fileURL)
    }
    
    func clear() {
        try? fileManager.removeItem(at: cacheDirectory)
        try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        fileInfos.removeAll()
        currentSize = 0
    }
    
    private func fileURL(for key: String) -> URL {
        let filename = key.data(using: .utf8)!.base64EncodedString()
            .replacingOccurrences(of: "/", with: "_")
        return cacheDirectory.appendingPathComponent(filename)
    }
    
    private func evictOldest() async {
        guard let oldest = fileInfos.min(by: { $0.value.lastAccess < $1.value.lastAccess }) else {
            return
        }
        remove(oldest.key)
    }
    
    private func calculateCurrentSize() async {
        let enumerator = fileManager.enumerator(at: cacheDirectory, includingPropertiesForKeys: [.fileSizeKey])
        
        var totalSize = 0
        while let fileURL = enumerator?.nextObject() as? URL {
            let size = (try? fileURL.resourceValues(forKeys: [.fileSizeKey]).fileSize) ?? 0
            totalSize += size
            
            let key = fileURL.lastPathComponent
            fileInfos[key] = FileInfo(size: size, lastAccess: Date())
        }
        currentSize = totalSize
    }
}
```

## Request Deduplication

```swift
actor RequestCoalescer {
    private var inFlightRequests: [String: Task<UIImage?, Never>] = [:]
    private var waiters: [String: [CheckedContinuation<UIImage?, Never>]] = [:]
    
    func deduplicate<T>(
        key: String,
        operation: @escaping () async -> T
    ) async -> T {
        // If request already in flight, wait for it
        if let existingTask = inFlightRequests[key] as? Task<T, Never> {
            return await existingTask.value
        }
        
        // Start new request
        let task = Task {
            await operation()
        }
        
        inFlightRequests[key] = task as? Task<UIImage?, Never>
        
        let result = await task.value
        
        inFlightRequests.removeValue(forKey: key)
        
        return result
    }
}
```

---

# A — Architecture & Patterns

## Pattern 1: Facade (ImageLoader)

### What it does:
Provides a simple interface hiding cache layers and network complexity.

```swift
public class ImageLoader {
    public static let shared = ImageLoader()
    
    private let memoryCache: MemoryCache
    private let diskCache: DiskCache
    private let networkFetcher: NetworkImageFetcher
    private let decoder: ImageDecoder
    private let requestCoalescer: RequestCoalescer
    
    private var activeTasks: [String: Task<UIImage?, Never>] = [:]
    private let taskLock = NSLock()
    
    private init() {
        self.memoryCache = MemoryCache(maxSizeMB: 100)
        self.diskCache = try! DiskCache(maxSizeMB: 500)
        self.networkFetcher = NetworkImageFetcher()
        self.decoder = ImageDecoder()
        self.requestCoalescer = RequestCoalescer()
        
        setupMemoryWarningObserver()
    }
    
    // MARK: - Public API
    
    /// Load image with simple URL
    public func load(url: URL) async -> UIImage? {
        await load(ImageRequest(url: url, targetSize: nil, priority: .normal, options: ImageOptions()))
    }
    
    /// Load image with full options
    public func load(_ request: ImageRequest) async -> UIImage? {
        let key = cacheKey(for: request)
        
        // 1. Check memory cache (fastest)
        if let cached = await memoryCache.get(key) {
            return cached
        }
        
        // 2. Deduplicate concurrent requests for same URL
        return await requestCoalescer.deduplicate(key: key) {
            await self.fetchImage(request, key: key)
        }
    }
    
    /// Prefetch images (lower priority, no return)
    public func prefetch(urls: [URL]) {
        for url in urls {
            Task(priority: .low) {
                _ = await load(url: url)
            }
        }
    }
    
    /// Cancel all prefetch operations
    public func cancelPrefetching(urls: [URL]) {
        for url in urls {
            let key = url.absoluteString
            cancelTask(for: key)
        }
    }
    
    /// Clear all caches
    public func clearCache() async {
        await memoryCache.clear()
        await diskCache.clear()
    }
    
    // MARK: - Private Implementation
    
    private func fetchImage(_ request: ImageRequest, key: String) async -> UIImage? {
        // 2. Check disk cache
        if let diskData = await diskCache.get(key) {
            if let image = await decoder.decode(data: diskData, targetSize: request.targetSize) {
                await memoryCache.set(key, image: image, data: diskData)
                return image
            }
        }
        
        // 3. Fetch from network
        do {
            let data = try await networkFetcher.fetch(url: request.url)
            
            guard let image = await decoder.decode(data: data, targetSize: request.targetSize) else {
                return nil
            }
            
            // Cache based on policy
            if request.options.cachePolicy != .none {
                if request.options.cachePolicy != .diskOnly {
                    await memoryCache.set(key, image: image, data: data)
                }
                if request.options.cachePolicy != .memoryOnly {
                    await diskCache.set(key, data: data)
                }
            }
            
            return image
        } catch {
            print("Image load failed: \(error)")
            return nil
        }
    }
    
    private func cacheKey(for request: ImageRequest) -> String {
        var key = request.url.absoluteString
        if let size = request.targetSize {
            key += "_\(Int(size.width))x\(Int(size.height))"
        }
        return key
    }
    
    private func cancelTask(for key: String) {
        taskLock.lock()
        activeTasks[key]?.cancel()
        activeTasks.removeValue(forKey: key)
        taskLock.unlock()
    }
    
    private func setupMemoryWarningObserver() {
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            Task {
                await self?.memoryCache.handleMemoryWarning()
            }
        }
    }
}
```

### Why Facade:

| Without Facade | With Facade |
|----------------|-------------|
| Client manages cache layers | Single `load()` call |
| Complex error handling | Simple optional return |
| Manual cache coordination | Automatic fallthrough |

---

## Pattern 2: Strategy (Decoding Strategies)

```swift
protocol ImageDecodingStrategy {
    func decode(data: Data, targetSize: CGSize?) async -> UIImage?
}

// Default decoding
class StandardDecodingStrategy: ImageDecodingStrategy {
    func decode(data: Data, targetSize: CGSize?) async -> UIImage? {
        guard let image = UIImage(data: data) else { return nil }
        
        if let targetSize = targetSize {
            return await downsample(image, to: targetSize)
        }
        return image
    }
    
    private func downsample(_ image: UIImage, to targetSize: CGSize) async -> UIImage {
        let renderer = UIGraphicsImageRenderer(size: targetSize)
        return renderer.image { _ in
            image.draw(in: CGRect(origin: .zero, size: targetSize))
        }
    }
}

// Memory-efficient decoding using ImageIO
class DownsamplingDecodingStrategy: ImageDecodingStrategy {
    func decode(data: Data, targetSize: CGSize?) async -> UIImage? {
        guard let targetSize = targetSize else {
            return UIImage(data: data)
        }
        
        let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
        guard let imageSource = CGImageSourceCreateWithData(data as CFData, imageSourceOptions) else {
            return nil
        }
        
        let maxDimension = max(targetSize.width, targetSize.height) * UIScreen.main.scale
        
        let downsampleOptions = [
            kCGImageSourceCreateThumbnailFromImageAlways: true,
            kCGImageSourceShouldCacheImmediately: true,
            kCGImageSourceCreateThumbnailWithTransform: true,
            kCGImageSourceThumbnailMaxPixelSize: maxDimension
        ] as CFDictionary
        
        guard let downsampledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else {
            return nil
        }
        
        return UIImage(cgImage: downsampledImage)
    }
}

// WebP decoding (requires libwebp)
class WebPDecodingStrategy: ImageDecodingStrategy {
    func decode(data: Data, targetSize: CGSize?) async -> UIImage? {
        // WebP decoding implementation
        // Would use libwebp or system decoder on iOS 14+
        return nil
    }
}

// Decoder using strategies
class ImageDecoder {
    private var strategies: [ImageDecodingStrategy] = [
        DownsamplingDecodingStrategy(),
        StandardDecodingStrategy()
    ]
    
    func decode(data: Data, targetSize: CGSize?) async -> UIImage? {
        for strategy in strategies {
            if let image = await strategy.decode(data: data, targetSize: targetSize) {
                return image
            }
        }
        return nil
    }
}
```

### Why Strategy:

- Different formats need different decoders
- Downsampling strategy saves memory
- Easy to add new format support

---

## Pattern 3: Observer (Loading State)

```swift
import Combine

class ImageLoadingViewModel: ObservableObject {
    @Published private(set) var state: ImageLoadingState = .idle
    @Published private(set) var image: UIImage?
    
    private var loadTask: Task<Void, Never>?
    private let imageLoader: ImageLoader
    
    init(imageLoader: ImageLoader = .shared) {
        self.imageLoader = imageLoader
    }
    
    func load(url: URL, targetSize: CGSize? = nil) {
        // Cancel previous load
        loadTask?.cancel()
        
        state = .loading(progress: 0)
        
        loadTask = Task { @MainActor in
            let request = ImageRequest(
                url: url,
                targetSize: targetSize,
                priority: .high,
                options: ImageOptions()
            )
            
            if let loadedImage = await imageLoader.load(request) {
                guard !Task.isCancelled else { return }
                self.image = loadedImage
                self.state = .success(loadedImage)
            } else {
                guard !Task.isCancelled else { return }
                self.state = .failure(.decodingFailed)
            }
        }
    }
    
    func cancel() {
        loadTask?.cancel()
        state = .idle
    }
    
    func retry(url: URL) {
        load(url: url)
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Image Load (Cache Miss → Network)

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  View  │ │ImageLoader │ │MemoryCache │ │ DiskCache  │ │  Network   │ │  Decoder   │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │              │
    │ load(url)  │              │              │              │              │
    │───────────▶│              │              │              │              │
    │            │              │              │              │              │
    │            │ get(key)     │              │              │              │
    │            │─────────────▶│              │              │              │
    │            │              │              │              │              │
    │            │   nil ❌     │              │              │              │
    │            │◀─────────────│              │              │              │
    │            │              │              │              │              │
    │            │ get(key)     │              │              │              │
    │            │─────────────────────────────▶│              │              │
    │            │              │              │              │              │
    │            │   nil ❌     │              │              │              │
    │            │◀─────────────────────────────│              │              │
    │            │              │              │              │              │
    │            │ fetch(url)   │              │              │              │
    │            │───────────────────────────────────────────▶│              │
    │            │              │              │              │              │
    │            │              │              │              │──┐           │
    │            │              │              │              │  │ HTTP GET  │
    │            │              │              │              │◀─┘           │
    │            │              │              │              │              │
    │            │   Data       │              │              │              │
    │            │◀──────────────────────────────────────────│              │
    │            │              │              │              │              │
    │            │ decode(data, size)          │              │              │
    │            │───────────────────────────────────────────────────────────▶
    │            │              │              │              │              │
    │            │              │              │              │              │
    │            │   UIImage    │              │              │              │
    │            │◀──────────────────────────────────────────────────────────│
    │            │              │              │              │              │
    │            │ set(key, image)             │              │              │
    │            │─────────────▶│              │              │              │
    │            │              │              │              │              │
    │            │ set(key, data)              │              │              │
    │            │─────────────────────────────▶│              │              │
    │            │              │              │              │              │
    │ UIImage ✅ │              │              │              │              │
    │◀───────────│              │              │              │              │
    │            │              │              │              │              │
```

## Sequence Diagram: Cell Reuse Cancellation

```
┌────────────┐ ┌────────────┐ ┌────────────┐
│    Cell    │ │  ViewModel │ │ImageLoader │
└─────┬──────┘ └─────┬──────┘ └─────┬──────┘
      │              │              │
      │ prepareForReuse()           │
      │─────────────▶│              │
      │              │              │
      │              │ cancel()     │
      │              │─────────────▶│
      │              │              │
      │              │              │ cancel task
      │              │              │──┐
      │              │              │◀─┘
      │              │              │
      │ configure(newURL)           │
      │─────────────▶│              │
      │              │              │
      │              │ load(newURL) │
      │              │─────────────▶│
      │              │              │
```

---

# E — Edge Cases

## Edge Case 1: Cell Reuse During Load

**Scenario:** User scrolls, cell reused before image loads

**Handling:**
```swift
class FeedImageCell: UICollectionViewCell {
    private var loadTask: Task<Void, Never>?
    private var currentURL: URL?
    
    override func prepareForReuse() {
        super.prepareForReuse()
        
        // Cancel in-flight request
        loadTask?.cancel()
        loadTask = nil
        currentURL = nil
        
        // Reset to placeholder
        imageView.image = nil
    }
    
    func configure(with url: URL) {
        currentURL = url
        
        loadTask = Task { @MainActor in
            if let image = await ImageLoader.shared.load(url: url) {
                // Verify URL still matches (cell not reused)
                guard currentURL == url else { return }
                
                UIView.transition(with: imageView, duration: 0.2, options: .transitionCrossDissolve) {
                    self.imageView.image = image
                }
            }
        }
    }
}
```

## Edge Case 2: Memory Warning

**Scenario:** System running low on memory

**Handling:**
```swift
class ImageLoader {
    private func setupMemoryWarningObserver() {
        NotificationCenter.default.addObserver(
            forName: UIApplication.didReceiveMemoryWarningNotification,
            object: nil,
            queue: .main
        ) { [weak self] _ in
            Task {
                // Clear memory cache entirely or partially
                await self?.memoryCache.handleMemoryWarning()
                
                // Log for monitoring
                Analytics.log("image_cache_memory_warning")
            }
        }
    }
}

actor MemoryCache {
    func handleMemoryWarning() {
        // Aggressive: clear all
        // Moderate: clear 75%
        // Conservative: clear 50%
        
        let targetCount = cache.count / 2
        while cache.count > targetCount, let oldest = accessOrder.first {
            evict(oldest)
        }
    }
}
```

## Edge Case 3: Same URL Requested Multiple Times

**Scenario:** Same image visible in multiple cells

**Handling:**
```swift
actor RequestCoalescer {
    private var inFlightRequests: [String: Task<UIImage?, Never>] = [:]
    
    func deduplicate(
        key: String,
        operation: @escaping () async -> UIImage?
    ) async -> UIImage? {
        // If already loading, wait for existing request
        if let existingTask = inFlightRequests[key] {
            return await existingTask.value
        }
        
        // Start new request
        let task = Task {
            await operation()
        }
        
        inFlightRequests[key] = task
        
        let result = await task.value
        
        // Remove after completion
        inFlightRequests.removeValue(forKey: key)
        
        return result
    }
}
```

## Edge Case 4: Large Images (Memory Explosion)

**Scenario:** Loading 4K images for small thumbnails

**Handling:**
```swift
class DownsamplingDecodingStrategy: ImageDecodingStrategy {
    func decode(data: Data, targetSize: CGSize?) async -> UIImage? {
        // Use ImageIO to decode at target size without loading full image
        let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
        
        guard let imageSource = CGImageSourceCreateWithData(data as CFData, imageSourceOptions) else {
            return nil
        }
        
        // Calculate max pixel size based on target + screen scale
        let scale = await UIScreen.main.scale
        let maxDimension = (targetSize?.width ?? 1000) * scale
        
        let downsampleOptions = [
            kCGImageSourceCreateThumbnailFromImageAlways: true,
            kCGImageSourceShouldCacheImmediately: true,
            kCGImageSourceThumbnailMaxPixelSize: maxDimension
        ] as CFDictionary
        
        guard let cgImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions) else {
            return nil
        }
        
        return UIImage(cgImage: cgImage)
    }
}
```

## Edge Case 5: App Backgrounded During Load

**Scenario:** User backgrounds app while images loading

**Handling:**
```swift
class ImageLoader {
    private var backgroundTask: UIBackgroundTaskIdentifier = .invalid
    
    func load(_ request: ImageRequest) async -> UIImage? {
        // Start background task for critical loads
        if request.priority >= .high {
            backgroundTask = await UIApplication.shared.beginBackgroundTask {
                // Handle expiration
                self.cancelAllTasks()
            }
        }
        
        defer {
            if backgroundTask != .invalid {
                Task { @MainActor in
                    UIApplication.shared.endBackgroundTask(self.backgroundTask)
                    self.backgroundTask = .invalid
                }
            }
        }
        
        return await fetchImage(request, key: cacheKey(for: request))
    }
}
```

## Edge Case Checklist

```markdown
□ Cell reused during load
□ Memory warning received
□ Same URL requested concurrently
□ Large image for small view (downsampling)
□ App backgrounded during load
□ Network failure mid-download
□ Corrupt image data
□ Cache full during write
□ Disk full
□ Invalid URL format
□ Slow network (show progress)
□ SSL certificate error
```

---

# D — Design Trade-offs

## Trade-off 1: Memory vs Disk Priority

| Memory First | Disk First |
|--------------|------------|
| Faster access | Survives app restart |
| Uses RAM | Uses storage |
| Lost on terminate | Persistent |

**Decision:** Check memory first, then disk, then network

**Rationale:** Memory is fastest, disk is persistent fallback

---

## Trade-off 2: Full vs Downsampled Cache

| Full Size | Downsampled |
|-----------|-------------|
| Flexible | Fixed size |
| More storage | Less storage |
| Decode every load | Pre-decoded |

**Decision:** Cache original data on disk, decoded size in memory

**Rationale:** Disk has more space for originals; memory needs ready-to-display

---

## Trade-off 3: LRU vs LFU Eviction

| LRU (Least Recently Used) | LFU (Least Frequently Used) |
|---------------------------|----------------------------|
| Recent wins | Popular wins |
| Simple | Complex |
| Good for feeds | Good for favorites |

**Decision:** LRU for both caches

**Rationale:** Feed images accessed once typically; LRU is simpler

---

## Trade-off 4: Sync vs Async Disk Access

| Synchronous | Asynchronous |
|-------------|--------------|
| Simpler code | Non-blocking |
| Blocks thread | Complex coordination |
| Predictable | Better for UI thread |

**Decision:** Async with Actor isolation

**Rationale:** Never block main thread for I/O

---

# iOS Implementation

## Complete SwiftUI Integration

```swift
// MARK: - SwiftUI View

struct AsyncCachedImage: View {
    let url: URL
    var targetSize: CGSize? = nil
    var placeholder: Image = Image(systemName: "photo")
    var failureImage: Image = Image(systemName: "exclamationmark.triangle")
    
    @StateObject private var viewModel = ImageLoadingViewModel()
    
    var body: some View {
        Group {
            switch viewModel.state {
            case .idle, .loading:
                placeholder
                    .resizable()
                    .aspectRatio(contentMode: .fill)
                    .redacted(reason: .placeholder)
                    .shimmering()
                
            case .success(let image):
                Image(uiImage: image)
                    .resizable()
                    .aspectRatio(contentMode: .fill)
                    .transition(.opacity)
                
            case .failure:
                failureImage
                    .resizable()
                    .aspectRatio(contentMode: .fit)
                    .foregroundColor(.gray)
                    .onTapGesture {
                        viewModel.retry(url: url)
                    }
            }
        }
        .onAppear {
            viewModel.load(url: url, targetSize: targetSize)
        }
        .onDisappear {
            viewModel.cancel()
        }
        .onChange(of: url) { newURL in
            viewModel.load(url: newURL, targetSize: targetSize)
        }
    }
}

// MARK: - Shimmer Effect

struct ShimmerModifier: ViewModifier {
    @State private var phase: CGFloat = 0
    
    func body(content: Content) -> some View {
        content
            .overlay(
                LinearGradient(
                    gradient: Gradient(colors: [.clear, .white.opacity(0.5), .clear]),
                    startPoint: .leading,
                    endPoint: .trailing
                )
                .offset(x: phase)
            )
            .onAppear {
                withAnimation(.linear(duration: 1.5).repeatForever(autoreverses: false)) {
                    phase = 300
                }
            }
    }
}

extension View {
    func shimmering() -> some View {
        modifier(ShimmerModifier())
    }
}
```

## UIKit Integration

```swift
// MARK: - UIImageView Extension

extension UIImageView {
    private static var taskKey: UInt8 = 0
    private static var urlKey: UInt8 = 1
    
    private var loadTask: Task<Void, Never>? {
        get { objc_getAssociatedObject(self, &Self.taskKey) as? Task<Void, Never> }
        set { objc_setAssociatedObject(self, &Self.taskKey, newValue, .OBJC_ASSOCIATION_RETAIN) }
    }
    
    private var currentURL: URL? {
        get { objc_getAssociatedObject(self, &Self.urlKey) as? URL }
        set { objc_setAssociatedObject(self, &Self.urlKey, newValue, .OBJC_ASSOCIATION_RETAIN) }
    }
    
    func loadImage(
        from url: URL,
        placeholder: UIImage? = nil,
        targetSize: CGSize? = nil
    ) {
        // Cancel previous
        loadTask?.cancel()
        
        // Set placeholder
        image = placeholder
        currentURL = url
        
        // Load
        loadTask = Task { @MainActor in
            let size = targetSize ?? bounds.size
            let request = ImageRequest(
                url: url,
                targetSize: size,
                priority: .high,
                options: ImageOptions()
            )
            
            if let loadedImage = await ImageLoader.shared.load(request) {
                guard currentURL == url else { return }  // Verify not reused
                
                UIView.transition(with: self, duration: 0.2, options: .transitionCrossDissolve) {
                    self.image = loadedImage
                }
            }
        }
    }
    
    func cancelImageLoad() {
        loadTask?.cancel()
        loadTask = nil
        currentURL = nil
    }
}

// MARK: - Collection View Prefetching

class ImagePrefetcher: NSObject, UICollectionViewDataSourcePrefetching {
    private var prefetchTasks: [URL: Task<Void, Never>] = [:]
    private let imageLoader: ImageLoader
    
    init(imageLoader: ImageLoader = .shared) {
        self.imageLoader = imageLoader
    }
    
    func prefetch(urls: [URL]) {
        for url in urls where prefetchTasks[url] == nil {
            prefetchTasks[url] = Task(priority: .low) {
                _ = await imageLoader.load(url: url)
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
    
    // UICollectionViewDataSourcePrefetching
    func collectionView(
        _ collectionView: UICollectionView,
        prefetchItemsAt indexPaths: [IndexPath]
    ) {
        // Subclass should override to provide URLs
    }
    
    func collectionView(
        _ collectionView: UICollectionView,
        cancelPrefetchingForItemsAt indexPaths: [IndexPath]
    ) {
        // Subclass should override to provide URLs
    }
}
```

---

# Interview Tips

## What to Say

```markdown
1. "I'll use a two-level cache - memory for speed, disk for persistence..."

2. "The facade pattern hides the complexity of cache coordination..."

3. "Request deduplication prevents multiple network calls for same URL..."

4. "I use ImageIO for downsampling to avoid memory spikes..."

5. "Cell reuse requires careful cancellation to avoid wrong images..."

6. "Memory warning handling is critical for iOS..."
```

## Red Flags to Avoid

```markdown
❌ "I'll just use URLSession's built-in caching"
   → No memory cache, no control

❌ "I'll load full size images and resize in UIImageView"
   → Memory explosion

❌ "I don't need to handle prepareForReuse"
   → Wrong images will appear

❌ "NSCache handles everything"
   → No disk persistence, less control

❌ Forgetting request deduplication
   → Wasteful duplicate downloads
```

## Common Follow-up Questions

| Question | Key Points |
|----------|------------|
| "How do you handle memory warnings?" | Clear/reduce memory cache |
| "What about animated GIFs?" | Different caching strategy |
| "How do you measure cache hit rate?" | Analytics logging |
| "What if disk is full?" | Graceful fallback, no crash |
| "How do you handle 404s?" | Cache negative result briefly |

---

*This is how an Amazon iOS LLD interview expects you to approach the Feed Image Loader problem!*
