# Offline-First Architecture for iOS
## A Complete Deep Dive: Beginner to Advanced

---

## Table of Contents
1. [Beginner Level](#beginner-level)
2. [Intermediate Level](#intermediate-level)
3. [Advanced Level](#advanced-level)
4. [Interview Preparation](#interview-preparation)

---

# Beginner Level

## What is "Offline-First"?

**Simple Explanation:**
Offline-first means your app works WITHOUT internet first, and syncs WITH internet later. The app stores data locally on the device, so users can still read, write, and interact with content even when disconnected.

**Think of it like this:**
- **Notes app on your iPhone**: You write notes anytime, anywhere. They sync to iCloud when you have internet.
- **Maps app**: Downloads map tiles for your area. Works underground or in airplane mode.
- **Kindle app**: Downloads books. Read offline, bookmarks sync later.

## Why Offline-First Matters for Mobile Apps

### 1. **Network Reliability**
- Elevators, subways, basements = no signal
- Poor 3G/4G connections = slow or failed requests
- Airplane mode, roaming restrictions

### 2. **Battery Life**
- Network requests drain battery fast
- Offline-first reduces unnecessary API calls
- Local reads from disk/memory use less power

### 3. **User Trust & Experience**
- Users expect instant responses
- "No internet" errors feel broken
- Data should never disappear

### 4. **Performance**
- Local reads: **~1ms**
- API call: **200ms - 2000ms+**
- User perception: Instant = better

## Real-Life Analogies

| Scenario | Offline-First Behavior |
|----------|------------------------|
| **Grocery List** | You write items on paper (local). Later, you share the list via WhatsApp (sync). |
| **DVR Recording** | You watch recorded shows anytime (offline). New episodes download when connected (sync). |
| **Library Books** | Borrow books (download). Read at home (offline). Return later (sync status). |
| **Cash Wallet** | You spend cash offline. Bank account updates when you deposit/withdraw (eventual sync). |

## Three Approaches: Comparison

### 1. **Online-First (Network-Dependent)**
```
User opens app → API call → Show data
No internet? → Error screen 😞
```
**Example:** Stock trading apps (need real-time prices)

### 2. **Cache-Only (Read Cache, No Writes)**
```
User opens app → Check cache → If empty, API call
User writes data → Direct to server (fails offline)
```
**Example:** Simple news apps

### 3. **Offline-First (Best for Mobile)**
```
User opens app → Show local data instantly
User writes data → Save locally + queue for sync
Has internet? → Background sync
```
**Example:** Uber, Notion, Instagram

## Simple iOS Example: Offline-First Feed

### Scenario: Social Feed App (like Instagram)

**Step 1: User Opens App (Offline)**
```swift
class FeedViewController: UIViewController {
    let database = CoreDataManager.shared
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // ✅ Always load from local database first
        let localPosts = database.fetchPosts()
        displayPosts(localPosts)
        
        // 🌐 Then try to refresh from network
        if NetworkMonitor.isConnected {
            refreshFromNetwork()
        }
    }
    
    func refreshFromNetwork() {
        APIClient.fetchLatestPosts { [weak self] result in
            switch result {
            case .success(let newPosts):
                // Save to local database
                self?.database.savePosts(newPosts)
                // Reload UI
                self?.displayPosts(newPosts)
            case .failure:
                // Silent failure - already showing local data
                print("Sync failed, using cached data")
            }
        }
    }
}
```

**Step 2: User Likes a Post (Offline)**
```swift
func likePost(_ post: Post) {
    // 1. Update local database immediately
    database.markAsLiked(post, byUser: currentUser)
    
    // 2. Update UI instantly (no waiting)
    post.isLiked = true
    updateUI(for: post)
    
    // 3. Queue action for sync
    let action = SyncAction(type: .like, postId: post.id)
    SyncQueue.shared.enqueue(action)
    
    // 4. Sync when online
    if NetworkMonitor.isConnected {
        SyncQueue.shared.processQueue()
    }
}
```

**Key Principles:**
1. Local database is **source of truth**
2. UI updates **immediately** from local data
3. Network operations happen **in background**
4. Failures are **silent** (user already sees their action)

---

# Intermediate Level

## Local Data Storage Strategies on iOS

### 1. **In-Memory Storage**
**When to Use:** Temporary session data, current user state
```swift
class InMemoryCache {
    private var cache: [String: Any] = [:]
    
    func store<T>(_ value: T, forKey key: String) {
        cache[key] = value
    }
    
    func retrieve<T>(forKey key: String) -> T? {
        return cache[key] as? T
    }
}

// Example: Current user session
let sessionCache = InMemoryCache()
sessionCache.store(currentUser, forKey: "user")
```

**Pros:** Fastest (nanoseconds), simple  
**Cons:** Lost on app termination, limited by RAM

### 2. **Disk Persistence (UserDefaults, Plist, JSON)**
**When to Use:** Settings, small key-value data
```swift
// UserDefaults (max ~1MB recommended)
UserDefaults.standard.set(theme, forKey: "appTheme")

// JSON to disk (for structured data)
struct AppSettings: Codable {
    var notificationsEnabled: Bool
    var preferredLanguage: String
}

func saveSettings(_ settings: AppSettings) {
    let encoder = JSONEncoder()
    if let data = try? encoder.encode(settings) {
        let url = FileManager.default.urls(for: .documentDirectory, in: .userDomainMask)[0]
            .appendingPathComponent("settings.json")
        try? data.write(to: url)
    }
}
```

**Pros:** Simple, persists across launches  
**Cons:** Slow for large data, no relationships

### 3. **Database (Core Data, SQLite, Realm)**
**When to Use:** Relational data, complex queries, large datasets

**Core Data Example:**
```swift
// Define entity relationships
class Post: NSManagedObject {
    @NSManaged var id: String
    @NSManaged var content: String
    @NSManaged var createdAt: Date
    @NSManaged var author: User
    @NSManaged var comments: Set<Comment>
    @NSManaged var isSynced: Bool
}

class CoreDataManager {
    func fetchUnsyncedPosts() -> [Post] {
        let request: NSFetchRequest<Post> = Post.fetchRequest()
        request.predicate = NSPredicate(format: "isSynced == NO")
        return try? context.fetch(request) ?? []
    }
}
```

**Pros:** Relationships, indexing, efficient queries  
**Cons:** Complex setup, learning curve

### Decision Matrix

| Data Type | Storage | Example |
|-----------|---------|---------|
| User session | In-memory | Current cart items |
| Settings | UserDefaults | Dark mode preference |
| Small lists | JSON file | Favorite locations |
| Large datasets | Core Data | Chat messages, feed posts |
| Binary data | File system + DB reference | Images, videos |

## Sync Models

### 1. **Read-Through Cache**
```
Read request → Check cache → If miss, fetch from API → Save to cache → Return
```

**iOS Implementation:**
```swift
class ReadThroughCache {
    let cache = NSCache<NSString, Post>()
    
    func getPost(id: String, completion: @escaping (Post?) -> Void) {
        // Check cache first
        if let cached = cache.object(forKey: id as NSString) {
            completion(cached)
            return
        }
        
        // Cache miss - fetch from network
        APIClient.fetchPost(id: id) { [weak self] result in
            if case .success(let post) = result {
                self?.cache.setObject(post, forKey: id as NSString)
                completion(post)
            } else {
                completion(nil)
            }
        }
    }
}
```

**Use Case:** Product details in shopping app

### 2. **Write-Through Cache**
```
Write request → Write to cache AND server simultaneously → Confirm on both success
```

**iOS Implementation:**
```swift
func updateProfile(_ profile: UserProfile, completion: @escaping (Bool) -> Void) {
    // Write to local database
    database.save(profile)
    
    // Immediately write to server
    APIClient.updateProfile(profile) { result in
        switch result {
        case .success:
            profile.isSynced = true
            database.save(profile)
            completion(true)
        case .failure:
            // Mark as needs sync
            profile.isSynced = false
            database.save(profile)
            completion(false)
        }
    }
}
```

**Use Case:** Critical data (payment info, account settings)

### 3. **Write-Back / Eventual Sync**
```
Write request → Write to local cache immediately → Queue for background sync → Sync later
```

**iOS Implementation:**
```swift
class WriteBackSync {
    let queue = DispatchQueue(label: "sync.queue")
    var pendingActions: [SyncAction] = []
    
    func createPost(_ post: Post) {
        // 1. Save locally instantly
        database.save(post)
        post.syncStatus = .pending
        
        // 2. Update UI immediately
        NotificationCenter.default.post(name: .postCreated, object: post)
        
        // 3. Queue for sync
        let action = SyncAction(type: .create, entity: post)
        pendingActions.append(action)
        
        // 4. Sync in background
        scheduleBackgroundSync()
    }
    
    func scheduleBackgroundSync() {
        queue.asyncAfter(deadline: .now() + 2) { [weak self] in
            self?.processPendingActions()
        }
    }
    
    func processPendingActions() {
        guard NetworkMonitor.isConnected else { return }
        
        for action in pendingActions {
            syncAction(action) { success in
                if success {
                    action.entity.syncStatus = .synced
                    self.removeAction(action)
                }
            }
        }
    }
}
```

**Use Case:** Social media posts, likes, comments

## Handling Stale Data

**Problem:** Local cache shows 2-hour-old feed. How do we know it's stale?

### Strategy 1: Time-Based Expiration
```swift
struct CachedData<T> {
    let data: T
    let timestamp: Date
    let ttl: TimeInterval // Time to live
    
    var isStale: Bool {
        return Date().timeIntervalSince(timestamp) > ttl
    }
}

class FeedCache {
    func getFeed() -> [Post]? {
        guard let cached = database.fetchCachedFeed() else { return nil }
        
        // Check if stale (e.g., 5 minutes old)
        if cached.isStale {
            // Trigger background refresh
            refreshFeed()
            // Still return stale data (better than nothing)
            return cached.data
        }
        
        return cached.data
    }
}
```

### Strategy 2: Version/ETag Tracking
```swift
struct SyncMetadata {
    var lastSyncVersion: String?
    var serverETag: String?
}

func refreshIfNeeded() {
    let metadata = database.fetchSyncMetadata()
    
    APIClient.checkVersion(currentVersion: metadata.lastSyncVersion) { serverVersion in
        if serverVersion != metadata.lastSyncVersion {
            // Data changed on server, pull updates
            fetchAndMergeUpdates()
        }
    }
}
```

## Conflict Resolution Basics

**Scenario:** User edits profile offline. Meanwhile, they edit same field on web.

### Approach 1: **Last-Write-Wins (Timestamp)**
```swift
struct UserProfile {
    var name: String
    var nameUpdatedAt: Date
    var email: String
    var emailUpdatedAt: Date
}

func resolveConflict(local: UserProfile, server: UserProfile) -> UserProfile {
    var resolved = UserProfile()
    
    // Compare timestamps field-by-field
    resolved.name = local.nameUpdatedAt > server.nameUpdatedAt ? local.name : server.name
    resolved.email = local.emailUpdatedAt > server.emailUpdatedAt ? local.email : server.email
    
    return resolved
}
```

### Approach 2: **Server Authority (Server Wins)**
```swift
func sync(localPost: Post, serverPost: Post) -> Post {
    // Server is always right for critical fields
    localPost.likesCount = serverPost.likesCount
    localPost.isPinned = serverPost.isPinned
    
    // Keep local changes for user-editable fields if pending
    if localPost.syncStatus == .pending {
        // Keep local content
        return localPost
    }
    
    return serverPost
}
```

## Background Sync & Retry Logic

### iOS Background Fetch
```swift
// AppDelegate
func application(_ application: UIApplication, 
                 performFetchWithCompletionHandler completion: @escaping (UIBackgroundFetchResult) -> Void) {
    SyncEngine.shared.backgroundSync { hasNewData in
        completion(hasNewData ? .newData : .noData)
    }
}

// Background task
class SyncEngine {
    func backgroundSync(completion: @escaping (Bool) -> Void) {
        let task = URLSession.shared.dataTask(with: syncRequest) { data, response, error in
            // Process sync
            completion(data != nil)
        }
        task.resume()
    }
}
```

### Exponential Backoff
```swift
class RetryManager {
    var retryCount = 0
    let maxRetries = 5
    
    func syncWithRetry(action: SyncAction) {
        attemptSync(action) { [weak self] success in
            guard let self = self else { return }
            
            if success {
                self.retryCount = 0
                return
            }
            
            // Failed - retry with backoff
            if self.retryCount < self.maxRetries {
                let delay = self.calculateBackoff()
                DispatchQueue.main.asyncAfter(deadline: .now() + delay) {
                    self.retryCount += 1
                    self.syncWithRetry(action: action)
                }
            } else {
                // Max retries reached - show error
                self.handlePermanentFailure(action)
            }
        }
    }
    
    func calculateBackoff() -> Double {
        // 2^retry * base delay
        // 1s, 2s, 4s, 8s, 16s
        let baseDelay = 1.0
        return pow(2.0, Double(retryCount)) * baseDelay
    }
}
```

## Network Detection & State Transitions

### NWPathMonitor (iOS 12+)
```swift
import Network

class NetworkMonitor {
    static let shared = NetworkMonitor()
    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "NetworkMonitor")
    
    var isConnected: Bool = false
    var connectionType: ConnectionType = .unknown
    
    enum ConnectionType {
        case wifi, cellular, unknown
    }
    
    func startMonitoring() {
        monitor.pathUpdateHandler = { [weak self] path in
            self?.isConnected = path.status == .satisfied
            
            if path.usesInterfaceType(.wifi) {
                self?.connectionType = .wifi
            } else if path.usesInterfaceType(.cellular) {
                self?.connectionType = .cellular
            }
            
            // Trigger sync when connected
            if path.status == .satisfied {
                NotificationCenter.default.post(name: .networkConnected, object: nil)
            }
        }
        
        monitor.start(queue: queue)
    }
}

// Usage
class SyncCoordinator {
    init() {
        NotificationCenter.default.addObserver(
            self, 
            selector: #selector(networkConnected), 
            name: .networkConnected, 
            object: nil
        )
    }
    
    @objc func networkConnected() {
        // WiFi? Sync images/videos
        if NetworkMonitor.shared.connectionType == .wifi {
            syncHeavyMedia()
        }
        
        // Any connection? Sync lightweight data
        syncPendingActions()
    }
}
```

### State Machine for Sync
```swift
enum SyncState {
    case idle
    case syncing
    case offline
    case error(Error)
}

class SyncStateMachine {
    private(set) var state: SyncState = .idle {
        didSet {
            handleStateChange(from: oldValue, to: state)
        }
    }
    
    func handleStateChange(from old: SyncState, to new: SyncState) {
        switch (old, new) {
        case (.offline, .idle):
            // Just came online - trigger sync
            triggerFullSync()
            
        case (_, .error):
            // Show retry UI
            showRetryBanner()
            
        case (.syncing, .idle):
            // Sync completed
            refreshUI()
            
        default:
            break
        }
    }
}
```

## Common Offline-First Mistakes

### ❌ **Mistake 1: Not Showing Local Data First**
```swift
// BAD
func loadFeed() {
    APIClient.fetchFeed { posts in
        self.displayPosts(posts)
    }
}

// GOOD
func loadFeed() {
    // Show cached data immediately
    let cachedPosts = database.fetchPosts()
    displayPosts(cachedPosts)
    
    // Then refresh from network
    APIClient.fetchFeed { newPosts in
        database.savePosts(newPosts)
        self.displayPosts(newPosts)
    }
}
```

### ❌ **Mistake 2: Not Persisting User Actions**
```swift
// BAD - Lost if app crashes
func deletePost(_ post: Post) {
    APIClient.deletePost(post.id) { success in
        if success {
            self.removeFromUI(post)
        }
    }
}

// GOOD - Persist immediately
func deletePost(_ post: Post) {
    // Mark as deleted locally
    post.isDeleted = true
    database.save(post)
    removeFromUI(post)
    
    // Sync in background
    queueDeletion(post.id)
}
```

### ❌ **Mistake 3: Ignoring Conflict Resolution**
```swift
// BAD - Blindly overwrite
func sync() {
    let serverData = APIClient.fetch()
    database.replaceAll(serverData) // Lost local changes!
}

// GOOD - Merge intelligently
func sync() {
    let localChanges = database.fetchPending()
    let serverData = APIClient.fetch()
    let merged = merge(local: localChanges, server: serverData)
    database.save(merged)
}
```

### ❌ **Mistake 4: Not Handling Partial Failures**
```swift
// BAD - All or nothing
func syncPosts(_ posts: [Post]) {
    APIClient.bulkUpload(posts) { success in
        if success {
            posts.forEach { $0.isSynced = true }
        }
    }
}

// GOOD - Individual tracking
func syncPosts(_ posts: [Post]) {
    for post in posts {
        APIClient.upload(post) { success in
            if success {
                post.isSynced = true
                database.save(post)
            } else {
                post.retryCount += 1
                scheduleRetry(post)
            }
        }
    }
}
```

### ❌ **Mistake 5: Not Considering Storage Limits**
```swift
// BAD - Unlimited growth
func savePost(_ post: Post) {
    database.save(post) // What if 100,000 posts?
}

// GOOD - Cleanup old data
func savePost(_ post: Post) {
    database.save(post)
    cleanupOldPosts(keepLast: 1000)
}

func cleanupOldPosts(keepLast: Int) {
    let allPosts = database.fetchPosts(sortedBy: "createdAt", ascending: false)
    if allPosts.count > keepLast {
        let toDelete = Array(allPosts.dropFirst(keepLast))
        database.delete(toDelete)
    }
}
```

---

# Advanced Level

## Robust Sync Engine Architecture

### **Sync Engine Components**
```swift
protocol SyncEngine {
    func enqueue(_ action: SyncAction)
    func processQueue()
    func handleConflict(_ conflict: Conflict) -> Resolution
    func rollback(_ action: SyncAction)
}

class ProductionSyncEngine: SyncEngine {
    let queue: SyncQueue
    let conflictResolver: ConflictResolver
    let networkMonitor: NetworkMonitor
    let storage: LocalStorage
    
    // Components
    private let actionSerializer = ActionSerializer()
    private let retryManager = RetryManager()
    private let batchProcessor = BatchProcessor()
    
    func enqueue(_ action: SyncAction) {
        // 1. Persist action to queue
        storage.save(action)
        
        // 2. Deduplicate
        deduplicateQueue()
        
        // 3. Prioritize
        queue.insert(action, priority: action.priority)
        
        // 4. Process if online
        if networkMonitor.isConnected {
            processQueue()
        }
    }
    
    func processQueue() {
        guard networkMonitor.isConnected else { return }
        
        let batch = queue.nextBatch(size: 10)
        
        batchProcessor.process(batch) { results in
            for (action, result) in zip(batch, results) {
                switch result {
                case .success:
                    queue.remove(action)
                    action.entity.syncStatus = .synced
                    
                case .conflict(let serverEntity):
                    let resolution = conflictResolver.resolve(
                        local: action.entity,
                        server: serverEntity
                    )
                    handleResolution(resolution, for: action)
                    
                case .failure(let error):
                    retryManager.scheduleRetry(action, error: error)
                }
            }
        }
    }
}
```

### **Priority Queue for Sync Actions**
```swift
enum SyncPriority: Int {
    case critical = 0  // Payment, booking
    case high = 1      // User-generated content
    case normal = 2    // Likes, views
    case low = 3       // Analytics, telemetry
}

class PrioritySyncQueue {
    private var queues: [SyncPriority: [SyncAction]] = [
        .critical: [],
        .high: [],
        .normal: [],
        .low: []
    ]
    
    func insert(_ action: SyncAction, priority: SyncPriority) {
        queues[priority, default: []].append(action)
    }
    
    func nextBatch(size: Int) -> [SyncAction] {
        var batch: [SyncAction] = []
        
        // Process critical first
        for priority in [SyncPriority.critical, .high, .normal, .low] {
            let available = queues[priority, default: []]
            let take = min(size - batch.count, available.count)
            batch.append(contentsOf: available.prefix(take))
            
            if batch.count >= size {
                break
            }
        }
        
        return batch
    }
}
```

## Advanced Conflict Resolution

### **Operational Transformation (for Text)**
```swift
// Example: Two users editing same document
class OperationalTransform {
    func transform(local: TextOperation, server: TextOperation) -> TextOperation {
        // If operations don't overlap, both apply
        if local.range.end < server.range.start {
            return local
        }
        
        // If server inserted before local, adjust local position
        if server.type == .insert && server.range.start < local.range.start {
            var adjusted = local
            adjusted.range.start += server.insertedLength
            adjusted.range.end += server.insertedLength
            return adjusted
        }
        
        // More complex cases...
        return resolveComplex(local, server)
    }
}

struct TextOperation {
    enum OpType { case insert, delete, replace }
    var type: OpType
    var range: Range<Int>
    var text: String
    var insertedLength: Int
}
```

### **Three-Way Merge**
```swift
class ThreeWayMerge {
    func merge(base: Entity, local: Entity, server: Entity) -> Entity {
        var result = Entity()
        
        // For each field, determine which version to use
        if local.field != base.field && server.field == base.field {
            // Only local changed
            result.field = local.field
        } else if server.field != base.field && local.field == base.field {
            // Only server changed
            result.field = server.field
        } else if local.field == server.field {
            // Both agree
            result.field = local.field
        } else {
            // Real conflict - both changed differently
            result.field = resolveConflict(local.field, server.field)
        }
        
        return result
    }
}
```

### **Field-Level Conflict Resolution**
```swift
struct Post {
    var id: String
    var content: String
    var contentModifiedAt: Date
    var likesCount: Int
    var likesModifiedAt: Date
    var isPinned: Bool
    var pinnedModifiedAt: Date
}

class FieldLevelResolver {
    func resolve(local: Post, server: Post) -> Post {
        var resolved = Post(id: local.id)
        
        // Content: last-write-wins by timestamp
        if local.contentModifiedAt > server.contentModifiedAt {
            resolved.content = local.content
            resolved.contentModifiedAt = local.contentModifiedAt
        } else {
            resolved.content = server.content
            resolved.contentModifiedAt = server.contentModifiedAt
        }
        
        // Likes: server authority (server knows truth)
        resolved.likesCount = server.likesCount
        resolved.likesModifiedAt = server.likesModifiedAt
        
        // Pinned: user preference wins
        if local.pinnedModifiedAt > server.pinnedModifiedAt {
            resolved.isPinned = local.isPinned
            resolved.pinnedModifiedAt = local.pinnedModifiedAt
        } else {
            resolved.isPinned = server.isPinned
            resolved.pinnedModifiedAt = server.pinnedModifiedAt
        }
        
        return resolved
    }
}
```

## Handling Partial Failures

### **Saga Pattern for Distributed Transactions**
```swift
class RideBookingSaga {
    enum Step {
        case reserveRide
        case chargePayment
        case notifyDriver
        case confirmBooking
    }
    
    var completedSteps: Set<Step> = []
    
    func executeBooking(_ booking: RideBooking) {
        do {
            // Step 1
            try reserveRide(booking)
            completedSteps.insert(.reserveRide)
            
            // Step 2
            try chargePayment(booking)
            completedSteps.insert(.chargePayment)
            
            // Step 3
            try notifyDriver(booking)
            completedSteps.insert(.notifyDriver)
            
            // Step 4
            try confirmBooking(booking)
            completedSteps.insert(.confirmBooking)
            
        } catch {
            // Rollback in reverse order
            rollback(booking)
        }
    }
    
    func rollback(_ booking: RideBooking) {
        if completedSteps.contains(.confirmBooking) {
            cancelBooking(booking)
        }
        if completedSteps.contains(.notifyDriver) {
            cancelDriverNotification(booking)
        }
        if completedSteps.contains(.chargePayment) {
            refundPayment(booking)
        }
        if completedSteps.contains(.reserveRide) {
            releaseRide(booking)
        }
    }
}
```

### **Idempotency for Safe Retries**
```swift
struct SyncAction {
    let id: UUID  // Unique ID for this action
    let type: ActionType
    let payload: Data
    let timestamp: Date
}

class IdempotentSync {
    func uploadPost(_ post: Post) {
        let actionId = post.localId  // Stable ID
        
        var request = URLRequest(url: endpoint)
        request.addValue(actionId.uuidString, forHTTPHeaderField: "Idempotency-Key")
        
        // Server checks: "Did I already process this UUID?"
        // If yes: return cached result (no duplicate)
        // If no: process and cache result
        
        URLSession.shared.dataTask(with: request) { data, response, error in
            // Safe to retry with same actionId
        }.resume()
    }
}
```

## Offline-First with Real-Time Features

### **Chat App: Hybrid Approach**
```swift
class ChatEngine {
    let localDB: ChatDatabase
    let websocket: WebSocket
    let syncQueue: SyncQueue
    
    // Send message
    func sendMessage(_ message: Message) {
        // 1. Save locally with pending status
        message.status = .pending
        localDB.save(message)
        updateUI(message)
        
        // 2. Try WebSocket if connected
        if websocket.isConnected {
            sendViaWebSocket(message)
        } else {
            // 3. Fall back to queue
            syncQueue.enqueue(SendMessageAction(message))
        }
    }
    
    func sendViaWebSocket(_ message: Message) {
        websocket.send(message.json) { [weak self] result in
            switch result {
            case .success(let ack):
                message.status = .sent
                message.serverId = ack.messageId
                message.timestamp = ack.serverTimestamp
                self?.localDB.update(message)
                
            case .failure:
                // WebSocket failed, queue for retry
                message.status = .pending
                self?.syncQueue.enqueue(SendMessageAction(message))
            }
        }
    }
    
    // Receive messages
    func onWebSocketMessage(_ data: Data) {
        let message = parse(data)
        
        // Check if duplicate (already in local DB)
        if localDB.exists(message.serverId) {
            return
        }
        
        // Save and display
        localDB.save(message)
        updateUI(message)
    }
    
    // On reconnect
    func onWebSocketReconnect() {
        // 1. Fetch missed messages
        let lastMessageId = localDB.lastMessageId()
        websocket.send(FetchMissedRequest(since: lastMessageId))
        
        // 2. Process pending queue
        syncQueue.processQueue()
    }
}
```

### **Live Location Tracking (Uber Driver)**
```swift
class LiveLocationTracker {
    var currentLocation: CLLocation?
    var locationBuffer: [LocationUpdate] = []
    
    func startTracking() {
        locationManager.startUpdatingLocation()
    }
    
    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        guard let location = locations.last else { return }
        currentLocation = location
        
        // Always save locally for trip history
        database.saveLocation(location)
        
        // Send to server
        if NetworkMonitor.isConnected {
            sendLocationUpdate(location)
        } else {
            // Buffer locations for later
            bufferLocation(location)
        }
    }
    
    func sendLocationUpdate(_ location: CLLocation) {
        // Throttle: only send every 5 seconds
        guard shouldSendUpdate(location) else { return }
        
        let update = LocationUpdate(
            lat: location.coordinate.latitude,
            lng: location.coordinate.longitude,
            timestamp: Date()
        )
        
        APIClient.updateLocation(update) { success in
            if !success {
                self.bufferLocation(location)
            }
        }
    }
    
    func onNetworkReconnect() {
        // Compress and send buffered locations
        let compressed = compressLocationBuffer(locationBuffer)
        APIClient.bulkUpdateLocations(compressed)
        locationBuffer.removeAll()
    }
}
```

## Performance, Battery, & Storage Trade-offs

### **Battery Optimization**
```swift
class BatteryAwareSync {
    func sync() {
        let batteryLevel = UIDevice.current.batteryLevel
        let batteryState = UIDevice.current.batteryState
        
        if batteryLevel < 0.15 && batteryState != .charging {
            // Low battery - only critical sync
            syncCriticalData()
        } else if batteryLevel < 0.30 {
            // Medium battery - skip heavy operations
            syncWithoutMedia()
        } else {
            // Good battery - full sync
            fullSync()
        }
    }
    
    func syncCriticalData() {
        // Only user-initiated actions
        syncQueue.filter { $0.priority == .critical }.forEach { sync($0) }
    }
    
    func syncWithoutMedia() {
        // Text data only, skip images/videos
        syncQueue.filter { !$0.isMediaHeavy }.forEach { sync($0) }
    }
}
```

### **Storage Management**
```swift
class StorageManager {
    func enforceStorageLimit() {
        let usedSpace = calculateUsedSpace()
        let maxAllowedSpace: Int64 = 500_000_000  // 500 MB
        
        if usedSpace > maxAllowedSpace {
            freeUpSpace(target: maxAllowedSpace * 0.8)  // Free to 80%
        }
    }
    
    func freeUpSpace(target: Int64) {
        // Strategy 1: Delete old media cache
        deleteOldMedia(olderThan: .days(30))
        
        // Strategy 2: Keep only recent messages
        if calculateUsedSpace() > target {
            keepOnlyRecentMessages(days: 90)
        }
        
        // Strategy 3: Compress old data
        if calculateUsedSpace() > target {
            compressOldEntities()
        }
    }
    
    func deleteOldMedia(olderThan interval: TimeInterval) {
        let cutoff = Date().addingTimeInterval(-interval)
        let oldMedia = database.fetchMedia(before: cutoff)
        
        for media in oldMedia {
            // Keep metadata, delete file
            try? FileManager.default.removeItem(at: media.fileURL)
            media.isDownloaded = false
            database.save(media)
        }
    }
}
```

### **Performance: Lazy Loading & Pagination**
```swift
class OptimizedFeedLoader {
    let pageSize = 20
    var currentPage = 0
    
    func loadInitialFeed() {
        // Load first page from local DB
        let posts = database.fetchPosts(limit: pageSize, offset: 0)
        display(posts)
        
        // Background refresh from server
        refreshFromServer()
    }
    
    func loadNextPage() {
        currentPage += 1
        let posts = database.fetchPosts(limit: pageSize, offset: currentPage * pageSize)
        append(posts)
    }
    
    func refreshFromServer() {
        APIClient.fetchLatestPosts(limit: pageSize) { newPosts in
            // Merge intelligently
            let merged = self.mergeWithLocal(newPosts)
            self.database.savePosts(merged)
            self.display(merged)
        }
    }
}
```

## Security & Privacy for Offline Data

### **Encryption at Rest**
```swift
import CryptoKit

class SecureStorage {
    func saveEncrypted(_ data: Data, forKey key: String) {
        // Generate encryption key from user's passcode/biometrics
        let encryptionKey = deriveKey(from: userPasscode)
        
        // Encrypt data
        let sealed = try? AES.GCM.seal(data, using: encryptionKey)
        
        // Save encrypted data
        UserDefaults.standard.set(sealed?.combined, forKey: key)
    }
    
    func loadEncrypted(forKey key: String) -> Data? {
        guard let combined = UserDefaults.standard.data(forKey: key) else {
            return nil
        }
        
        let encryptionKey = deriveKey(from: userPasscode)
        let sealedBox = try? AES.GCM.SealedBox(combined: combined)
        
        return try? AES.GCM.open(sealedBox!, using: encryptionKey)
    }
    
    func deriveKey(from password: String) -> SymmetricKey {
        let salt = "app-specific-salt".data(using: .utf8)!
        let passwordData = password.data(using: .utf8)!
        
        // Use PBKDF2 to derive key
        // In production, use KeyChain for key storage
        return SymmetricKey(data: SHA256.hash(data: passwordData + salt))
    }
}
```

### **Sensitive Data Policies**
```swift
enum DataSensitivity {
    case public       // Can cache indefinitely
    case private      // Encrypt, delete on logout
    case sensitive    // Never cache, always fetch fresh
}

class SensitiveDataManager {
    func save(_ entity: Entity) {
        switch entity.sensitivity {
        case .public:
            database.save(entity)
            
        case .private:
            encryptedDatabase.save(entity)
            
        case .sensitive:
            // Only in-memory, never persist
            inMemoryCache.store(entity)
        }
    }
    
    func onUserLogout() {
        // Delete all private data
        encryptedDatabase.deleteAll()
        database.delete(where: "sensitivity == private")
        
        // Clear sensitive in-memory cache
        inMemoryCache.clear()
    }
}
```

### **Data Retention Policies**
```swift
class DataRetentionPolicy {
    func enforcePolicy() {
        // GDPR: Delete user data after account deletion
        deleteDataForDeletedUsers()
        
        // PCI: Don't cache credit card details
        deleteCreditCardCache()
        
        // HIPAA: Encrypt health data, expire after 90 days
        expireHealthData(olderThan: .days(90))
    }
    
    func deleteCreditCardCache() {
        database.delete(where: "type == 'creditCard'")
    }
}
```

## How Large-Scale Apps Implement Offline-First

### **Uber: Ride Booking Flow**
```swift
class UberOfflineFlow {
    func requestRide(_ request: RideRequest) {
        // 1. Validate locally
        guard hasValidPayment() && hasPickupLocation() else {
            showError()
            return
        }
        
        // 2. Create local booking immediately
        let booking = createLocalBooking(request)
        booking.status = .requesting
        database.save(booking)
        showRequestingUI(booking)
        
        // 3. Try to request via API
        if NetworkMonitor.isConnected {
            APIClient.requestRide(request) { result in
                switch result {
                case .success(let confirmedBooking):
                    booking.status = .confirmed
                    booking.driverId = confirmedBooking.driverId
                    booking.eta = confirmedBooking.eta
                    self.database.save(booking)
                    self.showConfirmedUI(booking)
                    
                case .failure:
                    booking.status = .failed
                    self.database.save(booking)
                    self.showRetryUI(booking)
                }
            }
        } else {
            // 4. Offline - queue and show helpful message
            booking.status = .pending
            syncQueue.enqueue(RequestRideAction(booking))
            showOfflineMessage("Will request when connected")
        }
    }
}
```

### **Instagram: Post Upload**
```swift
class InstagramOfflinePost {
    func uploadPost(_ post: Post, images: [UIImage]) {
        // 1. Generate local ID
        post.localId = UUID()
        post.status = .uploading
        
        // 2. Save to local database
        database.save(post)
        
        // 3. Compress and cache images
        let compressed = images.map { compress($0, quality: 0.8) }
        cacheImages(compressed, for: post.localId)
        
        // 4. Show in feed immediately with "uploading" indicator
        feed.insert(post, at: 0)
        
        // 5. Upload in background
        uploadManager.enqueue(post) { [weak self] result in
            switch result {
            case .success(let serverPost):
                // Replace local ID with server ID
                post.serverId = serverPost.id
                post.status = .published
                self?.database.save(post)
                self?.deleteCachedImages(for: post.localId)
                
            case .failure:
                post.status = .failed
                self?.database.save(post)
                self?.showRetryOption(for: post)
            }
        }
    }
}
```

### **Google Maps: Offline Navigation**
```swift
class MapsOfflineNavigation {
    func downloadRegion(_ region: MapRegion) {
        // 1. Download map tiles
        let tiles = tileServer.fetchTiles(for: region)
        tiles.forEach { database.save($0) }
        
        // 2. Download POI data
        let pois = poiServer.fetch(for: region)
        pois.forEach { database.save($0) }
        
        // 3. Download routing graph
        let routingGraph = routingServer.fetchGraph(for: region)
        database.save(routingGraph)
        
        // Mark region as available offline
        region.isOfflineReady = true
        database.save(region)
    }
    
    func navigate(from start: Coordinate, to end: Coordinate) {
        // 1. Check if route is in cached region
        let route: Route
        if database.hasOfflineData(covering: [start, end]) {
            // Use local routing engine
            route = localRoutingEngine.calculateRoute(from: start, to: end)
        } else if NetworkMonitor.isConnected {
            // Fetch from server
            route = APIClient.fetchRoute(from: start, to: end)
            // Cache for offline use
            database.saveRoute(route)
        } else {
            showError("Route not available offline")
            return
        }
        
        // 2. Start turn-by-turn navigation
        startNavigation(route)
    }
}
```

---

# Interview Preparation

## Common Interview Questions

### **Q1: Design an offline-first social media feed (like Instagram/Twitter)**

**Structured Answer:**

**1. Clarify Requirements**
- Read-only feed or can users post/like?
- Media (images/videos) or text-only?
- How much offline storage? (100 posts? 1000?)
- Real-time updates needed?

**2. High-Level Architecture**
```
┌─────────────────┐
│  UI Layer       │
│  (Feed View)    │
└────────┬────────┘
         │
┌────────▼────────┐
│ Feed Manager    │  ← Coordinates everything
└────┬───────┬────┘
     │       │
┌────▼────┐ ┌▼────────┐
│ Local   │ │ Sync    │
│ Database│ │ Engine  │
└────┬────┘ └┬────────┘
     │       │
     └───┬───┘
         │
    ┌────▼────┐
    │ Network │
    │ Layer   │
    └─────────┘
```

**3. Data Model**
```swift
class Post {
    var id: String           // Server ID (nil if not synced)
    var localId: UUID        // Local unique ID
    var authorId: String
    var content: String
    var imageURL: String?
    var createdAt: Date
    var likesCount: Int
    var isLikedByUser: Bool
    
    // Sync metadata
    var syncStatus: SyncStatus  // synced, pending, failed
    var lastModified: Date
}

enum SyncStatus {
    case synced
    case pending
    case failed(Error)
}
```

**4. Read Flow (Load Feed)**
```swift
func loadFeed() {
    // Step 1: Immediately show local cache
    let cachedPosts = database.fetchPosts(limit: 20, offset: 0)
    displayPosts(cachedPosts)
    showCacheIndicator()  // "Showing cached content"
    
    // Step 2: Refresh from server if online
    guard NetworkMonitor.isConnected else { return }
    
    APIClient.fetchFeed { [weak self] result in
        switch result {
        case .success(let serverPosts):
            // Merge: keep local pending posts, add new server posts
            let merged = self?.merge(local: cachedPosts, server: serverPosts)
            self?.database.savePosts(merged)
            self?.displayPosts(merged)
            self?.hideCacheIndicator()
            
        case .failure:
            // Silent failure - already showing cache
            break
        }
    }
}
```

**5. Write Flow (Create Post)**
```swift
func createPost(content: String, image: UIImage?) {
    // Step 1: Create local post immediately
    let post = Post(
        localId: UUID(),
        content: content,
        createdAt: Date(),
        syncStatus: .pending
    )
    database.save(post)
    
    // Step 2: Show in feed instantly
    feed.insert(post, at: 0)
    showUploadingIndicator(for: post)
    
    // Step 3: Upload in background
    if let image = image {
        let compressed = compress(image)
        uploadImage(compressed) { imageURL in
            post.imageURL = imageURL
            self.uploadPost(post)
        }
    } else {
        uploadPost(post)
    }
}

func uploadPost(_ post: Post) {
    APIClient.createPost(post) { result in
        switch result {
        case .success(let serverPost):
            post.id = serverPost.id
            post.syncStatus = .synced
            database.save(post)
            hideUploadingIndicator(for: post)
            
        case .failure(let error):
            post.syncStatus = .failed(error)
            database.save(post)
            showRetryButton(for: post)
        }
    }
}
```

**6. Sync Strategy**
- Pull: Every 5 minutes, fetch new posts since last sync
- Push: Queue pending actions, retry with exponential backoff
- Conflict: Server authority for likes/comments, user authority for own posts

**7. Storage Limits**
- Keep last 1000 posts in database
- Cache images for last 100 posts
- Delete older data automatically

**8. Trade-offs**
| Decision | Pro | Con |
|----------|-----|-----|
| Show cache first | Instant UX | Might show stale data |
| SQLite vs Realm | Familiarity | Learning curve |
| Aggressive caching | Works offline | Storage usage |

---

### **Q2: Design offline sync for a ride-hailing app (like Uber)**

**Structured Answer:**

**1. Critical User Flows**
- View nearby drivers (read)
- Request ride (write)
- Track ride in progress (real-time)
- Rate driver (write)

**2. Offline Priorities**
```
High Priority:
✅ View ride history (offline)
✅ View saved addresses (offline)
✅ Request ride → queue if offline

Low Priority:
❌ Real-time driver location (requires network)
❌ Live ETA updates (requires network)
```

**3. Request Ride Flow**
```swift
func requestRide(pickup: Location, destination: Location) {
    // Step 1: Validate locally
    guard validatePickup(pickup) && hasPaymentMethod() else {
        showError("Invalid request")
        return
    }
    
    // Step 2: Create pending booking
    let booking = RideBooking(
        id: UUID(),
        pickup: pickup,
        destination: destination,
        status: .requesting,
        createdAt: Date()
    )
    database.save(booking)
    showRequestingUI()
    
    // Step 3: Try to book via API
    if NetworkMonitor.isConnected {
        APIClient.requestRide(booking) { result in
            switch result {
            case .success(let ride):
                booking.status = .confirmed
                booking.driverId = ride.driverId
                booking.driver = ride.driver
                database.save(booking)
                showConfirmedUI(ride)
                
            case .failure:
                booking.status = .failed
                showError("Couldn't find driver. Retry?")
            }
        }
    } else {
        // Step 4: Queue for when online
        syncQueue.enqueue(RequestRideAction(booking))
        showOfflineUI("Will request when connected")
    }
}
```

**4. Edge Cases**
- **Conflict**: User requests ride offline on phone, cancels on web → Use timestamps
- **Partial failure**: Driver accepts but notification fails → Retry notification
- **Stale data**: Cached driver location is 5 mins old → Show warning

**5. Real-Time Tracking**
- **Requires network** - can't be offline-first
- Fallback: Show last known location from cache
- Display: "Last updated 5 mins ago"

---

### **Q3: How would you handle conflicts in an offline-first chat app?**

**Structured Answer:**

**1. Conflict Scenarios**
```
Scenario A: User sends message offline, same message ID used twice
→ Solution: Use UUID for local messages, replace with server ID on sync

Scenario B: User deletes message offline, other user edits it
→ Solution: Deletion wins (most destructive action)

Scenario C: Two devices send message at same time
→ Solution: Server assigns timestamp, order by that
```

**2. Message State Machine**
```swift
enum MessageStatus {
    case pending      // Sending from local device
    case sent         // Server received
    case delivered    // Server delivered to recipient
    case read         // Recipient read
    case failed       // Send failed
}
```

**3. Conflict Resolution Strategy**
```swift
func resolveConflict(local: Message, server: Message) -> Message {
    // Deletion always wins
    if local.isDeleted || server.isDeleted {
        return markDeleted(local)
    }
    
    // Server timestamp is source of truth for ordering
    var resolved = local
    resolved.timestamp = server.timestamp
    resolved.serverId = server.id
    
    // Content: last-write-wins by editedAt
    if server.editedAt > local.editedAt {
        resolved.content = server.content
    }
    
    // Status: server authority
    resolved.status = server.status
    
    return resolved
}
```

**4. Idempotency**
```swift
func sendMessage(_ message: Message) {
    // Use stable client-generated ID
    let clientId = "\(userId)_\(message.localId)"
    
    APIClient.sendMessage(message, clientId: clientId) { result in
        // Server deduplicates by clientId
        // Retry is safe
    }
}
```

---

## How to Explain Trade-offs Out Loud

### **Framework: SCOT**
**S**ituation → **C**hoices → **O**utcome → **T**rade-offs

**Example:**
> "In the Instagram feed design (Situation), I had to choose between showing cached data immediately vs waiting for fresh data (Choices). I chose to show cache first because it gives instant feedback, which is critical for perceived performance (Outcome). The trade-off is that users might see stale posts for a few seconds, but we mitigate this with a refresh indicator and fade-in animation for new content (Trade-offs)."

### **Common Trade-offs to Articulate**

**1. Storage vs Freshness**
- More cache = works better offline
- More cache = disk space issues
- **Say:** "I'd cache last 1000 posts (about 50MB) and delete older data automatically"

**2. Battery vs Real-time**
- Frequent polling = battery drain
- Less polling = stale data
- **Say:** "Use WebSockets for real-time when critical (chat), polling every 5 mins for feed"

**3. Consistency vs Availability**
- Strong consistency = wait for server confirmation
- Availability = show optimistic update
- **Say:** "For likes, I'd use optimistic UI (instant feedback) with eventual sync, since a failed like isn't critical"

**4. Simplicity vs Conflict Resolution**
- Last-write-wins = simple but loses data
- Operational transform = complex but mergeable
- **Say:** "For comments, last-write-wins is fine. For collaborative docs, need operational transform"

---

## Common Follow-Up Questions

### **Q: "What if the user has no internet for 3 days?"**

**Answer:**
> "Great question. Here's my approach:
> 1. **Read operations**: Still work perfectly - user sees last synced data
> 2. **Write operations**: Queue locally, but warn user after 24 hours: 'Changes not synced. Will upload when connected.'
> 3. **Storage**: Limit pending queue to 100 actions to avoid unbounded growth
> 4. **Expiry**: Mark data older than 7 days as 'stale' with visual indicator
> 5. **On reconnect**: Use batch API to sync all 3 days of changes efficiently"

### **Q: "How do you prevent data loss during conflicts?"**

**Answer:**
> "I use a multi-layered approach:
> 1. **Preserve both versions**: Never discard data during conflict, store both
> 2. **Field-level merge**: Merge non-conflicting fields automatically
> 3. **User prompt**: For true conflicts, show UI: 'Your version / Server version / Keep both'
> 4. **Audit log**: Keep conflict history for debugging
> 5. **Backup**: Server stores previous versions for 30 days"

### **Q: "What about security for cached data?"**

**Answer:**
> "Security has multiple layers:
> 1. **Encryption at rest**: Use iOS Data Protection API (NSFileProtectionComplete)
> 2. **Sensitive data**: Never cache PII offline (credit cards, SSN)
> 3. **Time limits**: Expire sensitive cache after 24 hours
> 4. **Device lock**: Clear cache when device hasn't been unlocked in 7 days
> 5. **Logout**: Wipe all cached data on logout
> 6. **Jailbreak detection**: Refuse to cache if device is jailbroken"

### **Q: "How do you test offline-first behavior?"**

**Answer:**
> "Testing strategy:
> 1. **Unit tests**: Mock network layer, test sync logic
> 2. **Integration tests**: Use network link conditioner (Apple tool) to simulate poor network
> 3. **Manual tests**: 
>    - Airplane mode on/off during operations
>    - Kill app mid-sync
>    - Switch wifi/cellular during upload
> 4. **Chaos testing**: Randomly drop 30% of requests
> 5. **Monkey testing**: Random user actions for 1 hour offline
> 6. **Beta testing**: Recruit users with poor connectivity"

### **Q: "What metrics would you track for offline-first?"**

**Answer:**
> "Key metrics:
> 1. **Sync success rate**: % of queued actions successfully synced
> 2. **Time to sync**: How long until offline changes appear on server
> 3. **Conflict rate**: How often conflicts occur
> 4. **Cache hit rate**: % of reads served from cache vs API
> 5. **Storage usage**: Average disk space per user
> 6. **Perceived performance**: Time from app open to UI displayed
> 7. **Error rate**: % of operations that fail permanently (not just queued)"

---

## Interview Practice Template

**When asked: "Design [X] with offline-first"**

**1. Clarify (2 minutes)**
- What are the critical user flows?
- Read-heavy or write-heavy?
- Real-time requirements?
- Storage constraints?

**2. High-Level Design (3 minutes)**
- Draw architecture diagram
- Identify layers: UI, Local Storage, Sync Engine, Network

**3. Deep Dive (10 minutes)**
- Data model
- Read flow (cache-first)
- Write flow (optimistic UI + queue)
- Conflict resolution
- Edge cases

**4. Trade-offs (3 minutes)**
- Storage vs freshness
- Consistency vs availability
- Battery vs real-time
- Complexity vs features

**5. Follow-ups (5 minutes)**
- Metrics
- Testing
- Security
- Scale

---

## Key Takeaways for Interviews

### ✅ **Do**
- Always show local data first
- Queue write operations
- Use optimistic UI for instant feedback
- Plan for conflict resolution upfront
- Mention storage limits
- Discuss battery impact
- Think about failure cases

### ❌ **Don't**
- Don't make users wait for network
- Don't lose user data on conflicts
- Don't cache indefinitely (storage limits)
- Don't ignore security for offline data
- Don't forget idempotency for retries
- Don't assume network is reliable

### 🎯 **Key Phrases to Use**
- "I'd show cached data immediately for instant perceived performance"
- "Queue this action with exponential backoff retry"
- "Use last-write-wins for this field since conflicts are rare"
- "Limit cache to 500MB to avoid storage issues"
- "Mark data older than 7 days as stale"
- "Encrypt sensitive data at rest using iOS Data Protection"
- "Batch sync every 5 minutes to save battery"

---

## Summary: Offline-First Principles

1. **Local-First**: Local storage is source of truth, sync to server asynchronously
2. **Optimistic UI**: Show user actions immediately, sync in background
3. **Queue & Retry**: Network failures are normal, queue and retry intelligently
4. **Conflict Resolution**: Plan for conflicts, never lose data
5. **Storage Limits**: Manage cache size, delete old data
6. **Battery Aware**: Batch operations, respect battery level
7. **Security**: Encrypt offline data, expire sensitive cache
8. **User Trust**: Show sync status, be transparent about failures

**Remember:** The best offline-first app is one where the user never notices they're offline. That's the gold standard.
