# Swift Concurrency: Complete Deep Dive (GCD, OperationQueue, async/await, Actors)

## 🎈 Level 0: Complete Beginner - What is Concurrency?

### Imagine a Restaurant Kitchen

**Without Concurrency (One Chef):**
1. Chef takes order 1 → cooks it → serves it (5 minutes)
2. Chef takes order 2 → cooks it → serves it (5 minutes)
3. Chef takes order 3 → cooks it → serves it (5 minutes)

Total time: 15 minutes. Customers are waiting forever!

**With Concurrency (Multiple Chefs/Smart Organization):**
1. Chef 1 cooks order 1 (5 minutes)
2. Chef 2 cooks order 2 (5 minutes) - **at the same time**
3. Chef 3 cooks order 3 (5 minutes) - **at the same time**

Total time: 5 minutes! Everyone is happy!

### In iOS Apps

**Without Concurrency:**
```swift
// Download image (takes 3 seconds)
let image = downloadImage()  // App FREEZES for 3 seconds!

// Update UI
imageView.image = image
```

Your app freezes, user can't scroll, tap, or do anything. App Store reviews: ⭐️ "Terrible app, always freezing!"

**With Concurrency:**
```swift
// Download image on background thread
DispatchQueue.global().async {
    let image = downloadImage()  // 3 seconds, but app doesn't freeze!
    
    // Update UI on main thread
    DispatchQueue.main.async {
        imageView.image = image
    }
}
```

App remains smooth, user can scroll and tap. Reviews: ⭐️⭐️⭐️⭐️⭐️ "So smooth!"

### The Golden Rule

**Main Thread = UI Thread**
- Only use main thread for UI updates
- NEVER block main thread with slow operations
- All network calls, file operations, heavy calculations must be on background threads

***

## 🌟 Part 1: Grand Central Dispatch (GCD) - The Foundation

GCD is Apple's low-level API for managing concurrent operations.

### What Are Queues?

Think of queues as **lines at a theme park**:

**Serial Queue = Single-file line:**
```
Person 1 → Person 2 → Person 3
(one at a time, in order)
```

**Concurrent Queue = Multiple lines:**
```
Person 1 → Person 4
Person 2 → Person 5  (all at once!)
Person 3 → Person 6
```

### The Basic Queues

```swift
// 1. Main Queue - UI work ONLY (Serial)
DispatchQueue.main.async {
    self.label.text = "Updated!"  // Update UI
}

// 2. Global Queue - Background work (Concurrent)
DispatchQueue.global().async {
    let data = downloadData()  // Heavy work
}

// 3. Custom Queue - Your own queue
let myQueue = DispatchQueue(label: "com.myapp.myqueue")
```

### Real Example: Downloading an Image

```swift
func loadImage(from url: URL) {
    // Step 1: Go to background thread
    DispatchQueue.global(qos: .userInitiated).async {
        // Step 2: Download image (slow operation)
        guard let data = try? Data(contentsOf: url),
              let image = UIImage(data: data) else {
            return
        }
        
        // Step 3: Go back to main thread for UI update
        DispatchQueue.main.async {
            self.imageView.image = image
        }
    }
}
```

**Why this works:**
1. Download happens on background thread (app doesn't freeze)
2. UI update happens on main thread (required by iOS)

### Quality of Service (QoS) Levels

QoS tells the system how important your task is:

```swift
// 1. User Interactive - Highest priority
// Use for: UI updates, animations
DispatchQueue.global(qos: .userInteractive).async {
    // Critical UI work
}

// 2. User Initiated - High priority
// Use for: User requested actions (button tap)
DispatchQueue.global(qos: .userInitiated).async {
    let data = fetchUserData()
}

// 3. Default - Normal priority
DispatchQueue.global(qos: .default).async {
    // General work
}

// 4. Utility - Low priority
// Use for: Downloads, processing
DispatchQueue.global(qos: .utility).async {
    processLargeFile()
}

// 5. Background - Lowest priority
// Use for: Cleanup, maintenance
DispatchQueue.global(qos: .background).async {
    cleanupOldFiles()
}
```

### Serial vs Concurrent Queues

**Serial Queue - One at a time:**
```swift
let serialQueue = DispatchQueue(label: "serial")

serialQueue.async { print("Task 1 start") }  // Runs first
serialQueue.async { print("Task 2 start") }  // Waits for Task 1
serialQueue.async { print("Task 3 start") }  // Waits for Task 2

// Output (always in order):
// Task 1 start
// Task 2 start
// Task 3 start
```

**Concurrent Queue - All at once:**
```swift
let concurrentQueue = DispatchQueue(label: "concurrent", 
                                   attributes: .concurrent)

concurrentQueue.async { print("Task 1 start") }
concurrentQueue.async { print("Task 2 start") }
concurrentQueue.async { print("Task 3 start") }

// Output (random order!):
// Task 2 start
// Task 1 start
// Task 3 start
```

### sync vs async

**async - Don't wait, continue immediately:**
```swift
print("Before")

DispatchQueue.global().async {
    print("Inside async")
}

print("After")

// Output:
// Before
// After
// Inside async  (happens later)
```

**sync - Wait until finished:**
```swift
print("Before")

DispatchQueue.global().sync {
    print("Inside sync")
}

print("After")

// Output:
// Before
// Inside sync  (must finish first)
// After
```

⚠️ **WARNING: NEVER use sync on main queue from main thread - DEADLOCK!**

```swift
// On main thread - DON'T DO THIS!
DispatchQueue.main.sync {  // 💥 DEADLOCK - App freezes forever!
    print("This never prints")
}
```

***

## 🚀 Level 2: Intermediate - Advanced GCD Patterns

### Pattern 1: Dispatch Groups

**Problem:** How do you wait for multiple async tasks to complete?

```swift
let group = DispatchGroup()

// Task 1
group.enter()
DispatchQueue.global().async {
    downloadImage1()
    group.leave()
}

// Task 2
group.enter()
DispatchQueue.global().async {
    downloadImage2()
    group.leave()
}

// Task 3
group.enter()
DispatchQueue.global().async {
    downloadImage3()
    group.leave()
}

// Wait for all tasks
group.notify(queue: .main) {
    print("All downloads complete!")
    self.displayImages()
}
```

**Real-world example: Loading a profile screen**

```swift
func loadProfileScreen(userId: String) {
    let group = DispatchGroup()
    var user: User?
    var posts: [Post]?
    var followers: [User]?
    
    // Load user data
    group.enter()
    fetchUser(id: userId) { result in
        user = result
        group.leave()
    }
    
    // Load posts
    group.enter()
    fetchPosts(for: userId) { result in
        posts = result
        group.leave()
    }
    
    // Load followers
    group.enter()
    fetchFollowers(for: userId) { result in
        followers = result
        group.leave()
    }
    
    // All data loaded - update UI
    group.notify(queue: .main) {
        guard let user = user, let posts = posts, let followers = followers else {
            self.showError()
            return
        }
        self.displayProfile(user: user, posts: posts, followers: followers)
    }
}
```

### Pattern 2: Dispatch Barriers

**Problem:** Reading and writing to the same data from multiple threads = CRASH!

**Solution:** Barriers ensure exclusive access:

```swift
class ThreadSafeArray<T> {
    private var array: [T] = []
    private let queue = DispatchQueue(label: "array.queue", 
                                     attributes: .concurrent)
    
    // Multiple reads can happen at once
    func get(at index: Int) -> T? {
        queue.sync {
            return array.indices.contains(index) ? array[index] : nil
        }
    }
    
    // Write must be exclusive (barrier)
    func append(_ element: T) {
        queue.async(flags: .barrier) {
            self.array.append(element)
        }
    }
}
```

**How barriers work:**
```
Time →
Thread 1: Read → Read → Read
Thread 2: Read → Read → Read
Thread 3:       WRITE (barrier - blocks all others)
Thread 4:               Read → Read
```

### Pattern 3: DispatchWorkItem

**Problem:** Need to cancel a task?

```swift
class SearchViewController: UIViewController {
    private var searchWorkItem: DispatchWorkItem?
    
    func searchTextDidChange(_ text: String) {
        // Cancel previous search
        searchWorkItem?.cancel()
        
        // Create new search task
        let workItem = DispatchWorkItem {
            // Perform search
            let results = self.performSearch(text)
            
            DispatchQueue.main.async {
                self.displayResults(results)
            }
        }
        
        searchWorkItem = workItem
        
        // Delay search by 0.5 seconds (debouncing)
        DispatchQueue.global().asyncAfter(deadline: .now() + 0.5, 
                                         execute: workItem)
    }
}
```

### Pattern 4: DispatchSemaphore

**Problem:** Limit number of concurrent operations

```swift
class ImageDownloader {
    // Only 3 downloads at a time
    private let semaphore = DispatchSemaphore(value: 3)
    
    func downloadImages(urls: [URL]) {
        for url in urls {
            DispatchQueue.global().async {
                self.semaphore.wait()  // Block if 3 downloads active
                
                // Download image
                self.download(url)
                
                self.semaphore.signal()  // Release one slot
            }
        }
    }
}
```

***

## 🎯 Part 2: OperationQueue - High-Level Abstraction

OperationQueue is built on top of GCD with extra features:

### Basic Operation

```swift
// Create operation
let operation = BlockOperation {
    print("Doing work...")
    Thread.sleep(forTimeInterval: 2)
    print("Work done!")
}

// Create queue
let queue = OperationQueue()

// Add operation
queue.addOperation(operation)
```

### Why OperationQueue vs GCD?

| Feature | GCD | OperationQueue |
|---------|-----|----------------|
| Cancellation | ❌ Limited | ✅ Easy |
| Dependencies | ❌ Manual | ✅ Built-in |
| KVO | ❌ No | ✅ Yes |
| Max concurrent | ❌ System decides | ✅ You control |
| Reusability | ❌ One-time | ✅ Reusable objects |

### Real Example: Image Processing Pipeline

```swift
class ImageProcessor {
    let queue = OperationQueue()
    
    func processImages(_ images: [UIImage]) {
        queue.maxConcurrentOperationCount = 2  // Only 2 at a time
        
        for (index, image) in images.enumerated() {
            // Step 1: Resize
            let resizeOp = BlockOperation {
                let resized = self.resize(image)
                self.cache.store(resized, key: "resized_\(index)")
            }
            
            // Step 2: Apply filter (depends on resize)
            let filterOp = BlockOperation {
                let resized = self.cache.get(key: "resized_\(index)")!
                let filtered = self.applyFilter(resized)
                self.cache.store(filtered, key: "filtered_\(index)")
            }
            filterOp.addDependency(resizeOp)  // Wait for resize!
            
            // Step 3: Save (depends on filter)
            let saveOp = BlockOperation {
                let filtered = self.cache.get(key: "filtered_\(index)")!
                self.save(filtered, name: "image_\(index)")
            }
            saveOp.addDependency(filterOp)  // Wait for filter!
            
            queue.addOperations([resizeOp, filterOp, saveOp], 
                              waitUntilFinished: false)
        }
    }
}
```

**Dependency chain:**
```
Resize → Filter → Save
  ↓        ↓       ↓
 2sec    3sec    1sec

Total time: 6 seconds (sequential)
```

### Custom Operation Subclass

```swift
class DownloadOperation: Operation {
    let url: URL
    var downloadedData: Data?
    
    init(url: URL) {
        self.url = url
        super.init()
    }
    
    override func main() {
        // Check if cancelled
        guard !isCancelled else { return }
        
        // Perform download
        do {
            downloadedData = try Data(contentsOf: url)
        } catch {
            print("Download failed: \(error)")
        }
        
        // Check again after download
        guard !isCancelled else {
            downloadedData = nil
            return
        }
    }
}

// Usage
let downloadOp = DownloadOperation(url: imageURL)
let processOp = BlockOperation {
    guard let data = downloadOp.downloadedData else { return }
    processImage(data)
}
processOp.addDependency(downloadOp)

queue.addOperations([downloadOp, processOp], waitUntilFinished: false)
```

### Cancellation

```swift
class VideoProcessor {
    let queue = OperationQueue()
    
    func processVideo(_ video: Video) {
        let operation = BlockOperation {
            for frame in video.frames {
                // Check if cancelled
                if operation.isCancelled {
                    print("Processing cancelled")
                    return
                }
                
                processFrame(frame)
            }
        }
        
        queue.addOperation(operation)
        
        // Cancel after 5 seconds
        DispatchQueue.main.asyncAfter(deadline: .now() + 5) {
            operation.cancel()
        }
    }
}
```

***

## 💎 Part 3: Modern Swift Concurrency (async/await)

Introduced in Swift 5.5, this is the **future of Swift concurrency**.

### Why async/await is Better

**Old way (GCD):**
```swift
func fetchUser(completion: @escaping (User?) -> Void) {
    DispatchQueue.global().async {
        let data = downloadData()
        DispatchQueue.main.async {
            let user = parseUser(data)
            completion(user)
        }
    }
}

// Usage - callback hell!
fetchUser { user in
    guard let user = user else { return }
    fetchPosts(for: user) { posts in
        guard let posts = posts else { return }
        fetchComments(for: posts) { comments in
            // So many nested callbacks!
        }
    }
}
```

**New way (async/await):**
```swift
func fetchUser() async throws -> User {
    let data = try await downloadData()
    let user = try parseUser(data)
    return user
}

// Usage - linear, readable!
Task {
    do {
        let user = try await fetchUser()
        let posts = try await fetchPosts(for: user)
        let comments = try await fetchComments(for: posts)
        // So clean!
    } catch {
        print("Error: \(error)")
    }
}
```

### Basic async/await

```swift
// Define async function
func fetchData() async -> Data {
    // Simulate network delay
    try? await Task.sleep(nanoseconds: 1_000_000_000)  // 1 second
    return Data()
}

// Call async function
Task {
    let data = await fetchData()
    print("Got data: \(data)")
}
```

### async/await with throws

```swift
enum NetworkError: Error {
    case badURL
    case noData
}

func fetchUser(id: String) async throws -> User {
    guard let url = URL(string: "https://api.com/user/\(id)") else {
        throw NetworkError.badURL
    }
    
    let (data, _) = try await URLSession.shared.data(from: url)
    
    guard !data.isEmpty else {
        throw NetworkError.noData
    }
    
    let user = try JSONDecoder().decode(User.self, from: data)
    return user
}

// Usage
Task {
    do {
        let user = try await fetchUser(id: "123")
        print("User: \(user.name)")
    } catch {
        print("Error: \(error)")
    }
}
```

### Task - The Building Block

```swift
// Create a task
let task = Task {
    let user = try await fetchUser(id: "123")
    return user
}

// Wait for result
if let user = try? await task.value {
    print("User: \(user.name)")
}

// Cancel task
task.cancel()
```

### Task Priority

```swift
// High priority
Task(priority: .high) {
    await updateUI()
}

// Low priority
Task(priority: .low) {
    await syncData()
}

// Background
Task(priority: .background) {
    await cleanupCache()
}
```

### Parallel Execution with async let

```swift
func loadDashboard() async throws {
    // All three download at the same time!
    async let user = fetchUser()
    async let posts = fetchPosts()
    async let followers = fetchFollowers()
    
    // Wait for all results
    let (userData, postsData, followersData) = try await (user, posts, followers)
    
    displayDashboard(user: userData, posts: postsData, followers: followersData)
}
```

**Timing comparison:**
```
Sequential:
fetchUser()     → 1 sec
fetchPosts()    → 1 sec
fetchFollowers() → 1 sec
Total: 3 seconds

Parallel (async let):
fetchUser()     → 1 sec ┐
fetchPosts()    → 1 sec ├─ All at once!
fetchFollowers() → 1 sec ┘
Total: 1 second (3x faster!)
```

### TaskGroup - Dynamic Parallel Work

```swift
func downloadAllImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage.self) { group in
        // Add tasks for each URL
        for url in urls {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                return UIImage(data: data)!
            }
        }
        
        // Collect results
        var images: [UIImage] = []
        for try await image in group {
            images.append(image)
        }
        return images
    }
}
```

### MainActor - UI Thread Safety

```swift
@MainActor
class ViewModel: ObservableObject {
    @Published var users: [User] = []
    @Published var isLoading = false
    
    // This ALWAYS runs on main thread
    func loadUsers() async {
        isLoading = true
        
        // Network call on background
        let fetchedUsers = await fetchUsersFromAPI()
        
        // UI update automatically on main thread
        users = fetchedUsers
        isLoading = false
    }
}
```

**@MainActor ensures:**
- All properties accessed on main thread
- All methods run on main thread
- No more crashes from updating UI on background thread!

***

## 🛡️ Part 4: Actors - Thread-Safe by Design

Actors are Swift's answer to data races.

### The Problem: Data Races

```swift
class Counter {
    var count = 0
    
    func increment() {
        count += 1  // NOT thread-safe!
    }
}

let counter = Counter()

// 100 threads incrementing at once
DispatchQueue.concurrentPerform(iterations: 100) { _ in
    counter.increment()
}

print(counter.count)  // Should be 100, but might be 87! 💥
```

**Why?** Multiple threads reading/writing `count` simultaneously causes race conditions.

### The Solution: Actors

```swift
actor Counter {
    var count = 0
    
    func increment() {
        count += 1  // Thread-safe automatically!
    }
}

let counter = Counter()

// Now safe!
Task {
    await counter.increment()  // Note: must use await
}
```

**What changed:**
- `class` → `actor`
- Accessing actor properties requires `await`
- Actor ensures only one thread accesses at a time

### Real Example: Thread-Safe Cache

```swift
actor ImageCache {
    private var cache: [String: UIImage] = [:]
    
    func store(_ image: UIImage, forKey key: String) {
        cache[key] = image
    }
    
    func image(forKey key: String) -> UIImage? {
        return cache[key]
    }
    
    func clear() {
        cache.removeAll()
    }
}

// Usage - completely thread-safe!
let cache = ImageCache()

Task {
    await cache.store(myImage, forKey: "avatar")
}

Task {
    if let image = await cache.image(forKey: "avatar") {
        displayImage(image)
    }
}
```

### Actor Isolation

```swift
actor BankAccount {
    private var balance: Double = 0
    
    // Internal access - no await needed
    func deposit(_ amount: Double) {
        balance += amount  // No await - we're inside the actor
        checkBalance()     // No await - calling another actor method
    }
    
    private func checkBalance() {
        if balance < 0 {
            print("Overdraft!")
        }
    }
    
    // External access - await required
    func getBalance() -> Double {
        return balance
    }
}

// Outside the actor - await required
Task {
    let account = BankAccount()
    await account.deposit(100)
    let balance = await account.getBalance()
    print("Balance: \(balance)")
}
```

### Actor Reentrancy (Important!)

```swift
actor UserManager {
    var users: [User] = []
    
    func loadUsers() async {
        print("1. Starting load")
        
        // Suspension point - other code can run!
        let fetchedUsers = await fetchFromAPI()
        
        print("2. Got users: \(fetchedUsers.count)")
        users = fetchedUsers
    }
}

// Two tasks calling loadUsers
Task {
    await manager.loadUsers()  // Task 1
}

Task {
    await manager.loadUsers()  // Task 2
}

// Output might be:
// 1. Starting load (Task 1)
// 1. Starting load (Task 2)  ← Task 2 starts before Task 1 finishes!
// 2. Got users: 5 (Task 1)
// 2. Got users: 5 (Task 2)
```

**Why?** At `await`, Task 1 suspends, allowing Task 2 to start. This is **reentrancy**.

**Solution - use nonisolated when needed:**

```swift
actor Database {
    private var cache: [String: Data] = [:]
    
    // This can be called from multiple threads at once
    nonisolated func logQuery(_ query: String) {
        print("Query: \(query)")  // No state access, just logging
    }
}
```

### Global Actor

```swift
@globalActor
actor DatabaseActor {
    static let shared = DatabaseActor()
}

@DatabaseActor
class DatabaseManager {
    var connection: DatabaseConnection?
    
    func connect() {
        // All methods automatically isolated to DatabaseActor
        connection = DatabaseConnection()
    }
}
```

***

## 📊 Complete Comparison Table

| Feature | GCD | OperationQueue | async/await | Actors |
|---------|-----|----------------|-------------|--------|
| **Level** | Low-level | Mid-level | High-level | High-level |
| **API Style** | C-based | Objective-C | Swift-native | Swift-native |
| **Cancellation** | Manual | Built-in | Built-in | Built-in |
| **Dependencies** | Manual | Built-in | Sequential | N/A |
| **Thread Safety** | Manual | Manual | Manual | Automatic |
| **Error Handling** | Callbacks | Callbacks | try/catch | try/catch |
| **Performance** | Fastest | Slower | Optimized | Optimized |
| **Learning Curve** | Medium | Medium | Easy | Easy |
| **Use Cases** | Quick tasks | Complex workflows | Modern code | Shared state |

***

## 🏆 Real-World Examples

### Example 1: Image Feed (Instagram-like)

```swift
@MainActor
class FeedViewModel: ObservableObject {
    @Published var posts: [Post] = []
    @Published var isLoading = false
    
    private let imageCache = ImageCache()  // Actor
    
    func loadFeed() async {
        isLoading = true
        
        do {
            // Fetch posts
            let fetchedPosts = try await fetchPosts()
            
            // Download all images in parallel
            await withTaskGroup(of: (String, UIImage?).self) { group in
                for post in fetchedPosts {
                    group.addTask {
                        // Check cache first
                        if let cached = await self.imageCache.image(forKey: post.imageURL) {
                            return (post.id, cached)
                        }
                        
                        // Download and cache
                        if let image = try? await self.downloadImage(url: post.imageURL) {
                            await self.imageCache.store(image, forKey: post.imageURL)
                            return (post.id, image)
                        }
                        return (post.id, nil)
                    }
                }
                
                // Collect results
                for await (postId, image) in group {
                    if let image = image,
                       let index = fetchedPosts.firstIndex(where: { $0.id == postId }) {
                        var updatedPost = fetchedPosts[index]
                        updatedPost.image = image
                        posts.append(updatedPost)
                    }
                }
            }
        } catch {
            print("Error loading feed: \(error)")
        }
        
        isLoading = false
    }
}

actor ImageCache {
    private var cache: [String: UIImage] = [:]
    private let maxSize = 100
    
    func image(forKey key: String) -> UIImage? {
        return cache[key]
    }
    
    func store(_ image: UIImage, forKey key: String) {
        if cache.count >= maxSize {
            cache.removeAll()
        }
        cache[key] = image
    }
}
```

### Example 2: File Upload with Progress

```swift
actor UploadManager {
    private var uploads: [String: Upload] = [:]
    
    struct Upload {
        let file: URL
        var progress: Double
        var status: Status
        
        enum Status {
            case pending, uploading, completed, failed
        }
    }
    
    func startUpload(id: String, file: URL) async throws {
        uploads[id] = Upload(file: file, progress: 0, status: .uploading)
        
        // Upload in chunks
        let chunkSize = 1024 * 1024  // 1MB chunks
        let data = try Data(contentsOf: file)
        let totalChunks = (data.count + chunkSize - 1) / chunkSize
        
        for chunkIndex in 0..<totalChunks {
            // Check if cancelled
            if Task.isCancelled {
                uploads[id]?.status = .failed
                throw UploadError.cancelled
            }
            
            let start = chunkIndex * chunkSize
            let end = min(start + chunkSize, data.count)
            let chunk = data[start..<end]
            
            // Upload chunk
            try await uploadChunk(chunk, for: id)
            
            // Update progress
            let progress = Double(chunkIndex + 1) / Double(totalChunks)
            uploads[id]?.progress = progress
        }
        
        uploads[id]?.status = .completed
    }
    
    func getProgress(for id: String) -> Double? {
        return uploads[id]?.progress
    }
    
    func cancelUpload(id: String) {
        uploads[id]?.status = .failed
    }
}
```

### Example 3: Real-Time Chat

```swift
actor ChatManager {
    private var messages: [Message] = []
    private var listeners: [UUID: (Message) -> Void] = [:]
    
    func sendMessage(_ message: Message) async {
        messages.append(message)
        
        // Notify all listeners
        for listener in listeners.values {
            listener(message)
        }
        
        // Save to database
        await database.save(message)
    }
    
    func addListener(_ listener: @escaping (Message) -> Void) -> UUID {
        let id = UUID()
        listeners[id] = listener
        return id
    }
    
    func removeListener(id: UUID) {
        listeners.removeValue(forKey: id)
    }
    
    func loadHistory(limit: Int) async -> [Message] {
        return Array(messages.suffix(limit))
    }
}

// Usage in SwiftUI
@MainActor
class ChatViewModel: ObservableObject {
    @Published var messages: [Message] = []
    private let chatManager = ChatManager()
    private var listenerID: UUID?
    
    func setup() {
        listenerID = await chatManager.addListener { [weak self] message in
            Task { @MainActor in
                self?.messages.append(message)
            }
        }
    }
    
    func sendMessage(_ text: String) async {
        let message = Message(text: text, sender: currentUser)
        await chatManager.sendMessage(message)
    }
}
```

***

## 🎤 FAANG Interview Questions & Answers

### Q1: Explain the difference between sync and async in GCD. When would you use each?

**Expert Answer:** 

`async` dispatches work to a queue and returns immediately without waiting. The calling code continues executing while the dispatched work runs on another thread. Use async for most background work to avoid blocking the current thread.

`sync` dispatches work and **waits** for it to complete before returning. The calling thread is blocked until the work finishes. Use sync when you need the result immediately or must ensure sequential ordering.

**Critical warning:** Never call `sync` on the main queue from the main thread - this creates a deadlock where the main thread waits for itself to finish work it can't start.

Example use cases: Use async for network requests, file I/O, image processing. Use sync only when reading from thread-safe data structures where you need the value immediately to continue execution.

### Q2: What are data races and how do you prevent them?

**Expert Answer:**

Data races occur when multiple threads access the same memory location concurrently, with at least one thread writing, and without synchronization. This causes undefined behavior - crashes, corrupted data, or inconsistent state.

**Prevention strategies:**

1. **Serial queues** - Ensure all access happens serially on one queue
2. **Barriers** - Use `async(flags: .barrier)` for writes on concurrent queues to ensure exclusive access
3. **Actors** - Swift's built-in solution that automatically serializes access to mutable state
4. **Locks** - NSLock, os_unfair_lock (but actors are preferred in modern Swift)
5. **Immutability** - Value types (structs) copied on write prevent shared mutable state

Modern Swift strongly favors actors for thread safety because they're enforced at compile-time and integrated with async/await.

### Q3: Explain the difference between OperationQueue and GCD. When would you choose one over the other?

**Expert Answer:**

**GCD** is a low-level C API providing lightweight task dispatch with minimal overhead. It's great for simple async tasks but lacks high-level features.

**OperationQueue** wraps GCD with an object-oriented API providing: cancellation, dependencies between operations, KVO observation, max concurrent operation limits, and operation reuse.

**Choose GCD when:**
- Simple one-off async tasks
- Maximum performance needed
- Fire-and-forget operations
- Quick UI updates

**Choose OperationQueue when:**
- Complex workflows with dependencies (e.g., download → process → save pipeline)
- Need cancellation (user cancels long operation)
- Want to limit concurrency (max 3 simultaneous downloads)
- Building reusable operation classes

Example: Instagram feed loading uses OperationQueue for image download pipeline with dependencies, but simple button tap handlers use GCD for simplicity.

### Q4: How does async/await improve upon callback-based APIs?

**Expert Answer:**

async/await solves several problems with callbacks:

**1. Callback hell** - Nested callbacks create unreadable "pyramid of doom". async/await allows linear, sequential-looking code.

**2. Error handling** - Callbacks require separate error parameters. async/await uses standard try/catch with throws.

**3. Resource management** - Callbacks make it unclear when resources are released. Structured concurrency in async/await automatically cancels child tasks when parent exits.

**4. Thread context** - Callbacks require manual dispatch to correct threads. @MainActor and actor isolation handle this automatically.

**5. Composability** - Callbacks don't compose well. async/await functions compose naturally like synchronous code.

**Performance:** async/await also enables compiler optimizations through structured concurrency that aren't possible with callbacks, reducing thread overhead.

The Swift runtime's cooperative thread pool for async/await can handle far more concurrent tasks than traditional threads.

### Q5: Explain actor reentrancy. Why is it important to understand?

**Expert Answer:**

Actor reentrancy means an actor can start executing another call before a previous call completes, specifically at suspension points (await).

Example:
```swift
actor Counter {
    var value = 0
    
    func increment() async {
        let current = value  // Read: 0
        await Task.sleep(...)  // ← Suspension point!
        value = current + 1  // Write: 1
    }
}

// Two calls
await counter.increment()  // Call 1
await counter.increment()  // Call 2
```

Call 1 suspends at await, allowing Call 2 to start. Both read value = 0, both write 1. Final value: 1 (not 2!)

**Why it matters:** Assumptions about state can be invalidated at any await. State read before await might have changed after.

**Solutions:**
1. Minimize work after await
2. Re-validate state after suspension points
3. Use synchronous methods for atomic operations
4. Document reentrancy assumptions in comments

Understanding reentrancy is critical for senior engineers because it's a subtle source of bugs in actor-based code.

### Q6: How would you implement a thread-safe singleton in Swift?

**Expert Answer:**

**Modern approach - Actor:**
```swift
actor DatabaseManager {
    static let shared = DatabaseManager()
    private init() {}
    
    func query(_ sql: String) async -> [Row] {
        // Thread-safe by actor isolation
    }
}
```

**Traditional approach - Dispatch once:**
```swift
class DatabaseManager {
    static let shared = DatabaseManager()
    private init() {}
    
    private let queue = DispatchQueue(label: "db")
    
    func query(_ sql: String, completion: @escaping ([Row]) -> Void) {
        queue.async {
            // Thread-safe on serial queue
            let results = self.executeQuery(sql)
            DispatchQueue.main.async {
                completion(results)
            }
        }
    }
}
```

**Why static let is safe:** Swift's static let uses dispatch_once internally, guaranteeing single initialization even with concurrent access.

**Critical points:** 
- Private init prevents additional instances
- Actor isolation or serial queue protects mutable state
- Modern Swift prefers actors for automatic thread safety

### Q7: Explain Quality of Service (QoS) in GCD and when to use each level.

**Expert Answer:**

QoS tells the system how important your work is, affecting CPU allocation, timer coalescing, and energy efficiency.

**userInteractive** (.userInteractive) - Highest priority. Use for work directly updating UI that user is actively interacting with. Example: smooth scrolling animations, button response. Duration: milliseconds.

**userInitiated** (.userInitiated) - High priority. Use for work user requested that they're waiting for. Example: loading content after button tap, search results. Duration: seconds.

**default** (.default) - Normal priority. Use for general work not fitting other categories.

**utility** (.utility) - Low priority, long-running. Use for work user doesn't need immediately. Example: downloading content, syncing data. Duration: minutes.

**background** (.background) - Lowest priority. Use for maintenance work user isn't aware of. Example: cleanup, backups, indexing. Duration: hours.

**Pro tip:** Wrong QoS can degrade battery life or UI responsiveness. UI work on .background causes janky animations. Background work on .userInteractive wastes energy.

### Q8: How do you handle cancellation in modern Swift concurrency?

**Expert Answer:**

Swift concurrency provides cooperative cancellation through Task:

```swift
let task = Task {
    for item in items {
        // Check cancellation
        if Task.isCancelled {
            cleanup()
            return
        }
        
        // Or use try with Task.checkCancellation()
        try Task.checkCancellation()  // Throws CancellationError
        
        await process(item)
    }
}

// Cancel from elsewhere
task.cancel()
```

**Key points:**

1. **Cooperative** - You must check Task.isCancelled explicitly. Work doesn't stop automatically.

2. **Automatic propagation** - Cancelling parent task cancels all child tasks automatically with structured concurrency.

3. **withTaskCancellationHandler** - Register cleanup code:
```swift
await withTaskCancellationHandler {
    await longRunningWork()
} onCancel: {
    cleanup()
}
```

4. **Actor awareness** - Actors respect cancellation but you must check between operations.

**Best practice:** Check cancellation at loop iterations, before expensive operations, and after await points.

### Q9: What's the difference between Task, Task.detached, and async let?

**Expert Answer:**

**Task** - Creates child task inheriting priority and context from current task:
```swift
Task {  // Inherits priority, @MainActor context
    await work()
}
```

**Task.detached** - Creates independent task with no inheritance:
```swift
Task.detached {  // No inheritance, runs independently
    await work()
}
```

**async let** - Structured concurrency for parallel work within same scope:
```swift
async let a = fetchA()
async let b = fetchB()
let (resultA, resultB) = await (a, b)
// Cancellation of parent cancels both
```

**Key differences:**

- Task inherits context and priority; detached doesn't
- Task is cancelled when parent is; detached isn't
- async let variables are scope-bound; Task can escape scope
- async let forces await before scope exit; Task can continue

**Use Task** for most cases, **async let** for parallel work you'll wait for, **Task.detached** rarely when you need independence from current context.

### Q10: How would you implement a download manager handling multiple concurrent downloads with priority?

**Expert Answer:**

```swift
actor DownloadManager {
    private var activeDownloads: [UUID: DownloadTask] = [:]
    private let maxConcurrent = 3
    
    struct DownloadTask {
        let id: UUID
        let url: URL
        let priority: TaskPriority
        var progress: Double
        var status: Status
        var task: Task<Data, Error>?
    }
    
    enum Status {
        case queued, downloading, completed, failed, cancelled
    }
    
    func download(url: URL, priority: TaskPriority = .medium) async throws -> Data {
        // Wait if at max capacity
        while activeDownloads.count >= maxConcurrent {
            await Task.yield()
        }
        
        let id = UUID()
        let task = Task(priority: priority) { [weak self] in
            let (data, response) = try await URLSession.shared.data(from: url)
            await self?.updateProgress(id: id, progress: 1.0)
            return data
        }
        
        activeDownloads[id] = DownloadTask(
            id: id,
            url: url,
            priority: priority,
            progress: 0,
            status: .downloading,
            task: task
        )
        
        do {
            let data = try await task.value
            activeDownloads[id]?.status = .completed
            activeDownloads.removeValue(forKey: id)
            return data
        } catch {
            activeDownloads[id]?.status = .failed
            activeDownloads.removeValue(forKey: id)
            throw error
        }
    }
    
    func cancelDownload(id: UUID) {
        activeDownloads[id]?.task?.cancel()
        activeDownloads[id]?.status = .cancelled
    }
    
    private func updateProgress(id: UUID, progress: Double) {
        activeDownloads[id]?.progress = progress
    }
}
```

This demonstrates: actor isolation, concurrent limiting, priority handling, cancellation, and progress tracking - common FAANG interview topics.

***

## 🎓 Summary: Which Should You Use?

### Use GCD When:
- ✅ Simple, quick async tasks
- ✅ Legacy code compatibility
- ✅ Fire-and-forget operations
- ✅ Need maximum performance
- ✅ Simple UI updates

### Use OperationQueue When:
- ✅ Complex task dependencies
- ✅ Need cancellation
- ✅ Want to limit concurrency (max 3 simultaneous downloads)
- ✅ Reusable operation objects
- ✅ Need KVO observation

### Use async/await When:
- ✅ New Swift code (Swift 5.5+)
- ✅ Sequential async operations
- ✅ Network requests
- ✅ Readable, maintainable code
- ✅ Error handling with throws

### Use Actors When:
- ✅ Shared mutable state
- ✅ Thread safety required
- ✅ Replace locks and semaphores
- ✅ Cache implementations
- ✅ Modern Swift concurrency

**The Future:** Apple strongly recommends async/await + actors for new code. GCD and OperationQueue remain for legacy support and specific use cases.

Master all four because:
- **Interviews** test knowledge of all approaches
- **Legacy codebases** use GCD/OperationQueue
- **Modern apps** use async/await/actors
- **Senior roles** require knowing when to use each

You're now equipped with complete concurrency knowledge from beginner to FAANG interview level! 🚀
