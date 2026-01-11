# iOS Design Patterns: Complete Deep Dive
## From Absolute Beginner to Big Tech Interview Ready

---

## Table of Contents

1. [Beginner Level: What Are Design Patterns?](#-beginner-level-what-are-design-patterns)
2. [Singleton Pattern](#-singleton-pattern)
3. [Factory Pattern](#-factory-pattern)
4. [Builder Pattern](#-builder-pattern)
5. [Observer Pattern](#-observer-pattern)
6. [Adapter Pattern](#-adapter-pattern)
7. [Decorator Pattern](#-decorator-pattern)
8. [Strategy Pattern](#-strategy-pattern)
9. [Advanced: Combining Patterns](#-advanced-combining-patterns)
10. [Interview Preparation](#-interview-preparation)

---

# 🎈 Beginner Level: What Are Design Patterns?

## The Simplest Explanation

Imagine you're building LEGO. Instead of figuring out how to build a car from scratch every time, LEGO gives you **instruction booklets**. These booklets show proven ways to build things that work well.

**Design patterns are instruction booklets for code.**

They're solutions to common problems that smart developers have figured out over decades. Instead of reinventing the wheel, you use these proven approaches.

## Real-Life Analogies

| Pattern | Real-Life Analogy |
|---------|-------------------|
| **Singleton** | There's only ONE president of a country at a time |
| **Factory** | A car factory decides which car model to produce based on order |
| **Builder** | Subway sandwich - add ingredients step by step |
| **Observer** | YouTube subscription - you get notified of new videos |
| **Adapter** | Power adapter converts EU plug to US socket |
| **Decorator** | Adding toppings to pizza - base stays same, extras added |
| **Strategy** | GPS app choosing route - fastest, shortest, no tolls |

## Why Patterns Matter in Mobile Apps

```
Without Patterns:
┌─────────────────────────────────────┐
│     SPAGHETTI CODE 🍝               │
│  • Hard to test                     │
│  • Hard to change                   │
│  • Hard for others to understand    │
│  • Bugs hide everywhere             │
│  • App crashes randomly             │
└─────────────────────────────────────┘

With Patterns:
┌─────────────────────────────────────┐
│     CLEAN ARCHITECTURE 🏛️           │
│  ✅ Easy to test                    │
│  ✅ Easy to change one thing        │
│  ✅ Team can work together          │
│  ✅ Bugs are isolated               │
│  ✅ App is stable                   │
└─────────────────────────────────────┘
```

### Three Key Benefits

1. **Scalability**: Code grows without becoming a mess
2. **Testability**: You can write unit tests easily
3. **Readability**: Any developer can understand your code

---

# 🔷 Singleton Pattern

## What It Is (Simple English)

A Singleton ensures **only ONE instance** of a class exists in your entire app. Everyone who needs it uses the same instance.

## Real-Life Analogy

Think of the **Settings app** on your iPhone. There's only ONE Settings app. Every app that needs to know your notification preferences accesses the SAME Settings. You don't have 50 copies of Settings.

## When to Use It ✅

| Good Use Cases | Why It Works |
|---------------|--------------|
| App configuration | One source of truth for settings |
| Analytics tracker | Single queue for events |
| Network monitor | One observer for connection state |
| Audio player | One player controls sound |
| Authentication state | One source for logged-in user |

## When to AVOID It ❌

| Bad Use Cases | Why It's a Problem |
|--------------|-------------------|
| ViewModels | Creates hidden dependencies |
| Data repositories | Hard to test, hard to reset |
| Everything "just because" | Over-engineering |

## Beginner Implementation

```swift
// The simplest Singleton in Swift
class AnalyticsManager {
    // 1. Static shared instance
    static let shared = AnalyticsManager()
    
    // 2. Private init prevents creating more instances
    private init() { }
    
    // 3. Methods that operate on the single instance
    func trackEvent(_ name: String) {
        print("📊 Tracking: \(name)")
    }
}

// Usage - same instance everywhere
AnalyticsManager.shared.trackEvent("app_opened")
AnalyticsManager.shared.trackEvent("button_tapped")
```

## Intermediate Implementation (Thread-Safe with State)

```swift
final class NetworkMonitor {
    static let shared = NetworkMonitor()
    
    // Thread-safe property
    private let queue = DispatchQueue(label: "networkMonitor.queue")
    private var _isConnected = false
    
    var isConnected: Bool {
        queue.sync { _isConnected }
    }
    
    private init() {
        startMonitoring()
    }
    
    private func startMonitoring() {
        // NWPathMonitor setup...
    }
    
    func updateConnectionStatus(_ connected: Bool) {
        queue.async { [weak self] in
            self?._isConnected = connected
        }
    }
}
```

## Advanced: Dependency-Injectable Singleton

**The Problem with Basic Singleton:**
```swift
class UserService {
    func fetchUser() {
        // ❌ Hard-coded dependency - impossible to test!
        let user = NetworkManager.shared.fetch("/user")
    }
}
```

**The Solution - Protocol + Injection:**
```swift
// 1. Define protocol
protocol NetworkManagerProtocol {
    func fetch<T: Decodable>(_ endpoint: String) async throws -> T
}

// 2. Singleton conforms to protocol
final class NetworkManager: NetworkManagerProtocol {
    static let shared = NetworkManager()
    private init() {}
    
    func fetch<T: Decodable>(_ endpoint: String) async throws -> T {
        // Real implementation
    }
}

// 3. Mock for testing
class MockNetworkManager: NetworkManagerProtocol {
    var mockResult: Any?
    
    func fetch<T: Decodable>(_ endpoint: String) async throws -> T {
        return mockResult as! T
    }
}

// 4. Service accepts protocol, not concrete type
class UserService {
    private let network: NetworkManagerProtocol
    
    // Default to shared, but allow injection
    init(network: NetworkManagerProtocol = NetworkManager.shared) {
        self.network = network
    }
    
    func fetchUser() async throws -> User {
        try await network.fetch("/user")
    }
}

// 5. Testing is now easy!
func testFetchUser() async throws {
    let mockNetwork = MockNetworkManager()
    mockNetwork.mockResult = User(id: "1", name: "Test")
    
    let service = UserService(network: mockNetwork)
    let user = try await service.fetchUser()
    
    XCTAssertEqual(user.name, "Test")
}
```

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Using Singleton for everything | Creates hidden dependencies | Use DI instead |
| Not making init private | Others can create instances | Add `private init()` |
| Mutable shared state | Race conditions | Use serial queue or actor |
| Not using protocol | Can't test | Extract protocol |

## Interview Question

> **Q: "When would you use a Singleton in an iOS app?"**

**Good Answer:**
"I'd use Singleton for truly app-wide, shared resources like analytics tracking or network monitoring. For example, our analytics manager needs to batch events and send them to one endpoint - having multiple instances would cause duplicate events.

However, I'd make it injectable through a protocol so I can mock it in tests. And I'd never use Singleton for things like ViewModels or repositories, because that makes testing hard and creates hidden dependencies."

---

# 🏭 Factory Pattern

## What It Is (Simple English)

A Factory creates objects for you without you knowing the exact class being created. You ask for something, the factory figures out what to give you.

## Real-Life Analogy

Think of **ordering food at a restaurant**. You say "I want a burger." You don't go to the kitchen and cook it yourself. The kitchen (factory) decides which chef makes it, what ingredients to use, and delivers the finished burger.

## When to Use It ✅

| Scenario | Why Factory Helps |
|----------|------------------|
| Creating view controllers | Different VCs for different user states |
| Network responses | Parse JSON into correct model type |
| Feature flags | Return mock vs real implementation |
| Payment processing | Stripe, Apple Pay, PayPal handlers |
| Theming | Light mode vs dark mode components |

## Beginner Implementation

```swift
// 1. Define what we want to create
protocol PaymentProcessor {
    func process(amount: Decimal) async throws -> PaymentResult
}

// 2. Concrete implementations
class StripeProcessor: PaymentProcessor {
    func process(amount: Decimal) async throws -> PaymentResult {
        // Stripe SDK logic
        return PaymentResult(success: true)
    }
}

class ApplePayProcessor: PaymentProcessor {
    func process(amount: Decimal) async throws -> PaymentResult {
        // Apple Pay logic
        return PaymentResult(success: true)
    }
}

class PayPalProcessor: PaymentProcessor {
    func process(amount: Decimal) async throws -> PaymentResult {
        // PayPal SDK logic
        return PaymentResult(success: true)
    }
}

// 3. Factory creates the right one
enum PaymentMethod {
    case stripe
    case applePay
    case paypal
}

class PaymentProcessorFactory {
    static func create(for method: PaymentMethod) -> PaymentProcessor {
        switch method {
        case .stripe:
            return StripeProcessor()
        case .applePay:
            return ApplePayProcessor()
        case .paypal:
            return PayPalProcessor()
        }
    }
}

// 4. Usage - caller doesn't know concrete type
let processor = PaymentProcessorFactory.create(for: .applePay)
let result = try await processor.process(amount: 99.99)
```

## Intermediate: ViewController Factory

```swift
// Screen definitions
enum Screen {
    case home
    case profile(userId: String)
    case settings
    case productDetail(productId: String)
}

// Factory creates configured ViewControllers
class ViewControllerFactory {
    private let dependencies: AppDependencies
    
    init(dependencies: AppDependencies) {
        self.dependencies = dependencies
    }
    
    func make(_ screen: Screen) -> UIViewController {
        switch screen {
        case .home:
            let vm = HomeViewModel(
                userService: dependencies.userService,
                feedService: dependencies.feedService
            )
            return HomeViewController(viewModel: vm)
            
        case .profile(let userId):
            let vm = ProfileViewModel(
                userId: userId,
                userService: dependencies.userService
            )
            return ProfileViewController(viewModel: vm)
            
        case .settings:
            let vm = SettingsViewModel(
                settingsService: dependencies.settingsService
            )
            return SettingsViewController(viewModel: vm)
            
        case .productDetail(let productId):
            let vm = ProductDetailViewModel(
                productId: productId,
                productService: dependencies.productService,
                cartService: dependencies.cartService
            )
            return ProductDetailViewController(viewModel: vm)
        }
    }
}

// Usage in Coordinator
class AppCoordinator {
    private let factory: ViewControllerFactory
    
    func navigate(to screen: Screen) {
        let vc = factory.make(screen)
        navigationController.pushViewController(vc, animated: true)
    }
}
```

## Advanced: Abstract Factory for Feature Flags

```swift
// Abstract factory protocol
protocol ServiceFactory {
    func makeAnalytics() -> AnalyticsService
    func makeFeatureFlags() -> FeatureFlagService
    func makeNetworking() -> NetworkService
}

// Production factory
class ProductionServiceFactory: ServiceFactory {
    func makeAnalytics() -> AnalyticsService {
        return FirebaseAnalytics()
    }
    
    func makeFeatureFlags() -> FeatureFlagService {
        return LaunchDarklyService()
    }
    
    func makeNetworking() -> NetworkService {
        return URLSessionNetworkService()
    }
}

// Debug/Testing factory
class MockServiceFactory: ServiceFactory {
    func makeAnalytics() -> AnalyticsService {
        return PrintAnalytics() // Just prints to console
    }
    
    func makeFeatureFlags() -> FeatureFlagService {
        return LocalFeatureFlags() // Reads from plist
    }
    
    func makeNetworking() -> NetworkService {
        return MockNetworkService() // Returns canned responses
    }
}

// App uses whichever factory is injected
class AppDependencies {
    let analytics: AnalyticsService
    let featureFlags: FeatureFlagService
    let networking: NetworkService
    
    init(factory: ServiceFactory) {
        self.analytics = factory.makeAnalytics()
        self.featureFlags = factory.makeFeatureFlags()
        self.networking = factory.makeNetworking()
    }
}

// In AppDelegate
#if DEBUG
let factory = MockServiceFactory()
#else
let factory = ProductionServiceFactory()
#endif

let dependencies = AppDependencies(factory: factory)
```

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Factory knows too many details | Violates Single Responsibility | Split into focused factories |
| Returning concrete types | Defeats the purpose | Return protocols |
| Static-only factory | Hard to test | Allow instance factories |

## Interview Question

> **Q: "How would you design a system that can switch between different payment providers?"**

**Good Answer:**
"I'd use the Factory pattern with a protocol defining the payment interface. Each provider (Stripe, Apple Pay, PayPal) implements this protocol. The factory takes a payment method enum and returns the correct implementation.

The key benefit is the checkout flow doesn't know or care which provider is used - it just calls `process()` on whatever the factory returns. This makes adding new providers easy (just add a new enum case and implementation) and testing simple (inject a mock processor)."

---

# 🔨 Builder Pattern

## What It Is (Simple English)

The Builder pattern constructs complex objects step by step. You can produce different types and representations of an object using the same building process.

## Real-Life Analogy

Think of **Subway sandwich ordering**:
1. Choose bread
2. Choose protein
3. Choose veggies
4. Choose sauce
5. Toast or not?
6. **Final sandwich!**

You build your sandwich step by step, and the final result depends on your choices.

## When to Use It ✅

| Scenario | Why Builder Helps |
|----------|------------------|
| URLRequest construction | Headers, body, method, params |
| Alert/notification setup | Title, message, buttons, actions |
| Complex model creation | Many optional parameters |
| Test data setup | Create variations easily |
| Configuration objects | Build different configs |

## Beginner Implementation

```swift
// Without Builder - too many parameters!
let request = URLRequest(
    url: url,
    method: .post,
    headers: ["Authorization": "Bearer xxx"],
    body: jsonData,
    cachePolicy: .reloadIgnoringLocalCache,
    timeout: 30
)

// With Builder - clear and readable
class URLRequestBuilder {
    private var url: URL
    private var method: String = "GET"
    private var headers: [String: String] = [:]
    private var body: Data?
    private var timeout: TimeInterval = 30
    
    init(url: URL) {
        self.url = url
    }
    
    func setMethod(_ method: String) -> Self {
        self.method = method
        return self
    }
    
    func addHeader(key: String, value: String) -> Self {
        headers[key] = value
        return self
    }
    
    func setBody(_ body: Data) -> Self {
        self.body = body
        return self
    }
    
    func setTimeout(_ timeout: TimeInterval) -> Self {
        self.timeout = timeout
        return self
    }
    
    func build() -> URLRequest {
        var request = URLRequest(url: url)
        request.httpMethod = method
        request.allHTTPHeaderFields = headers
        request.httpBody = body
        request.timeoutInterval = timeout
        return request
    }
}

// Usage - fluent, readable API
let request = URLRequestBuilder(url: apiURL)
    .setMethod("POST")
    .addHeader(key: "Authorization", value: "Bearer \(token)")
    .addHeader(key: "Content-Type", value: "application/json")
    .setBody(jsonData)
    .setTimeout(60)
    .build()
```

## Intermediate: Alert Builder

```swift
class AlertBuilder {
    private var title: String?
    private var message: String?
    private var style: UIAlertController.Style = .alert
    private var actions: [UIAlertAction] = []
    private var textFields: [(UITextField) -> Void] = []
    
    func setTitle(_ title: String) -> Self {
        self.title = title
        return self
    }
    
    func setMessage(_ message: String) -> Self {
        self.message = message
        return self
    }
    
    func setStyle(_ style: UIAlertController.Style) -> Self {
        self.style = style
        return self
    }
    
    func addAction(
        title: String,
        style: UIAlertAction.Style = .default,
        handler: ((UIAlertAction) -> Void)? = nil
    ) -> Self {
        let action = UIAlertAction(title: title, style: style, handler: handler)
        actions.append(action)
        return self
    }
    
    func addDestructiveAction(
        title: String,
        handler: @escaping (UIAlertAction) -> Void
    ) -> Self {
        return addAction(title: title, style: .destructive, handler: handler)
    }
    
    func addCancelAction(title: String = "Cancel") -> Self {
        return addAction(title: title, style: .cancel)
    }
    
    func addTextField(configuration: @escaping (UITextField) -> Void) -> Self {
        textFields.append(configuration)
        return self
    }
    
    func build() -> UIAlertController {
        let alert = UIAlertController(
            title: title,
            message: message,
            preferredStyle: style
        )
        
        textFields.forEach { config in
            alert.addTextField(configurationHandler: config)
        }
        
        actions.forEach { alert.addAction($0) }
        
        return alert
    }
}

// Usage
let deleteAlert = AlertBuilder()
    .setTitle("Delete Item?")
    .setMessage("This action cannot be undone.")
    .addDestructiveAction(title: "Delete") { _ in
        self.deleteItem()
    }
    .addCancelAction()
    .build()

present(deleteAlert, animated: true)

// Login dialog with text fields
let loginAlert = AlertBuilder()
    .setTitle("Login Required")
    .addTextField { textField in
        textField.placeholder = "Email"
        textField.keyboardType = .emailAddress
    }
    .addTextField { textField in
        textField.placeholder = "Password"
        textField.isSecureTextEntry = true
    }
    .addAction(title: "Login") { [weak self] _ in
        let email = alert.textFields?[0].text ?? ""
        let password = alert.textFields?[1].text ?? ""
        self?.login(email: email, password: password)
    }
    .addCancelAction()
    .build()
```

## Advanced: Test Data Builder

```swift
// Product model
struct Product {
    let id: String
    let name: String
    let price: Decimal
    let description: String?
    let imageURL: URL?
    let category: Category
    let inStock: Bool
    let rating: Double?
    let reviewCount: Int
}

// Test builder with sensible defaults
class ProductBuilder {
    private var id = UUID().uuidString
    private var name = "Test Product"
    private var price: Decimal = 9.99
    private var description: String? = nil
    private var imageURL: URL? = nil
    private var category: Category = .electronics
    private var inStock = true
    private var rating: Double? = 4.5
    private var reviewCount = 100
    
    func withId(_ id: String) -> Self {
        self.id = id
        return self
    }
    
    func withName(_ name: String) -> Self {
        self.name = name
        return self
    }
    
    func withPrice(_ price: Decimal) -> Self {
        self.price = price
        return self
    }
    
    func withDescription(_ description: String) -> Self {
        self.description = description
        return self
    }
    
    func outOfStock() -> Self {
        self.inStock = false
        return self
    }
    
    func withRating(_ rating: Double, reviewCount: Int) -> Self {
        self.rating = rating
        self.reviewCount = reviewCount
        return self
    }
    
    func withNoRating() -> Self {
        self.rating = nil
        self.reviewCount = 0
        return self
    }
    
    func build() -> Product {
        return Product(
            id: id,
            name: name,
            price: price,
            description: description,
            imageURL: imageURL,
            category: category,
            inStock: inStock,
            rating: rating,
            reviewCount: reviewCount
        )
    }
}

// Tests become very readable
func testOutOfStockBadge() {
    let product = ProductBuilder()
        .withName("iPhone")
        .outOfStock()
        .build()
    
    let cell = ProductCell()
    cell.configure(with: product)
    
    XCTAssertFalse(cell.outOfStockBadge.isHidden)
}

func testNoRatingState() {
    let product = ProductBuilder()
        .withNoRating()
        .build()
    
    let cell = ProductCell()
    cell.configure(with: product)
    
    XCTAssertTrue(cell.ratingView.isHidden)
}
```

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Not returning `Self` | Can't chain methods | Return `self` from setters |
| Forgetting `build()` | Object never created | Make `build()` required |
| Too many required params | Defeats the purpose | Use defaults |

## Interview Question

> **Q: "How would you design an API for building complex network requests?"**

**Good Answer:**
"I'd use the Builder pattern. The builder would start with just the URL, then allow chaining methods like `.setMethod()`, `.addHeader()`, `.setBody()`, and `.setTimeout()`. Each method returns `self` for fluent chaining, and finally `.build()` produces the `URLRequest`.

Benefits: it's readable (you can see each part being set), flexible (skip optional parts), and prevents invalid objects (build() can validate before returning). It also makes tests cleaner - you can create request variations easily without huge initializers."

---

# 👁️ Observer Pattern

## What It Is (Simple English)

The Observer pattern lets one object (the **subject**) notify many other objects (the **observers**) when something changes. Observers "subscribe" to updates.

## Real-Life Analogy

Think of **YouTube subscriptions**:
- You subscribe to a channel
- When the channel uploads a new video, you get notified
- You can unsubscribe anytime
- The channel doesn't need to know who you are - it just broadcasts to all subscribers

## When to Use It ✅

| Scenario | Why Observer Helps |
|----------|------------------|
| UI updates from ViewModel | View observes state changes |
| Network status changes | Multiple screens need to react |
| User login/logout | Update UI across the app |
| Cart updates | Badge, checkout, list all update |
| Settings changes | Apply theme/language everywhere |

## iOS Observer Implementations

### 1. NotificationCenter (Built-in)

```swift
// POSTING a notification (Subject)
class AuthService {
    func login(user: User) {
        // ... login logic ...
        
        NotificationCenter.default.post(
            name: .userDidLogin,
            object: nil,
            userInfo: ["user": user]
        )
    }
    
    func logout() {
        NotificationCenter.default.post(
            name: .userDidLogout,
            object: nil
        )
    }
}

// Define notification names
extension Notification.Name {
    static let userDidLogin = Notification.Name("userDidLogin")
    static let userDidLogout = Notification.Name("userDidLogout")
}

// OBSERVING (Observer)
class ProfileViewController: UIViewController {
    private var observers: [NSObjectProtocol] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Modern block-based observer
        let observer = NotificationCenter.default.addObserver(
            forName: .userDidLogin,
            object: nil,
            queue: .main
        ) { [weak self] notification in
            guard let user = notification.userInfo?["user"] as? User else { return }
            self?.updateUI(for: user)
        }
        
        observers.append(observer)
    }
    
    deinit {
        // Always remove observers!
        observers.forEach { NotificationCenter.default.removeObserver($0) }
    }
}
```

### 2. Combine Framework (Modern Apple Way)

```swift
import Combine

// ViewModel as Subject
class CartViewModel {
    // Published properties automatically notify observers
    @Published private(set) var items: [CartItem] = []
    @Published private(set) var totalPrice: Decimal = 0
    @Published private(set) var itemCount: Int = 0
    
    func addItem(_ product: Product) {
        items.append(CartItem(product: product, quantity: 1))
        recalculate()
    }
    
    func removeItem(at index: Int) {
        items.remove(at: index)
        recalculate()
    }
    
    private func recalculate() {
        totalPrice = items.reduce(0) { $0 + $1.product.price * Decimal($1.quantity) }
        itemCount = items.reduce(0) { $0 + $1.quantity }
    }
}

// View as Observer
class CartViewController: UIViewController {
    private let viewModel: CartViewModel
    private var cancellables = Set<AnyCancellable>()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // Observe items
        viewModel.$items
            .receive(on: DispatchQueue.main)
            .sink { [weak self] items in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
        
        // Observe total price
        viewModel.$totalPrice
            .map { "$\($0)" }
            .assign(to: \.text, on: totalLabel)
            .store(in: &cancellables)
        
        // Observe item count for badge
        viewModel.$itemCount
            .sink { [weak self] count in
                self?.tabBarItem.badgeValue = count > 0 ? "\(count)" : nil
            }
            .store(in: &cancellables)
    }
}
```

### 3. Custom Observer Implementation

```swift
// When you need fine-grained control

protocol CartObserver: AnyObject {
    func cartDidUpdate(_ cart: Cart)
    func cartDidAddItem(_ item: CartItem)
    func cartDidRemoveItem(_ item: CartItem)
}

class CartService {
    static let shared = CartService()
    
    private var observers = NSHashTable<AnyObject>.weakObjects()
    private(set) var cart = Cart()
    
    func addObserver(_ observer: CartObserver) {
        observers.add(observer)
    }
    
    func removeObserver(_ observer: CartObserver) {
        observers.remove(observer)
    }
    
    func addItem(_ item: CartItem) {
        cart.items.append(item)
        
        // Notify all observers
        for case let observer as CartObserver in observers.allObjects {
            observer.cartDidAddItem(item)
            observer.cartDidUpdate(cart)
        }
    }
    
    func removeItem(_ item: CartItem) {
        cart.items.removeAll { $0.id == item.id }
        
        for case let observer as CartObserver in observers.allObjects {
            observer.cartDidRemoveItem(item)
            observer.cartDidUpdate(cart)
        }
    }
}

// Usage
class CartBadgeView: UIView, CartObserver {
    override init(frame: CGRect) {
        super.init(frame: frame)
        CartService.shared.addObserver(self)
    }
    
    func cartDidUpdate(_ cart: Cart) {
        updateBadge(count: cart.items.count)
    }
    
    func cartDidAddItem(_ item: CartItem) { }
    func cartDidRemoveItem(_ item: CartItem) { }
}
```

## Advanced: Reactive State Management

```swift
// Observable store pattern (like Redux)
final class Store<State, Action> {
    private(set) var state: State
    private let reducer: (State, Action) -> State
    private var subscribers: [(State) -> Void] = []
    private let queue = DispatchQueue(label: "store.queue")
    
    init(initialState: State, reducer: @escaping (State, Action) -> State) {
        self.state = initialState
        self.reducer = reducer
    }
    
    func dispatch(_ action: Action) {
        queue.async { [weak self] in
            guard let self = self else { return }
            self.state = self.reducer(self.state, action)
            
            DispatchQueue.main.async {
                self.subscribers.forEach { $0(self.state) }
            }
        }
    }
    
    func subscribe(_ subscriber: @escaping (State) -> Void) {
        subscriber(state) // Immediate update with current state
        subscribers.append(subscriber)
    }
}

// Usage
struct AppState {
    var user: User?
    var cart: Cart
    var isLoading: Bool
}

enum AppAction {
    case login(User)
    case logout
    case addToCart(Product)
    case setLoading(Bool)
}

let store = Store<AppState, AppAction>(
    initialState: AppState(user: nil, cart: Cart(), isLoading: false)
) { state, action in
    var newState = state
    switch action {
    case .login(let user):
        newState.user = user
    case .logout:
        newState.user = nil
    case .addToCart(let product):
        newState.cart.add(product)
    case .setLoading(let loading):
        newState.isLoading = loading
    }
    return newState
}
```

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Forgetting to remove observer | Memory leak, crashes | Use `deinit` or weak references |
| Strong reference in closure | Retain cycle | Use `[weak self]` |
| Too many notifications | Performance, spaghetti | Use Combine or direct binding |
| Not using main queue for UI | Crashes, undefined behavior | `.receive(on: .main)` |

## Interview Question

> **Q: "How would you propagate state changes across multiple screens?"**

**Good Answer:**
"I'd use Combine with a shared ViewModel or Store pattern. For example, when the cart updates, the CartViewModel publishes changes through `@Published` properties. Any screen that needs cart data subscribes to these publishers.

The key is using weak references to avoid retain cycles and receiving updates on the main queue for UI. For app-wide state like login status, I might use NotificationCenter for simplicity, or a singleton Store that screens subscribe to."

---

# 🔌 Adapter Pattern

## What It Is (Simple English)

The Adapter pattern converts the interface of one class into an interface that another class expects. It lets incompatible classes work together.

## Real-Life Analogy

Think of a **travel power adapter**. Your iPhone charger has a US plug, but you're in Europe with EU sockets. The adapter doesn't change your charger or the socket - it just makes them compatible.

## When to Use It ✅

| Scenario | Why Adapter Helps |
|----------|------------------|
| Third-party SDK integration | Wrap SDK in your interface |
| Legacy code migration | Old API → new API |
| API response transformation | Server model → app model |
| Multiple analytics providers | Unified analytics interface |
| Different image loaders | SDWebImage, Kingfisher, custom |

## Beginner Implementation

```swift
// Your app's analytics interface
protocol AnalyticsTracker {
    func track(event: String, properties: [String: Any]?)
    func identify(userId: String)
    func reset()
}

// Third-party SDK (let's say Mixpanel)
// Has its own API that doesn't match yours
class MixpanelSDK {
    func trackEvent(_ name: String, params: NSDictionary?) { }
    func identifyUser(_ id: String) { }
    func clearUserData() { }
}

// Adapter makes Mixpanel work with your interface
class MixpanelAdapter: AnalyticsTracker {
    private let mixpanel = MixpanelSDK()
    
    func track(event: String, properties: [String: Any]?) {
        mixpanel.trackEvent(event, params: properties as NSDictionary?)
    }
    
    func identify(userId: String) {
        mixpanel.identifyUser(userId)
    }
    
    func reset() {
        mixpanel.clearUserData()
    }
}

// Now you can swap analytics providers easily
class AnalyticsService {
    private let tracker: AnalyticsTracker
    
    init(tracker: AnalyticsTracker = MixpanelAdapter()) {
        self.tracker = tracker
    }
    
    func trackScreenView(_ screenName: String) {
        tracker.track(event: "screen_view", properties: ["screen": screenName])
    }
}
```

## Intermediate: API Response Adapter

```swift
// Server returns this format
struct ServerUserResponse: Codable {
    let user_id: String
    let first_name: String
    let last_name: String
    let avatar_url: String?
    let created_at: String // ISO8601 string
}

// Your app uses this model
struct User {
    let id: String
    let fullName: String
    let avatarURL: URL?
    let createdAt: Date
}

// Adapter converts server response to app model
class UserAdapter {
    private let dateFormatter: ISO8601DateFormatter = {
        let formatter = ISO8601DateFormatter()
        formatter.formatOptions = [.withInternetDateTime]
        return formatter
    }()
    
    func adapt(_ response: ServerUserResponse) -> User {
        return User(
            id: response.user_id,
            fullName: "\(response.first_name) \(response.last_name)",
            avatarURL: response.avatar_url.flatMap { URL(string: $0) },
            createdAt: dateFormatter.date(from: response.created_at) ?? Date()
        )
    }
    
    func adapt(_ responses: [ServerUserResponse]) -> [User] {
        responses.map { adapt($0) }
    }
}

// Usage in network layer
class UserAPIService {
    private let adapter = UserAdapter()
    
    func fetchUser(id: String) async throws -> User {
        let response: ServerUserResponse = try await network.get("/users/\(id)")
        return adapter.adapt(response)
    }
}
```

## Advanced: Universal Image Loader Adapter

```swift
// Your app's image loading interface
protocol ImageLoader {
    func load(
        url: URL,
        into imageView: UIImageView,
        placeholder: UIImage?,
        completion: ((Result<UIImage, Error>) -> Void)?
    )
    
    func cancelLoad(for imageView: UIImageView)
    func clearCache()
}

// Adapter for SDWebImage
import SDWebImage

class SDWebImageAdapter: ImageLoader {
    func load(
        url: URL,
        into imageView: UIImageView,
        placeholder: UIImage?,
        completion: ((Result<UIImage, Error>) -> Void)?
    ) {
        imageView.sd_setImage(with: url, placeholderImage: placeholder) { image, error, _, _ in
            if let image = image {
                completion?(.success(image))
            } else if let error = error {
                completion?(.failure(error))
            }
        }
    }
    
    func cancelLoad(for imageView: UIImageView) {
        imageView.sd_cancelCurrentImageLoad()
    }
    
    func clearCache() {
        SDImageCache.shared.clearMemory()
        SDImageCache.shared.clearDisk()
    }
}

// Adapter for Kingfisher
import Kingfisher

class KingfisherAdapter: ImageLoader {
    func load(
        url: URL,
        into imageView: UIImageView,
        placeholder: UIImage?,
        completion: ((Result<UIImage, Error>) -> Void)?
    ) {
        imageView.kf.setImage(with: url, placeholder: placeholder) { result in
            switch result {
            case .success(let value):
                completion?(.success(value.image))
            case .failure(let error):
                completion?(.failure(error))
            }
        }
    }
    
    func cancelLoad(for imageView: UIImageView) {
        imageView.kf.cancelDownloadTask()
    }
    
    func clearCache() {
        KingfisherManager.shared.cache.clearMemoryCache()
        KingfisherManager.shared.cache.clearDiskCache()
    }
}

// Your app code stays the same regardless of which library you use
class ProductCell: UICollectionViewCell {
    @IBOutlet var imageView: UIImageView!
    
    // Injected, can be swapped without changing cell code
    var imageLoader: ImageLoader!
    
    func configure(with product: Product) {
        imageLoader.load(
            url: product.imageURL,
            into: imageView,
            placeholder: UIImage(named: "placeholder"),
            completion: nil
        )
    }
    
    override func prepareForReuse() {
        super.prepareForReuse()
        imageLoader.cancelLoad(for: imageView)
    }
}
```

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Leaking third-party types | Defeats the purpose | Keep SDK types internal |
| Too thin adapter | Doesn't add value | Add meaningful abstraction |
| Adapting too early | Over-engineering | Adapt only when swapping is needed |

## Interview Question

> **Q: "How would you integrate multiple analytics SDKs without coupling your code to specific vendors?"**

**Good Answer:**
"I'd use the Adapter pattern. First, define my app's analytics protocol with methods like `track(event:)` and `identify(userId:)`. Then create an adapter for each SDK - `MixpanelAdapter`, `FirebaseAdapter`, etc. - that implements this protocol.

My app code only knows about the protocol, not the SDKs. This means I can switch providers without touching any analytics call sites. I can also create a composite adapter that sends events to multiple providers simultaneously, or a mock adapter for testing."

---

# 🎨 Decorator Pattern

## What It Is (Simple English)

The Decorator pattern adds new behavior to objects without changing their code. You "wrap" an object with another object that adds functionality.

## Real-Life Analogy

Think of **pizza toppings**. You start with a base pizza (cheese), then add pepperoni, then add mushrooms. Each topping "decorates" the pizza, adding to the base without replacing it.

## When to Use It ✅

| Scenario | Why Decorator Helps |
|----------|---------------------|
| Adding logging to network calls | Wrap network service |
| Adding retry logic | Wrap any async operation |
| Adding caching | Wrap repository |
| Request authentication | Wrap URLSession |
| UI enhancement | Wrap base views |

## Beginner Implementation

```swift
// Base protocol
protocol DataFetcher {
    func fetch(from url: URL) async throws -> Data
}

// Basic implementation
class NetworkDataFetcher: DataFetcher {
    func fetch(from url: URL) async throws -> Data {
        let (data, _) = try await URLSession.shared.data(from: url)
        return data
    }
}

// Logging Decorator - adds logging without changing NetworkDataFetcher
class LoggingDataFetcher: DataFetcher {
    private let wrapped: DataFetcher
    
    init(wrapping fetcher: DataFetcher) {
        self.wrapped = fetcher
    }
    
    func fetch(from url: URL) async throws -> Data {
        print("📤 Fetching: \(url)")
        let start = Date()
        
        do {
            let data = try await wrapped.fetch(from: url)
            let duration = Date().timeIntervalSince(start)
            print("📥 Success: \(data.count) bytes in \(duration)s")
            return data
        } catch {
            print("❌ Failed: \(error)")
            throw error
        }
    }
}

// Usage - compose decorators
let baseFetcher = NetworkDataFetcher()
let loggingFetcher = LoggingDataFetcher(wrapping: baseFetcher)

// Now all fetches are logged automatically
let data = try await loggingFetcher.fetch(from: someURL)
```

## Intermediate: Caching Decorator

```swift
// Add caching without modifying the network layer
class CachingDataFetcher: DataFetcher {
    private let wrapped: DataFetcher
    private let cache: URLCache
    
    init(wrapping fetcher: DataFetcher, cache: URLCache = .shared) {
        self.wrapped = fetcher
        self.cache = cache
    }
    
    func fetch(from url: URL) async throws -> Data {
        let request = URLRequest(url: url)
        
        // Check cache first
        if let cached = cache.cachedResponse(for: request) {
            print("📦 Cache hit for \(url)")
            return cached.data
        }
        
        // Fetch from network
        let data = try await wrapped.fetch(from: url)
        
        // Store in cache
        let response = URLResponse(
            url: url,
            mimeType: nil,
            expectedContentLength: data.count,
            textEncodingName: nil
        )
        let cachedResponse = CachedURLResponse(response: response, data: data)
        cache.storeCachedResponse(cachedResponse, for: request)
        
        return data
    }
}

// Retry Decorator
class RetryingDataFetcher: DataFetcher {
    private let wrapped: DataFetcher
    private let maxRetries: Int
    private let delay: TimeInterval
    
    init(wrapping fetcher: DataFetcher, maxRetries: Int = 3, delay: TimeInterval = 1.0) {
        self.wrapped = fetcher
        self.maxRetries = maxRetries
        self.delay = delay
    }
    
    func fetch(from url: URL) async throws -> Data {
        var lastError: Error?
        
        for attempt in 0..<maxRetries {
            do {
                return try await wrapped.fetch(from: url)
            } catch {
                lastError = error
                print("⚠️ Attempt \(attempt + 1) failed, retrying...")
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            }
        }
        
        throw lastError ?? NetworkError.unknown
    }
}

// Stack decorators for full-featured fetching
let fetcher = LoggingDataFetcher(
    wrapping: RetryingDataFetcher(
        wrapping: CachingDataFetcher(
            wrapping: NetworkDataFetcher()
        )
    )
)

// Now: logging → retry → cache → network
```

## Advanced: Authentication Decorator

```swift
protocol APIClient {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T
}

class BaseAPIClient: APIClient {
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        let request = endpoint.urlRequest
        let (data, _) = try await URLSession.shared.data(for: request)
        return try JSONDecoder().decode(T.self, from: data)
    }
}

// Adds authentication to any API client
class AuthenticatedAPIClient: APIClient {
    private let wrapped: APIClient
    private let tokenProvider: TokenProvider
    
    init(wrapping client: APIClient, tokenProvider: TokenProvider) {
        self.wrapped = client
        self.tokenProvider = tokenProvider
    }
    
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        // Add auth header
        var authenticatedEndpoint = endpoint
        
        if let token = await tokenProvider.accessToken {
            authenticatedEndpoint.headers["Authorization"] = "Bearer \(token)"
        }
        
        do {
            return try await wrapped.request(authenticatedEndpoint)
        } catch let error as APIError where error == .unauthorized {
            // Try to refresh token
            try await tokenProvider.refreshToken()
            
            // Retry with new token
            if let newToken = await tokenProvider.accessToken {
                authenticatedEndpoint.headers["Authorization"] = "Bearer \(newToken)"
            }
            return try await wrapped.request(authenticatedEndpoint)
        }
    }
}

// Usage
let baseClient = BaseAPIClient()
let authClient = AuthenticatedAPIClient(
    wrapping: baseClient,
    tokenProvider: KeychainTokenProvider()
)

let user: User = try await authClient.request(.getUser)
```

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Too many decorators | Hard to debug | Limit nesting depth |
| Breaking interface contract | Unexpected behavior | Maintain same semantics |
| Heavy decorators | Performance overhead | Keep decorators lightweight |

## Interview Question

> **Q: "How would you add logging and retry logic to an existing network layer without modifying it?"**

**Good Answer:**
"I'd use the Decorator pattern. Both the network layer and decorators implement the same protocol (like `DataFetcher`). The LoggingDecorator wraps the network layer and logs before/after calling through. The RetryDecorator wraps anything and adds retry logic.

You can stack them: `Retry → Logging → Network`. Each layer is independent and testable. Adding new behavior (like caching) is just adding another decorator - no changes to existing code."

---

# 🎯 Strategy Pattern

## What It Is (Simple English)

The Strategy pattern defines a family of algorithms and makes them interchangeable. You can switch between different behaviors at runtime.

## Real-Life Analogy

Think of **navigation apps** like Google Maps. You choose a strategy:
- Fastest route
- Shortest route
- No highways
- No tolls

The destination is the same, but the way to get there changes based on your choice.

## When to Use It ✅

| Scenario | Why Strategy Helps |
|----------|-------------------|
| Sorting options | Sort by price, rating, distance |
| Payment methods | Card, Apple Pay, PayPal |
| Image compression | JPEG, PNG, WebP |
| Data export | CSV, JSON, XML |
| Validation rules | Email, phone, credit card |

## Beginner Implementation

```swift
// Strategy protocol
protocol SortStrategy {
    func sort(_ products: [Product]) -> [Product]
}

// Concrete strategies
class PriceSortStrategy: SortStrategy {
    let ascending: Bool
    
    init(ascending: Bool = true) {
        self.ascending = ascending
    }
    
    func sort(_ products: [Product]) -> [Product] {
        products.sorted { 
            ascending ? $0.price < $1.price : $0.price > $1.price 
        }
    }
}

class RatingSortStrategy: SortStrategy {
    func sort(_ products: [Product]) -> [Product] {
        products.sorted { $0.rating > $1.rating }
    }
}

class PopularitySortStrategy: SortStrategy {
    func sort(_ products: [Product]) -> [Product] {
        products.sorted { $0.orderCount > $1.orderCount }
    }
}

// Context that uses strategy
class ProductListViewModel {
    private var products: [Product] = []
    private var sortStrategy: SortStrategy = PriceSortStrategy()
    
    var sortedProducts: [Product] {
        sortStrategy.sort(products)
    }
    
    // Change strategy at runtime
    func setSortStrategy(_ strategy: SortStrategy) {
        self.sortStrategy = strategy
    }
}

// Usage
let viewModel = ProductListViewModel()
viewModel.setSortStrategy(PriceSortStrategy(ascending: true))  // Low to high
viewModel.setSortStrategy(RatingSortStrategy())                 // Best rated
viewModel.setSortStrategy(PopularitySortStrategy())            // Most popular
```

## Intermediate: Validation Strategy

```swift
// Validation result
struct ValidationResult {
    let isValid: Bool
    let errorMessage: String?
    
    static let valid = ValidationResult(isValid: true, errorMessage: nil)
    
    static func invalid(_ message: String) -> ValidationResult {
        ValidationResult(isValid: false, errorMessage: message)
    }
}

// Strategy protocol
protocol ValidationStrategy {
    func validate(_ input: String) -> ValidationResult
}

// Email validation
class EmailValidationStrategy: ValidationStrategy {
    func validate(_ input: String) -> ValidationResult {
        let emailRegex = #"^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$"#
        let isValid = input.range(of: emailRegex, options: .regularExpression) != nil
        return isValid ? .valid : .invalid("Please enter a valid email address")
    }
}

// Password validation
class PasswordValidationStrategy: ValidationStrategy {
    let minLength: Int
    let requiresUppercase: Bool
    let requiresNumber: Bool
    
    init(minLength: Int = 8, requiresUppercase: Bool = true, requiresNumber: Bool = true) {
        self.minLength = minLength
        self.requiresUppercase = requiresUppercase
        self.requiresNumber = requiresNumber
    }
    
    func validate(_ input: String) -> ValidationResult {
        if input.count < minLength {
            return .invalid("Password must be at least \(minLength) characters")
        }
        if requiresUppercase && !input.contains(where: { $0.isUppercase }) {
            return .invalid("Password must contain an uppercase letter")
        }
        if requiresNumber && !input.contains(where: { $0.isNumber }) {
            return .invalid("Password must contain a number")
        }
        return .valid
    }
}

// Phone validation
class PhoneValidationStrategy: ValidationStrategy {
    func validate(_ input: String) -> ValidationResult {
        let digits = input.filter { $0.isNumber }
        let isValid = digits.count >= 10 && digits.count <= 15
        return isValid ? .valid : .invalid("Please enter a valid phone number")
    }
}

// Generic form field that can use any validation
class FormField {
    let textField: UITextField
    let errorLabel: UILabel
    var validationStrategy: ValidationStrategy?
    
    func validate() -> Bool {
        guard let strategy = validationStrategy else { return true }
        
        let result = strategy.validate(textField.text ?? "")
        errorLabel.text = result.errorMessage
        errorLabel.isHidden = result.isValid
        return result.isValid
    }
}

// Usage in form
class RegistrationViewController: UIViewController {
    let emailField = FormField()
    let passwordField = FormField()
    let phoneField = FormField()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        emailField.validationStrategy = EmailValidationStrategy()
        passwordField.validationStrategy = PasswordValidationStrategy(minLength: 8)
        phoneField.validationStrategy = PhoneValidationStrategy()
    }
    
    @IBAction func submitTapped() {
        let allValid = [emailField, passwordField, phoneField].allSatisfy { $0.validate() }
        if allValid {
            // Submit form
        }
    }
}
```

## Advanced: Payment Strategy with Factory

```swift
// Payment strategy
protocol PaymentStrategy {
    var name: String { get }
    var icon: UIImage { get }
    func canProcess(amount: Decimal) -> Bool
    func process(amount: Decimal) async throws -> PaymentResult
}

// Apple Pay
class ApplePayStrategy: PaymentStrategy {
    let name = "Apple Pay"
    var icon: UIImage { UIImage(systemName: "applelogo")! }
    
    func canProcess(amount: Decimal) -> Bool {
        return PKPaymentAuthorizationController.canMakePayments()
    }
    
    func process(amount: Decimal) async throws -> PaymentResult {
        // Apple Pay processing
        return PaymentResult(success: true, transactionId: UUID().uuidString)
    }
}

// Credit Card
class CreditCardStrategy: PaymentStrategy {
    private let stripeService: StripeService
    
    let name = "Credit Card"
    var icon: UIImage { UIImage(systemName: "creditcard")! }
    
    init(stripeService: StripeService) {
        self.stripeService = stripeService
    }
    
    func canProcess(amount: Decimal) -> Bool {
        return true // Always available if card is saved
    }
    
    func process(amount: Decimal) async throws -> PaymentResult {
        return try await stripeService.charge(amount: amount)
    }
}

// PayPal
class PayPalStrategy: PaymentStrategy {
    let name = "PayPal"
    var icon: UIImage { UIImage(named: "paypal")! }
    
    func canProcess(amount: Decimal) -> Bool {
        return amount < 10000 // PayPal limit example
    }
    
    func process(amount: Decimal) async throws -> PaymentResult {
        // PayPal SDK processing
        return PaymentResult(success: true, transactionId: UUID().uuidString)
    }
}

// Checkout uses strategy
class CheckoutViewModel {
    private var selectedStrategy: PaymentStrategy?
    private let amount: Decimal
    
    init(amount: Decimal) {
        self.amount = amount
    }
    
    var availablePaymentMethods: [PaymentStrategy] {
        let all: [PaymentStrategy] = [
            ApplePayStrategy(),
            CreditCardStrategy(stripeService: .shared),
            PayPalStrategy()
        ]
        return all.filter { $0.canProcess(amount: amount) }
    }
    
    func selectPaymentMethod(_ strategy: PaymentStrategy) {
        self.selectedStrategy = strategy
    }
    
    func checkout() async throws -> PaymentResult {
        guard let strategy = selectedStrategy else {
            throw CheckoutError.noPaymentSelected
        }
        return try await strategy.process(amount: amount)
    }
}
```

## Common Mistakes

| Mistake | Why It's Bad | Fix |
|---------|-------------|-----|
| Strategy knows context | Tight coupling | Strategy should be independent |
| Too many strategies | Complex hierarchy | Merge similar strategies |
| Stateful strategies | Unexpected behavior | Keep strategies stateless |

## Interview Question

> **Q: "How would you design a system that supports multiple payment providers?"**

**Good Answer:**
"I'd use the Strategy pattern. Define a `PaymentStrategy` protocol with methods like `canProcess(amount:)` and `process(amount:)`. Each provider (Apple Pay, Stripe, PayPal) is a concrete strategy.

The checkout flow holds a reference to the selected strategy without knowing its concrete type. The UI can show available options by filtering strategies that return true for `canProcess()`. This makes adding new payment methods a matter of adding a new class - no changes to checkout logic."

---

# 🧩 Advanced: Combining Patterns

## Real-World Example: Image Loading Pipeline

This is how production apps like Instagram, Twitter, or Uber structure image loading:

```
            ┌─────────────────────────────────────────────┐
User taps → │            ImageLoader (Facade)              │
            │  ┌─────────────────────────────────────────┐ │
            │  │         Decorator Chain                  │ │
            │  │  Logging → Retry → Cache → Download     │ │
            │  └─────────────────────────────────────────┘ │
            │  ┌─────────────────────────────────────────┐ │
            │  │         Strategy: Transformer            │ │
            │  │  (resize, crop, filter)                  │ │
            │  └─────────────────────────────────────────┘ │
            │  ┌─────────────────────────────────────────┐ │
            │  │         Factory: Cache                   │ │
            │  │  (memory, disk, hybrid)                  │ │
            │  └─────────────────────────────────────────┘ │
            │  ┌─────────────────────────────────────────┐ │
            │  │         Observer: Completion             │ │
            │  │  (notify UI when ready)                  │ │
            │  └─────────────────────────────────────────┘ │
            └─────────────────────────────────────────────┘
```

## When NOT to Use Patterns

| Scenario | Why Skip Pattern |
|----------|------------------|
| Simple CRUD app | Over-engineering |
| Prototype/MVP | Slows development |
| 1-2 person team | No one else to understand |
| Unlikely to change | YAGNI (You Aren't Gonna Need It) |

## Pattern Misuse Warning Signs

```swift
// 🚩 RED FLAG: Factory that creates only one type
class UserFactory {
    static func create() -> User { ... } // Pointless factory!
}

// 🚩 RED FLAG: Singleton for data that varies
class ProductService {
    static let shared = ProductService()
    var currentProduct: Product? // Different per screen!
}

// 🚩 RED FLAG: Strategy with one implementation
protocol NetworkStrategy { }
class URLSessionStrategy: NetworkStrategy { }
// No other implementations = wasted abstraction

// 🚩 RED FLAG: Decorator that doesn't add behavior
class PassthroughDecorator: DataFetcher {
    func fetch(url: URL) async throws -> Data {
        return try await wrapped.fetch(url: url) // Does nothing!
    }
}
```

---

# 📝 Interview Preparation

## Common Interview Questions

### Question 1: "Explain Singleton and its problems"

**Answer:**
"Singleton ensures one instance exists globally. In iOS, it's good for truly shared resources like analytics or network monitoring.

Problems:
1. **Hidden dependencies** - code that uses `ServiceManager.shared` has an invisible dependency
2. **Hard to test** - you can't inject mocks easily
3. **Global state** - changes affect entire app unpredictably

**Solution:** Use Singleton for the instance, but inject it through protocols so you can mock it in tests."

---

### Question 2: "Design a pluggable analytics system"

**Answer:**
"I'd combine Factory and Adapter patterns:

1. **Protocol** defines `track(event:)`, `identify(userId:)`
2. **Adapters** for each SDK (Mixpanel, Firebase, Amplitude) implement the protocol
3. **Factory** creates the right adapter based on configuration
4. **Composite adapter** can wrap multiple adapters to send to all providers

This way, app code is decoupled from SDKs. Swapping providers or adding new ones requires no changes to tracking call sites."

---

### Question 3: "How would you add retry logic without touching existing code?"

**Answer:**
"Decorator pattern. I'd create a `RetryingService` that wraps any service conforming to a protocol.

```swift
class RetryingDataFetcher: DataFetcher {
    private let wrapped: DataFetcher
    
    func fetch(...) async throws -> Data {
        for attempt in 0..<3 {
            do {
                return try await wrapped.fetch(...)
            } catch {
                if attempt == 2 { throw error }
                await wait(seconds: pow(2, attempt))
            }
        }
    }
}
```

Benefits: original service unchanged, retry logic is testable independently, can stack with other decorators."

---

### Question 4: "When would you NOT use a design pattern?"

**Answer:**
"I'd skip patterns when:
1. **Simple problem** - patterns add complexity; if the direct solution is clear, use it
2. **Unlikely to change** - patterns prepare for variation; if there won't be variation, it's over-engineering
3. **Prototype phase** - speed matters more than architecture initially
4. **Only one implementation** - a Factory for one product or Strategy with one algorithm is pointless

I follow YAGNI - You Aren't Gonna Need It. Add abstraction when the need is real, not imagined."

---

## Quick Pattern Selection Guide

| Need | Pattern |
|------|---------|
| One instance only | Singleton |
| Create objects without knowing exact type | Factory |
| Build complex objects step by step | Builder |
| React to changes | Observer |
| Make incompatible interfaces work | Adapter |
| Add behavior without changing class | Decorator |
| Switch algorithms at runtime | Strategy |

---

## Final Checklist Before Interview

- [ ] Can explain each pattern in simple English with real-life analogy
- [ ] Can write basic implementation from scratch (no looking up syntax)
- [ ] Know when to use AND when NOT to use each pattern
- [ ] Can identify pattern misuse (over-engineering)
- [ ] Have real project examples for each pattern
- [ ] Understand trade-offs (testability, complexity, performance)
- [ ] Can combine patterns for complex scenarios

---

> **Remember:** Patterns are tools, not rules. Use them when they solve real problems, not to impress. The best code is the simplest code that does the job well. 🚀
