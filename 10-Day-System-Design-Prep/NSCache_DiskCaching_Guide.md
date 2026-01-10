# NSCache & Disk Caching: Complete iOS Guide

## 📚 Table of Contents

1. [NSCache Deep Dive](#nscache-deep-dive)
   - Beginner: What is NSCache?
   - Intermediate: Features & Configuration
   - Advanced: Production Implementation
2. [Disk Caching Deep Dive](#disk-caching-deep-dive)
   - Beginner: File-based Caching
   - Intermediate: Cache Management
   - Advanced: LRU Disk Cache
3. [Two-Level Caching](#two-level-caching)
4. [Real-World Examples](#real-world-examples)
5. [Interview Questions](#interview-questions)

---

## 🎯 NSCache Deep Dive

### 🎈 BEGINNER: What is NSCache?

**Simple Analogy:**

Think of NSCache like a smart refrigerator:
- It automatically throws away food when it's running out of space
- It removes the oldest or least-used items first
- It knows when the kitchen (your app) is running out of memory
- Multiple family members (threads) can access it safely at the same time

**Why NSCache exists:**

Before NSCache, developers used `Dictionary` for caching. But dictionaries have problems:
- ❌ They never release memory automatically
- ❌ They're not thread-safe
- ❌ They can cause memory crashes
- ❌ They don't respond to memory warnings

NSCache solves all these problems! ✅

**Basic NSCache Example:**

```swift
import UIKit

// Simple image cache using NSCache
class BasicImageCache {
    // NSCache stores NSString keys and UIImage values
    private let cache = NSCache<NSString, UIImage>()
    
    init() {
        // Set maximum number of items
        cache.countLimit = 50  // Store max 50 images
        
        print("✅ Image cache initialized with limit of 50 images")
    }
    
    // Store an image in cache
    func cacheImage(_ image: UIImage, forKey key: String) {
        cache.setObject(image, forKey: key as NSString)
        print("📦 Cached image: \(key)")
    }
    
    // Retrieve an image from cache
    func getImage(forKey key: String) -> UIImage? {
        if let image = cache.object(forKey: key as NSString) {
            print("🎯 Cache HIT: \(key)")
            return image
        } else {
            print("❌ Cache MISS: \(key)")
            return nil
        }
    }
    
    // Clear all cached images
    func clearCache() {
        cache.removeAllObjects()
        print("🗑️ Cache cleared")
    }
}

// Usage
let imageCache = BasicImageCache()

// Cache profile picture
if let profileImage = UIImage(named: "avatar") {
    imageCache.cacheImage(profileImage, forKey: "user_123_avatar")
}

// Retrieve it later (instant!)
if let cachedImage = imageCache.getImage(forKey: "user_123_avatar") {
    // Use image - no need to reload from disk or network
    imageView.image = cachedImage
}
```

**Key Concepts:**

1. **Generic Type**: `NSCache<KeyType, ObjectType>` where both must be classes
2. **No Subscripting**: Must use `setObject()` and `object(forKey:)`, not `cache[key]`
3. **Automatic Cleanup**: iOS clears cache when memory is low
4. **Thread-Safe**: Can be accessed from multiple threads safely

***

### 🚀 INTERMEDIATE: NSCache Features & Configuration

**NSCache has 3 main limits you can configure:**

```swift
class ConfiguredImageCache {
    private let cache = NSCache<NSString, UIImage>()
    
    init() {
        // LIMIT 1: Maximum number of objects
        cache.countLimit = 100  // Max 100 images
        
        // LIMIT 2: Maximum total cost (in bytes)
        cache.totalCostLimit = 50 * 1024 * 1024  // 50 MB
        
        // LIMIT 3: Eviction policy (handled automatically by NSCache)
        // NSCache uses LRU-like algorithm internally
        
        // Optional: Name for debugging
        cache.name = "com.myapp.imageCache"
        
        // Listen to memory warnings
        setupMemoryWarningObserver()
    }
    
    private func setupMemoryWarningObserver() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleMemoryWarning),
            name: UIApplication.didReceiveMemoryWarningNotification,
            object: nil
        )
    }
    
    @objc private func handleMemoryWarning() {
        // NSCache already auto-evicts, but we can help
        cache.removeAllObjects()
        print("⚠️ Memory warning received - Cache cleared")
    }
    
    // IMPORTANT: Store with cost for better eviction decisions
    func cacheImage(_ image: UIImage, forKey key: String) {
        // Calculate cost (image size in bytes)
        let cost = calculateImageCost(image)
        
        // Store with cost - helps NSCache decide what to evict
        cache.setObject(image, forKey: key as NSString, cost: cost)
        
        print("📦 Cached '\(key)' with cost: \(cost / 1024)KB")
    }
    
    private func calculateImageCost(_ image: UIImage) -> Int {
        // Estimate memory footprint
        let pixelCount = Int(image.size.width * image.size.height)
        let bytesPerPixel = 4  // RGBA
        return pixelCount * bytesPerPixel
    }
    
    func getImage(forKey key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
    
    // Remove specific image
    func removeImage(forKey key: String) {
        cache.removeObject(forKey: key as NSString)
    }
}
```

**Understanding Cost:**

The `cost` parameter is crucial for smart eviction:

```swift
/*
WITHOUT COST (bad):
- Cache has: 50MB limit
- Image 1: 100KB (no cost specified)
- Image 2: 10MB (no cost specified)
- NSCache counts: 2 images = fine (under countLimit)
- PROBLEM: Actually using 10.1MB but NSCache doesn't know!

WITH COST (good):
- Cache has: 50MB limit
- Image 1: 100KB (cost: 100KB)
- Image 2: 10MB (cost: 10MB)
- NSCache tracks: 10.1MB total
- When adding more images, NSCache evicts based on totalCostLimit
*/
```

**NSCache vs Dictionary Comparison:**

```swift
// ❌ BAD: Using Dictionary
class BadImageCache {
    private var cache: [String: UIImage] = [:]
    
    func cache(_ image: UIImage, key: String) {
        cache[key] = image  // Grows forever! Memory crash!
    }
}

// ✅ GOOD: Using NSCache
class GoodImageCache {
    private let cache = NSCache<NSString, UIImage>()
    
    func cache(_ image: UIImage, key: String) {
        cache.setObject(image, forKey: key as NSString)
        // Auto-evicts when memory is low ✅
        // Thread-safe ✅
        // Has size limits ✅
    }
}
```

**Real-World Usage Pattern:**

```swift
class UserProfileImageCache {
    static let shared = UserProfileImageCache()
    private let cache = NSCache<NSString, UIImage>()
    
    private init() {
        // Configure for profile images (typically small)
        cache.countLimit = 200  // 200 user avatars
        cache.totalCostLimit = 20 * 1024 * 1024  // 20MB max
        cache.name = "UserProfileImageCache"
    }
    
    func loadProfileImage(userId: String, completion: @escaping (UIImage?) -> Void) {
        let cacheKey = "profile_\(userId)"
        
        // 1. Check cache first
        if let cachedImage = cache.object(forKey: cacheKey as NSString) {
            print("🎯 Loaded from cache: \(userId)")
            completion(cachedImage)
            return
        }
        
        // 2. Not in cache - download from network
        print("📡 Downloading profile image: \(userId)")
        downloadProfileImage(userId: userId) { [weak self] image in
            guard let self = self, let image = image else {
                completion(nil)
                return
            }
            
            // 3. Cache the downloaded image
            let cost = self.estimateImageSize(image)
            self.cache.setObject(image, forKey: cacheKey as NSString, cost: cost)
            
            completion(image)
        }
    }
    
    private func downloadProfileImage(userId: String, completion: @escaping (UIImage?) -> Void) {
        // Network download implementation
        // ...
    }
    
    private func estimateImageSize(_ image: UIImage) -> Int {
        return Int(image.size.width * image.size.height * 4)
    }
}

// Usage in a UITableViewCell
class UserCell: UITableViewCell {
    @IBOutlet weak var avatarImageView: UIImageView!
    
    func configure(userId: String) {
        UserProfileImageCache.shared.loadProfileImage(userId: userId) { [weak self] image in
            DispatchQueue.main.async {
                self?.avatarImageView.image = image
            }
        }
    }
}
```

***

### 💎 ADVANCED: Production NSCache Implementation

**Enterprise-Grade Image Cache with All Best Practices:**

```swift
import UIKit

/// Production-ready image cache with NSCache
/// Features:
/// - Thread-safe image caching
/// - Automatic memory management
/// - Cost-based eviction
/// - Memory warning handling
/// - Task cancellation support
/// - Metrics tracking
actor ProductionImageCache {
    // Singleton instance
    static let shared = ProductionImageCache()
    
    // NSCache for memory storage
    private let cache = NSCache<NSString, UIImage>()
    
    // Track ongoing downloads to avoid duplicates
    private var ongoingTasks: [String: Task<UIImage?, Error>] = [:]
    
    // Metrics
    private var cacheHits: Int = 0
    private var cacheMisses: Int = 0
    
    // Configuration
    struct Configuration {
        let maxMemoryCount: Int
        let maxMemoryCost: Int  // in bytes
        let imageCompressionQuality: CGFloat
        
        static let `default` = Configuration(
            maxMemoryCount: 100,
            maxMemoryCost: 50 * 1024 * 1024,  // 50MB
            imageCompressionQuality: 0.8
        )
    }
    
    private let config: Configuration
    
    private init(config: Configuration = .default) {
        self.config = config
        
        // Configure NSCache
        cache.countLimit = config.maxMemoryCount
        cache.totalCostLimit = config.maxMemoryCost
        cache.name = "ProductionImageCache"
        
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
        
        print("✅ Image cache initialized")
        print("   Max count: \(config.maxMemoryCount)")
        print("   Max memory: \(config.maxMemoryCost / 1024 / 1024)MB")
    }
    
    // MARK: - Public API
    
    /// Load image from cache or download
    func loadImage(from url: URL, priority: TaskPriority = .medium) async throws -> UIImage {
        let cacheKey = url.absoluteString
        
        // Check cache first
        if let cachedImage = getFromCache(key: cacheKey) {
            cacheHits += 1
            print("🎯 Cache HIT (\(cacheHits)/\(cacheHits + cacheMisses)): \(url.lastPathComponent)")
            return cachedImage
        }
        
        cacheMisses += 1
        print("❌ Cache MISS (\(cacheHits)/\(cacheHits + cacheMisses)): \(url.lastPathComponent)")
        
        // Check if already downloading
        if let existingTask = ongoingTasks[cacheKey] {
            print("⏳ Joining existing download: \(url.lastPathComponent)")
            return try await existingTask.value ?? UIImage()
        }
        
        // Start new download
        let downloadTask = Task(priority: priority) {
            try await self.downloadAndCache(url: url, key: cacheKey)
        }
        
        ongoingTasks[cacheKey] = downloadTask
        
        do {
            let image = try await downloadTask.value
            ongoingTasks.removeValue(forKey: cacheKey)
            return image ?? UIImage()
        } catch {
            ongoingTasks.removeValue(forKey: cacheKey)
            throw error
        }
    }
    
    /// Prefetch images (low priority background task)
    func prefetchImages(urls: [URL]) async {
        await withTaskGroup(of: Void.self) { group in
            for url in urls {
                group.addTask(priority: .low) {
                    _ = try? await self.loadImage(from: url, priority: .low)
                }
            }
        }
    }
    
    /// Get cache statistics
    func getStatistics() -> CacheStatistics {
        let hitRate = cacheHits + cacheMisses > 0
            ? Double(cacheHits) / Double(cacheHits + cacheMisses)
            : 0.0
        
        return CacheStatistics(
            hits: cacheHits,
            misses: cacheMisses,
            hitRate: hitRate
        )
    }
    
    /// Clear all cached images
    func clearCache() {
        cache.removeAllObjects()
        cacheHits = 0
        cacheMisses = 0
        print("🗑️ Cache cleared")
    }
    
    // MARK: - Private Methods
    
    private func getFromCache(key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
    
    private func downloadAndCache(url: URL, key: String) async throws -> UIImage? {
        print("📡 Downloading: \(url.lastPathComponent)")
        
        let (data, _) = try await URLSession.shared.data(from: url)
        
        guard let image = UIImage(data: data) else {
            throw CacheError.invalidImageData
        }
        
        // Cache the image with cost
        let cost = estimateMemoryCost(for: image)
        cache.setObject(image, forKey: key as NSString, cost: cost)
        
        print("✅ Downloaded & cached: \(url.lastPathComponent) (\(cost / 1024)KB)")
        
        return image
    }
    
    private func estimateMemoryCost(for image: UIImage) -> Int {
        let width = Int(image.size.width * image.scale)
        let height = Int(image.size.height * image.scale)
        let bytesPerPixel = 4  // RGBA
        return width * height * bytesPerPixel
    }
    
    private func handleMemoryWarning() {
        print("⚠️ Memory warning - Clearing image cache")
        cache.removeAllObjects()
        
        // Reset statistics
        cacheHits = 0
        cacheMisses = 0
    }
    
    // MARK: - Supporting Types
    
    struct CacheStatistics {
        let hits: Int
        let misses: Int
        let hitRate: Double
        
        var description: String {
            """
            Cache Statistics:
            - Hits: \(hits)
            - Misses: \(misses)
            - Hit Rate: \(String(format: "%.1f%%", hitRate * 100))
            """
        }
    }
    
    enum CacheError: Error {
        case invalidImageData
        case downloadFailed
    }
}

// MARK: - UIImageView Extension for Easy Loading

extension UIImageView {
    private static var loadTaskKey: UInt8 = 0
    
    /// Load image from URL with caching
    func loadImage(from url: URL, placeholder: UIImage? = nil) {
        // Cancel previous load task
        cancelImageLoad()
        
        // Show placeholder immediately
        self.image = placeholder
        
        // Start new load task
        let task = Task {
            do {
                let image = try await ProductionImageCache.shared.loadImage(from: url)
                
                guard !Task.isCancelled else { return }
                
                await MainActor.run {
                    UIView.transition(
                        with: self,
                        duration: 0.3,
                        options: .transitionCrossDissolve
                    ) {
                        self.image = image
                    }
                }
            } catch {
                print("Failed to load image: \(error)")
            }
        }
        
        // Store task for cancellation
        objc_setAssociatedObject(self, &Self.loadTaskKey, task, .OBJC_ASSOCIATION_RETAIN)
    }
    
    /// Cancel ongoing image load
    func cancelImageLoad() {
        if let task = objc_getAssociatedObject(self, &Self.loadTaskKey) as? Task<Void, Never> {
            task.cancel()
        }
    }
}

// MARK: - Usage in UITableViewCell

class ProductCell: UITableViewCell {
    @IBOutlet weak var productImageView: UIImageView!
    
    func configure(imageURL: URL) {
        // Simple one-liner with automatic caching!
        productImageView.loadImage(
            from: imageURL,
            placeholder: UIImage(named: "placeholder")
        )
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        // Cancel ongoing loads when cell is reused
        productImageView.cancelImageLoad()
    }
}
```

---

## 💾 Disk Caching Deep Dive

### 🎈 BEGINNER: File-based Caching

**Why Disk Caching?**

NSCache is great but has one problem: **It's lost when the app closes!**

Think of it like this:
- **NSCache** = Post-it notes (fast but temporary)
- **Disk Cache** = Notebook (permanent storage)

**Where to Store Cache Files:**

iOS provides special directories:

```swift
class CacheDirectories {
    static func printDirectories() {
        let fileManager = FileManager.default
        
        // 1. CACHES DIRECTORY (recommended for cache)
        if let cachesDir = fileManager.urls(for: .cachesDirectory, in: .userDomainMask).first {
            print("📁 Caches: \(cachesDir.path)")
            print("   ✅ System can delete when storage is low")
            print("   ✅ Not backed up to iCloud")
            print("   ✅ PERFECT for cache!")
        }
        
        // 2. DOCUMENTS DIRECTORY (NOT for cache!)
        if let docsDir = fileManager.urls(for: .documentDirectory, in: .userDomainMask).first {
            print("📁 Documents: \(docsDir.path)")
            print("   ❌ Backed up to iCloud")
            print("   ❌ DON'T use for cache")
        }
        
        // 3. TEMPORARY DIRECTORY
        let tempDir = fileManager.temporaryDirectory
        print("📁 Temp: \(tempDir.path)")
        print("   ⚠️ Deleted when app closes")
        print("   ⚠️ Use only for very temporary files")
    }
}
```

**Simple Disk Cache:**

```swift
import Foundation

class SimpleDiskCache {
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    
    init(cacheName: String = "ImageCache") {
        // Get Caches directory
        let cachesDir = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        
        // Create subdirectory for our cache
        self.cacheDirectory = cachesDir.appendingPathComponent(cacheName)
        
        // Create directory if it doesn't exist
        createCacheDirectoryIfNeeded()
        
        print("📁 Disk cache ready: \(cacheDirectory.path)")
    }
    
    private func createCacheDirectoryIfNeeded() {
        if !fileManager.fileExists(atPath: cacheDirectory.path) {
            try? fileManager.createDirectory(
                at: cacheDirectory,
                withIntermediateDirectories: true,
                attributes: nil
            )
        }
    }
    
    // MARK: - Save to Disk
    
    func save(_ data: Data, forKey key: String) {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        
        do {
            try data.write(to: fileURL)
            print("✅ Saved to disk: \(key) (\(data.count / 1024)KB)")
        } catch {
            print("❌ Failed to save: \(error)")
        }
    }
    
    func saveImage(_ image: UIImage, forKey key: String, compressionQuality: CGFloat = 0.8) {
        guard let data = image.jpegData(compressionQuality: compressionQuality) else {
            print("❌ Failed to convert image to data")
            return
        }
        
        save(data, forKey: key)
    }
    
    // MARK: - Load from Disk
    
    func load(forKey key: String) -> Data? {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        
        do {
            let data = try Data(contentsOf: fileURL)
            print("✅ Loaded from disk: \(key) (\(data.count / 1024)KB)")
            return data
        } catch {
            print("❌ File not found: \(key)")
            return nil
        }
    }
    
    func loadImage(forKey key: String) -> UIImage? {
        guard let data = load(forKey: key) else {
            return nil
        }
        
        return UIImage(data: data)
    }
    
    // MARK: - Delete
    
    func delete(forKey key: String) {
        let fileURL = cacheDirectory.appendingPathComponent(key)
        
        try? fileManager.removeItem(at: fileURL)
        print("🗑️ Deleted: \(key)")
    }
    
    // MARK: - Clear All
    
    func clearAll() {
        guard let files = try? fileManager.contentsOfDirectory(
            at: cacheDirectory,
            includingPropertiesForKeys: nil
        ) else {
            return
        }
        
        for fileURL in files {
            try? fileManager.removeItem(at: fileURL)
        }
        
        print("🗑️ Cleared disk cache (\(files.count) files)")
    }
    
    // MARK: - Info
    
    func getTotalSize() -> Int {
        guard let files = try? fileManager.contentsOfDirectory(
            at: cacheDirectory,
            includingPropertiesForKeys: [.fileSizeKey]
        ) else {
            return 0
        }
        
        let totalSize = files.reduce(0) { total, fileURL in
            let fileSize = (try? fileURL.resourceValues(forKeys: [.fileSizeKey]))?.fileSize ?? 0
            return total + fileSize
        }
        
        print("📊 Cache size: \(totalSize / 1024 / 1024)MB")
        return totalSize
    }
}

// MARK: - Usage Example

let diskCache = SimpleDiskCache()

// Save image
if let image = UIImage(named: "photo") {
    diskCache.saveImage(image, forKey: "photo_123")
}

// Load image later (even after app restart!)
if let cachedImage = diskCache.loadImage(forKey: "photo_123") {
    imageView.image = cachedImage
}

// Check cache size
diskCache.getTotalSize()

// Clear old data
diskCache.clearAll()
```

***

### 🚀 INTERMEDIATE: Disk Cache Management

**Problems with Simple Disk Cache:**

1. **No size limit** - Can fill up device storage
2. **No expiration** - Old data stays forever
3. **No cleanup** - Never removes unused files
4. **Poor organization** - All files in one directory

**Improved Disk Cache with Management:**

```swift
import Foundation
import CryptoKit

class ManagedDiskCache {
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    private let maxCacheSize: Int  // in bytes
    private let maxCacheAge: TimeInterval  // in seconds
    
    // Metadata tracking
    private var metadata: [String: CacheMetadata] = [:]
    private let metadataQueue = DispatchQueue(label: "diskCache.metadata")
    
    struct CacheMetadata: Codable {
        let key: String
        let fileName: String
        let size: Int
        let createdAt: Date
        var lastAccessedAt: Date
        var accessCount: Int
    }
    
    init(
        cacheName: String = "ManagedCache",
        maxSizeMB: Int = 100,
        maxAgeDays: Int = 7
    ) {
        // Setup cache directory
        let cachesDir = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        self.cacheDirectory = cachesDir.appendingPathComponent(cacheName)
        
        // Convert to bytes and seconds
        self.maxCacheSize = maxSizeMB * 1024 * 1024
        self.maxCacheAge = TimeInterval(maxAgeDays * 24 * 60 * 60)
        
        // Create directory
        try? fileManager.createDirectory(
            at: cacheDirectory,
            withIntermediateDirectories: true
        )
        
        // Load metadata
        loadMetadata()
        
        // Initial cleanup
        cleanupIfNeeded()
        
        print("✅ Managed disk cache initialized")
        print("   Max size: \(maxSizeMB)MB")
        print("   Max age: \(maxAgeDays) days")
    }
    
    // MARK: - Save
    
    func save(_ data: Data, forKey key: String) {
        metadataQueue.sync {
            // Generate safe filename (MD5 hash)
            let fileName = key.md5Hash
            let fileURL = cacheDirectory.appendingPathComponent(fileName)
            
            // Write data
            do {
                try data.write(to: fileURL)
                
                // Update metadata
                metadata[key] = CacheMetadata(
                    key: key,
                    fileName: fileName,
                    size: data.count,
                    createdAt: Date(),
                    lastAccessedAt: Date(),
                    accessCount: 0
                )
                
                saveMetadata()
                
                // Cleanup if over limit
                cleanupIfNeeded()
                
                print("✅ Saved: \(key) (\(data.count / 1024)KB)")
            } catch {
                print("❌ Save failed: \(error)")
            }
        }
    }
    
    // MARK: - Load
    
    func load(forKey key: String) -> Data? {
        return metadataQueue.sync {
            guard let meta = metadata[key] else {
                print("❌ Key not found: \(key)")
                return nil
            }
            
            // Check if expired
            let age = Date().timeIntervalSince(meta.createdAt)
            if age > maxCacheAge {
                print("⏰ Expired: \(key) (age: \(Int(age / 86400)) days)")
                delete(forKey: key)
                return nil
            }
            
            // Load from disk
            let fileURL = cacheDirectory.appendingPathComponent(meta.fileName)
            guard let data = try? Data(contentsOf: fileURL) else {
                print("❌ File missing: \(key)")
                metadata.removeValue(forKey: key)
                return nil
            }
            
            // Update access metadata
            metadata[key]?.lastAccessedAt = Date()
            metadata[key]?.accessCount += 1
            
            print("✅ Loaded: \(key) (\(data.count / 1024)KB, accessed \(meta.accessCount + 1) times)")
            
            return data
        }
    }
    
    // MARK: - Delete
    
    func delete(forKey key: String) {
        metadataQueue.sync {
            guard let meta = metadata[key] else { return }
            
            let fileURL = cacheDirectory.appendingPathComponent(meta.fileName)
            try? fileManager.removeItem(at: fileURL)
            
            metadata.removeValue(forKey: key)
            saveMetadata()
            
            print("🗑️ Deleted: \(key)")
        }
    }
    
    // MARK: - Cleanup
    
    private func cleanupIfNeeded() {
        let totalSize = metadata.values.reduce(0) { $0 + $1.size }
        
        guard totalSize > maxCacheSize else {
            return
        }
        
        print("⚠️ Cache too large (\(totalSize / 1024 / 1024)MB > \(maxCacheSize / 1024 / 1024)MB)")
        print("🧹 Starting cleanup...")
        
        // Sort by last access (LRU)
        let sorted = metadata.values.sorted { $0.lastAccessedAt < $1.lastAccessedAt }
        
        var currentSize = totalSize
        var deletedCount = 0
        
        for meta in sorted {
            guard currentSize > maxCacheSize else { break }
            
            // Delete file
            let fileURL = cacheDirectory.appendingPathComponent(meta.fileName)
            try? fileManager.removeItem(at: fileURL)
            
            // Update metadata
            metadata.removeValue(forKey: meta.key)
            currentSize -= meta.size
            deletedCount += 1
        }
        
        saveMetadata()
        
        print("✅ Cleanup complete: Deleted \(deletedCount) files")
        print("   New size: \(currentSize / 1024 / 1024)MB")
    }
    
    func clearExpired() {
        metadataQueue.sync {
            let now = Date()
            var deletedCount = 0
            
            for (key, meta) in metadata {
                let age = now.timeIntervalSince(meta.createdAt)
                if age > maxCacheAge {
                    let fileURL = cacheDirectory.appendingPathComponent(meta.fileName)
                    try? fileManager.removeItem(at: fileURL)
                    metadata.removeValue(forKey: key)
                    deletedCount += 1
                }
            }
            
            if deletedCount > 0 {
                saveMetadata()
                print("🧹 Cleared \(deletedCount) expired files")
            }
        }
    }
    
    // MARK: - Metadata Persistence
    
    private func loadMetadata() {
        let metadataURL = cacheDirectory.appendingPathComponent("metadata.json")
        
        guard let data = try? Data(contentsOf: metadataURL),
              let decoded = try? JSONDecoder().decode([String: CacheMetadata].self, from: data) else {
            return
        }
        
        metadata = decoded
        print("📖 Loaded metadata: \(metadata.count) entries")
    }
    
    private func saveMetadata() {
        let metadataURL = cacheDirectory.appendingPathComponent("metadata.json")
        
        guard let encoded = try? JSONEncoder().encode(metadata) else {
            return
        }
        
        try? encoded.write(to: metadataURL)
    }
    
    // MARK: - Info
    
    func getStatistics() -> String {
        let totalSize = metadata.values.reduce(0) { $0 + $1.size }
        let fileCount = metadata.count
        
        return """
        📊 Disk Cache Statistics:
        - Files: \(fileCount)
        - Size: \(totalSize / 1024 / 1024)MB / \(maxCacheSize / 1024 / 1024)MB
        - Usage: \(String(format: "%.1f%%", Double(totalSize) / Double(maxCacheSize) * 100))
        """
    }
}

extension String {
    var md5Hash: String {
        let data = Data(self.utf8)
        let hash = Insecure.MD5.hash(data: data)
        return hash.map { String(format: "%02x", $0) }.joined()
    }
}
```

***

## 🔗 Two-Level Caching

**The Best of Both Worlds:**

Combine NSCache (fast, memory) + Disk Cache (persistent):

```swift
actor TwoLevelCache {
    // Level 1: Memory (fast)
    private let memoryCache = NSCache<NSString, UIImage>()
    
    // Level 2: Disk (persistent)
    private let diskCache: ManagedDiskCache
    
    init() {
        // Configure memory cache
        memoryCache.countLimit = 50
        memoryCache.totalCostLimit = 30 * 1024 * 1024  // 30MB
        
        // Configure disk cache
        diskCache = ManagedDiskCache(
            cacheName: "TwoLevelImageCache",
            maxSizeMB: 100,
            maxAgeDays: 7
        )
        
        print("✅ Two-level cache initialized")
    }
    
    // MARK: - Get Image
    
    func getImage(forKey key: String) async -> UIImage? {
        // L1: Check memory (fastest - microseconds)
        if let memoryImage = memoryCache.object(forKey: key as NSString) {
            print("🎯 L1 HIT (memory): \(key)")
            return memoryImage
        }
        
        // L2: Check disk (fast - milliseconds)
        if let diskData = diskCache.load(forKey: key),
           let diskImage = UIImage(data: diskData) {
            print("🎯 L2 HIT (disk): \(key)")
            
            // Promote to memory for next time
            let cost = estimateCost(for: diskImage)
            memoryCache.setObject(diskImage, forKey: key as NSString, cost: cost)
            
            return diskImage
        }
        
        print("❌ MISS (both levels): \(key)")
        return nil
    }
    
    // MARK: - Cache Image
    
    func cacheImage(_ image: UIImage, forKey key: String) {
        // Save to memory (instant access)
        let cost = estimateCost(for: image)
        memoryCache.setObject(image, forKey: key as NSString, cost: cost)
        
        // Save to disk (persistent)
        if let data = image.jpegData(compressionQuality: 0.8) {
            diskCache.save(data, forKey: key)
        }
        
        print("✅ Cached in both levels: \(key)")
    }
    
    // MARK: - Helper
    
    private func estimateCost(for image: UIImage) -> Int {
        return Int(image.size.width * image.size.height * 4)
    }
}

// Performance comparison:
/*
Network:       1000ms  (1 second)
Disk (L2):     50ms    (20x faster than network)
Memory (L1):   1ms     (1000x faster than network!)
*/
```

---

## 📝 Interview Questions

### Q1: Explain NSCache vs Dictionary. When would you use each?

**Expected Answer:**

"NSCache and Dictionary both store key-value pairs, but they serve different purposes:

**NSCache:**
- ✅ Thread-safe automatically
- ✅ Auto-evicts under memory pressure
- ✅ Has size/cost limits
- ❌ Can't be serialized (Codable)
- ❌ Items can disappear
- **Use for:** Disposable cached data like images, computed results

**Dictionary:**
- ✅ Full control over data
- ✅ Can be serialized
- ✅ Subscript syntax [key]
- ❌ Not thread-safe
- ❌ No automatic cleanup
- ❌ Can cause memory leaks
- **Use for:** Configuration, critical app state

**Example mistake:**
Never use NSCache for critical data like auth tokens - they might get evicted and user will be logged out unexpectedly!"

### Q2: How do you implement cache eviction in disk cache?

**Expected Answer:**

"I use LRU (Least Recently Used) with metadata tracking:

```swift
// Track metadata for each cached file
struct CacheMetadata {
    let createdAt: Date
    var lastAccessedAt: Date
    let size: Int
}

// When cache exceeds max size:
1. Sort entries by lastAccessedAt
2. Delete oldest until under limit
3. Update metadata
```

I also implement TTL (Time To Live):
- Check age on load: if `Date() - createdAt > maxAge`, delete it
- Run periodic cleanup to remove expired files"

### Q3: Design caching for Instagram feed

**Expected Answer:**

"Three-level approach:

**L1: NSCache (50MB, ~50 images)**
- Recent feed images
- Instant scrolling performance
- Auto-evicts on memory warning

**L2: Disk (200MB, ~500 images)**
- Persistent across app restarts
- LRU eviction when full
- Organized by feed position

**L3: Network (CDN)**
- Images we haven't seen yet
- Prefetch next 10 posts as user scrolls

**Prefetching strategy:**
- When user scrolls to row N, prefetch rows N+1 to N+10
- Use low priority background tasks
- Cancel prefetch if user scrolls away"

---

This comprehensive guide covers NSCache and Disk Caching from beginner to advanced levels, with production-ready code examples! 🚀
