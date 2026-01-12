# iOS Concurrency & Memory Management: Interview Mastery Guide
## Staff/Senior Engineer Level (Uber, Amazon, Google)

---

## 📚 Table of Contents
1. [Foundations (Beginner)](#part-1-foundations)
2. [Core iOS Concepts (SDE-2)](#part-2-core-ios-concepts)
3. [Advanced Topics (Big Tech Bar)](#part-3-advanced-topics)
4. [Real Interview Questions](#part-4-real-interview-questions)
5. [Integration with HLD](#part-5-using-concurrency--memory-in-hld)
6. [Interview Answer Templates](#part-6-interview-answer-templates)

---

# Part 1 — Foundations (Beginner Level)

## What is Concurrency in iOS?

**Simple English**: Concurrency means doing multiple things at the same time, or at least making it *look* like you're doing multiple things at once.

**Why it's needed in iOS**:
- **Responsiveness**: If you download an image while scrolling a feed, the screen should not freeze
- **Performance**: On multi-core iPhones, you can actually do multiple tasks simultaneously
- **User Experience**: Users expect instant interactions, even when data is loading

**Real-life Analogy**:
Imagine you're cooking dinner:
- **Serial (No concurrency)**: Boil water → Then chop vegetables → Then cook pasta. Very slow!
- **Concurrent**: Put water to boil, while it's heating, chop vegetables. Much faster!

In iOS:
- **Main thread** = The kitchen counter where you serve food (UI updates)
- **Background threads** = The stove, oven, and other appliances (heavy work like network calls, image processing)

---

## What is Memory Management in iOS?

**Simple English**: Memory management is about making sure your app doesn't waste space (memory) and doesn't crash by using objects that no longer exist.

**Why it matters**:
- iOS devices have limited RAM
- If your app uses too much memory, iOS will kill it
- If you access deleted objects, your app crashes

**Real-life Analogy**:
Think of your iPhone's memory as a parking lot:
- **strong reference** = You own a parking spot and the car stays there as long as you want
- **weak reference** = You know where a car is parked, but you don't own the spot. The car might leave without telling you
- **unowned** = You're sure the car will be there (but if it's not, you crash!)

---

## How ARC Works (Simple Terms)

**ARC = Automatic Reference Counting**

Every object in Swift has a "reference count" — a number that tracks how many places are using it.

```swift
class Car {
    var model: String
    init(model: String) {
        self.model = model
        print("\(model) is created")
    }
    deinit {
        print("\(model) is destroyed")
    }
}

var myCar: Car? = Car(model: "Tesla") // Reference count = 1
var friendCar = myCar                    // Reference count = 2
myCar = nil                               // Reference count = 1 (still alive!)
friendCar = nil                           // Reference count = 0 → DESTROYED
```

**Output**:
```
Tesla is created
Tesla is destroyed
```

**The Rule**: When reference count hits 0, the object is deallocated (destroyed) automatically.

---

## Real-Life Analogies

### 1. **Threads**
**Analogy**: Workers in a restaurant
- **Main thread** = The waiter who serves customers (must always be free to interact with users)
- **Background threads** = Chefs in the kitchen (do heavy work like cooking, which customers don't see)

### 2. **Queues**
**Analogy**: Lines at a coffee shop
- **Serial queue** = One line, one barista. Orders are processed one by one.
- **Concurrent queue** = Multiple baristas. Multiple orders can be processed at the same time.

### 3. **Async Tasks**
**Analogy**: Ordering food for delivery
- **Synchronous** = You wait at the door until food arrives (blocking)
- **Asynchronous** = You place an order and continue watching TV. You'll be notified when it arrives (non-blocking)

### 4. **Memory Ownership**
**Analogy**: Sharing a Netflix account
- **strong** = You pay for the account; it stays active as long as you want
- **weak** = You're using your friend's account; if they cancel, it's gone
- **unowned** = You're absolutely sure your friend will never cancel (but if they do, you crash!)

---

# Part 2 — Core iOS Concepts (SDE-2 Level)

## 2.1 GCD vs OperationQueue vs async/await

### **GCD (Grand Central Dispatch)**

**When to use**: Simple, fire-and-forget tasks. Lightweight concurrency.

```swift
// Example: Download image in background
DispatchQueue.global(qos: .userInitiated).async {
    let image = downloadImage() // Heavy work
    DispatchQueue.main.async {
        self.imageView.image = image // UI update on main thread
    }
}
```

**Pros**:
- Very fast and lightweight
- Built into iOS, no overhead

**Cons**:
- No cancellation support
- No dependency management
- Hard to manage complex workflows

---

### **OperationQueue**

**When to use**: Complex workflows with dependencies, cancellation, and priorities.

```swift
let queue = OperationQueue()

let downloadOp = BlockOperation {
    let data = downloadData()
    print("Downloaded: \(data)")
}

let processOp = BlockOperation {
    print("Processing data...")
}

// Set dependency: processOp runs AFTER downloadOp
processOp.addDependency(downloadOp)

queue.addOperations([downloadOp, processOp], waitUntilFinished: false)

// You can cancel individual operations
downloadOp.cancel()
```

**Pros**:
- Can cancel operations
- Can set dependencies (X runs before Y)
- Can set priorities

**Cons**:
- More overhead than GCD
- More complex API

---

### **async/await (Modern Swift)**

**When to use**: New code. Best for readability and structured concurrency.

```swift
func fetchUserProfile() async throws -> User {
    let data = try await networkManager.get("/user/profile")
    let user = try JSONDecoder().decode(User.self, from: data)
    return user
}

// Usage
Task {
    do {
        let user = try await fetchUserProfile()
        print("Welcome, \(user.name)!")
    } catch {
        print("Error: \(error)")
    }
}
```

**Pros**:
- Clean, readable code (looks synchronous but isn't)
- Compiler-enforced thread safety with actors
- Built-in cancellation via `Task`

**Cons**:
- Requires iOS 15+
- Learning curve if you're used to GCD

---

### **Comparison Table**

| Feature | GCD | OperationQueue | async/await |
|---------|-----|----------------|-------------|
| **Performance** | Fastest | Medium | Fast |
| **Cancellation** | ❌ No | ✅ Yes | ✅ Yes |
| **Dependencies** | ❌ Manual | ✅ Built-in | ✅ Structured |
| **Readability** | Medium | Low | ✅ Excellent |
| **iOS Version** | All | All | iOS 15+ |
| **Use Case** | Simple tasks | Complex workflows | Modern apps |

---

### **Common Mistakes**

#### ❌ Mistake 1: UI updates on background thread
```swift
DispatchQueue.global().async {
    self.label.text = "Done" // CRASH! UI can only be updated on main thread
}
```

#### ✅ Fix:
```swift
DispatchQueue.global().async {
    let result = heavyComputation()
    DispatchQueue.main.async {
        self.label.text = "Done"
    }
}
```

#### ❌ Mistake 2: Blocking the main thread
```swift
let data = URLSession.shared.data(from: url) // FREEZES UI!
```

#### ✅ Fix:
```swift
Task {
    let data = try await URLSession.shared.data(from: url)
    // UI updates here
}
```

#### ❌ Mistake 3: Not canceling operations
```swift
// User navigates away, but download continues
queue.addOperation {
    downloadLargeFile() // Wastes battery and bandwidth!
}
```

#### ✅ Fix:
```swift
let operation = BlockOperation {
    downloadLargeFile()
}
queue.addOperation(operation)

// In viewWillDisappear:
operation.cancel()
```

---

## 2.2 Actors and Thread Safety

### Why Actors Exist

**Problem**: When multiple threads access the same data, you get **data races**.

```swift
class Counter {
    var value = 0
    
    func increment() {
        value += 1 // NOT thread-safe!
    }
}

let counter = Counter()
DispatchQueue.concurrentPerform(iterations: 1000) { _ in
    counter.increment()
}
print(counter.value) // Expected: 1000, Actual: Random number (race condition!)
```

**Why it fails**: `value += 1` is actually three operations:
1. Read current value
2. Add 1
3. Write new value

If two threads do this simultaneously, they can overwrite each other's changes.

---

### **Solution 1: Locks (Old Way)**

```swift
class Counter {
    private var value = 0
    private let lock = NSLock()
    
    func increment() {
        lock.lock()
        value += 1
        lock.unlock()
    }
    
    func getValue() -> Int {
        lock.lock()
        defer { lock.unlock() }
        return value
    }
}
```

**Problems with locks**:
- Easy to forget to unlock → deadlock
- Verbose code
- No compiler help

---

### **Solution 2: Serial Queue (Better)**

```swift
class Counter {
    private var value = 0
    private let queue = DispatchQueue(label: "counter.queue")
    
    func increment() {
        queue.async {
            self.value += 1
        }
    }
    
    func getValue(completion: @escaping (Int) -> Void) {
        queue.async {
            completion(self.value)
        }
    }
}
```

**Better**, but still requires discipline.

---

### **Solution 3: Actors (Best for iOS 15+)**

```swift
actor Counter {
    private var value = 0
    
    func increment() {
        value += 1
    }
    
    func getValue() -> Int {
        return value
    }
}

// Usage
let counter = Counter()
Task {
    await counter.increment() // Compiler ensures thread-safe access!
    let val = await counter.getValue()
    print(val)
}
```

**Why actors are better**:
- ✅ **Compiler-enforced** thread safety (you can't forget)
- ✅ Clean, simple code
- ✅ No manual locks or queues

---

### **When to Use Each**

| Approach | When to Use |
|----------|-------------|
| **Actor** | New code, iOS 15+, best for data models |
| **Serial Queue** | Legacy code, need iOS 13+ support |
| **Locks** | Very rare, only if you need performance-critical sync code |

---

## 2.3 ARC Fundamentals

### **strong / weak / unowned**

```swift
class Person {
    var name: String
    var apartment: Apartment?
    
    init(name: String) {
        self.name = name
    }
    deinit {
        print("\(name) is deallocated")
    }
}

class Apartment {
    var unit: String
    weak var tenant: Person? // WEAK to avoid retain cycle
    
    init(unit: String) {
        self.unit = unit
    }
    deinit {
        print("Apartment \(unit) is deallocated")
    }
}

var john: Person? = Person(name: "John")
var unit4A: Apartment? = Apartment(unit: "4A")

john?.apartment = unit4A
unit4A?.tenant = john

john = nil // John is deallocated because apartment -> tenant is WEAK
unit4A = nil // Apartment is also deallocated
```

**Output**:
```
John is deallocated
Apartment 4A is deallocated
```

---

### **Closure Capture Lists**

#### ❌ **Retain Cycle Example**

```swift
class ViewController: UIViewController {
    var name = "Home"
    
    func setupHandler() {
        apiManager.onComplete = {
            print(self.name) // RETAIN CYCLE! self -> apiManager -> closure -> self
        }
    }
    
    deinit {
        print("ViewController deallocated") // This will NEVER print!
    }
}
```

#### ✅ **Fix with Weak Self**

```swift
func setupHandler() {
    apiManager.onComplete = { [weak self] in
        print(self?.name ?? "Unknown") // No retain cycle
    }
}
```

#### ✅ **Or Unowned (if you're 100% sure self will outlive the closure)**

```swift
func setupHandler() {
    apiManager.onComplete = { [unowned self] in
        print(self.name) // No retain cycle, but crashes if self is nil!
    }
}
```

**When to use**:
- **weak**: When self might be deallocated before the closure runs (most cases)
- **unowned**: When you're absolutely certain self will outlive the closure (rare)

---

### **Retain Cycles in Different Scenarios**

#### **1. Delegates (Classic Example)**

```swift
protocol DataManagerDelegate: AnyObject {
    func didReceiveData()
}

class DataManager {
    weak var delegate: DataManagerDelegate? // MUST be weak!
}

class ViewController: UIViewController, DataManagerDelegate {
    let dataManager = DataManager()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        dataManager.delegate = self // No retain cycle because delegate is weak
    }
    
    func didReceiveData() {
        print("Data received")
    }
}
```

---

#### **2. Closures in Async Tasks**

```swift
class ImageLoader {
    var image: UIImage?
    
    func loadImage(completion: @escaping (UIImage) -> Void) {
        DispatchQueue.global().async { [weak self] in
            guard let self = self else { return }
            let downloadedImage = self.downloadImage()
            completion(downloadedImage)
        }
    }
}
```

---

#### **3. Timers**

```swift
class TimerManager {
    var timer: Timer?
    
    func startTimer() {
        timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { [weak self] _ in
            self?.timerFired()
        }
    }
    
    func timerFired() {
        print("Tick")
    }
    
    deinit {
        timer?.invalidate() // IMPORTANT: Invalidate timer to break retain cycle
    }
}
```

---

#### **4. Combine Subscriptions**

```swift
import Combine

class ViewModel {
    var cancellables = Set<AnyCancellable>()
    
    func fetchData() {
        apiService.fetchUser()
            .sink { [weak self] user in
                self?.updateUI(with: user)
            }
            .store(in: &cancellables)
    }
}
```

---

# Part 3 — Advanced Topics (Big Tech Bar)

## 3.1 Designing Thread-Safe Components at Scale

### **Problem**: Building a thread-safe cache used by hundreds of view controllers

**Requirements**:
- Fast read access (many threads reading simultaneously)
- Safe write access (one thread writing at a time)
- No data races
- Minimal performance overhead

### **Solution: Reader-Writer Lock Pattern**

```swift
import Foundation

actor ImageCache {
    private var cache: [String: UIImage] = [:]
    private let maxCacheSize = 50
    
    func getImage(for key: String) -> UIImage? {
        return cache[key]
    }
    
    func setImage(_ image: UIImage, for key: String) {
        if cache.count >= maxCacheSize {
            // Evict oldest entry (simple LRU)
            if let firstKey = cache.keys.first {
                cache.removeValue(forKey: firstKey)
            }
        }
        cache[key] = image
    }
    
    func clear() {
        cache.removeAll()
    }
}

// Usage
let cache = ImageCache()

Task {
    await cache.setImage(myImage, for: "profile.jpg")
    let image = await cache.getImage(for: "profile.jpg")
}
```

**Why Actor is Perfect Here**:
- Automatic serialization of all access
- Compiler-enforced safety
- Clean, simple code

---

### **Alternative: NSCache (Built-in Thread-Safe Cache)**

```swift
class ImageCacheManager {
    private let cache = NSCache<NSString, UIImage>()
    
    init() {
        cache.countLimit = 50 // Max 50 images
        cache.totalCostLimit = 100 * 1024 * 1024 // 100 MB
    }
    
    func getImage(for key: String) -> UIImage? {
        return cache.object(forKey: key as NSString)
    }
    
    func setImage(_ image: UIImage, for key: String) {
        cache.setObject(image, forKey: key as NSString)
    }
}
```

**Pros**: Already thread-safe, automatic eviction
**Cons**: Less control, Objective-C API

---

## 3.2 Mixing async/await with Legacy GCD Code

### **Problem**: You have old GCD code and new async/await code interacting

### **Bridging GCD to async/await**

```swift
// Legacy GCD function
func fetchDataOldWay(completion: @escaping (Data?, Error?) -> Void) {
    DispatchQueue.global().async {
        // Simulate network call
        sleep(1)
        let data = "Response".data(using: .utf8)
        completion(data, nil)
    }
}

// Bridge to async/await
func fetchDataNewWay() async throws -> Data {
    return try await withCheckedThrowingContinuation { continuation in
        fetchDataOldWay { data, error in
            if let error = error {
                continuation.resume(throwing: error)
            } else if let data = data {
                continuation.resume(returning: data)
            } else {
                continuation.resume(throwing: NSError(domain: "Unknown", code: -1))
            }
        }
    }
}

// Usage
Task {
    let data = try await fetchDataNewWay()
    print("Data: \(data)")
}
```

**Key**: `withCheckedThrowingContinuation` lets you bridge completion handlers to async/await.

---

### **Bridging async/await to GCD**

```swift
// New async function
func modernFetch() async throws -> String {
    try await Task.sleep(nanoseconds: 1_000_000_000)
    return "Modern data"
}

// Bridge to completion handler for legacy code
func modernFetchWithCompletion(completion: @escaping (String?, Error?) -> Void) {
    Task {
        do {
            let result = try await modernFetch()
            completion(result, nil)
        } catch {
            completion(nil, error)
        }
    }
}
```

---

## 3.3 Actors vs Serial Queues vs Locks (Trade-offs)

| Aspect | Actor | Serial Queue | Lock (NSLock) |
|--------|-------|--------------|---------------|
| **Thread Safety** | ✅ Compiler-enforced | ⚠️ Manual discipline | ⚠️ Manual discipline |
| **Readability** | ✅ Excellent | Medium | Poor |
| **Performance** | Fast | Fast | Fastest |
| **Cancellation** | ✅ Built-in | ⚠️ Manual | ❌ No |
| **Deadlock Risk** | ❌ No | ⚠️ Possible | ⚠️ High |
| **iOS Version** | iOS 15+ | All | All |
| **Best For** | Data models, modern apps | Background tasks | Very tight critical sections |

---

## 3.4 Advanced ARC Behavior

### **Object Lifetime Gotcha**

```swift
class Logger {
    var name: String
    init(name: String) {
        self.name = name
        print("\(name) created")
    }
    deinit {
        print("\(name) destroyed")
    }
}

func test() {
    let logger = Logger(name: "Test")
    DispatchQueue.global().async { [weak logger] in
        sleep(1)
        print(logger?.name ?? "Already deallocated")
    }
    print("Exiting test()")
}

test()
// Wait a bit...
```

**Output**:
```
Test created
Exiting test()
Test destroyed
Already deallocated
```

**Key Insight**: `logger` is deallocated as soon as `test()` exits because the closure captured it **weakly**.

---

### **deinit Timing**

```swift
class Resource {
    init() {
        print("Resource acquired")
    }
    deinit {
        print("Resource released")
    }
}

func doWork() {
    let res = Resource()
    print("Working...")
} // deinit is called HERE, immediately after the function exits

doWork()
print("After doWork")
```

**Output**:
```
Resource acquired
Working...
Resource released
After doWork
```

**Takeaway**: `deinit` is called **as soon as** the last strong reference is removed.

---

## 3.5 Detecting and Fixing Memory Leaks Using Instruments

### **Steps to Find Leaks**:

1. **Run your app in Xcode**
2. **Product → Profile** (or Cmd+I)
3. **Choose "Leaks" instrument**
4. **Reproduce the scenario** (e.g., open and close a view controller multiple times)
5. **Look for red bars** in the Leaks timeline

### **Common Leak Patterns**:

#### **Leak 1: Delegate Retain Cycle**
```swift
class Manager {
    var delegate: SomeDelegate? // Should be WEAK!
}
```

#### **Leak 2: Closure Retain Cycle**
```swift
someClosure = {
    self.doSomething() // Should be [weak self]
}
```

#### **Leak 3: Timer Retain Cycle**
```swift
timer = Timer.scheduledTimer(withTimeInterval: 1.0, repeats: true) { _ in
    self.update() // Should be [weak self] and timer.invalidate() in deinit
}
```

---

## 3.6 Concurrency Bugs That Appear Only Under Load

### **Example: Data Race in Production**

```swift
class AnalyticsManager {
    private var events: [String] = []
    
    func logEvent(_ event: String) {
        events.append(event) // CRASH under heavy load!
    }
}
```

**Why it crashes**: If 1000 users trigger `logEvent` simultaneously, multiple threads modify `events` concurrently → crash.

**Fix**:
```swift
actor AnalyticsManager {
    private var events: [String] = []
    
    func logEvent(_ event: String) {
        events.append(event) // Safe!
    }
}
```

---

## 3.7 Battery and Performance Trade-offs

### **Problem**: Background location tracking drains battery

**Bad**:
```swift
locationManager.allowsBackgroundLocationUpdates = true
locationManager.desiredAccuracy = kCLLocationAccuracyBest // Drains battery!
locationManager.startUpdatingLocation()
```

**Good**:
```swift
locationManager.desiredAccuracy = kCLLocationAccuracyHundredMeters // Good enough for ride tracking
locationManager.distanceFilter = 100 // Only update every 100 meters
locationManager.startUpdatingLocation()
```

**Key**: Balance accuracy vs battery life based on the use case.

---

# Part 4 — Real Interview Questions

## Question 1: Implement a Thread-Safe Cache in Swift

### **Problem Statement**
Design a generic, thread-safe cache that:
- Stores key-value pairs
- Has a maximum size (evicts oldest when full)
- Supports concurrent reads
- Is safe for writes

### **Thought Process**
1. **Thread safety**: Use `actor` for automatic serialization
2. **Eviction policy**: Simple FIFO or LRU
3. **Performance**: Minimize lock contention

### **Solution**

```swift
actor ThreadSafeCache<Key: Hashable, Value> {
    private var storage: [Key: Value] = [:]
    private var accessOrder: [Key] = []
    private let maxSize: Int
    
    init(maxSize: Int = 100) {
        self.maxSize = maxSize
    }
    
    func get(_ key: Key) -> Value? {
        // Update access order for LRU
        if let index = accessOrder.firstIndex(of: key) {
            accessOrder.remove(at: index)
            accessOrder.append(key)
        }
        return storage[key]
    }
    
    func set(_ key: Key, value: Value) {
        // If key exists, update access order
        if storage[key] != nil {
            if let index = accessOrder.firstIndex(of: key) {
                accessOrder.remove(at: index)
            }
        }
        
        // Evict if at capacity
        if storage.count >= maxSize, storage[key] == nil {
            if let oldestKey = accessOrder.first {
                storage.removeValue(forKey: oldestKey)
                accessOrder.removeFirst()
            }
        }
        
        storage[key] = value
        accessOrder.append(key)
    }
    
    func remove(_ key: Key) {
        storage.removeValue(forKey: key)
        if let index = accessOrder.firstIndex(of: key) {
            accessOrder.remove(at: index)
        }
    }
    
    func clear() {
        storage.removeAll()
        accessOrder.removeAll()
    }
}

// Usage
let cache = ThreadSafeCache<String, UIImage>(maxSize: 50)
Task {
    await cache.set("profile", value: profileImage)
    let img = await cache.get("profile")
}
```

### **Time & Space Complexity**
- **get**: O(n) due to access order update (could optimize with linked list)
- **set**: O(n) worst case
- **Space**: O(maxSize)

### **Interview Tips**
- Mention you could optimize with a doubly-linked list for O(1) LRU
- Discuss trade-offs: actor vs NSCache

---

## Question 2: Why is This Code Leaking Memory?

### **Buggy Code**

```swift
class ChatViewController: UIViewController {
    var messageHandler: (() -> Void)?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        messageHandler = {
            self.refreshMessages()
        }
        
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(newMessageReceived),
            name: .newMessage,
            object: nil
        )
    }
    
    @objc func newMessageReceived() {
        messageHandler?()
    }
    
    func refreshMessages() {
        print("Refreshing...")
    }
}
```

### **Why It Leaks**
1. **Closure capture**: `messageHandler` captures `self` strongly → retain cycle
2. **Notification observer**: Not removed in `deinit` → observer keeps reference to `self`

### **Fixed Code**

```swift
class ChatViewController: UIViewController {
    var messageHandler: (() -> Void)?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        messageHandler = { [weak self] in
            self?.refreshMessages()
        }
        
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(newMessageReceived),
            name: .newMessage,
            object: nil
        )
    }
    
    @objc func newMessageReceived() {
        messageHandler?()
    }
    
    func refreshMessages() {
        print("Refreshing...")
    }
    
    deinit {
        NotificationCenter.default.removeObserver(self)
    }
}
```

---

## Question 3: Convert This GCD Code to async/await

### **Original GCD Code**

```swift
func fetchUserData(userId: String, completion: @escaping (User?, Error?) -> Void) {
    DispatchQueue.global().async {
        let url = URL(string: "https://api.example.com/users/\(userId)")!
        
        let task = URLSession.shared.dataTask(with: url) { data, response, error in
            if let error = error {
                completion(nil, error)
                return
            }
            
            guard let data = data else {
                completion(nil, NSError(domain: "NoData", code: -1))
                return
            }
            
            do {
                let user = try JSONDecoder().decode(User.self, from: data)
                completion(user, nil)
            } catch {
                completion(nil, error)
            }
        }
        task.resume()
    }
}
```

### **async/await Version**

```swift
func fetchUserData(userId: String) async throws -> User {
    let url = URL(string: "https://api.example.com/users/\(userId)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    let user = try JSONDecoder().decode(User.self, from: data)
    return user
}

// Usage
Task {
    do {
        let user = try await fetchUserData(userId: "123")
        print("User: \(user.name)")
    } catch {
        print("Error: \(error)")
    }
}
```

**Benefits**:
- ✅ Much cleaner, easier to read
- ✅ No completion handler nesting
- ✅ Automatic error propagation
- ✅ Built-in cancellation support

---

## Question 4: Design a Download Manager with Cancellation and Priority

### **Requirements**
- Download multiple files concurrently
- Support cancellation of individual downloads
- Support priority (high-priority downloads first)
- Thread-safe

### **Solution**

```swift
enum DownloadPriority: Int, Comparable {
    case low = 0
    case medium = 1
    case high = 2
    
    static func < (lhs: DownloadPriority, rhs: DownloadPriority) -> Bool {
        return lhs.rawValue < rhs.rawValue
    }
}

actor DownloadManager {
    private var activeTasks: [String: Task<Data, Error>] = [:]
    
    func download(
        from url: URL,
        priority: DownloadPriority = .medium
    ) async throws -> Data {
        let key = url.absoluteString
        
        // If already downloading, return existing task
        if let existingTask = activeTasks[key] {
            return try await existingTask.value
        }
        
        let task = Task(priority: taskPriority(from: priority)) {
            defer { Task { await self.removeTask(for: key) } }
            let (data, _) = try await URLSession.shared.data(from: url)
            return data
        }
        
        activeTasks[key] = task
        return try await task.value
    }
    
    func cancelDownload(for url: URL) {
        let key = url.absoluteString
        activeTasks[key]?.cancel()
        activeTasks.removeValue(forKey: key)
    }
    
    func cancelAllDownloads() {
        for task in activeTasks.values {
            task.cancel()
        }
        activeTasks.removeAll()
    }
    
    private func removeTask(for key: String) {
        activeTasks.removeValue(forKey: key)
    }
    
    private func taskPriority(from priority: DownloadPriority) -> TaskPriority {
        switch priority {
        case .low: return .low
        case .medium: return .medium
        case .high: return .high
        }
    }
}

// Usage
let downloadManager = DownloadManager()

Task {
    do {
        let data = try await downloadManager.download(
            from: URL(string: "https://example.com/file.pdf")!,
            priority: .high
        )
        print("Downloaded: \(data.count) bytes")
    } catch {
        print("Download failed: \(error)")
    }
}

// Cancel specific download
Task {
    await downloadManager.cancelDownload(for: URL(string: "https://example.com/file.pdf")!)
}
```

**Time Complexity**: O(1) for download/cancel operations (dictionary lookup)
**Space Complexity**: O(n) where n = number of active downloads
**Memory Impact**: Minimal overhead, tasks are cleaned up automatically

---

## Question 5: Explain When unowned is Unsafe with an Example

### **Problem Scenario**

```swift
class RideTracker {
    var currentLocation: Location?
    
    deinit {
        print("RideTracker deallocated")
    }
}

class NavigationView {
    var tracker: RideTracker?
    
    var updateHandler: (() -> Void)?
    
    func startTracking(tracker: RideTracker) {
        self.tracker = tracker
        
        // DANGEROUS: Using unowned
        updateHandler = { [unowned tracker] in
            print("Location: \(tracker.currentLocation)")
        }
        
        scheduleUpdates()
    }
    
    func scheduleUpdates() {
        DispatchQueue.main.asyncAfter(deadline: .now() + 2) {
            self.updateHandler?()
        }
    }
}

// Usage that CRASHES
let navView = NavigationView()
var tracker: RideTracker? = RideTracker()
navView.startTracking(tracker: tracker!)

tracker = nil // Tracker is deallocated

// 2 seconds later... CRASH! updateHandler tries to access deallocated tracker
```

### **Why It Crashes**
- `unowned` means "I promise this object will outlive the closure"
- When `tracker` is deallocated, `unowned tracker` becomes a **dangling pointer**
- Accessing it = crash

### **Safe Alternative: Use weak**

```swift
func startTracking(tracker: RideTracker) {
    self.tracker = tracker
    
    updateHandler = { [weak tracker] in
        guard let tracker = tracker else {
            print("Tracker deallocated, skipping update")
            return
        }
        print("Location: \(tracker.currentLocation)")
    }
    
    scheduleUpdates()
}
```

**When to use unowned**:
- **Almost never**, unless you're 100% certain the captured object outlives the closure
- Example: closure defined inside a method that executes synchronously

```swift
class Processor {
    func process() {
        let data = [1, 2, 3]
        data.forEach { [unowned self] value in
            self.handleValue(value) // Safe: self definitely outlives this synchronous closure
        }
    }
    
    func handleValue(_ value: Int) {
        print(value)
    }
}
```

---

# Part 5 — Using Concurrency & Memory in HLD

## Example: Ride-Tracking Screen (Uber-like)

### **High-Level System Architecture**

```
┌─────────────────────────────────────────┐
│         RideTrackingViewController      │
│  - Shows map, driver location, ETA      │
└───────────────┬─────────────────────────┘
                │
        ┌───────┴───────┐
        ▼               ▼
┌──────────────┐  ┌──────────────┐
│ LocationMgr  │  │ NetworkMgr   │
│  (Actor)     │  │  (Actor)     │
└──────────────┘  └──────────────┘
        │               │
        ▼               ▼
┌──────────────┐  ┌──────────────┐
│  MapView     │  │  API Service │
└──────────────┘  └──────────────┘
```

---

### **Where Concurrency is Required**

1. **Location updates**: Background thread (Core Location delegates run on a separate queue)
2. **Network calls**: Fetch driver location every 5 seconds → async/await
3. **Map rendering**: Main thread only (UI constraint)
4. **ETA calculation**: Background thread (CPU-intensive)

---

### **Implementation with async/await and Actors**

```swift
// Thread-safe location manager
actor LocationManager {
    private var currentLocation: CLLocation?
    private let manager = CLLocationManager()
    
    func startTracking() {
        manager.requestWhenInUseAuthorization()
        manager.startUpdatingLocation()
    }
    
    func updateLocation(_ location: CLLocation) {
        currentLocation = location
    }
    
    func getCurrentLocation() -> CLLocation? {
        return currentLocation
    }
}

// Thread-safe network manager
actor RideNetworkManager {
    func fetchDriverLocation(rideId: String) async throws -> DriverLocation {
        let url = URL(string: "https://api.example.com/rides/\(rideId)/driver")!
        let (data, _) = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode(DriverLocation.self, from: data)
    }
}

// Main view controller
class RideTrackingViewController: UIViewController {
    private let locationManager = LocationManager()
    private let networkManager = RideNetworkManager()
    private var updateTask: Task<Void, Never>?
    
    @IBOutlet weak var mapView: MKMapView!
    @IBOutlet weak var etaLabel: UILabel!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        startRideTracking()
    }
    
    func startRideTracking() {
        Task {
            await locationManager.startTracking()
        }
        
        // Poll driver location every 5 seconds
        updateTask = Task {
            while !Task.isCancelled {
                await fetchAndUpdateDriverLocation()
                try? await Task.sleep(nanoseconds: 5_000_000_000) // 5 seconds
            }
        }
    }
    
    func fetchAndUpdateDriverLocation() async {
        do {
            let driverLoc = try await networkManager.fetchDriverLocation(rideId: "123")
            
            // Update UI on main thread
            await MainActor.run {
                updateMapWithDriver(location: driverLoc)
                updateETA(driverLoc.eta)
            }
        } catch {
            print("Error fetching driver location: \(error)")
        }
    }
    
    @MainActor
    func updateMapWithDriver(location: DriverLocation) {
        let coordinate = CLLocationCoordinate2D(
            latitude: location.lat,
            longitude: location.lng
        )
        // Update map annotation
        mapView.centerCoordinate = coordinate
    }
    
    @MainActor
    func updateETA(_ eta: Int) {
        etaLabel.text = "\(eta) min"
    }
    
    deinit {
        updateTask?.cancel() // Stop polling when view is dismissed
    }
}
```

---

### **How Memory Ownership is Managed**

1. **LocationManager**: Owned by `RideTrackingViewController`, lives as long as the view controller
2. **updateTask**: Canceled in `deinit` to prevent retain cycles
3. **Closures**: Use `@MainActor.run` instead of capturing `self` weakly (safer)

---

### **How Retain Cycles are Prevented**

```swift
// ❌ BAD: Would create retain cycle
updateTask = Task {
    while true {
        self.fetchAndUpdateDriverLocation() // Captures self strongly!
        // ...
    }
}

// ✅ GOOD: No capture needed, async context handles it
updateTask = Task {
    while !Task.isCancelled {
        await fetchAndUpdateDriverLocation() // No self needed in async context
        // ...
    }
}
```

---

### **How Thread Safety is Guaranteed**

1. **Actors**: `LocationManager` and `RideNetworkManager` are actors → automatic serialization
2. **MainActor**: All UI updates wrapped in `@MainActor.run { ... }`
3. **Task cancellation**: `updateTask?.cancel()` ensures no dangling tasks

---

### **How This Scales as Users Grow**

| Aspect | Approach |
|--------|----------|
| **Multiple concurrent rides** | Each ride gets its own `updateTask`, isolated by actor |
| **Memory pressure** | Cancel background tasks when app enters background |
| **Battery optimization** | Increase polling interval (5s → 10s) when app is backgrounded |
| **Network efficiency** | Batch requests, use WebSockets for live updates |

---

### **HLD Discussion Points**

In an interview, you'd draw this diagram and explain:

1. **"I use actors for thread safety"** → LocationManager, NetworkManager
2. **"I use async/await for readability"** → Clean, sequential-looking async code
3. **"I use Task for structured concurrency"** → Automatic cancellation, no leaks
4. **"I use MainActor for UI updates"** → Compiler-enforced main thread safety
5. **"I cancel tasks in deinit"** → Prevents memory leaks and wasted resources

---

# Part 6 — Interview Answer Templates

## Template 1: Explaining Concurrency to an Interviewer

**Interviewer**: "Explain how you'd handle concurrency in this feature."

**Your Answer**:
> "I would use **async/await** for the asynchronous work because it provides clean, readable code and structured concurrency. For example, when fetching data from the API, I'd write:
> 
> ```swift
> let data = try await fetchData()
> ```
> 
> This runs on a background thread automatically, and when the data arrives, I'd update the UI using `MainActor`:
> 
> ```swift
> await MainActor.run {
>     self.tableView.reloadData()
> }
> ```
> 
> For thread-safe shared state, like a cache, I'd use an **actor** to ensure all access is serialized and prevent data races."

---

## Template 2: Justifying Design Choices

**Interviewer**: "Why did you choose an actor over a serial queue?"

**Your Answer**:
> "I chose an actor because:
> 1. **Compiler-enforced safety**: With actors, the compiler prevents me from accidentally accessing shared state from multiple threads. With a serial queue, I have to remember to dispatch every access, which is error-prone.
> 2. **Readability**: Actor syntax is cleaner – I just use `await` instead of wrapping everything in `queue.async`.
> 3. **Modern**: Actors are the Swift-native approach to concurrency, and they integrate seamlessly with async/await.
> 
> If I needed to support iOS 13 or 14, I'd fall back to a serial queue, but for new code targeting iOS 15+, actors are the better choice."

---

## Template 3: Handling Memory Management Questions

**Interviewer**: "How do you prevent retain cycles?"

**Your Answer**:
> "I follow these rules:
> 1. **Delegates**: Always mark as `weak` (e.g., `weak var delegate: SomeDelegate?`)
> 2. **Closures**: Use `[weak self]` in closures that capture `self`, especially in async blocks or callbacks
> 3. **Timers**: Invalidate timers in `deinit` and use `[weak self]` in the timer's closure
> 4. **Combine/async**: Use `[weak self]` in `sink` and `map` closures
> 
> I also use **Instruments** to profile memory leaks. The Leaks instrument shows me exactly which objects aren't being deallocated, and I can trace back to the source of the retain cycle."

---

## Template 4: Common Follow-Up Questions

### **Q: "What's the difference between weak and unowned?"**

**A**: 
> "Both break retain cycles, but:
> - **weak**: The reference becomes `nil` when the object is deallocated. Safe, but requires unwrapping.
> - **unowned**: Assumes the object will never be `nil`, so no unwrapping needed. But if the object IS deallocated, you crash.
> 
> I almost always use `weak` because it's safer. I only use `unowned` in very specific cases where I'm 100% certain the object will outlive the closure, like in synchronous closures."

### **Q: "How do you decide between GCD and async/await?"**

**A**:
> "For new code, I use **async/await** because:
> - It's more readable
> - It has built-in cancellation via `Task`
> - It integrates with actors for thread safety
> 
> I only use GCD if:
> - I need to support older iOS versions (pre-iOS 15)
> - I have existing GCD code that's working fine and doesn't need to change
> 
> For complex workflows with dependencies, I'd consider **OperationQueue**, but async/await with structured concurrency usually covers those cases too."

### **Q: "How do you test concurrent code?"**

**A**:
> "I use a few strategies:
> 1. **Unit tests with expectations**: Use `XCTestExpectation` to wait for async operations
> 2. **Thread sanitizer**: Xcode's Thread Sanitizer detects data races at runtime
> 3. **Stress testing**: Run tests with high concurrency (e.g., 1000 simultaneous requests) to expose race conditions
> 4. **Instruments**: Use the Time Profiler to detect performance bottlenecks caused by excessive locking
> 
> For example:
> ```swift
> func testConcurrentCacheAccess() {
>     let cache = ThreadSafeCache<String, Int>()
>     let expectation = expectation(description: "Concurrent access")
>     expectation.expectedFulfillmentCount = 100
>     
>     for i in 0..<100 {
>         Task {
>             await cache.set("\(i)", value: i)
>             expectation.fulfill()
>         }
>     }
>     
>     wait(for: [expectation], timeout: 5.0)
> }
> ```
> "

---

## Template 5: Discussing Performance vs Battery Trade-offs

**Interviewer**: "How do you balance performance and battery life?"

**Your Answer**:
> "I follow these principles:
> 1. **Reduce frequency**: For location tracking, I use `distanceFilter` to update only when the user moves significantly (e.g., 100 meters), not continuously.
> 2. **Lower accuracy when possible**: Use `kCLLocationAccuracyHundredMeters` instead of `kCLLocationAccuracyBest` for features like ride tracking.
> 3. **Debounce network requests**: Instead of making an API call on every keystroke, I wait until the user stops typing for 500ms.
> 4. **Background task management**: Use `BGTaskScheduler` for non-urgent work, which iOS schedules opportunistically when the device is plugged in and idle.
> 5. **Profile with Instruments**: Use the Energy Log instrument to see which parts of my app consume the most power.
> 
> For example, in a feed app, I'd:
> - Preload images only for visible cells
> - Pause image loading when scrolling fast
> - Cancel pending downloads when cells are reused"

---

## Summary: Key Takeaways

### **For Concurrency**:
- Use **async/await** for modern, readable asynchronous code
- Use **actors** for thread-safe shared state
- Use **MainActor** to enforce UI updates on the main thread
- Use **Task** for structured concurrency and automatic cancellation

### **For Memory Management**:
- Use **weak** for delegates, closures, and timers
- Use **[weak self]** in async closures to prevent retain cycles
- Invalidate timers in `deinit`
- Use **Instruments** to detect leaks

### **For Interviews**:
- Explain your choices clearly
- Mention trade-offs (actor vs queue, weak vs unowned)
- Show code examples
- Discuss testing and profiling strategies
- Connect to real-world scenarios (ride tracking, feeds, chat)

---

**You're now ready to ace concurrency and memory management questions at Uber, Amazon, Google, and any Big Tech company!** 🚀
