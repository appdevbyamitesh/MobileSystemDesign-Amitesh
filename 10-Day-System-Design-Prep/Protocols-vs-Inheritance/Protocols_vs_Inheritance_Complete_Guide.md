# Protocols vs Inheritance in Swift: Complete iOS System Design Guide

> A comprehensive deep dive from beginner to advanced level, focused on real iOS app architecture, interview preparation, and big tech practices.

---

## Table of Contents

1. [Beginner Level (Noob)](#beginner-level-noob)
2. [Intermediate Level (SDE-2 Ready)](#intermediate-level-sde-2-ready)
3. [Advanced Level (Big Tech Interviews)](#advanced-level-big-tech-interviews)
4. [Interview Preparation Section](#interview-preparation-section)

---

# Beginner Level (Noob)

## What is Inheritance? (Simple English)

**Inheritance** is like a family tree. Imagine you have a parent class that knows how to do certain things. When you create a child class, it automatically gets all the abilities of the parent — like how a child inherits traits from their parents.

> **Simple Definition:** Inheritance means one class gets all the properties and methods from another class automatically.

### Real-Life Analogy

Think of a **Toyota Car**:
- The parent is a generic "Car" that has wheels, engine, and can drive
- A "Toyota Camry" inherits everything from "Car" and adds its own features (like specific interior design)
- A "Toyota Corolla" also inherits from "Car" with different features

```swift
// Parent Class
class Car {
    var wheels = 4
    var hasEngine = true
    
    func drive() {
        print("The car is driving")
    }
}

// Child Class - inherits everything from Car
class ToyotaCamry: Car {
    var model = "Camry"
    
    func playMusic() {
        print("Playing music on premium speakers")
    }
}

// Usage
let myCar = ToyotaCamry()
myCar.drive()      // Inherited from Car
myCar.playMusic()  // Own method
print(myCar.wheels) // Inherited property: 4
```

---

## What are Protocols? (Simple English)

**Protocols** are like a contract or a promise. Think of it as a job description. When you apply for a job as a "Driver," the job description says:
- You must be able to drive
- You must have a license
- You must know traffic rules

The protocol doesn't tell you HOW to do these things — it just says you MUST be able to do them.

> **Simple Definition:** A protocol is a list of requirements (properties and methods) that any type must fulfill when it "signs the contract."

### Real-Life Analogy

Think of **USB Ports**:
- USB is a "protocol" — it defines the shape and how data transfers
- Any device that follows the USB protocol can plug into any USB port
- Doesn't matter if it's a mouse, keyboard, or flash drive — they all follow the same contract

```swift
// Protocol = Contract/Job Description
protocol Drivable {
    var numberOfWheels: Int { get }
    func drive()
}

// Any type that adopts this protocol MUST implement these requirements
class Car: Drivable {
    var numberOfWheels: Int = 4
    
    func drive() {
        print("Car is driving on road")
    }
}

class Motorcycle: Drivable {
    var numberOfWheels: Int = 2
    
    func drive() {
        print("Motorcycle is zooming fast")
    }
}

// Both can be treated as "Drivable"
let vehicles: [Drivable] = [Car(), Motorcycle()]
for vehicle in vehicles {
    vehicle.drive()
}
```

---

## Real-Life Analogies: Inheritance vs Protocols

| Concept | Inheritance | Protocols |
|---------|-------------|-----------|
| **Analogy** | Family tree | Job contract |
| **Relationship** | "Is a" (A Toyota IS a Car) | "Can do" (A Toyota CAN drive) |
| **Example** | Child inherits parent's DNA | Employee follows company rules |
| **Flexibility** | Rigid (only one parent in Swift) | Flexible (can adopt many protocols) |
| **Real World** | iPhone 15 IS an iPhone | iPhone CAN make calls, CAN take photos |

### The Outlet Example

**Inheritance:** A 3-pin plug inherits from "Plug" but can ONLY work with 3-pin outlets.

**Protocols:** USB-C is a protocol. Any device that implements USB-C can work with any USB-C port — phones, laptops, tablets, all different devices.

---

## Why Swift Encourages Protocol-Oriented Programming

Swift was designed to be **Protocol-Oriented** rather than Object-Oriented. Here's why:

### 1. Single Inheritance Limitation

In Swift, a class can only inherit from ONE parent class. This creates problems:

```swift
// Problem: Diamond Problem
class Bird {
    func fly() { print("Flying") }
}

class Fish {
    func swim() { print("Swimming") }
}

// ERROR: Swift doesn't allow multiple inheritance
// class Duck: Bird, Fish { } // ❌ Not possible
```

With protocols, no problem:

```swift
protocol Flyable {
    func fly()
}

protocol Swimmable {
    func swim()
}

// ✅ A Duck can adopt multiple protocols
class Duck: Flyable, Swimmable {
    func fly() { print("Duck is flying") }
    func swim() { print("Duck is swimming") }
}
```

### 2. Value Types Can't Use Inheritance

Structs and Enums (value types) cannot inherit, but they CAN adopt protocols:

```swift
// Structs can't inherit
struct User {
    var name: String
}

// But structs CAN adopt protocols
protocol Identifiable {
    var id: String { get }
}

struct User: Identifiable {
    var id: String
    var name: String
}
```

### 3. Apple's Standard Library Uses Protocols

Look at Swift's standard library:
- `Equatable` — allows `==` comparison
- `Hashable` — allows use in Sets and Dictionary keys
- `Codable` — allows JSON encoding/decoding
- `Identifiable` — used in SwiftUI lists

---

## Simple iOS Examples

### Example 1: View Controllers (Beginner Mistake vs Better Approach)

**❌ Inheritance Approach (Common Beginner Mistake)**

```swift
// Creating a base view controller with common functionality
class BaseViewController: UIViewController {
    func showLoader() {
        print("Showing loader...")
    }
    
    func hideLoader() {
        print("Hiding loader...")
    }
    
    func showError(_ message: String) {
        print("Error: \(message)")
    }
}

// Every VC inherits from Base
class HomeViewController: BaseViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        showLoader()
    }
}

class ProfileViewController: BaseViewController {
    override func viewDidLoad() {
        super.viewDidLoad()
        showLoader()
    }
}
```

**Problems:**
- What if you need a UITableViewController? You can't inherit from both
- BaseViewController keeps growing with unrelated functionality
- Tight coupling — hard to test

**✅ Protocol Approach (Better)**

```swift
// Define what loading behavior looks like
protocol LoadingShowable {
    func showLoader()
    func hideLoader()
}

// Provide default implementation
extension LoadingShowable where Self: UIViewController {
    func showLoader() {
        // Add loading view to self.view
        print("Showing loader on \(type(of: self))")
    }
    
    func hideLoader() {
        // Remove loading view
        print("Hiding loader")
    }
}

protocol ErrorShowable {
    func showError(_ message: String)
}

extension ErrorShowable where Self: UIViewController {
    func showError(_ message: String) {
        let alert = UIAlertController(title: "Error", message: message, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }
}

// Now ANY view controller can adopt these protocols
class HomeViewController: UIViewController, LoadingShowable, ErrorShowable {
    override func viewDidLoad() {
        super.viewDidLoad()
        showLoader() // Works!
    }
}

// Even UITableViewController can use the same protocols
class SettingsTableViewController: UITableViewController, LoadingShowable {
    override func viewDidLoad() {
        super.viewDidLoad()
        showLoader() // Works!
    }
}
```

---

### Example 2: Network Service (Simple iOS Example)

```swift
// Protocol defines what a network service must do
protocol NetworkServiceProtocol {
    func fetchData(from url: URL, completion: @escaping (Data?, Error?) -> Void)
}

// Real implementation
class NetworkService: NetworkServiceProtocol {
    func fetchData(from url: URL, completion: @escaping (Data?, Error?) -> Void) {
        URLSession.shared.dataTask(with: url) { data, _, error in
            completion(data, error)
        }.resume()
    }
}

// Mock for testing
class MockNetworkService: NetworkServiceProtocol {
    var mockData: Data?
    var mockError: Error?
    
    func fetchData(from url: URL, completion: @escaping (Data?, Error?) -> Void) {
        completion(mockData, mockError)
    }
}

// ViewModel doesn't know which implementation it's using
class UserListViewModel {
    private let networkService: NetworkServiceProtocol
    
    // Dependency injection via protocol
    init(networkService: NetworkServiceProtocol) {
        self.networkService = networkService
    }
    
    func loadUsers() {
        let url = URL(string: "https://api.example.com/users")!
        networkService.fetchData(from: url) { data, error in
            // Handle response
        }
    }
}

// Usage in app
let realViewModel = UserListViewModel(networkService: NetworkService())

// Usage in tests
let mockService = MockNetworkService()
mockService.mockData = "test data".data(using: .utf8)
let testViewModel = UserListViewModel(networkService: mockService)
```

---

### Example 3: Analytics Manager

```swift
// Protocol for analytics
protocol AnalyticsTrackable {
    func trackEvent(name: String, properties: [String: Any])
    func trackScreen(name: String)
}

// Firebase implementation
class FirebaseAnalytics: AnalyticsTrackable {
    func trackEvent(name: String, properties: [String: Any]) {
        // Firebase.Analytics.logEvent(name, parameters: properties)
        print("Firebase: Tracked event \(name)")
    }
    
    func trackScreen(name: String) {
        print("Firebase: Tracked screen \(name)")
    }
}

// Mixpanel implementation
class MixpanelAnalytics: AnalyticsTrackable {
    func trackEvent(name: String, properties: [String: Any]) {
        // Mixpanel.track(name, properties: properties)
        print("Mixpanel: Tracked event \(name)")
    }
    
    func trackScreen(name: String) {
        print("Mixpanel: Tracked screen \(name)")
    }
}

// App can switch analytics providers without changing code
class AnalyticsManager {
    static let shared = AnalyticsManager()
    
    var provider: AnalyticsTrackable = FirebaseAnalytics()
    
    func trackEvent(name: String, properties: [String: Any] = [:]) {
        provider.trackEvent(name: name, properties: properties)
    }
    
    func trackScreen(name: String) {
        provider.trackScreen(name: name)
    }
}

// Usage
AnalyticsManager.shared.trackEvent(name: "button_tapped", properties: ["button_id": "checkout"])
```

---

# Intermediate Level (SDE-2 Ready)

## Protocols vs Inheritance: Decision Framework

Use this decision tree when designing iOS architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    WHEN TO USE WHAT?                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Is it truly an "IS-A" relationship?                        │
│  (e.g., ToyotaCamry IS-A Car)                               │
│         │                                                    │
│         ├── YES ──> Does child need to override most        │
│         │           parent behavior?                         │
│         │           ├── YES ──> Consider PROTOCOL ✅         │
│         │           └── NO ───> INHERITANCE might work ⚠️    │
│         │                                                    │
│         └── NO ───> Use PROTOCOL ✅                          │
│                                                              │
│  Do you need multiple behaviors?                            │
│  (e.g., Flyable AND Swimmable)                              │
│         │                                                    │
│         └── YES ──> Use PROTOCOLS ✅                         │
│                                                              │
│  Are you working with structs?                              │
│         │                                                    │
│         └── YES ──> Must use PROTOCOLS ✅                    │
│                                                              │
│  Need to test easily with mocks?                            │
│         │                                                    │
│         └── YES ──> Use PROTOCOLS ✅                         │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### When Inheritance Makes Sense

1. **UIKit View Controllers** — UIViewController → UITableViewController hierarchy
2. **True hierarchical relationships** — where child truly extends parent
3. **When you need stored properties from parent**

```swift
// Good use of inheritance: Custom base cell
class BaseCardCell: UITableViewCell {
    let cardView: UIView = {
        let view = UIView()
        view.layer.cornerRadius = 12
        view.layer.shadowOpacity = 0.1
        return view
    }()
    
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupCardView()
    }
    
    private func setupCardView() {
        contentView.addSubview(cardView)
        // Constraints...
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
}

class ProductCell: BaseCardCell {
    // Inherits cardView setup, adds product-specific content
    let productImageView = UIImageView()
    let titleLabel = UILabel()
}
```

### When Protocols Are Better

1. **Defining capabilities** (Cacheable, Trackable, Persistable)
2. **Dependency injection** for testing
3. **Cross-cutting concerns** (logging, error handling)
4. **Struct-based models**

---

## Protocol Extensions and Default Implementations

Protocol extensions allow you to provide default behavior, so conforming types don't have to implement everything.

### Basic Example

```swift
protocol Describable {
    var name: String { get }
    func describe() -> String
}

// Default implementation
extension Describable {
    func describe() -> String {
        return "This is \(name)"
    }
}

struct Product: Describable {
    var name: String
    // describe() is automatically available with default implementation
}

struct User: Describable {
    var name: String
    
    // Can override default implementation
    func describe() -> String {
        return "User: \(name)"
    }
}

let product = Product(name: "iPhone")
print(product.describe()) // "This is iPhone" (default)

let user = User(name: "John")
print(user.describe()) // "User: John" (custom)
```

### Real iOS Example: Reusable Cells

```swift
protocol ReusableCell {
    static var reuseIdentifier: String { get }
}

// Default implementation using class name
extension ReusableCell {
    static var reuseIdentifier: String {
        return String(describing: self)
    }
}

// Any cell can adopt this
class UserCell: UITableViewCell, ReusableCell {}
class ProductCell: UITableViewCell, ReusableCell {}

// Usage
tableView.register(UserCell.self, forCellReuseIdentifier: UserCell.reuseIdentifier)
let cell = tableView.dequeueReusableCell(withIdentifier: UserCell.reuseIdentifier, for: indexPath)
```

### Real iOS Example: Nib Loadable Views

```swift
protocol NibLoadable: AnyObject {
    static var nibName: String { get }
}

extension NibLoadable {
    static var nibName: String {
        return String(describing: self)
    }
}

extension NibLoadable where Self: UIView {
    static func loadFromNib() -> Self {
        let bundle = Bundle(for: self)
        let nib = UINib(nibName: nibName, bundle: bundle)
        return nib.instantiate(withOwner: nil, options: nil).first as! Self
    }
}

// Usage
class CustomCardView: UIView, NibLoadable {}

let cardView = CustomCardView.loadFromNib()
```

---

## Using Protocols to Reduce Tight Coupling

**Tight coupling** means one component directly depends on another's concrete implementation. This makes code hard to test and modify.

### ❌ Tight Coupling Example

```swift
class UserRepository {
    private let coreDataManager = CoreDataManager() // Direct dependency!
    
    func saveUser(_ user: User) {
        coreDataManager.save(user) // Can't swap storage
    }
    
    func fetchUsers() -> [User] {
        return coreDataManager.fetchAll()
    }
}

// Problem: Can't test without CoreData, can't switch to Realm or UserDefaults
```

### ✅ Loose Coupling with Protocols

```swift
// Define the contract
protocol DataStorageProtocol {
    associatedtype Entity
    func save(_ entity: Entity)
    func fetchAll() -> [Entity]
    func delete(_ entity: Entity)
}

// CoreData implementation
class CoreDataStorage<T>: DataStorageProtocol {
    typealias Entity = T
    
    func save(_ entity: T) {
        print("Saving to CoreData")
    }
    
    func fetchAll() -> [T] {
        print("Fetching from CoreData")
        return []
    }
    
    func delete(_ entity: T) {
        print("Deleting from CoreData")
    }
}

// UserDefaults implementation
class UserDefaultsStorage<T: Codable>: DataStorageProtocol {
    typealias Entity = T
    private let key: String
    
    init(key: String) {
        self.key = key
    }
    
    func save(_ entity: T) {
        if let data = try? JSONEncoder().encode(entity) {
            UserDefaults.standard.set(data, forKey: key)
        }
    }
    
    func fetchAll() -> [T] {
        guard let data = UserDefaults.standard.data(forKey: key),
              let entities = try? JSONDecoder().decode([T].self, from: data) else {
            return []
        }
        return entities
    }
    
    func delete(_ entity: T) {
        // Implementation
    }
}

// Repository depends on protocol, not concrete class
class UserRepository<Storage: DataStorageProtocol> where Storage.Entity == User {
    private let storage: Storage
    
    init(storage: Storage) {
        self.storage = storage
    }
    
    func saveUser(_ user: User) {
        storage.save(user)
    }
    
    func fetchUsers() -> [User] {
        return storage.fetchAll()
    }
}
```

---

## Associated Types: What Problem They Solve

**Associated types** make protocols generic. They're like a placeholder for a type that will be defined later.

### The Problem Without Associated Types

```swift
// Without associated types, you'd need separate protocols for each type
protocol IntContainer {
    func add(_ item: Int)
    func getAll() -> [Int]
}

protocol StringContainer {
    func add(_ item: String)
    func getAll() -> [String]
}

// This doesn't scale!
```

### Solution: Associated Types

```swift
protocol Container {
    associatedtype Item // Placeholder type
    
    func add(_ item: Item)
    func getAll() -> [Item]
    var count: Int { get }
}

// Concrete type specifies what Item is
class IntBox: Container {
    typealias Item = Int
    private var items: [Int] = []
    
    func add(_ item: Int) {
        items.append(item)
    }
    
    func getAll() -> [Int] {
        return items
    }
    
    var count: Int { items.count }
}

class StringBox: Container {
    typealias Item = String
    private var items: [String] = []
    
    func add(_ item: String) {
        items.append(item)
    }
    
    func getAll() -> [String] {
        return items
    }
    
    var count: Int { items.count }
}
```

### Real iOS Example: Repository Pattern

```swift
protocol Repository {
    associatedtype Entity
    associatedtype Identifier
    
    func fetch(by id: Identifier) -> Entity?
    func fetchAll() -> [Entity]
    func save(_ entity: Entity)
    func delete(by id: Identifier)
}

struct User: Identifiable {
    let id: UUID
    var name: String
    var email: String
}

class UserRepository: Repository {
    typealias Entity = User
    typealias Identifier = UUID
    
    private var users: [UUID: User] = [:]
    
    func fetch(by id: UUID) -> User? {
        return users[id]
    }
    
    func fetchAll() -> [User] {
        return Array(users.values)
    }
    
    func save(_ entity: User) {
        users[entity.id] = entity
    }
    
    func delete(by id: UUID) {
        users.removeValue(forKey: id)
    }
}
```

---

## Common Mistakes iOS Engineers Make with Protocols

### Mistake 1: Creating "God Protocols"

```swift
// ❌ Bad: Too many requirements
protocol NetworkManagerProtocol {
    func fetchUsers() -> [User]
    func fetchProducts() -> [Product]
    func fetchOrders() -> [Order]
    func login(email: String, password: String)
    func logout()
    func uploadImage(_ image: UIImage)
    func downloadFile(url: URL)
    // ... 20 more methods
}

// ✅ Good: Split into focused protocols
protocol UserFetching {
    func fetchUsers() -> [User]
}

protocol AuthenticationService {
    func login(email: String, password: String)
    func logout()
}

protocol FileService {
    func uploadImage(_ image: UIImage)
    func downloadFile(url: URL)
}
```

### Mistake 2: Not Using Protocol Extensions

```swift
// ❌ Bad: Every conforming type must implement this
protocol Trackable {
    func trackScreenView(screenName: String, properties: [String: Any])
}

// ✅ Good: Provide defaults
protocol Trackable {
    func trackScreenView(screenName: String, properties: [String: Any])
}

extension Trackable {
    func trackScreenView(screenName: String, properties: [String: Any] = [:]) {
        // Default implementation
        AnalyticsManager.shared.track(screenName, properties: properties)
    }
}
```

### Mistake 3: Forgetting Protocol Can't Store Properties

```swift
// ❌ This won't compile
protocol CacheableProtocol {
    var cache: [String: Any] = [:] // ERROR: Protocols can't have stored properties
}

// ✅ Use computed properties or methods
protocol CacheableProtocol {
    var cache: [String: Any] { get set }
}

// Or use protocol extension with object association
extension CacheableProtocol where Self: AnyObject {
    // Use associated objects or other techniques
}
```

### Mistake 4: Overusing Protocols

```swift
// ❌ Bad: Protocol for something that will never have multiple implementations
protocol UserDefaultsManagerProtocol {
    func save(key: String, value: Any)
    func get(key: String) -> Any?
}

class UserDefaultsManager: UserDefaultsManagerProtocol {
    // Only implementation ever
}

// ✅ Good: Just use the concrete class if there's only one implementation
// Only use protocols when you need abstraction (testing, multiple implementations)
```

---

# Advanced Level (Big Tech Interviews)

## Type Erasure in Swift: Why It Exists and How to Implement It

### The Problem

Protocols with associated types cannot be used as types directly:

```swift
protocol Fetchable {
    associatedtype Model
    func fetch() -> Model
}

class UserFetcher: Fetchable {
    typealias Model = User
    func fetch() -> User {
        return User(id: UUID(), name: "John", email: "john@example.com")
    }
}

class ProductFetcher: Fetchable {
    typealias Model = Product
    func fetch() -> Product {
        return Product(id: UUID(), name: "iPhone", price: 999)
    }
}

// ❌ This won't compile!
// Error: Protocol 'Fetchable' can only be used as a generic constraint
var fetchers: [Fetchable] = [] // ❌ ERROR
```

### Why Does This Happen?

Swift needs to know exact types at compile time. When you have an array of `Fetchable`, Swift doesn't know what `Model` type each element will return.

### Solution: Type Erasure

Type erasure wraps the protocol in a concrete type that hides the associated type.

```swift
// Step 1: Define the protocol
protocol Fetchable {
    associatedtype Model
    func fetch() -> Model
}

// Step 2: Create a type-erased wrapper
struct AnyFetchable<Model>: Fetchable {
    private let _fetch: () -> Model
    
    init<F: Fetchable>(_ fetcher: F) where F.Model == Model {
        self._fetch = fetcher.fetch
    }
    
    func fetch() -> Model {
        return _fetch()
    }
}

// Step 3: Usage
class UserFetcher: Fetchable {
    typealias Model = User
    func fetch() -> User {
        return User(id: UUID(), name: "John", email: "john@example.com")
    }
}

class AnotherUserFetcher: Fetchable {
    typealias Model = User
    func fetch() -> User {
        return User(id: UUID(), name: "Jane", email: "jane@example.com")
    }
}

// ✅ Now we can have an array of user fetchers
var userFetchers: [AnyFetchable<User>] = [
    AnyFetchable(UserFetcher()),
    AnyFetchable(AnotherUserFetcher())
]

for fetcher in userFetchers {
    let user = fetcher.fetch()
    print(user.name)
}
```

### Real iOS Example: Type-Erased Repository

```swift
protocol Repository {
    associatedtype Entity
    associatedtype ID
    
    func fetch(by id: ID) async throws -> Entity?
    func save(_ entity: Entity) async throws
}

// Type-erased wrapper
final class AnyRepository<Entity, ID>: Repository {
    private let _fetch: (ID) async throws -> Entity?
    private let _save: (Entity) async throws -> Void
    
    init<R: Repository>(_ repository: R) where R.Entity == Entity, R.ID == ID {
        self._fetch = repository.fetch
        self._save = repository.save
    }
    
    func fetch(by id: ID) async throws -> Entity? {
        try await _fetch(id)
    }
    
    func save(_ entity: Entity) async throws {
        try await _save(entity)
    }
}

// Usage
class UserRepository: Repository {
    typealias Entity = User
    typealias ID = UUID
    
    func fetch(by id: UUID) async throws -> User? {
        // Fetch from database
        return nil
    }
    
    func save(_ entity: User) async throws {
        // Save to database
    }
}

// Can now store in a property or pass around
let anyUserRepo: AnyRepository<User, UUID> = AnyRepository(UserRepository())
```

### Swift 5.7+ Simplification: `any` and `some`

```swift
// Modern Swift allows this with 'any'
var fetchers: [any Fetchable] = [UserFetcher(), ProductFetcher()]

// 'some' for opaque return types
func makeFetcher() -> some Fetchable {
    return UserFetcher()
}
```

---

## Protocols with Associated Types vs Generics (Trade-offs)

| Aspect | Protocol with Associated Types (PAT) | Generics |
|--------|--------------------------------------|----------|
| **Definition** | Protocol defines placeholder type | Function/class defines placeholder type |
| **Type determined by** | Conforming type decides | Caller decides |
| **Flexibility** | Less flexible, one type per conformance | More flexible, caller chooses type |
| **Use as type** | Cannot use as standalone type (needs type erasure) | Can use directly |
| **Best for** | Defining contracts with related types | Writing reusable algorithms |

### Example Comparison

```swift
// Protocol with Associated Type
protocol Cache {
    associatedtype Value
    func get(key: String) -> Value?
    func set(key: String, value: Value)
}

class UserCache: Cache {
    typealias Value = User // Type is fixed by conforming type
    private var storage: [String: User] = [:]
    
    func get(key: String) -> User? { storage[key] }
    func set(key: String, value: User) { storage[key] = value }
}

// Generic Class
class GenericCache<Value> {
    private var storage: [String: Value] = [:]
    
    func get(key: String) -> Value? { storage[key] }
    func set(key: String, value: Value) { storage[key] = value }
}

// Caller decides the type
let userCache = GenericCache<User>()
let productCache = GenericCache<Product>()
```

### When to Use Each

**Use Protocol with Associated Types when:**
- Defining a contract that types conform to
- The conforming type determines the associated type
- You want type safety at the protocol level

**Use Generics when:**
- Writing reusable algorithms or data structures
- The caller should decide the type
- You need more flexibility

---

## Dependency Injection Using Protocols

### Types of Dependency Injection

#### 1. Constructor (Initializer) Injection

The most common and recommended approach.

```swift
protocol UserServiceProtocol {
    func fetchUser(id: String) async throws -> User
}

protocol AnalyticsProtocol {
    func track(event: String)
}

class UserProfileViewModel {
    private let userService: UserServiceProtocol
    private let analytics: AnalyticsProtocol
    
    // Dependencies injected through initializer
    init(userService: UserServiceProtocol, analytics: AnalyticsProtocol) {
        self.userService = userService
        self.analytics = analytics
    }
    
    func loadProfile(userId: String) async {
        analytics.track(event: "profile_view_started")
        do {
            let user = try await userService.fetchUser(id: userId)
            // Update UI
            analytics.track(event: "profile_loaded")
        } catch {
            analytics.track(event: "profile_load_failed")
        }
    }
}

// Production
let viewModel = UserProfileViewModel(
    userService: RealUserService(),
    analytics: FirebaseAnalytics()
)

// Testing
let testViewModel = UserProfileViewModel(
    userService: MockUserService(),
    analytics: MockAnalytics()
)
```

#### 2. Property Injection

Useful when dependency can be changed after initialization.

```swift
class ImageDownloader {
    // Property injection - can be swapped
    var networkSession: URLSessionProtocol = URLSession.shared
    
    func downloadImage(from url: URL) async throws -> UIImage {
        let (data, _) = try await networkSession.data(from: url)
        guard let image = UIImage(data: data) else {
            throw ImageError.invalidData
        }
        return image
    }
}

// In tests
let downloader = ImageDownloader()
downloader.networkSession = MockURLSession()
```

#### 3. Method Injection

Pass dependency only when needed.

```swift
protocol Logger {
    func log(_ message: String)
}

class PaymentProcessor {
    func processPayment(amount: Double, logger: Logger) {
        logger.log("Starting payment of \(amount)")
        // Process payment
        logger.log("Payment completed")
    }
}

// Different loggers for different scenarios
let processor = PaymentProcessor()
processor.processPayment(amount: 99.99, logger: ConsoleLogger())
processor.processPayment(amount: 99.99, logger: FileLogger())
```

### Real-World DI Container Example

```swift
protocol DIContainer {
    func resolve<T>(_ type: T.Type) -> T
    func register<T>(_ type: T.Type, factory: @escaping () -> T)
}

class AppDIContainer: DIContainer {
    static let shared = AppDIContainer()
    
    private var factories: [String: () -> Any] = [:]
    
    func register<T>(_ type: T.Type, factory: @escaping () -> T) {
        let key = String(describing: type)
        factories[key] = factory
    }
    
    func resolve<T>(_ type: T.Type) -> T {
        let key = String(describing: type)
        guard let factory = factories[key] else {
            fatalError("No registered factory for \(key)")
        }
        return factory() as! T
    }
}

// Registration
extension AppDIContainer {
    func registerDependencies() {
        register(UserServiceProtocol.self) { RealUserService() }
        register(AnalyticsProtocol.self) { FirebaseAnalytics() }
        register(CacheProtocol.self) { NSCacheWrapper() }
    }
}

// Usage
let userService: UserServiceProtocol = AppDIContainer.shared.resolve(UserServiceProtocol.self)
```

---

## Designing Testable, Scalable Architectures Using Protocols

### The SOLID Principles Applied to iOS

#### 1. Single Responsibility via Protocol Segregation

```swift
// ❌ Bad: One protocol doing too much
protocol UserManager {
    func login(email: String, password: String)
    func logout()
    func fetchProfile() -> User
    func updateProfile(_ user: User)
    func uploadAvatar(_ image: UIImage)
    func sendPasswordReset(email: String)
}

// ✅ Good: Segregated protocols
protocol AuthenticationService {
    func login(email: String, password: String) async throws -> AuthToken
    func logout() async throws
}

protocol ProfileService {
    func fetchProfile() async throws -> User
    func updateProfile(_ user: User) async throws
}

protocol AvatarService {
    func uploadAvatar(_ image: UIImage) async throws -> URL
}

protocol PasswordResetService {
    func sendPasswordReset(email: String) async throws
}
```

#### 2. Dependency Inversion for Testability

```swift
// High-level module
class CheckoutViewModel {
    private let paymentService: PaymentServiceProtocol
    private let inventoryService: InventoryServiceProtocol
    private let analyticsService: AnalyticsServiceProtocol
    
    init(
        paymentService: PaymentServiceProtocol,
        inventoryService: InventoryServiceProtocol,
        analyticsService: AnalyticsServiceProtocol
    ) {
        self.paymentService = paymentService
        self.inventoryService = inventoryService
        self.analyticsService = analyticsService
    }
    
    func checkout(cart: Cart) async throws {
        analyticsService.track("checkout_started")
        
        // Verify inventory
        for item in cart.items {
            let available = try await inventoryService.checkAvailability(item.productId)
            guard available else {
                throw CheckoutError.itemUnavailable(item)
            }
        }
        
        // Process payment
        try await paymentService.process(amount: cart.total)
        
        analyticsService.track("checkout_completed")
    }
}

// Testing is now easy
class CheckoutViewModelTests: XCTestCase {
    func testCheckoutSuccess() async throws {
        // Arrange
        let mockPayment = MockPaymentService()
        let mockInventory = MockInventoryService()
        mockInventory.availabilityResult = true
        let mockAnalytics = MockAnalyticsService()
        
        let viewModel = CheckoutViewModel(
            paymentService: mockPayment,
            inventoryService: mockInventory,
            analyticsService: mockAnalytics
        )
        
        let cart = Cart(items: [CartItem(productId: "123", quantity: 1)])
        
        // Act
        try await viewModel.checkout(cart: cart)
        
        // Assert
        XCTAssertTrue(mockPayment.processWasCalled)
        XCTAssertEqual(mockAnalytics.trackedEvents, ["checkout_started", "checkout_completed"])
    }
}
```

### Scalable Architecture Pattern: Clean Architecture with Protocols

```
┌─────────────────────────────────────────────────────────────────┐
│                         Presentation Layer                       │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  ViewModels     │  │  Views          │  │  Coordinators   │ │
│  │  (Observable)   │  │  (SwiftUI/UIKit)│  │  (Navigation)   │ │
│  └────────┬────────┘  └─────────────────┘  └─────────────────┘ │
│           │                                                      │
│           ▼ Uses protocol                                        │
├─────────────────────────────────────────────────────────────────┤
│                         Domain Layer                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Use Cases      │  │  Domain Models  │  │  Repository     │ │
│  │  (Interactors)  │  │  (Entities)     │  │  Protocols      │ │
│  └────────┬────────┘  └─────────────────┘  └─────────────────┘ │
│           │                                                      │
│           ▼ Uses protocol                                        │
├─────────────────────────────────────────────────────────────────┤
│                         Data Layer                               │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐ │
│  │  Repository     │  │  Network        │  │  Local Storage  │ │
│  │  Implementations│  │  Services       │  │  Services       │ │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

```swift
// Domain Layer - Protocol definition
protocol UserRepositoryProtocol {
    func fetchUser(id: String) async throws -> User
    func saveUser(_ user: User) async throws
    func deleteUser(id: String) async throws
}

// Data Layer - Implementation
class UserRepository: UserRepositoryProtocol {
    private let networkService: NetworkServiceProtocol
    private let cacheService: CacheServiceProtocol
    
    init(networkService: NetworkServiceProtocol, cacheService: CacheServiceProtocol) {
        self.networkService = networkService
        self.cacheService = cacheService
    }
    
    func fetchUser(id: String) async throws -> User {
        // Try cache first
        if let cachedUser: User = cacheService.get(key: "user_\(id)") {
            return cachedUser
        }
        
        // Fetch from network
        let user: User = try await networkService.request(endpoint: .user(id: id))
        
        // Cache the result
        cacheService.set(key: "user_\(id)", value: user)
        
        return user
    }
    
    func saveUser(_ user: User) async throws {
        try await networkService.request(endpoint: .updateUser(user))
        cacheService.set(key: "user_\(user.id)", value: user)
    }
    
    func deleteUser(id: String) async throws {
        try await networkService.request(endpoint: .deleteUser(id: id))
        cacheService.remove(key: "user_\(id)")
    }
}

// Domain Layer - Use Case
class FetchUserProfileUseCase {
    private let userRepository: UserRepositoryProtocol
    
    init(userRepository: UserRepositoryProtocol) {
        self.userRepository = userRepository
    }
    
    func execute(userId: String) async throws -> UserProfile {
        let user = try await userRepository.fetchUser(id: userId)
        return UserProfile(user: user)
    }
}

// Presentation Layer - ViewModel
@MainActor
class UserProfileViewModel: ObservableObject {
    @Published var profile: UserProfile?
    @Published var isLoading = false
    @Published var error: Error?
    
    private let fetchProfileUseCase: FetchUserProfileUseCase
    
    init(fetchProfileUseCase: FetchUserProfileUseCase) {
        self.fetchProfileUseCase = fetchProfileUseCase
    }
    
    func loadProfile(userId: String) {
        isLoading = true
        error = nil
        
        Task {
            do {
                profile = try await fetchProfileUseCase.execute(userId: userId)
            } catch {
                self.error = error
            }
            isLoading = false
        }
    }
}
```

---

## Performance, Memory, and Compile-Time Trade-offs

### 1. Static vs Dynamic Dispatch

```swift
// Static dispatch (faster) - using generics
func processStatic<T: Processable>(_ item: T) {
    item.process() // Compiler knows exact type at compile time
}

// Dynamic dispatch (slower) - using protocol type
func processDynamic(_ item: Processable) {
    item.process() // Runtime lookup via witness table
}

// Protocol extension methods are statically dispatched when called on concrete type
protocol Printable {
    func printDescription()
}

extension Printable {
    func printDescription() {
        print("Default description")
    }
}

struct Item: Printable {
    func printDescription() {
        print("Item description")
    }
}

let item = Item()
item.printDescription() // "Item description" - static dispatch

let printable: Printable = Item()
printable.printDescription() // "Item description" - dynamic dispatch
```

### 2. Protocol Witness Tables

When you use a protocol type (not generic constraint), Swift creates a witness table for method lookup:

```swift
// Using protocol type creates overhead
var services: [ServiceProtocol] = [] // Each element has witness table

// Using generics avoids witness tables
func configure<S: ServiceProtocol>(_ service: S) {
    // No witness table, type is known at compile time
}
```

### 3. Memory Considerations

```swift
// Existential containers for protocol types
protocol DataProtocol {
    var data: Data { get }
}

// Protocol types use existential containers (typically 40 bytes on 64-bit)
// This includes:
// - 24 bytes for value buffer (or heap pointer if larger)
// - 8 bytes for metadata
// - 8 bytes for protocol witness table

// For performance-critical code, prefer generics
func processData<D: DataProtocol>(_ data: D) {
    // No existential container overhead
}
```

### 4. Compile-Time Trade-offs

| Approach | Compile Time | Binary Size | Runtime Performance |
|----------|--------------|-------------|---------------------|
| Protocol types (`any P`) | Faster | Smaller | Slower (dynamic dispatch) |
| Generics (`<T: P>`) | Slower | Larger (specialization) | Faster (static dispatch) |
| `@inlinable` generics | Slower | Larger | Fastest |

```swift
// Mark critical path methods as inlinable for best performance
protocol FastOperation {
    @inlinable
    func performFast()
}
```

---

## How Big Tech iOS Apps Use Protocols

### Uber's Approach: RIBs Architecture

Uber's RIBs (Router, Interactor, Builder) architecture heavily uses protocols:

```swift
// Every component defines its dependencies via protocol
protocol HomeDependency: Dependency {
    var userService: UserServiceProtocol { get }
    var rideService: RideServiceProtocol { get }
    var analyticsService: AnalyticsServiceProtocol { get }
}

// Interactor communicates with Router via protocol
protocol HomeRouting: ViewableRouting {
    func routeToRideRequest()
    func routeToPayment()
    func dismissCurrentChild()
}

// Builder creates the RIB with dependencies
protocol HomeBuildable: Buildable {
    func build(withListener listener: HomeListener) -> HomeRouting
}

final class HomeBuilder: Builder<HomeDependency>, HomeBuildable {
    func build(withListener listener: HomeListener) -> HomeRouting {
        let component = HomeComponent(dependency: dependency)
        let viewController = HomeViewController()
        let interactor = HomeInteractor(presenter: viewController, 
                                         userService: component.userService,
                                         rideService: component.rideService)
        interactor.listener = listener
        
        let router = HomeRouter(interactor: interactor, 
                                viewController: viewController,
                                rideRequestBuilder: component.rideRequestBuilder)
        return router
    }
}
```

### Amazon's Approach: Plugin Architecture

Amazon uses protocols for plugin-based feature development:

```swift
// Feature plugin protocol
protocol FeaturePlugin {
    var featureId: String { get }
    var isEnabled: Bool { get }
    
    func register(with container: ServiceContainer)
    func onAppLaunch()
    func onUserLogin(userId: String)
}

// Each team implements their own plugin
class SearchPlugin: FeaturePlugin {
    let featureId = "search"
    
    var isEnabled: Bool {
        FeatureFlags.isSearchEnabled
    }
    
    func register(with container: ServiceContainer) {
        container.register(SearchServiceProtocol.self) { SearchService() }
        container.register(SearchViewModelFactory.self) { SearchViewModelFactory(container: container) }
    }
    
    func onAppLaunch() {
        // Pre-warm search index
    }
    
    func onUserLogin(userId: String) {
        // Sync search preferences
    }
}

// App bootstrapper
class AppBootstrapper {
    private var plugins: [FeaturePlugin] = []
    
    func registerPlugins() {
        plugins = [
            SearchPlugin(),
            CartPlugin(),
            PaymentPlugin(),
            RecommendationsPlugin()
        ]
        
        let container = ServiceContainer.shared
        for plugin in plugins where plugin.isEnabled {
            plugin.register(with: container)
        }
    }
}
```

### Google's Approach: Modular Services

Google emphasizes protocol-based service discovery:

```swift
// Service protocol with versioning
protocol ServiceDescriptor {
    static var serviceName: String { get }
    static var version: Int { get }
}

protocol UserDataService: ServiceDescriptor {
    func fetchUserData(userId: String) async throws -> UserData
    func updateUserData(_ data: UserData) async throws
}

extension UserDataService {
    static var serviceName: String { "UserDataService" }
    static var version: Int { 2 }
}

// Service locator pattern
protocol ServiceLocator {
    func getService<T>(_ type: T.Type) -> T?
    func registerService<T>(_ type: T.Type, instance: T)
}

class DefaultServiceLocator: ServiceLocator {
    static let shared = DefaultServiceLocator()
    private var services: [String: Any] = [:]
    
    func registerService<T>(_ type: T.Type, instance: T) {
        let key = String(describing: type)
        services[key] = instance
    }
    
    func getService<T>(_ type: T.Type) -> T? {
        let key = String(describing: type)
        return services[key] as? T
    }
}
```

---

# Interview Preparation Section

## Real Interview Questions and Answers

### Question 1: "Explain the difference between protocols and inheritance in Swift. When would you use each?"

**Structured Answer:**

> "Protocols and inheritance serve different purposes in Swift.
>
> **Inheritance** establishes an 'is-a' relationship where a subclass inherits all properties and methods from its parent class. It's useful when you have a true hierarchical relationship, like UITableViewController being a specialized UIViewController.
>
> **Protocols** establish a 'can-do' relationship, defining a contract of capabilities. Any type can conform to multiple protocols, including structs.
>
> In practice, I prefer protocols for most iOS architecture because:
> 1. Swift only allows single inheritance, but multiple protocol conformance
> 2. Protocols work with value types (structs), which inheritance doesn't
> 3. Protocols enable better testability through dependency injection
> 4. Protocol extensions provide default implementations without tight coupling
>
> I use inheritance primarily with UIKit classes where Apple's framework requires it, or when I need shared stored properties across a hierarchy."

**Follow-up:** "Can you give a specific example from your experience?"

> "In our networking layer, we defined a `NetworkServiceProtocol` that specified methods like `request<T: Decodable>(endpoint:) async throws -> T`. This allowed us to have `ProductionNetworkService` and `MockNetworkService` implementations. When writing tests, we could inject the mock and control exactly what responses it returned, making our tests deterministic and fast."

---

### Question 2: "What are associated types in protocols? Why would you use them?"

**Structured Answer:**

> "Associated types make protocols generic. They're placeholders for types that conforming types will specify.
>
> For example, the `Collection` protocol in Swift has an associated type `Element`. When `Array<Int>` conforms to `Collection`, its `Element` is `Int`.
>
> I use associated types when:
> 1. A protocol needs to work with different types but in a type-safe way
> 2. The conforming type should determine what type to use
> 3. Multiple methods in the protocol need to use the same related type
>
> A real example is a `Repository` protocol:
>
> ```swift
> protocol Repository {
>     associatedtype Entity
>     func fetch(id: String) -> Entity?
>     func save(_ entity: Entity)
> }
> ```
>
> A `UserRepository` would set `Entity` to `User`, ensuring type safety between fetch and save operations."

**Follow-up:** "What's the limitation of associated types?"

> "You can't use protocols with associated types as standalone types—you can't write `var repos: [Repository]`. You need type erasure (like `AnyRepository`), generics, or the newer `any Repository` syntax. This is because Swift needs to know the concrete type at compile time for type safety."

---

### Question 3: "Explain type erasure and when you'd use it."

**Structured Answer:**

> "Type erasure is a pattern that hides a protocol's associated type behind a concrete wrapper type.
>
> The problem: Protocols with associated types can't be used as types directly. You can't create an array of `Fetchable` because Swift doesn't know what `Model` type each element returns.
>
> The solution: Create a wrapper struct like `AnyFetchable<Model>` that:
> 1. Stores closures that forward to the wrapped instance's methods
> 2. Exposes the associated type as a generic parameter
>
> I've used this in apps where we had multiple analytics providers. We created `AnyAnalyticsProvider` that wrapped different implementations, allowing us to store them in an array and iterate through them.
>
> In Swift 5.7+, the `any` keyword simplifies this, but type erasure is still valuable for performance-critical paths and pre-iOS 16 support."

---

### Question 4: "How would you design a testable networking layer using protocols?"

**Structured Answer:**

> "I'd design it with three key protocols:
>
> 1. **HTTPClient protocol** — handles raw HTTP requests
> 2. **Endpoints as types** — encapsulate request details
> 3. **Service layer** — domain-specific operations
>
> ```swift
> protocol HTTPClientProtocol {
>     func execute<T: Decodable>(_ request: URLRequest) async throws -> T
> }
> 
> protocol UserServiceProtocol {
>     func fetchUser(id: String) async throws -> User
>     func updateUser(_ user: User) async throws
> }
> ```
>
> The `UserServiceProtocol` depends on `HTTPClientProtocol` through constructor injection:
>
> ```swift
> class UserService: UserServiceProtocol {
>     private let httpClient: HTTPClientProtocol
>     
>     init(httpClient: HTTPClientProtocol) {
>         self.httpClient = httpClient
>     }
> }
> ```
>
> For tests, I create `MockHTTPClient` that returns predefined responses, making tests fast and deterministic. No network calls needed."

---

### Question 5: "What's the performance difference between using protocol types vs generics?"

**Structured Answer:**

> "There are measurable differences:
>
> **Protocol types (existentials)** use dynamic dispatch through witness tables. Each protocol type instance requires extra memory for the existential container—typically 40 bytes on 64-bit systems. Method calls go through an indirection.
>
> **Generics** use static dispatch when possible. The compiler generates specialized code for each concrete type used, resulting in direct function calls.
>
> In benchmarks, generic code can be 2-5x faster for tight loops.
>
> When to care:
> - Hot paths (called thousands of times)
> - Large collections of protocol types
> - Embedded/performance-critical apps
>
> When not to worry:
> - Dependency injection and architecture boundaries
> - UI code (user interactions are the bottleneck)
> - Most app-level code
>
> In my experience, I use protocols for architecture and dependency injection, and switch to generics if profiling shows a specific bottleneck."

---

### Question 6: "Design a feature flag system using protocols."

**Structured Answer:**

> "I'd design it with the following components:
>
> ```swift
> // Feature flag provider protocol
> protocol FeatureFlagProvider {
>     func isEnabled(_ flag: FeatureFlag) -> Bool
>     func value<T>(for flag: FeatureFlag, default: T) -> T
>     func refresh() async
> }
> 
> // Can have multiple implementations
> class RemoteFeatureFlagProvider: FeatureFlagProvider {
>     // Fetches from Firebase Remote Config, LaunchDarkly, etc.
> }
> 
> class LocalFeatureFlagProvider: FeatureFlagProvider {
>     // For development and testing
> }
> 
> class CachedFeatureFlagProvider: FeatureFlagProvider {
>     private let remote: FeatureFlagProvider
>     private let cache: FeatureFlagCache
>     
>     // Decorator pattern - checks cache first
> }
> ```
>
> Benefits:
> 1. Can swap implementations (remote → local for testing)
> 2. Can compose providers (cached + remote)
> 3. ViewModels take `FeatureFlagProvider` as dependency, enabling testing
> 4. A/B testing becomes easy to mock
>
> In tests:
> ```swift
> let mockFlags = MockFeatureFlagProvider()
> mockFlags.setEnabled(.newCheckout, true)
> let viewModel = CheckoutViewModel(featureFlags: mockFlags)
> ```"

---

## How to Explain Protocol-Oriented Design Decisions Out Loud

### Framework for Answering

1. **State the problem** — What challenge are you solving?
2. **Explain the approach** — Why protocols/generics/inheritance?
3. **Show the benefit** — Testing, flexibility, scalability
4. **Acknowledge trade-offs** — Nothing is perfect

### Example Explanation Script

> "We needed to build an analytics layer that could send events to multiple providers—Firebase, Amplitude, and our custom backend.
>
> I chose protocols because we needed a common interface but different implementations. Each provider handles events differently internally, but from the app's perspective, they all just 'track events.'
>
> The benefit was that we could add or remove providers without changing app code. We also created a mock provider for tests, ensuring our test suite doesn't make actual network calls.
>
> The trade-off is added complexity—we have more files and protocols to maintain. But for a cross-cutting concern like analytics that's used everywhere, the flexibility was worth it."

---

## Common Follow-up Questions and Responses

### "Why not just use classes with a common base class?"

> "A base class would work, but it creates tight coupling. All providers would need to inherit from the same parent, which limits flexibility. For example, we might want a provider that also conforms to other protocols or has its own inheritance hierarchy. Protocols give us composition over inheritance."

### "Doesn't this add unnecessary complexity?"

> "It can, yes. I follow the rule of waiting for abstraction. If I have only one implementation and no tests, I might not create a protocol initially. But once I need a second implementation or want to write unit tests, I extract the protocol. The key is not abstracting prematurely."

### "How do you decide what goes in the protocol?"

> "I follow Interface Segregation—clients shouldn't depend on methods they don't use. If a protocol is growing too large, I split it. For example, instead of one `UserManagerProtocol` with 15 methods, I'd have `UserAuthenticationProtocol`, `UserProfileProtocol`, and `UserPreferencesProtocol`."

### "What about protocol extensions vs default implementations in abstract classes?"

> "Protocol extensions are more flexible because:
> 1. They work with structs and enums
> 2. A type can adopt multiple protocols with extensions
> 3. Extensions can be constrained (e.g., only apply to UIViewController conformers)
>
> Abstract classes work well when you need stored properties shared across the hierarchy. In UIKit, we often use base view controllers for this reason."

---

## Quick Reference Cheat Sheet

### When to Use Protocols

✅ Defining contracts for dependency injection  
✅ Multiple implementations needed (real + mock)  
✅ Working with value types (structs)  
✅ Need multiple conformances  
✅ Cross-cutting concerns (logging, analytics)  

### When to Use Inheritance

✅ UIKit view controller hierarchy  
✅ Need shared stored properties  
✅ True "is-a" relationships  
✅ Incremental behavior modification (override)  

### Protocol Best Practices

✅ Keep protocols small and focused  
✅ Use protocol extensions for default implementations  
✅ Prefer composition over deep hierarchies  
✅ Use associated types for type-safe generic protocols  
✅ Consider type erasure when storing protocol types  

### Common Mistakes to Avoid

❌ Creating "God protocols" with too many requirements  
❌ Using protocols when only one implementation exists  
❌ Forgetting that protocol types use dynamic dispatch  
❌ Not testing mock implementations  
❌ Overusing associated types when simple generics suffice  

---

## Summary

| Level | Key Takeaways |
|-------|---------------|
| **Beginner** | Inheritance = family tree, Protocols = job contract. Swift prefers protocols for flexibility. |
| **Intermediate** | Use protocols for testability and loose coupling. Protocol extensions provide defaults. Associated types make protocols generic. |
| **Advanced** | Type erasure solves associated type limitations. Understand dispatch and performance trade-offs. Design plugin architectures with protocols. |
| **Interview** | Structure answers with problem → approach → benefit → trade-off. Be ready with real examples. Know when NOT to use protocols. |

---

*This guide is part of the iOS System Design Interview Preparation series.*
