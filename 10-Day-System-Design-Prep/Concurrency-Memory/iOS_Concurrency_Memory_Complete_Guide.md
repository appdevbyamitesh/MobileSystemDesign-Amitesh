# iOS Concurrency & Memory Management
## Complete Deep Dive: From Beginner to Big Tech Interview Ready

---

## Table of Contents

1. [Beginner: What is Concurrency?](#-beginner-what-is-concurrency)
2. [Beginner: What is Memory Management?](#-beginner-what-is-memory-management)
3. [Intermediate: GCD, OperationQueue, async/await](#-intermediate-gcd-operationqueue-asyncawait)
4. [Intermediate: Thread Safety & Race Conditions](#-intermediate-thread-safety--race-conditions)
5. [Intermediate: Actors](#-intermediate-actors)
6. [Intermediate: ARC Deep Dive](#-intermediate-arc-deep-dive)
7. [Intermediate: Retain Cycles](#-intermediate-retain-cycles)
8. [Advanced: Thread-Safe Components](#-advanced-thread-safe-components)
9. [Advanced: Mixing Concurrency Models](#-advanced-mixing-concurrency-models)
10. [Advanced: Memory Debugging](#-advanced-memory-debugging)
11. [Advanced: Big Tech Patterns](#-advanced-big-tech-patterns)
12. [Interview Preparation](#-interview-preparation)

---

# 🎈 Beginner: What is Concurrency?

## The Simplest Explanation

Imagine you're in a kitchen making breakfast:

**Without Concurrency (Single-threaded):**
```
1. Start boiling water ───────────────────── 5 minutes
2. Wait... doing nothing
3. Toast bread ─────────────────────────── 3 minutes
4. Wait... doing nothing
5. Fry eggs ────────────────────────────── 4 minutes
Total: 12 minutes 😩
```

**With Concurrency (Multi-threaded):**
```
1. Start boiling water ──────┐
2. While water boils, toast ─┼──── All happening together!
3. While both going, fry eggs┘
Total: 5 minutes 😎
```

**In iOS terms:** Your app can do multiple things "at the same time" instead of waiting for each thing to finish.

## Why Mobile Apps NEED Concurrency

### Problem: The Frozen App

```
User taps "Load Photos" button
    ↓
App downloads 100 photos (takes 5 seconds)
    ↓
FROZEN SCREEN 🥶 - user can't scroll, tap, or do anything
    ↓
User thinks app crashed → deletes app → 1-star review
```

### Solution: Concurrency

```
User taps "Load Photos" button
    ↓
App starts download on BACKGROUND thread
    ↓
UI stays responsive ✨ - user can scroll, tap, cancel
    ↓
Photos appear as they download
    ↓
Happy user → 5-star review ⭐️
```

## Real-Life Analogies

| Concept | Real-Life Analogy |
|---------|------------------|
| **Main Thread** | The front desk at a hotel - handles all guest interactions |
| **Background Thread** | Housekeeping - works behind the scenes |
| **Queue** | Line at a coffee shop - tasks wait their turn |
| **Async** | Ordering Uber - you don't wait on the curb, you get a notification |
| **Sync** | Waiting in line at DMV - you can't leave until done |

## The Golden Rule of iOS

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│   🚨 MAIN THREAD = UI THREAD                                │
│                                                              │
│   ✅ DO on Main Thread:      ❌ NEVER do on Main Thread:    │
│   • Update UI elements       • Network calls                 │
│   • Handle user taps         • File operations               │
│   • Navigate screens         • Database queries              │
│   • Animate views            • Image processing              │
│                              • Heavy calculations            │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

## Your First Concurrency Code

```swift
// ❌ BAD: Freezes UI
func loadPhotos() {
    let photos = downloadPhotosFromServer() // Takes 5 seconds!
    imageView.image = photos[0] // UI finally updates
}

// ✅ GOOD: UI stays responsive
func loadPhotos() {
    // 1. Do heavy work on background thread
    DispatchQueue.global().async {
        let photos = self.downloadPhotosFromServer() // Takes 5 seconds on background
        
        // 2. Update UI on main thread
        DispatchQueue.main.async {
            self.imageView.image = photos[0]
        }
    }
}
```

---

# 🧠 Beginner: What is Memory Management?

## The Simplest Explanation

Your iPhone has limited memory (RAM) - like a desk with limited space.

**Without memory management:**
```
Open app → create objects → more objects → MORE OBJECTS → 💥 CRASH!
(desk overflows, everything falls)
```

**With memory management:**
```
Open app → create objects → REMOVE objects when done → space freed
(clean desk policy)
```

## What is ARC?

**ARC = Automatic Reference Counting**

Think of it like a library book tracking system:

```
📚 Library Book System:
1. You borrow a book       → Library adds 1 to borrow count
2. Your friend borrows too → Count = 2
3. You return it           → Count = 1
4. Friend returns it       → Count = 0
5. Count = 0?              → Book goes back to shelf (freed)

💻 iOS Memory:
1. You create an object    → Reference count = 1
2. Another var holds it    → Reference count = 2
3. First var goes away     → Reference count = 1
4. Second var goes away    → Reference count = 0
5. Count = 0?              → Object deallocated (memory freed)
```

## Why Developers Must Care About ARC

Even though ARC is "automatic," you can still create problems:

### The Retain Cycle Problem

```
Imagine two friends who refuse to leave a party until the other leaves first:

Person A: "I won't leave until you leave"
Person B: "I won't leave until YOU leave"

Result: Neither leaves → PARTY NEVER ENDS (memory never freed)
```

In code:
```swift
class Person {
    var friend: Person? // Strong reference
}

let john = Person()
let jane = Person()
john.friend = jane  // John holds Jane
jane.friend = john  // Jane holds John

// Even if we set john = nil and jane = nil,
// they hold each other → memory never freed → MEMORY LEAK!
```

## Real-Life Memory Analogies

| Concept | Real-Life Analogy |
|---------|-------------------|
| **Strong Reference** | Holding someone's hand - they can't leave |
| **Weak Reference** | Knowing someone's name - they can leave anytime |
| **Reference Count** | Number of people holding the person's hands |
| **Deallocation** | Person leaves the room when no one holds them |
| **Memory Leak** | People stuck forever holding each other |

---

# 🛠️ Intermediate: GCD, OperationQueue, async/await

## Quick Comparison

| Feature | GCD | OperationQueue | async/await |
|---------|-----|---------------|-------------|
| **Complexity** | Simple | Medium | Simplest |
| **Cancellation** | Manual | Built-in | Built-in |
| **Dependencies** | Manual | Built-in | Natural |
| **Debugging** | Hard | Medium | Easiest |
| **When to use** | Quick tasks | Complex pipelines | Modern iOS 15+ |

## GCD (Grand Central Dispatch)

### Core Concept: Queues

```swift
// MAIN QUEUE - for UI work
DispatchQueue.main.async {
    self.label.text = "Updated!"
}

// GLOBAL QUEUE - for background work
DispatchQueue.global(qos: .userInitiated).async {
    let data = self.processLargeFile()
}

// CUSTOM QUEUE - for your own control
let myQueue = DispatchQueue(label: "com.myapp.processing")
myQueue.async {
    self.doCustomWork()
}
```

### Quality of Service (QoS) Priority

```swift
// Highest → Lowest priority:

.userInteractive  // Animation, event handling (use sparingly!)
.userInitiated    // User clicked something, expects quick result
.default          // Normal priority
.utility          // Long tasks user is aware of (progress bar)
.background       // User doesn't care when it finishes (backup)
.unspecified      // Let system decide
```

### Real iOS Example: Loading a Feed

```swift
class FeedViewController: UIViewController {
    func loadFeed() {
        // Show loading spinner on main thread
        activityIndicator.startAnimating()
        
        // Fetch data on background thread
        DispatchQueue.global(qos: .userInitiated).async { [weak self] in
            do {
                let posts = try self?.fetchPostsFromAPI()
                
                // Parse and transform data (still on background)
                let viewModels = posts?.map { PostViewModel($0) }
                
                // Update UI on main thread
                DispatchQueue.main.async {
                    self?.activityIndicator.stopAnimating()
                    self?.posts = viewModels ?? []
                    self?.tableView.reloadData()
                }
            } catch {
                DispatchQueue.main.async {
                    self?.showError(error)
                }
            }
        }
    }
}
```

### Serial vs Concurrent Queues

```swift
// SERIAL QUEUE - one task at a time (safe for shared data)
let serialQueue = DispatchQueue(label: "serial")
serialQueue.async { print("1") }  // Runs first
serialQueue.async { print("2") }  // Waits for 1
serialQueue.async { print("3") }  // Waits for 2
// Output: 1, 2, 3 (always in order)

// CONCURRENT QUEUE - multiple tasks at once (faster)
let concurrentQueue = DispatchQueue(label: "concurrent", attributes: .concurrent)
concurrentQueue.async { print("1") }  // Runs immediately
concurrentQueue.async { print("2") }  // Runs immediately
concurrentQueue.async { print("3") }  // Runs immediately
// Output: could be 2, 1, 3 or 1, 3, 2 or any order
```

### DispatchGroup: Wait for Multiple Tasks

```swift
// Load profile screen: need user, posts, and followers
func loadProfileScreen(userId: String) {
    let group = DispatchGroup()
    var user: User?
    var posts: [Post]?
    var followers: [User]?
    
    // Fetch user
    group.enter()
    fetchUser(id: userId) { result in
        user = result
        group.leave()
    }
    
    // Fetch posts (runs parallel with user fetch)
    group.enter()
    fetchPosts(for: userId) { result in
        posts = result
        group.leave()
    }
    
    // Fetch followers (runs parallel with both)
    group.enter()
    fetchFollowers(for: userId) { result in
        followers = result
        group.leave()
    }
    
    // When ALL are done, update UI
    group.notify(queue: .main) { [weak self] in
        self?.updateUI(user: user, posts: posts, followers: followers)
    }
}
```

## OperationQueue

Better for complex scenarios with dependencies and cancellation.

### Basic Usage

```swift
let queue = OperationQueue()
queue.maxConcurrentOperationCount = 3  // Limit parallel operations

let operation = BlockOperation {
    print("Doing work...")
}

operation.completionBlock = {
    print("Work done!")
}

queue.addOperation(operation)
```

### Dependencies: This Must Finish Before That

```swift
// Image processing pipeline
let downloadOp = BlockOperation { downloadImage() }
let resizeOp = BlockOperation { resizeImage() }
let filterOp = BlockOperation { applyFilter() }
let saveOp = BlockOperation { saveToCache() }

// Set up dependencies
resizeOp.addDependency(downloadOp)   // Resize waits for download
filterOp.addDependency(resizeOp)     // Filter waits for resize
saveOp.addDependency(filterOp)       // Save waits for filter

// Add all to queue - they execute in correct order
queue.addOperations([downloadOp, resizeOp, filterOp, saveOp], 
                    waitUntilFinished: false)

// Timeline:
// Download ──→ Resize ──→ Filter ──→ Save
```

### Cancellation

```swift
class ImageGalleryViewController: UIViewController {
    let imageQueue = OperationQueue()
    
    override func viewWillDisappear(_ animated: Bool) {
        super.viewWillDisappear(animated)
        // Cancel all pending downloads when user leaves
        imageQueue.cancelAllOperations()
    }
    
    func loadImages(_ urls: [URL]) {
        for url in urls {
            let operation = BlockOperation { [weak self] in
                // Check if cancelled BEFORE doing work
                if operation.isCancelled { return }
                
                let data = try? Data(contentsOf: url)
                
                // Check again AFTER expensive work
                if operation.isCancelled { return }
                
                DispatchQueue.main.async {
                    self?.displayImage(data)
                }
            }
            imageQueue.addOperation(operation)
        }
    }
}
```

## async/await (Modern Swift)

The cleanest way to write concurrent code (iOS 15+).

### Basic Syntax

```swift
// Old callback way
func fetchUser(completion: @escaping (User?) -> Void) {
    URLSession.shared.dataTask(with: url) { data, _, _ in
        let user = try? JSONDecoder().decode(User.self, from: data!)
        completion(user)
    }.resume()
}

// New async/await way
func fetchUser() async throws -> User {
    let (data, _) = try await URLSession.shared.data(from: url)
    return try JSONDecoder().decode(User.self, from: data)
}

// Usage - reads like synchronous code!
Task {
    do {
        let user = try await fetchUser()
        let posts = try await fetchPosts(for: user.id)
        let followers = try await fetchFollowers(for: user.id)
        updateUI(user: user, posts: posts, followers: followers)
    } catch {
        showError(error)
    }
}
```

### Parallel Execution with async let

```swift
// SEQUENTIAL - one after another (slower)
let user = try await fetchUser()           // 1 second
let posts = try await fetchPosts()         // 1 second
let followers = try await fetchFollowers() // 1 second
// Total: 3 seconds

// PARALLEL - all at once (faster)
async let user = fetchUser()               // Starts immediately
async let posts = fetchPosts()             // Starts immediately
async let followers = fetchFollowers()     // Starts immediately

let (u, p, f) = try await (user, posts, followers)
// Total: 1 second (all ran at same time)
```

### TaskGroup for Dynamic Parallel Work

```swift
// Download many images in parallel
func downloadAllImages(urls: [URL]) async throws -> [UIImage] {
    try await withThrowingTaskGroup(of: UIImage.self) { group in
        for url in urls {
            group.addTask {
                let (data, _) = try await URLSession.shared.data(from: url)
                guard let image = UIImage(data: data) else {
                    throw ImageError.invalidData
                }
                return image
            }
        }
        
        var images: [UIImage] = []
        for try await image in group {
            images.append(image)
        }
        return images
    }
}
```

### When to Use Which

| Scenario | Best Choice | Why |
|----------|-------------|-----|
| Simple background task | GCD | Quick, no dependencies |
| Image/video pipeline | OperationQueue | Dependencies, cancellation |
| Modern networking | async/await | Clean code, easy cancellation |
| Legacy codebase | GCD | Compatibility |
| Complex state machines | OperationQueue | Fine control |
| New projects (iOS 15+) | async/await | Best experience |

---

# 🔐 Intermediate: Thread Safety & Race Conditions

## What is a Race Condition?

Two threads trying to access the same data at the same time.

```
Thread 1: Reads counter (value: 5)
Thread 2: Reads counter (value: 5)
Thread 1: Adds 1, writes 6
Thread 2: Adds 1, writes 6  ← WRONG! Should be 7!

Expected: 7
Actual: 6
Result: BUG! 🐛
```

## Real iOS Race Condition Example

```swift
// ❌ DANGEROUS: Race condition in image cache
class UnsafeImageCache {
    private var cache: [URL: UIImage] = [:]
    
    func getImage(for url: URL) -> UIImage? {
        return cache[url]  // Thread A reads
    }
    
    func setImage(_ image: UIImage, for url: URL) {
        cache[url] = image  // Thread B writes at same time → CRASH!
    }
}
```

## Detecting Race Conditions

**Symptoms:**
- App crashes randomly, hard to reproduce
- Data corruption
- UI shows wrong values sometimes
- Xcode shows "Thread Sanitizer" warnings

**Enable Thread Sanitizer:**
```
Xcode → Product → Scheme → Edit Scheme → Run → Diagnostics → Thread Sanitizer ✓
```

## Solutions for Thread Safety

### Solution 1: Serial Queue (Simple & Safe)

```swift
class ThreadSafeCache {
    private var cache: [URL: UIImage] = [:]
    private let queue = DispatchQueue(label: "cache.queue")
    
    func getImage(for url: URL) -> UIImage? {
        queue.sync {  // Blocks until read complete
            return cache[url]
        }
    }
    
    func setImage(_ image: UIImage, for url: URL) {
        queue.async {  // Async write is fine
            self.cache[url] = image
        }
    }
}
```

### Solution 2: Reader-Writer Lock (Better Performance)

```swift
class FastThreadSafeCache {
    private var cache: [URL: UIImage] = [:]
    private let queue = DispatchQueue(label: "cache.queue", attributes: .concurrent)
    
    func getImage(for url: URL) -> UIImage? {
        queue.sync {  // Multiple reads can happen together
            return cache[url]
        }
    }
    
    func setImage(_ image: UIImage, for url: URL) {
        queue.async(flags: .barrier) {  // Barrier = exclusive access
            self.cache[url] = image
        }
    }
}

// Timeline:
// Read ─┬─ Read ─┬─ Read     (parallel reads OK)
//       │        │
//       └────────┴───── BARRIER WRITE ─── (exclusive)
//                              │
//                    Read ─┬─ Read (parallel reads resume)
```

### Solution 3: NSLock (Low-level)

```swift
class LockBasedCache {
    private var cache: [URL: UIImage] = [:]
    private let lock = NSLock()
    
    func getImage(for url: URL) -> UIImage? {
        lock.lock()
        defer { lock.unlock() }
        return cache[url]
    }
    
    func setImage(_ image: UIImage, for url: URL) {
        lock.lock()
        defer { lock.unlock() }
        cache[url] = image
    }
}
```

### Solution 4: Actor (Modern Swift - Best!)

```swift
actor ModernCache {
    private var cache: [URL: UIImage] = [:]
    
    func getImage(for url: URL) -> UIImage? {
        cache[url]  // Automatically thread-safe!
    }
    
    func setImage(_ image: UIImage, for url: URL) {
        cache[url] = image  // Automatically thread-safe!
    }
}

// Usage
let cache = ModernCache()
Task {
    await cache.setImage(image, for: url)
    let cached = await cache.getImage(for: url)
}
```

---

# 🎭 Intermediate: Actors

## What is an Actor?

An Actor is a **reference type** that **protects its mutable state** from concurrent access. Think of it as a class with a built-in bodyguard.

```swift
// Regular class - anyone can access data anytime → DANGEROUS
class UnsafeCounter {
    var count = 0
    func increment() { count += 1 }
}

// Actor - only one caller can access at a time → SAFE
actor SafeCounter {
    var count = 0
    func increment() { count += 1 }
}
```

## How Actors Work

```
Without Actor:
Thread 1 ─────────────→ [count] ←───────────── Thread 2
                         💥 Race condition!

With Actor:
Thread 1 → [Waits] → [Bodyguard] → [count]
Thread 2 → [Waits] ↗
              ↑
        Only one thread allowed in at a time
```

## Using Actors

```swift
actor ShoppingCart {
    private var items: [Product] = []
    
    var itemCount: Int {
        items.count
    }
    
    var totalPrice: Decimal {
        items.reduce(0) { $0 + $1.price }
    }
    
    func add(_ product: Product) {
        items.append(product)
    }
    
    func remove(_ product: Product) {
        items.removeAll { $0.id == product.id }
    }
    
    func clear() {
        items.removeAll()
    }
}

// Usage - must use await because actor protects access
let cart = ShoppingCart()

Task {
    await cart.add(product1)
    await cart.add(product2)
    let count = await cart.itemCount
    print("Cart has \(count) items")
}
```

## MainActor: The UI Actor

```swift
// Automatically runs on main thread
@MainActor
class ProfileViewModel: ObservableObject {
    @Published var user: User?
    @Published var isLoading = false
    
    func loadUser() async {
        isLoading = true  // Safe - on main thread
        
        do {
            user = try await userService.fetchCurrentUser()
        } catch {
            // Handle error
        }
        
        isLoading = false  // Safe - on main thread
    }
}

// For individual functions
class DataManager {
    @MainActor
    func updateUI(with data: Data) {
        // Guaranteed to run on main thread
        label.text = String(data: data, encoding: .utf8)
    }
}
```

## Actor vs Serial Queue vs Lock

| Feature | Actor | Serial Queue | NSLock |
|---------|-------|--------------|--------|
| Syntax | Cleanest | Medium | Verbose |
| Async support | Native | Manual | None |
| Deadlock risk | Low | Medium | High |
| Performance | Good | Good | Best |
| Debugging | Easy | Medium | Hard |
| iOS version | 15+ | All | All |

---

# 💪 Intermediate: ARC Deep Dive

## Reference Types Explained

### Strong (Default)

```swift
class Dog {
    let name: String
    init(name: String) { 
        self.name = name
        print("\(name) is born!")
    }
    deinit { print("\(name) is gone!") }
}

var dog1: Dog? = Dog(name: "Max")  // "Max is born!" - count: 1
var dog2 = dog1                      // count: 2
var dog3 = dog1                      // count: 3

dog1 = nil  // count: 2 - Max still alive
dog2 = nil  // count: 1 - still alive
dog3 = nil  // count: 0 - "Max is gone!"
```

### Weak

```swift
class Person {
    let name: String
    weak var pet: Dog?  // Weak = doesn't keep Dog alive
    
    init(name: String) { self.name = name }
}

var john: Person? = Person(name: "John")
var dog: Dog? = Dog(name: "Max")

john?.pet = dog  // Weak reference - Dog's count: 1 (just dog var)

dog = nil        // count: 0 - "Max is gone!"
print(john?.pet) // nil - weak auto-becomes nil
```

### Unowned

```swift
class CreditCard {
    unowned let owner: Person  // Must ALWAYS have valid owner
    let number: String
    
    init(owner: Person, number: String) {
        self.owner = owner
        self.number = number
    }
}

class Person {
    var card: CreditCard?
    let name: String
    
    init(name: String) { self.name = name }
}

var john: Person? = Person(name: "John")
john?.card = CreditCard(owner: john!, number: "1234")

// Card exists only while John exists
// If John deallocated while card still exists → CRASH!
```

### When to Use Which

| Type | Use When | Example |
|------|----------|---------|
| **strong** | You own the object | ViewModel owns service |
| **weak** | Object might outlive you | Delegate, parent reference |
| **unowned** | Same lifetime, but back-reference | Credit card → owner |

## Memory Graph Example

```
STRONG References:
ViewController ─── strong ───→ ViewModel ─── strong ───→ Service

WEAK References:
TableView ─── weak ───→ DataSource (the VC)

UNOWNED References:
CreditCard ─── unowned ───→ Person (same lifetime)
```

---

# 🔄 Intermediate: Retain Cycles

## What is a Retain Cycle?

Two (or more) objects holding strong references to each other, preventing deallocation.

```
┌─────────┐ strong  ┌─────────┐
│ Object A│ ──────→ │ Object B│
│         │ ←────── │         │
└─────────┘ strong  └─────────┘

Neither can be deallocated → MEMORY LEAK!
```

## Common Retain Cycle Scenarios

### 1. Closures Capturing self

```swift
// ❌ RETAIN CYCLE
class ViewController: UIViewController {
    var name = "Home"
    
    func setupButton() {
        button.onTap = {
            print(self.name)  // self captured strongly!
        }
        // VC → button → closure → VC (cycle!)
    }
    
    deinit { print("VC deallocated") }  // Never called!
}

// ✅ FIX: Capture list with weak
func setupButton() {
    button.onTap = { [weak self] in
        guard let self = self else { return }
        print(self.name)
    }
}

// ✅ FIX: Capture list with unowned (if guaranteed to exist)
func setupButton() {
    button.onTap = { [unowned self] in
        print(self.name)  // Crashes if self is nil
    }
}
```

### 2. Delegate Pattern

```swift
// ❌ RETAIN CYCLE
protocol NetworkDelegate {
    func didFinish()
}

class NetworkManager {
    var delegate: NetworkDelegate?  // Strong reference!
}

class ViewController: UIViewController, NetworkDelegate {
    let network = NetworkManager()
    
    override func viewDidLoad() {
        network.delegate = self  // VC → network → VC (cycle!)
    }
}

// ✅ FIX: Weak delegate
protocol NetworkDelegate: AnyObject {  // Must be AnyObject for weak
    func didFinish()
}

class NetworkManager {
    weak var delegate: NetworkDelegate?  // Weak breaks cycle
}
```

### 3. Timer

```swift
// ❌ RETAIN CYCLE
class ViewController: UIViewController {
    var timer: Timer?
    
    override func viewDidLoad() {
        timer = Timer.scheduledTimer(
            timeInterval: 1,
            target: self,        // Strong reference to self!
            selector: #selector(tick),
            userInfo: nil,
            repeats: true
        )
    }
    // Timer → self → timer (through var) → cycle!
    
    deinit { print("Never called!") }
}

// ✅ FIX: Use block-based API
override func viewDidLoad() {
    timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
        self?.tick()
    }
}

override func viewWillDisappear(_ animated: Bool) {
    super.viewWillDisappear(animated)
    timer?.invalidate()  // Always invalidate!
}
```

### 4. NotificationCenter

```swift
// ❌ RETAIN CYCLE (older API)
class ViewController: UIViewController {
    override func viewDidLoad() {
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(handleNotification),
            name: .someNotification,
            object: nil
        )
    }
    // Must manually remove in deinit!
}

// ✅ FIX: Block-based with weak self
class ViewController: UIViewController {
    var observer: NSObjectProtocol?
    
    override func viewDidLoad() {
        observer = NotificationCenter.default.addObserver(
            forName: .someNotification,
            object: nil,
            queue: .main
        ) { [weak self] notification in
            self?.handleNotification(notification)
        }
    }
    
    deinit {
        if let observer = observer {
            NotificationCenter.default.removeObserver(observer)
        }
    }
}
```

### 5. Combine Subscriptions

```swift
// ❌ RETAIN CYCLE
class ViewController: UIViewController {
    var cancellables = Set<AnyCancellable>()
    
    func subscribe() {
        publisher
            .sink { value in
                self.updateUI(value)  // Strong capture!
            }
            .store(in: &cancellables)
    }
}

// ✅ FIX
func subscribe() {
    publisher
        .sink { [weak self] value in
            self?.updateUI(value)
        }
        .store(in: &cancellables)
}
```

## Retain Cycle Prevention Checklist

| Scenario | Fix |
|----------|-----|
| Closure captures self | `[weak self]` or `[unowned self]` |
| Delegate | `weak var delegate` |
| Timer target | Block-based API + invalidate |
| NotificationCenter | Block-based + remove observer |
| Combine sink | `[weak self]` in closure |
| Parent-child relationship | Child uses `weak` parent reference |

---

# 🏛️ Advanced: Thread-Safe Components

## Designing a Thread-Safe Network Layer

```swift
actor NetworkManager {
    private var activeTasks: [URL: Task<Data, Error>] = [:]
    
    func fetch(url: URL) async throws -> Data {
        // Coalesce duplicate requests
        if let existingTask = activeTasks[url] {
            return try await existingTask.value
        }
        
        let task = Task {
            defer { activeTasks.removeValue(forKey: url) }
            let (data, _) = try await URLSession.shared.data(from: url)
            return data
        }
        
        activeTasks[url] = task
        return try await task.value
    }
    
    func cancelAll() {
        activeTasks.values.forEach { $0.cancel() }
        activeTasks.removeAll()
    }
}
```

## Thread-Safe Observable Store

```swift
@MainActor
final class Store<State, Action> {
    @Published private(set) var state: State
    private let reducer: (State, Action) -> State
    
    init(initialState: State, reducer: @escaping (State, Action) -> State) {
        self.state = initialState
        self.reducer = reducer
    }
    
    func dispatch(_ action: Action) {
        state = reducer(state, action)
    }
}

// Usage
struct AppState {
    var user: User?
    var cartItems: [Product] = []
}

enum AppAction {
    case login(User)
    case addToCart(Product)
}

@MainActor
let store = Store<AppState, AppAction>(
    initialState: AppState()
) { state, action in
    var newState = state
    switch action {
    case .login(let user):
        newState.user = user
    case .addToCart(let product):
        newState.cartItems.append(product)
    }
    return newState
}
```

## Thread-Safe Image Cache

```swift
actor ImageCache {
    private let memoryCache = NSCache<NSURL, UIImage>()
    private var diskCachePath: URL
    private var loadingTasks: [URL: Task<UIImage, Error>] = [:]
    
    init() {
        let paths = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask)
        diskCachePath = paths[0].appendingPathComponent("ImageCache")
        try? FileManager.default.createDirectory(at: diskCachePath, withIntermediateDirectories: true)
    }
    
    func image(for url: URL) async throws -> UIImage {
        // 1. Check memory cache
        if let cached = memoryCache.object(forKey: url as NSURL) {
            return cached
        }
        
        // 2. Check if already loading
        if let existingTask = loadingTasks[url] {
            return try await existingTask.value
        }
        
        // 3. Check disk cache
        let diskPath = diskCachePath.appendingPathComponent(url.lastPathComponent)
        if let diskImage = UIImage(contentsOfFile: diskPath.path) {
            memoryCache.setObject(diskImage, forKey: url as NSURL)
            return diskImage
        }
        
        // 4. Download
        let task = Task {
            let (data, _) = try await URLSession.shared.data(from: url)
            guard let image = UIImage(data: data) else {
                throw ImageError.invalidData
            }
            
            // Save to caches
            memoryCache.setObject(image, forKey: url as NSURL)
            try? data.write(to: diskPath)
            
            loadingTasks.removeValue(forKey: url)
            return image
        }
        
        loadingTasks[url] = task
        return try await task.value
    }
    
    func clearMemory() {
        memoryCache.removeAllObjects()
    }
    
    func clearDisk() {
        try? FileManager.default.removeItem(at: diskCachePath)
        try? FileManager.default.createDirectory(at: diskCachePath, withIntermediateDirectories: true)
    }
}
```

---

# 🔀 Advanced: Mixing Concurrency Models

## async/await with GCD

```swift
// Wrap GCD in async/await
extension DispatchQueue {
    func asyncAwait<T>(_ work: @escaping () throws -> T) async throws -> T {
        try await withCheckedThrowingContinuation { continuation in
            self.async {
                do {
                    let result = try work()
                    continuation.resume(returning: result)
                } catch {
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}

// Usage
let result = try await DispatchQueue.global().asyncAwait {
    // Heavy synchronous work
    return processLargeFile()
}
```

## async/await with Completion Handlers

```swift
// Wrap completion handler in async/await
func fetchUser() async throws -> User {
    try await withCheckedThrowingContinuation { continuation in
        oldAPIWithCompletion { result in
            switch result {
            case .success(let user):
                continuation.resume(returning: user)
            case .failure(let error):
                continuation.resume(throwing: error)
            }
        }
    }
}

// For non-throwing
func fetchData() async -> Data? {
    await withCheckedContinuation { continuation in
        legacyFetch { data in
            continuation.resume(returning: data)
        }
    }
}
```

## Actors with Legacy Code

```swift
actor DataStore {
    private var data: [String: Any] = [:]
    
    func setValue(_ value: Any, forKey key: String) {
        data[key] = value
    }
    
    func getValue(forKey key: String) -> Any? {
        data[key]
    }
    
    // Bridge for non-async callers
    nonisolated func setValueSync(_ value: Any, forKey key: String) {
        Task {
            await setValue(value, forKey: key)
        }
    }
}
```

---

# 🔍 Advanced: Memory Debugging

## Using Instruments - Leaks

1. **Open Instruments**: Xcode → Product → Profile (⌘I)
2. **Select "Leaks"** template
3. **Run your app** and reproduce the leak scenario
4. **Look for leaked objects** in the timeline

## Using Memory Graph Debugger

1. **Run app** in debug mode
2. **Click memory graph** button: ![Debug Memory Graph](memory-graph-button)
3. **Look for cycles**: Objects with arrows going both ways

## Debug Deinit

```swift
class ViewController: UIViewController {
    deinit {
        print("✅ \(Self.self) was deallocated")
    }
}

// If you never see this print, you have a retain cycle!
```

## Weak-Self Debugging Helper

```swift
#if DEBUG
func checkRetainCycle<T: AnyObject>(_ object: T, file: String = #file, line: Int = #line) {
    let pointer = Unmanaged.passUnretained(object).toOpaque()
    let typeName = String(describing: type(of: object))
    
    DispatchQueue.main.asyncAfter(deadline: .now() + 2) { [weak object] in
        if object != nil {
            print("⚠️ POSSIBLE RETAIN CYCLE: \(typeName) at \(file):\(line)")
            print("   Object pointer: \(pointer)")
        }
    }
}
#endif

// Usage
class ViewController: UIViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        #if DEBUG
        checkRetainCycle(self)
        #endif
    }
}
```

## Autoreleasepool for Batch Operations

```swift
// ❌ Memory spike when processing large dataset
func processAllImages(_ images: [UIImage]) {
    for image in images {
        let processed = applyFilter(to: image)
        save(processed)
        // Memory builds up!
    }
}

// ✅ Controlled memory with autoreleasepool
func processAllImages(_ images: [UIImage]) {
    for image in images {
        autoreleasepool {
            let processed = applyFilter(to: image)
            save(processed)
            // Memory released each iteration
        }
    }
}
```

---

# 🏢 Advanced: Big Tech Patterns

## How Uber Handles Concurrency

**Problem:** Real-time map updates, location tracking, ride state changes

**Solution:**
```swift
// Centralized state with actor
actor RideStateManager {
    private(set) var currentState: RideState = .idle
    private var stateObservers: [(RideState) -> Void] = []
    
    func transition(to newState: RideState) {
        guard canTransition(from: currentState, to: newState) else { return }
        currentState = newState
        notifyObservers()
    }
    
    private func canTransition(from: RideState, to: RideState) -> Bool {
        // State machine validation
        switch (from, to) {
        case (.idle, .searching): return true
        case (.searching, .matched): return true
        case (.matched, .enRoute): return true
        case (.enRoute, .arrived): return true
        case (.arrived, .inProgress): return true
        case (.inProgress, .completed): return true
        case (_, .cancelled): return true
        default: return false
        }
    }
}
```

## How Amazon Handles Image Loading

**Problem:** Thousands of product images, variable network, memory constraints

**Solution:**
```swift
actor ImagePipeline {
    private let maxConcurrentDownloads = 4
    private var activeDownloads = 0
    private var pendingQueue: [(URL, CheckedContinuation<UIImage, Error>)] = []
    
    func loadImage(url: URL) async throws -> UIImage {
        // Check caches first (memory, disk)
        if let cached = await checkCaches(url: url) {
            return cached
        }
        
        // Throttle concurrent downloads
        if activeDownloads >= maxConcurrentDownloads {
            return try await withCheckedThrowingContinuation { continuation in
                pendingQueue.append((url, continuation))
            }
        }
        
        activeDownloads += 1
        defer { 
            activeDownloads -= 1
            processNextPending()
        }
        
        return try await download(url: url)
    }
    
    private func processNextPending() {
        guard !pendingQueue.isEmpty, activeDownloads < maxConcurrentDownloads else { return }
        let (url, continuation) = pendingQueue.removeFirst()
        
        Task {
            do {
                let image = try await download(url: url)
                continuation.resume(returning: image)
            } catch {
                continuation.resume(throwing: error)
            }
        }
    }
}
```

## How Google Handles Background Sync

**Problem:** Sync data when user not active, battery efficiency

**Solution:**
```swift
class BackgroundSyncManager {
    func registerBackgroundTasks() {
        BGTaskScheduler.shared.register(
            forTaskWithIdentifier: "com.app.sync",
            using: nil
        ) { task in
            self.handleSync(task as! BGProcessingTask)
        }
    }
    
    func scheduleSync() {
        let request = BGProcessingTaskRequest(identifier: "com.app.sync")
        request.requiresNetworkConnectivity = true
        request.requiresExternalPower = false
        
        try? BGTaskScheduler.shared.submit(request)
    }
    
    private func handleSync(_ task: BGProcessingTask) {
        let syncOperation = SyncOperation()
        
        task.expirationHandler = {
            syncOperation.cancel()
        }
        
        syncOperation.completionBlock = {
            task.setTaskCompleted(success: !syncOperation.isCancelled)
        }
        
        OperationQueue().addOperation(syncOperation)
    }
}
```

---

# 📝 Interview Preparation

## Question 1: "Explain the difference between GCD and async/await"

**Answer:**
"GCD is Apple's low-level concurrency framework using queues and closures. It's powerful but can lead to nested callbacks (pyramid of doom) and makes error handling awkward.

async/await, introduced in Swift 5.5, offers cleaner syntax that reads like synchronous code. It has built-in cancellation, better error handling with try/catch, and the compiler catches threading errors at compile time.

I'd use GCD for legacy codebases or when I need specific queue control. For new code on iOS 15+, I prefer async/await for readability and safety."

---

## Question 2: "What is a retain cycle and how do you prevent it?"

**Answer:**
"A retain cycle occurs when two or more objects hold strong references to each other, preventing ARC from deallocating them.

Common scenarios:
1. **Closures capturing self** - use `[weak self]`
2. **Delegates** - declare as `weak var`
3. **Timers** - use block-based API with weak self

Prevention checklist:
- Make delegate properties weak
- Use capture lists in closures
- Add deinit logging during development
- Use Memory Graph Debugger to visualize issues"

---

## Question 3: "How would you design a thread-safe cache?"

**Answer:**
"For iOS 15+, I'd use an Actor:

```swift
actor ImageCache {
    private var cache: [URL: UIImage] = [:]
    
    func get(_ url: URL) -> UIImage? { cache[url] }
    func set(_ image: UIImage, for url: URL) { cache[url] = image }
}
```

For older iOS, I'd use a concurrent queue with barrier writes:

```swift
class ThreadSafeCache {
    private var cache: [URL: UIImage] = [:]
    private let queue = DispatchQueue(label: "cache", attributes: .concurrent)
    
    func get(_ url: URL) -> UIImage? {
        queue.sync { cache[url] }
    }
    
    func set(_ image: UIImage, for url: URL) {
        queue.async(flags: .barrier) { self.cache[url] = image }
    }
}
```

Actor is simpler and compiler-enforced safe. The queue version offers more control but requires careful implementation."

---

## Question 4: "How would you handle concurrent network requests?"

**Answer:**
"I'd use TaskGroup for parallel requests with controlled concurrency:

```swift
func fetchAllProducts(ids: [String]) async throws -> [Product] {
    try await withThrowingTaskGroup(of: Product.self) { group in
        for id in ids {
            group.addTask { try await fetchProduct(id: id) }
        }
        
        var products: [Product] = []
        for try await product in group {
            products.append(product)
        }
        return products
    }
}
```

For controlling max concurrent requests, I'd wrap in an Actor with a semaphore pattern. For cancellation, each Task checks `Task.isCancelled` before expensive work."

---

## Question 5: "What happens when you access a property on a deallocated weak reference?"

**Answer:**
"Weak references automatically become nil when the object they point to is deallocated. Accessing a weak property after deallocation returns nil, not a crash.

```swift
weak var delegate: MyDelegate?
delegate?.doSomething()  // Safe - does nothing if nil
```

This is different from unowned, which assumes the object exists. Accessing an unowned reference after deallocation causes a crash.

Use weak when the referenced object might be deallocated while you still hold the reference. Use unowned only when you're certain the object will outlive the reference."

---

## Quick Reference

| Concept | Key Point |
|---------|-----------|
| Main thread | UI only, never block it |
| GCD | Low-level, powerful, verbose |
| async/await | High-level, clean, iOS 15+ |
| Actor | Thread-safe state container |
| strong | Default, increases retain count |
| weak | Optional, doesn't prevent deallocation |
| unowned | Non-optional, crashes if deallocated |
| Retain cycle | Strong references in both directions |
| Race condition | Concurrent access to shared mutable state |

---

> **Remember:** Concurrency bugs are the hardest to debug because they're non-deterministic. Design for safety first (Actors, serial queues), then optimize for performance only where profiling shows it's needed. 🚀
