# Feed with Pagination & Caching - Complete iOS System Design

> [!NOTE]
> This is a 60-minute Uber L4 interview simulation following the **[SCALED framework](./00_SCALED_Framework_Guide.md)**.

---

## 🎯 SCALED Framework Coverage

This design follows the SCALED principles:

| SCALED | Section in This File | What You'll Learn |
|--------|----------------------|-------------------|
| **S** - System Requirements | [0-10 min: Requirements](#0-10-min-requirements-clarification) | Feed type, pagination requirements, offline expectations |
| **C** - Design Considerations | [10-25 min: HLD](#10-25-min-high-level-design-hld) | Cursor pagination choice, image loading strategy |
| **A** - Architecture | [10-25 min: HLD](#10-25-min-high-level-design-hld) | MVVM + Separate ImageLoader, prefetching design |
| **L** - Low-Level Design | [25-45 min: LLD](#25-45-min-low-level-design-lld) | FeedViewModel, ImageLoader, LRU cache, prefetching |
| **E** - Evaluating NFRs | [45-55 min: Deep Dives](#45-55-min-deep-dives) | 60 FPS scrolling, memory limits, image optimization |
| **D** - API Design | [Part 4: Feed API Design](#part-4-feed-api-design-cursor-based-pagination) | Cursor pagination, field selection, rate limiting, real-time |
| **T** - Trade-offs | Throughout + [Deep Dives](#45-55-min-deep-dives) | Cursor vs offset, memory vs disk cache, polling vs WebSocket |

> [!TIP]
> Feed pagination is extremely common at Meta, Instagram, Twitter. Master cursor-based pagination here!

---

## 0-10 min: Requirements Clarification

### Questions to Ask Interviewer

**You:** "Let me clarify the feed requirements. First, what type of content?
- Text posts only, or with images/videos?
- User-generated or algorithmically ranked?
- Real-time updates or pull-to-refresh?"

**Interviewer:** "Focus on text posts with images. Chronological feed for simplicity. Pull-to-refresh, no real-time for now."

**You:** "For pagination:
- How many posts per page?
- Cursor-based or offset-based pagination?
- Should we support bi-directional scroll (load older AND newer)?"

**Interviewer:** "20 posts per page. Cursor-based. Only load older posts (infinite scroll down)."

**You:** "For images:
- What size/resolution are we displaying?
- Multiple images per post or single?
- Should we prefetch images?"

**Interviewer:** "Single image per post, thumbnail size (300x300). Yes, prefetch next page."

**You:** "For offline and caching:
- Should cached feed be available offline?
- How long should we cache images?
- Memory budget constraints?"

**Interviewer:** "Cache first 2 pages for offline viewing. Images cached for 7 days with max 100 MB disk usage."

### Documented Scope

```
✅ IN SCOPE:
- Chronological feed with text + single image per post
- Infinite scroll pagination (20 posts/page, cursor-based)
- Pull-to-refresh
- Image caching (memory + disk, LRU eviction)
- Prefetching next page
- Offline viewing of first 2 pages
- Smooth 60 FPS scrolling

❌ OUT OF SCOPE:
- Algorithmic ranking
- Real-time updates (WebSocket)
- Video content
- Stories/Reels
- Post creation
- Bi-directional scroll
```

**Non-Functional Requirements:**
- First page load: < 1 second
- Smooth scrolling: 60 FPS
- Image cache: Max 100 MB disk
- Offline: First 2 pages available

---

## 10-25 min: High-Level Design (HLD)

### Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                UI Layer                             │
│  ┌──────────────────────────────────────────┐       │
│  │  FeedViewController (UITableView)        │       │
│  │  - Prefetching data source               │       │
│  │  - Cell reusing                          │       │
│  └──────────────────────────────────────────┘       │
└────────────────────┬────────────────────────────────┘
                     │ observes
┌────────────────────▼────────────────────────────────┐
│           Presentation Layer                        │
│  ┌──────────────────────────────────────────┐       │
│  │  FeedViewModel                           │       │
│  │  @Published var posts: [Post]            │       │
│  │  @Published var state: LoadingState      │       │
│  │  func loadInitialFeed()                  │       │
│  │  func loadMorePosts()                    │       │
│  │  func refresh()                          │       │
│  └───────────────┬────────────┬─────────────┘       │
└──────────────────┼────────────┼─────────────────────┘
                   │            │
      ┌────────────▼────┐   ┌───▼─────────────┐
      │  FeedRepository │   │  ImageLoader    │
      │  - Network      │   │  - Memory cache │
      │  - Disk cache   │   │  - Disk cache   │
      │  - Cursor mgmt  │   │  - Prefetching  │
      └────────┬────────┘   └───┬─────────────┘
               │                │
        ┌──────▼──────┐  ┌──────▼──────┐
        │NetworkSvc   │  │  FileSystem │
        │(URLSession) │  │  (LRU Cache)│
        └─────────────┘  └─────────────┘
```

### Why This Architecture?

**You:** "I'm using MVVM + Repository with a separate ImageLoader:

**1. Separate Image Loading:**
- Images are heavy, deserve dedicated system
- Shared across app (profile pics, post images)
- Independent lifecycle from feed data

**2. Repository Pattern:**
- Handles feed data caching
- Cursor management for pagination
- Separates network from ViewModels

**3. Prefetching:**
- UITableView's prefetching API
- Load images before cells appear
- Smooth scrolling

**Trade-offs:**
- ✅ Separation of concerns (data vs images)
- ✅ Reusable image loader
- ❌ More complexity than monolithic approach
- **Alternative:** Could use SDWebImage/Kingfisher library (production choice)"

### Data Flow - Initial Load

```
App Launch / Pull-to-refresh
    │
    ▼
ViewModel.loadInitialFeed()
    │
    ▼
Repository.fetchPosts(cursor: nil)
    │
    ├─→ Check disk cache (offline-first)
    │   └─ Return cached page 1
    │
    └─→ Network: GET /feed?cursor=nil&limit=20
        │
        ▼
    Response: { posts: [...], nextCursor: "abc123" }
        │
        ▼
    Cache to disk (first 2 pages only)
        │
        ▼
    Return to ViewModel
        │
        ▼
    @Published posts = response.posts
        │
        ▼
    UI renders cells
        │
        ▼
    Prefetch images for visible + next 5 cells
```

### Data Flow - Pagination

```
User scrolls to row 17/20
    │
    ▼
UITableView.prefetchRowsAt([18, 19, 20])
    │
    ▼
ViewModel.loadMorePosts()
    │
    ├─→ Guard: already loading? Return
    ├─→ Guard: no more pages? Return
    │
    ▼
Repository.fetchPosts(cursor: currentCursor)
    │
    ▼
Network: GET /feed?cursor=abc123&limit=20
    │
    ▼
Response: { posts: [...], nextCursor: "xyz789" }
    │
    ▼
Append to existing posts (deduplicate)
    │
    ▼
Update cursor: currentCursor = "xyz789"
    │
    ▼
UI: Insert rows 21-40
    │
    ▼
Prefetch images for rows 18-25
```

---

## 25-45 min: Low-Level Design (LLD)

### Data Models

```swift
struct Post: Identifiable, Codable, Hashable {
    let id: String
    let authorId: String
    let authorName: String
    let authorAvatarURL: URL
    let content: String
    let imageURL: URL?
    let timestamp: Date
    let likeCount: Int
    let commentCount: Int
}

struct FeedResponse: Codable {
    let posts: [Post]
    let nextCursor: String?
    let hasMore: Bool
}
```

### ViewModel Implementation

```swift
@MainActor
class FeedViewModel: ObservableObject {
    @Published private(set) var posts: [Post] = []
    @Published private(set) var state: LoadingState = .idle
    
    private let repository: FeedRepositoryProtocol
    private let imageLoader: ImageLoader
    
    private var currentCursor: String?
    private var hasMorePages = true
    
    init(repository: FeedRepositoryProtocol, imageLoader: ImageLoader) {
        self.repository = repository
        self.imageLoader = imageLoader
    }
    
    func loadInitialFeed() async {
        guard state != .loading else { return }
        state = .loading
        
        do {
            let response = try await repository.fetchFeed(cursor: nil)
            posts = response.posts
            currentCursor = response.nextCursor
            hasMorePages = response.hasMore
            state = .success
            
            // Prefetch first page images
            prefetchImages(for: response.posts)
        } catch {
            state = .error(error)
        }
    }
    
    func loadMorePosts() async {
        guard state != .loadingMore,
              hasMorePages,
              let cursor = currentCursor else { return }
        
        state = .loadingMore
        
        do {
            let response = try await repository.fetchFeed(cursor: cursor)
            
            // Deduplicate before appending
            let newPosts = response.posts.filter { newPost in
                !posts.contains(where: { $0.id == newPost.id })
            }
            
            posts.append(contentsOf: newPosts)
            currentCursor = response.nextCursor
            hasMorePages = response.hasMore
            state = .success
            
            // Prefetch next page images
            prefetchImages(for: newPosts)
        } catch {
            state = .error(error)
        }
    }
    
    func refresh() async {
        currentCursor = nil
        hasMorePages = true
        await loadInitialFeed()
    }
    
    private func prefetchImages(for posts: [Post]) {
        let imageURLs = posts.compactMap { $0.imageURL }
        imageLoader.prefetch(urls: imageURLs)
    }
}
```

### Feed Repository

```swift
protocol FeedRepositoryProtocol {
    func fetchFeed(cursor: String?) async throws -> FeedResponse
}

class FeedRepository: FeedRepositoryProtocol {
    private let networkService: NetworkServiceProtocol
    private let cacheService: FeedCacheService
    
    init(networkService: NetworkServiceProtocol,
         cacheService: FeedCacheService) {
        self.networkService = networkService
        self.cacheService = cacheService
    }
    
    func fetchFeed(cursor: String?) async throws -> FeedResponse {
        // Offline-first for first 2 pages
        if cursor == nil || isSecondPage(cursor) {
            if let cached = await cacheService.getCachedFeed(cursor: cursor) {
                // Return cache, refresh in background
                Task {
                    try? await refreshFromNetwork(cursor: cursor)
                }
                return cached
            }
        }
        
        // Fetch from network
        let response = try await networkService.getFeed(cursor: cursor, limit: 20)
        
        // Cache first 2 pages only
        if shouldCache(cursor: cursor) {
            await cacheService.saveFeed(response, cursor: cursor)
        }
        
        return response
    }
    
    private func shouldCache(cursor: String?) -> Bool {
        // Simple heuristic: cache page 1 (nil cursor) and page 2
        // In production, track page number explicitly
        return cursor == nil || isSecondPage(cursor)
    }
    
    private func isSecondPage(_ cursor: String?) -> Bool {
        // Implementation depends on cursor format
        // Could track page numbers or decode cursor
        return false // Simplified
    }
    
    private func refreshFromNetwork(cursor: String?) async throws {
        let response = try await networkService.getFeed(cursor: cursor, limit: 20)
        await cacheService.saveFeed(response, cursor: cursor)
    }
}
```

### Feed Cache Service

```swift
actor FeedCacheService {
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    private let memoryCache = NSCache<NSString, FeedResponse>()
    
    init() {
        let caches = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        cacheDirectory = caches.appendingPathComponent("FeedCache")
        try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        
        memoryCache.countLimit = 5 // Only keep 5 pages in memory
    }
    
    func getCachedFeed(cursor: String?) -> FeedResponse? {
        let key = cacheKey(cursor: cursor)
        
        // Try memory first
        if let cached = memoryCache.object(forKey: key as NSString) {
            return cached
        }
        
        // Try disk
        let fileURL = cacheDirectory.appendingPathComponent("\(key).json")
        guard let data = try? Data(contentsOf: fileURL),
              let cached = try? JSONDecoder().decode(CachedFeed.self, from: data) else {
            return nil
        }
        
        // Check TTL
        guard cached.expiryDate > Date() else {
            try? fileManager.removeItem(at: fileURL)
            return nil
        }
        
        // Promote to memory
        memoryCache.setObject(cached.response, forKey: key as NSString)
        
        return cached.response
    }
    
    func saveFeed(_ response: FeedResponse, cursor: String?) {
        let key = cacheKey(cursor: cursor)
        
        // Save to memory
        memoryCache.setObject(response, forKey: key as NSString)
        
        // Save to disk
        let cached = CachedFeed(response: response, expiryDate: Date().addingTimeInterval(3600))
        let fileURL = cacheDirectory.appendingPathComponent("\(key).json")
        
        if let data = try? JSONEncoder().encode(cached) {
            try? data.write(to: fileURL)
        }
    }
    
    private func cacheKey(cursor: String?) -> String {
        cursor ?? "page_1"
    }
}

struct CachedFeed: Codable {
    let response: FeedResponse
    let expiryDate: Date
}
```

### Image Loader (Production-Quality)

```swift
class ImageLoader {
    static let shared = ImageLoader()
    
    private let memoryCache = NSCache<NSURL, UIImage>()
    private let diskCache: DiskImageCache
    private let session: URLSession
    private let downloadQueue = DispatchQueue(label: "com.app.imageloader", qos: .userInitiated)
    
    // Track ongoing downloads to avoid duplicates
    private var activeDownloads: [URL: Task<UIImage, Error>] = [:]
    private let lock = NSLock()
    
    init(diskCache: DiskImageCache = DiskImageCache.shared,
         session: URLSession = .shared) {
        self.diskCache = diskCache
        self.session = session
        
        // Configure memory cache
        memoryCache.countLimit = 100
        memoryCache.totalCostLimit = 50 * 1024 * 1024 // 50 MB
    }
    
    func loadImage(from url: URL) async throws -> UIImage {
        // 1. Check memory cache
        if let cached = memoryCache.object(forKey: url as NSURL) {
            return cached
        }
        
        // 2. Check if already downloading
        lock.lock()
        if let existingTask = activeDownloads[url] {
            lock.unlock()
            return try await existingTask.value
        }
        lock.unlock()
        
        // 3. Check disk cache
        if let diskImage = await diskCache.getImage(for: url) {
            memoryCache.setObject(diskImage, forKey: url as NSURL)
            return diskImage
        }
        
        // 4. Download from network
        let downloadTask = Task {
            try await downloadImage(from: url)
        }
        
        lock.lock()
        activeDownloads[url] = downloadTask
        lock.unlock()
        
        defer {
            lock.lock()
            activeDownloads.removeValue(forKey: url)
            lock.unlock()
        }
        
        return try await downloadTask.value
    }
    
    private func downloadImage(from url: URL) async throws -> UIImage {
        let (data, response) = try await session.data(from: url)
        
        guard let httpResponse = response as? HTTPURLResponse,
              (200...299).contains(httpResponse.statusCode) else {
            throw ImageLoaderError.invalidResponse
        }
        
        guard let image = UIImage(data: data) else {
            throw ImageLoaderError.invalidImageData
        }
        
        // Cache to memory and disk
        memoryCache.setObject(image, forKey: url as NSURL)
        await diskCache.saveImage(image, for: url)
        
        return image
    }
    
    func prefetch(urls: [URL]) {
        for url in urls {
            Task {
                try? await loadImage(from: url)
            }
        }
    }
    
    func clearMemoryCache() {
        memoryCache.removeAllObjects()
    }
}

enum ImageLoaderError: Error {
    case invalidResponse
    case invalidImageData
}
```

### Disk Image Cache (LRU with Size Limit)

```swift
actor DiskImageCache {
    static let shared = DiskImageCache()
    
    private let fileManager = FileManager.default
    private let cacheDirectory: URL
    private let maxCacheSize: Int64 = 100 * 1024 * 1024 // 100 MB
    
    // Track file access times for LRU
    private var accessTimes: [String: Date] = [:]
    
    init() {
        let caches = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        cacheDirectory = caches.appendingPathComponent("ImageCache")
        try? fileManager.createDirectory(at: cacheDirectory, withIntermediateDirectories: true)
        
        // Load access times from metadata
        loadAccessTimes()
    }
    
    func getImage(for url: URL) -> UIImage? {
        let filename = cacheFilename(for: url)
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        
        guard let data = try? Data(contentsOf: fileURL),
              let image = UIImage(data: data) else {
            return nil
        }
        
        // Update access time (LRU)
        accessTimes[filename] = Date()
        
        return image
    }
    
    func saveImage(_ image: UIImage, for url: URL) {
        let filename = cacheFilename(for: url)
        let fileURL = cacheDirectory.appendingPathComponent(filename)
        
        guard let data = image.jpegData(compressionQuality: 0.8) else { return }
        
        try? data.write(to: fileURL)
        accessTimes[filename] = Date()
        
        // Check cache size, evict if needed
        Task {
            await evictIfNeeded()
        }
    }
    
    private func evictIfNeeded() async {
        let currentSize = await calculateCacheSize()
        
        guard currentSize > maxCacheSize else { return }
        
        // Sort files by access time (LRU first)
        let sortedFiles = accessTimes.sorted { $0.value < $1.value }
        
        var freedSpace: Int64 = 0
        let targetFreeSpace = maxCacheSize / 4 // Free 25% when evicting
        
        for (filename, _) in sortedFiles {
            let fileURL = cacheDirectory.appendingPathComponent(filename)
            
            if let attributes = try? fileManager.attributesOfItem(atPath: fileURL.path),
               let fileSize = attributes[.size] as? Int64 {
                
                try? fileManager.removeItem(at: fileURL)
                accessTimes.removeValue(forKey: filename)
                freedSpace += fileSize
                
                if freedSpace >= targetFreeSpace {
                    break
                }
            }
        }
        
        saveAccessTimes()
    }
    
    private func calculateCacheSize() async -> Int64 {
        guard let enumerator = fileManager.enumerator(at: cacheDirectory, includingPropertiesForKeys: [.fileSizeKey]) else {
            return 0
        }
        
        var totalSize: Int64 = 0
        
        for case let fileURL as URL in enumerator {
            if let attributes = try? fileManager.attributesOfItem(atPath: fileURL.path),
               let fileSize = attributes[.size] as? Int64 {
                totalSize += fileSize
            }
        }
        
        return totalSize
    }
    
    private func cacheFilename(for url: URL) -> String {
        // Use MD5 or simple hash
        url.absoluteString.data(using: .utf8)!.base64EncodedString()
            .replacingOccurrences(of: "/", with: "_")
    }
    
    private func loadAccessTimes() {
        let metadataURL = cacheDirectory.appendingPathComponent("metadata.json")
        if let data = try? Data(contentsOf: metadataURL),
           let times = try? JSONDecoder().decode([String: Date].self, from: data) {
            accessTimes = times
        }
    }
    
    private func saveAccessTimes() {
        let metadataURL = cacheDirectory.appendingPathComponent("metadata.json")
        if let data = try? JSONEncoder().encode(accessTimes) {
            try? data.write(to: metadataURL)
        }
    }
}
```

### UI Layer (UITableView)

```swift
class FeedViewController: UIViewController {
    private let viewModel: FeedViewModel
    private let tableView = UITableView()
    private let refreshControl = UIRefreshControl()
    private var cancellables = Set<AnyCancellable>()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        bindViewModel()
        
        Task {
            await viewModel.loadInitialFeed()
        }
    }
    
    private func setupUI() {
        view.addSubview(tableView)
        tableView.frame = view.bounds
        tableView.delegate = self
        tableView.dataSource = self
        tableView.prefetchDataSource = self // Key for prefetching!
        tableView.register(PostCell.self, forCellReuseIdentifier: "PostCell")
        
        refreshControl.addTarget(self, action: #selector(handleRefresh), for: .valueChanged)
        tableView.refreshControl = refreshControl
    }
    
    private func bindViewModel() {
        viewModel.$posts
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
        
        viewModel.$state
            .receive(on: DispatchQueue.main)
            .sink { [weak self] state in
                self?.handleStateChange(state)
            }
            .store(in: &cancellables)
    }
    
    @objc private func handleRefresh() {
        Task {
            await viewModel.refresh()
        }
    }
}

extension FeedViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        viewModel.posts.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "PostCell", for: indexPath) as! PostCell
        let post = viewModel.posts[indexPath.row]
        cell.configure(with: post)
        return cell
    }
}

extension FeedViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        // Trigger pagination
        let threshold = viewModel.posts.count - 5
        if indexPath.row >= threshold {
            Task {
                await viewModel.loadMorePosts()
            }
        }
    }
}

// CRITICAL: Prefetching for smooth scrolling
extension FeedViewController: UITableViewDataSourcePrefetching {
    func tableView(_ tableView: UITableView, prefetchRowsAt indexPaths: [IndexPath]) {
        let urls = indexPaths.compactMap { viewModel.posts[$0.row].imageURL }
        ImageLoader.shared.prefetch(urls: urls)
    }
    
    func tableView(_ tableView: UITableView, cancelPrefetchingForRowsAt indexPaths: [IndexPath]) {
        // Could cancel ongoing downloads if using custom cancellation
    }
}
```

### Post Cell

```swift
class PostCell: UITableViewCell {
    private let postImageView = UIImageView()
    private let authorLabel = UILabel()
    private let contentLabel = UILabel()
    private var imageLoadTask: Task<Void, Never>?
    
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupUI()
    }
    
    func configure(with post: Post) {
        authorLabel.text = post.authorName
        contentLabel.text = post.content
        
        // Cancel previous image load
        imageLoadTask?.cancel()
        postImageView.image = nil
        
        // Load image asynchronously
        if let imageURL = post.imageURL {
            imageLoadTask = Task {
                do {
                    let image = try await ImageLoader.shared.loadImage(from: imageURL)
                    
                    // Check if cell wasn't reused
                    guard !Task.isCancelled else { return }
                    
                    await MainActor.run {
                        self.postImageView.image = image
                    }
                } catch {
                    // Show placeholder or error state
                }
            }
        }
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        imageLoadTask?.cancel()
        postImageView.image = nil
    }
}
```

---

## Part 4: Feed API Design (Cursor-Based Pagination)

### Feed API Endpoints

**You:** "For a social media feed, I need cursor-based pagination for consistency. Here's my API design:"

#### Core Feed Endpoint

```
GET /v1/feed
  Query Params:
    - cursor: String (optional, opaque pagination token)
    - limit: Int (default: 20, max: 50)
    - fields: String (optional, comma-separated fields to include)
  
  Response: 200 OK
  {
    "data": {
      "posts": [
        {
          "id": "post_abc123",
          "author": {
            "id": "user_xyz",
            "username": "johndoe",
            "displayName": "John Doe",
            "avatarUrl": "https://cdn.app.com/avatars/xyz_thumb.jpg"
          },
          "content": "Just finished an amazing hike!",
          "imageUrl": "https://cdn.app.com/images/abc123_medium.jpg",
          "imageThumbnailUrl": "https://cdn.app.com/images/abc123_thumb.jpg",  // Mobile optimization
          "timestamp": "2026-01-13T15:00:00Z",
          "stats": {
            "likeCount": 245,
            "commentCount": 18,
            "shareCount": 5
          },
          "userEngagement": {
            "liked": false,
            "saved": false
          }
        }
      ],
      "pagination": {
        "nextCursor": "eyJpZCI6InBvc3RfYWJjMTIzIiwidCI6MTY3MzYyODAwMH0=",  // Base64 encoded
        "hasMore": true
      }
    },
    "meta": {
      "serverTimestamp": "2026-01-13T18:00:00Z",
      "requestId": "req_uuid"
    }
  }

GET /v1/posts/{postId}
  Response: 200 OK (full post details)
  Response: 404 Not Found
```

### Why Cursor-Based Pagination?

**Interviewer:** "Why cursor instead of offset for feed?"

**You:** "Critical for feed consistency:

| Scenario | Offset Pagination | Cursor Pagination |
|----------|-------------------|-------------------|
| **User scrolls to page 2** | `?page=2&limit=20` | `?cursor=abc&limit=20` |
| **New post added at top** | Page 2 now has posts 2-21 (was 1-20) | Cursor ensures posts AFTER 'abc' |
| **Result** | ❌ Duplicate post on page 2 | ✅ No duplicates |
| **Performance** | `OFFSET 40 LIMIT 20` (slow for large offsets) | `WHERE id < 'abc' LIMIT 20` (indexed, fast) |

**Real example:**
```
// User on page 1 (posts 1-20)
GET /feed?limit=20
Response: posts [1-20], nextCursor="post_20"

// 5 new posts added while user reads

// User taps "Load More"
GET /feed?cursor=post_20&limit=20
// Cursor ensures: fetch posts OLDER than post_20
// Returns: posts [21-40], NOT [16-35] with duplicates
```

**Trade-off:** Can't jump to arbitrary page, but feed UX doesn't need that."

### Cursor Implementation

**Server-side cursor encoding:**
```json
// Cursor contains: post ID + timestamp
{
  "id": "post_abc123",
  "timestamp": 1673628000
}

// Base64 encoded: "eyJpZCI6InBvc3RfYWJjMTIzIiwidCI6MTY3MzYyODAwMH0="
```

**SQL query:**
```sql
-- Without cursor (first page)
SELECT * FROM posts
ORDER BY timestamp DESC, id DESC
LIMIT 20;

-- With cursor (subsequent pages)
SELECT * FROM posts
WHERE timestamp < '2026-01-13T15:00:00Z'
   OR (timestamp = '2026-01-13T15:00:00Z' AND id < 'post_abc123')
ORDER BY timestamp DESC, id DESC
LIMIT 20;
```

**iOS client handling:**
```swift
struct FeedResponse: Codable {
    let posts: [Post]
    let pagination: Pagination
    
    struct Pagination: Codable {
        let nextCursor: String?
        let hasMore: Bool
    }
}

// ViewModel tracks cursor
var currentCursor: String?

func loadMorePosts() async {
    let response = try await api.getFeed(cursor: currentCursor, limit: 20)
    posts.append(contentsOf: response.posts)
    currentCursor = response.pagination.nextCursor
}
```

### Field Selection (Bandwidth Optimization)

**Interviewer:** "How do you minimize response size for mobile?"

**You:** "Field selection pattern:

```
// Mobile app (light)
GET /feed?fields=id,author.id,author.username,author.avatarUrl,content,imageThumbnailUrl,timestamp,stats.likeCount

Response:
{
  "posts": [
    {
      "id": "post_123",
      "author": {
        "id": "user_xyz",
        "username": "johndoe",
        "avatarUrl": "..."
      },
      "content": "...",
      "imageThumbnailUrl": "...",  // Small 300x300
      "timestamp": "...",
      "stats": {"likeCount": 245}
    }
  ]
}

// Detail view (full)
GET /posts/{id}?fields=* // All fields
```

**Benefits:**
- ✅ List view: ~1KB per post (thumbnail URL only)
- ✅ Full view: ~5KB per post (high-res image, full stats)
- ✅ Saves 80% bandwidth for feed scrolling
- **Alternative:** Separate `/v1/feed/light` and `/v1/feed/full` endpoints"

### Image CDN URLs

```json
{
  "imageUrl": "https://cdn.app.com/images/{postId}_{size}.jpg",
  
  "sizes": {
    "thumbnail": "https://cdn.app.com/images/abc123_thumb.jpg",    // 300x300
    "medium": "https://cdn.app.com/images/abc123_medium.jpg",       // 1080x1080
    "large": "https://cdn.app.com/images/abc123_large.jpg"          // Original
  }
}
```

**iOS client:**
```swift
// For list view
let thumbnailURL = post.sizes.thumbnail

// For detail/fullscreen
let highResURL = post.sizes.large
```

### Rate Limiting for Feed

```
Headers:
  X-RateLimit-Limit: 100          // Requests per hour
  X-RateLimit-Remaining: 95
  X-RateLimit-Reset: 1673640000
  
  X-Feed-Refresh-After: 300      // Don't refresh feed more than every 5 min
```

**iOS client:**
```swift
actor FeedRateLimiter {
    private var lastRefreshTime: Date?
    private let minRefreshInterval: TimeInterval = 300  // 5 minutes
    
    func canRefresh() -> Bool {
        guard let last = lastRefreshTime else {
            lastRefreshTime = Date()
            return true
        }
        
        let elapsed = Date().timeIntervalSince(last)
        if elapsed >= minRefreshInterval {
            lastRefreshTime = Date()
            return true
        }
        
        return false
    }
}

// In ViewModel
func refresh() async {
    guard await rateLimiter.canRefresh() else {
        // Show "Please wait before refreshing" message
        return
    }
    
    await loadInitialFeed()
}
```

### Prefetching Optimization

**Server hint for prefetching:**
```json
{
  "posts": [...],
  "prefetchHints": {
    "nextPageCursor": "xyz789",
    "estimatedNextPageSize": 20,
    "prefetchRecommended": true  // Server suggests prefetch
  }
}
```

**iOS client:**
```swift
if response.prefetchHints.prefetchRecommended {
    // Prefetch next page in background
    Task.detached(priority: .utility) {
        try? await fetchFeed(cursor: response.prefetchHints.nextPageCursor)
    }
}
```

### Caching Headers

```
Response Headers:
  Cache-Control: private, max-age=60        // Cache feed for 1 minute
  ETag: "feed_version_abc123"
  Last-Modified: Mon, 13 Jan 2026 15:00:00 GMT
  Vary: Accept-Encoding, Authorization       // Cache per user

Conditional Request:
  If-None-Match: "feed_version_abc123"
  If-Modified-Since: Mon, 13 Jan 2026 15:00:00 GMT

Response: 304 Not Modified (use cache)
```

### Error Responses

```json
// Empty feed (new user)
{
  "data": {
    "posts": [],
    "pagination": {
      "nextCursor": null,
      "hasMore": false
    }
  },
  "meta": {
    "emptyState": {
      "reason": "no_following",
      "message": "Follow users to see their posts",
      "action": {
        "label": "Discover People",
        "url": "/discover"
      }
    }
  }
}

// Invalid cursor
{
  "error": {
    "code": "INVALID_CURSOR",
    "message": "Pagination cursor is invalid or expired",
    "retryable": false,
    "action": "Restart from beginning by omitting cursor parameter"
  }
}

// Content blocked
{
  "data": {
    "posts": [
      {
        "id": "post_123",
        "contentBlocked": true,
        "reason": "This post was removed for violating guidelines",
        "timestamp": "2026-01-13T15:00:00Z"
      }
    ]
  }
}
```

### Real-Time Updates (Pull-to-Refresh)

**Interviewer:** "How would you add real-time feed updates?"

**You:** "Two approaches:

**1. Poll with 'since' parameter:**
```
GET /feed/updates?since=2026-01-13T15:00:00Z

Response:
{
  "newPosts": [
    // Posts added since timestamp
  ],
  "updatedPosts": [
    // Like counts, comment counts changed
  ],
  "deletedPostIds": ["post_xyz"]
}
```

**iOS implementation:**
```swift
// Background poll every 30 seconds
timer = Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { _ in
    Task {
        let updates = try? await api.getFeedUpdates(since: lastUpdateTime)
        
        if let new = updates?.newPosts, !new.isEmpty {
            // Show "5 new posts" banner
            showNewPostsBanner(count: new.count)
        }
    }
}
```

**2. WebSocket (Real-time):**
```
// Connect to WebSocket
ws://api.app.com/feed/stream

// Server pushes events
{
  "event": "new_post",
  "data": {
    "postId": "post_new",
    "insertPosition": 0  // Add at top
  }
}

{
  "event": "post_updated",
  "data": {
    "postId": "post_123",
    "stats": {"likeCount": 250}  // Updated count
  }
}
```

**Trade-offs:**
- **Polling:** Simple, battery-friendly (30s interval)
- **WebSocket:** Real-time, but constant connection (battery drain)
- **My choice:** Polling for feed, WebSocket for messaging/notifications"

### Bi-Directional Scroll API

**Future enhancement:**
```
// Load newer posts (pull down at top)
GET /feed/newer?cursor=post_abc123&limit=20

// Load older posts (scroll down)
GET /feed/older?cursor=post_abc123&limit=20
// Same as GET /feed?cursor=post_abc123
```

### API Versioning

```swift
// Client specifies supported content types
Headers:
  Accept: application/vnd.app.v2+json
  X-Client-Version: 2.5.0

// Server response includes compatibility
{
  "data": {...},
  "meta": {
    "apiVersion": "2",
    "schemaVersion": "2.1",
    "deprecations": [
      {
        "field": "imageUrl",
        "replacedBy": "sizes.medium",
        "sunsetDate": "2026-06-01"
      }
    ]
  }
}
```

### Mobile-Specific Optimizations

**1. Adaptive Quality**
```
Headers:
  X-Network-Type: cellular  // or wifi
  X-Data-Saver: true        // User enabled data saver

Server response:
{
  "posts": [
    {
      "imageUrl": "https://cdn.app.com/images/abc123_low.jpg",  // Lower quality on cellular
      "videoUrl": null  // Exclude videos when data saver enabled
    }
  ]
}
```

**2. Batch Prefetch**
```
POST /v1/feed/batch-prefetch
{
  "cursors": ["cursor1", "cursor2", "cursor3"]  // Prefetch 3 pages
}

Response:
{
  "pages": [
    {"cursor": "cursor1", "posts": [...]},
    {"cursor": "cursor2", "posts": [...]},
    {"cursor": "cursor3", "posts": [...]}
  ]
}
```

**3. Compression**
```
Request Headers:
  Accept-Encoding: gzip, br  // Brotli preferred

Response:
  Content-Encoding: br
  // 70-80% smaller payload
```

### Interview Q&A: Feed API Design

**Q:** "Why include `userEngagement.liked` in feed response?"

**A:**
> "To avoid N+1 queries:
> 
> **Without it:**
> ```swift
> // Fetch feed
> let posts = await api.getFeed()
> 
> // For each post, check if user liked (20 requests!)
> for post in posts {
>     post.liked = try await api.checkIfLiked(postId: post.id)
> }
> ```
> 
> **With it:**
> ```sql
> -- Server query
> SELECT posts.*, 
>        EXISTS(SELECT 1 FROM likes 
>               WHERE post_id = posts.id 
>               AND user_id = current_user.id) AS liked
> FROM posts;
> ```
> 
> Single query, all engagement data included. Essential for mobile performance."

**Q:** "How do you handle deleted posts in cursor pagination?"

**A:**
> "Two approaches:
> 
> **1. Soft delete (my choice):**
> ```json
> {
>   \"id\": \"post_deleted\",
>   \"status\": \"deleted\",
>   \"content\": null,
>   \"timestamp\": \"...\"
> }
> ```
> Maintains cursor integrity, client filters out deleted posts
> 
> **2. Skip deleted, may return fewer items:**
> ```
> Request: ?cursor=abc&limit=20
> Response: 18 posts (2 were deleted)
> Client handles variable page sizes
> ```
> 
> Soft delete is more reliable for pagination consistency."

**Q:** "What if cursor expires (server clears old data)?"

**A:**
> "Graceful degradation:
> 
> ```
> Response: 410 Gone
> {
>   \"error\": {
>     \"code\": \"CURSOR_EXPIRED\",
>     \"message\": \"Pagination cursor has expired\",
>     \"suggestion\": \"Restart from beginnin

g by calling /feed without cursor\"
>   }
> }
> ```
> 
> iOS client:
> ```swift
> if error.code == \"CURSOR_EXPIRED\" {
>     // Clear feed, restart from top
>     currentCursor = nil
>     await loadInitialFeed()
> }
> ```"

---

## 45-55 min: Deep Dives

### 1. Pagination Strategy: Cursor vs Offset

**Interviewer:** "Why cursor-based over offset-based?"

**You:** "Great question. Here's the comparison:

| Aspect | Offset (LIMIT/OFFSET) | Cursor (Token/ID) |
|--------|----------------------|-------------------|
| **Consistency** | ❌ Duplicates if new items added | ✅ Stateful, no duplicates |
| **Performance** | ❌ Slow for large offsets (DB skip) | ✅ Fast (indexed WHERE id > cursor) |
| **Simplicity** | ✅ Easy to implement | ❌ Requires cursor management |
| **Jump to page** | ✅ Can jump to page N | ❌ Must scroll sequentially |

**Example problem with offset:**
```
Page 1: OFFSET 0  → Posts 1-20
(User posts new item)
Page 2: OFFSET 20 → Posts 2-21 (duplicate post #2!)
```

**Cursor solves this:**
```
Request: GET /feed?cursor=post_20_id
Response: Posts AFTER post_20_id
```

**Trade-off:** For feed, cursor is better. For search results with page numbers, offset works."

### 2. Image Caching Performance

**Interviewer:** "How do you ensure smooth scrolling with images?"

**You:** "Multiple optimizations:

**1. Three-tier caching:**
```
Memory (NSCache) → Disk → Network
     ↓ 50 MB         ↓ 100 MB
   Instant        < 50 ms    ~500 ms
```

**2. Prefetching (critical!):**
```swift
func tableView(_ tableView: UITableView, prefetchRowsAt indexPaths: [IndexPath]) {
    // Start loading images 5 rows ahead
    let urls = indexPaths.compactMap { posts[$0.row].imageURL }
    ImageLoader.shared.prefetch(urls: urls)
}
```

**3. Deduplication:**
- Track active downloads to avoid multiple requests for same image
- Use Task dictionary: `[URL: Task<UIImage, Error>]`

**4. Lazy decoding:**
```swift
let image = UIImage(data: data)
// Decode on background thread
DispatchQueue.global().async {
    let decoded = image.decodedImage() // Force decode
    DispatchQueue.main.async {
        imageView.image = decoded // Already decoded, fast render
    }
}
```

**5. Downsampling:**
```swift
// Don't load full 4K image if displaying 300x300
let downsampledImage = UIImage.downsample(from: url, to: CGSize(width: 300, height: 300))
```

**Result:** 60 FPS scrolling even with 100% new images."

### 3. LRU Eviction Strategy

**Interviewer:** "Walk me through your LRU implementation."

**You:** "My disk cache implements LRU with size-based eviction:

**Data structure:**
```swift
private var accessTimes: [String: Date] = [:]
```

**On access (get/save):**
```swift
accessTimes[filename] = Date() // Update last access
```

**On eviction (when cache > 100 MB):**
```swift
// 1. Sort files by access time (oldest first)
let sortedFiles = accessTimes.sorted { $0.value < $1.value }

// 2. Delete until we free 25% of max size
for (filename, _) in sortedFiles {
    delete(filename)
    freedSpace += fileSize
    if freedSpace >= 25 MB { break }
}
```

**Why 25% batch eviction:**
- Amortized cost: Don't evict on every save
- Hysteresis: Avoid thrashing (delete, re-download, delete...)

**Alternative:** Could use Least Frequently Used (LFU) for viral content, but LRU is simpler."

### 4. Offline Support

**Interviewer:** "How does offline work exactly?"

**You:** "Repository implements cache-first for first 2 pages:

```swift
func fetchFeed(cursor: String?) async throws -> FeedResponse {
    if shouldCache(cursor) {
        if let cached = await cacheService.getCachedFeed(cursor: cursor) {
            // Return cached immediately (fast)
            Task {
                // Silently refresh in background
                try? await refreshFromNetwork(cursor: cursor)
            }
            return cached
        }
    }
    
    // No cache or page 3+ → network required
    return try await networkService.getFeed(cursor: cursor)
}
```

**Offline UX:**
1. User opens app (no internet)
2. Repository returns cached page 1 from disk (< 50ms)
3. User sees feed instantly
4. When scrolls to page 3 → Show \"You're offline\" banner

**Cache invalidation:**
- TTL: 1 hour
- On app foreground: Silent background refresh
- Pull-to-refresh: Force network fetch

**Trade-off:** Stale data for 1 hour vs instant offline access."

### 5. Memory Management

**Interviewer:** "What if user scrolls through 1000 posts?"

**You:** "Several safeguards:

**1. NSCache auto-eviction:**
```swift
memoryCache.countLimit = 100 // Only 100 images
memoryCache.totalCostLimit = 50 * 1024 * 1024 // 50 MB max
```

**2. Cell cancellation:**
```swift
override func prepareForReuse() {
    imageLoadTask?.cancel() // Cancel download if cell scrolled away
    imageView.image = nil   // Release image memory
}
```

**3. Limit in-memory posts:**
```swift
// Keep only 200 posts in array
if posts.count > 200 {
    posts = Array(posts.suffix(200))
}
```

**4. Lazy image loading:**
- Load only when cell appears
- Use `willDisplay` instead of `cellForRow`

**Memory profile:**
- 200 posts × 200 bytes = ~40 KB (negligible)
- 100 images × 300x300 × 4 bytes = ~36 MB (managed by NSCache)
- Total: < 50 MB even after infinite scroll

**Trade-off:** User can't scroll back to post #1 after 200+ posts (acceptable for feed)."

### 6. Handle Duplicate Posts

**Interviewer:** "How do you prevent duplicate posts?"

**You:** "Deduplication at append:

```swift
func loadMorePosts() async {
    let response = try await repository.fetchFeed(cursor: cursor)
    
    // Use Set-based deduplication (O(n) instead of O(n²))
    let existingIds = Set(posts.map { $0.id })
    let newPosts = response.posts.filter { !existingIds.contains($0.id) }
    
    posts.append(contentsOf: newPosts)
}
```

**Why duplicates might occur:**
- Race condition: Two simultaneous pagination requests
- Server bug: Returns overlapping cursors
- Retry logic: Network failure → retry → duplicate page

**Additional safeguard:**
```swift
// Guard against race condition
guard state != .loadingMore else { return }
state = .loadingMore
```"

---

## Common L4 Follow-Up Questions

### Q1: "How would you add real-time updates?"

**Answer:**
"Add WebSocket for new post notifications:

```swift
class FeedViewModel {
    private let webSocketService: WebSocketService
    
    func connectToFeed() {
        Task {
            for await newPost in webSocketService.newPostStream {
                // Insert at top
                posts.insert(newPost, at: 0)
                
                // Or show \"3 new posts\" banner and let user tap to refresh
                newPostsCount += 1
            }
        }
    }
}
```

**UX choice:**
- Auto-insert: Jarring if user mid-scroll
- Banner: Better UX, user controls refresh

**Battery impact:** Keep-alive pings, stay in PLANNING mode with sleep timer."

### Q2: "How would you implement pull-down-to-load-newer?"

**Answer:**
"Add bi-directional pagination:

```swift
struct FeedResponse {
    let posts: [Post]
    let nextCursor: String?     // For older posts
    let previousCursor: String? // For newer posts
}

func loadNewerPosts() async {
    // GET /feed?cursor=firstPost.id&direction=before
    let response = try await repository.fetchFeed(cursor: posts.first?.id, direction: .before)
    posts.insert(contentsOf: response.posts, at: 0)
}
```

**Index path management:**
```swift
// Crucial: Update scroll offset to avoid jump
let oldOffset = tableView.contentOffset
tableView.reloadData()
let newOffset = CGPoint(x: 0, y: oldOffset.y + insertedHeight)
tableView.setContentOffset(newOffset, animated: false)
```"

### Q3: "How to handle video in feed?"

**Answer:**
"Video adds complexity:

**1. Autoplay logic:**
```swift
func scrollViewDidScroll(_ scrollView: UIScrollView) {
let visibleCells = tableView.visibleCells.compactMap { $0 as? PostCell }
    
    // Play video in most-visible cell, pause others
    if let mostVisible = getMostVisibleCell(visibleCells) {
        mostVisible.playVideo()
        otherCells.forEach { $0.pauseVideo() }
    }
}
```

**2. Preloading:**
- Prefetch first 5 seconds (not entire video)
- Use AVAssetResourceLoader for chunked loading

**3. Memory:**
- Only keep 3 video players in memory
- Aggressive cache eviction (videos are huge)

**Trade-off:** Videos 100x larger than images, huge cache/bandwidth impact."

---

## L4 Success Checklist

✅ Chose cursor-based pagination with justification
✅ Designed three-tier image caching (memory/disk/network)
✅ Implemented LRU eviction with size limit
✅ Used UITableViewDataSourcePrefetching for smooth scrolling
✅ Handled offline with cache-first strategy
✅ Prevented duplicate posts with Set-based deduplication
✅ Managed memory with NSCache limits and cell cancellation
✅ Considered performance (downsampling, lazy decoding)
✅ Discussed real-time and bi-directional extensions
✅ Covered testing strategy for async image loading

---

## Interview Mistakes to Avoid

| Mistake | Impact | Fix |
|---------|--------|-----|
| Not using prefetching | Laggy scrolling | Implement `UITableViewDataSourcePrefetching` |
| Loading full-res images | Memory blow-up | Downsample to display size |
| Offset-based pagination | Duplicates in feed | Use cursor-based |
| No cache size limit | Disk full errors | LRU eviction with max size |
| Blocking UI for images | Janky scroll | Async load in Task, cancel on reuse |
| Ignoring offline | Mobile-specific miss | Cache first N pages |

> [!TIP]
> The interviewer WILL ask about smooth scrolling. Master prefetching and image optimization!
