# LLD Problem 5: Food Ordering System (Zomato/Swiggy)

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Domain Relevance**: Amazon Fresh, Whole Foods
- **Real-time Tracking**: GPS, status updates
- **State Management**: Complex order lifecycle
- **Search & Filters**: Restaurant discovery

---

# R — Requirements

## Functional Requirements

```markdown
1. Restaurant Discovery
   - Search restaurants
   - Filter by cuisine, rating, delivery time
   - Sort by distance, popularity, ratings

2. Menu & Cart
   - Browse menu with categories
   - Add/remove items
   - Customize items (toppings, size)
   - Apply coupons

3. Order Management
   - Place order
   - Real-time order tracking
   - Order status updates
   - Cancel order (if allowed)

4. Delivery Tracking
   - Live driver location
   - Estimated arrival time
   - Driver contact
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Search latency | < 200ms | Quick discovery |
| Location updates | Every 5s | Smooth tracking |
| Cart persistence | Session | No data loss |
| Offline catalog | Yes | Browse offline |

## iOS-Specific Requirements

```markdown
- MapKit for restaurant locations
- Core Location for tracking
- Push notifications for status
- Widget for active order
- Siri shortcuts for reorder
```

## Clarifying Questions

1. **Scale**: How many restaurants? → 10,000 in a city
2. **Live tracking**: Built-in or 3rd party? → Built-in
3. **Ratings**: User reviews? → Yes, but simplified
4. **Multi-vendor**: Single order, multiple restaurants? → No

---

# E — Entities

## Core Classes

```swift
// MARK: - Restaurant & Menu

struct Restaurant: Identifiable, Codable {
    let id: String
    let name: String
    let cuisines: [String]
    let rating: Double
    let reviewCount: Int
    let deliveryTime: ClosedRange<Int>  // 25-35 min
    let minimumOrder: Decimal
    let deliveryFee: Decimal
    let isOpen: Bool
    let location: Location
    let imageUrl: String
}

struct Menu: Codable {
    let restaurantId: String
    let categories: [MenuCategory]
}

struct MenuCategory: Identifiable, Codable {
    let id: String
    let name: String
    let items: [MenuItem]
}

struct MenuItem: Identifiable, Codable {
    let id: String
    let name: String
    let description: String
    let price: Decimal
    let imageUrl: String?
    let isVeg: Bool
    let customizations: [Customization]?
    let isAvailable: Bool
}

struct Customization: Codable {
    let name: String  // "Size", "Toppings"
    let type: CustomizationType
    let options: [CustomizationOption]
    let isRequired: Bool
}

enum CustomizationType: String, Codable {
    case single   // Radio: Size (S/M/L)
    case multiple // Checkbox: Toppings
}

struct CustomizationOption: Codable {
    let name: String
    let priceModifier: Decimal  // +$2 for extra cheese
}

// MARK: - Cart

class Cart: ObservableObject {
    let restaurantId: String
    @Published var items: [CartItem] = []
    @Published var couponCode: String?
    
    var subtotal: Decimal {
        items.reduce(0) { $0 + $1.totalPrice }
    }
    
    var discount: Decimal {
        // Calculate based on coupon
        return 0
    }
    
    var total: Decimal {
        subtotal - discount
    }
}

struct CartItem: Identifiable {
    let id = UUID()
    let menuItem: MenuItem
    var quantity: Int
    var selectedCustomizations: [String: [String]]  // "Size": ["Large"]
    
    var totalPrice: Decimal {
        var price = menuItem.price * Decimal(quantity)
        // Add customization prices
        return price
    }
}

// MARK: - Order

struct Order: Identifiable, Codable {
    let id: String
    let userId: String
    let restaurantId: String
    let items: [OrderItem]
    let deliveryAddress: Address
    let paymentMethod: String
    let subtotal: Decimal
    let deliveryFee: Decimal
    let discount: Decimal
    let total: Decimal
    let status: OrderStatus
    let createdAt: Date
    let estimatedDelivery: Date?
    let driver: Driver?
}

struct OrderItem: Codable {
    let menuItemId: String
    let name: String
    let quantity: Int
    let price: Decimal
    let customizations: [String: [String]]
}

struct Driver: Codable {
    let id: String
    let name: String
    let phone: String
    let photoUrl: String
    let vehicleNumber: String
    var location: Location?
}

struct Location: Codable {
    let latitude: Double
    let longitude: Double
}
```

## Enums

```swift
enum OrderStatus: String, Codable {
    case placed
    case confirmed
    case preparing
    case readyForPickup
    case outForDelivery
    case delivered
    case cancelled
    
    var displayName: String {
        switch self {
        case .placed: return "Order Placed"
        case .confirmed: return "Confirmed"
        case .preparing: return "Preparing Food"
        case .readyForPickup: return "Ready for Pickup"
        case .outForDelivery: return "Out for Delivery"
        case .delivered: return "Delivered"
        case .cancelled: return "Cancelled"
        }
    }
    
    var canCancel: Bool {
        switch self {
        case .placed, .confirmed:
            return true
        default:
            return false
        }
    }
}
```

## Entity Relationships

```
┌─────────────┐       ┌─────────────┐
│ Restaurant  │──────▶│    Menu     │
└──────┬──────┘       └──────┬──────┘
       │ 1:N                 │ N:1
       │                     ▼
       │              ┌─────────────┐
       └─────────────▶│  MenuItem   │
                      └──────┬──────┘
                             │
                      ┌──────┴──────┐
                      ▼             ▼
               ┌──────────┐  ┌──────────┐
               │   Cart   │  │  Order   │
               └──────────┘  └────┬─────┘
                                  │ 0..1
                                  ▼
                           ┌──────────┐
                           │  Driver  │
                           └──────────┘
```

---

# S — States

## Order State Machine

```swift
enum OrderStatus: String, Codable {
    case placed
    case confirmed
    case preparing
    case readyForPickup
    case outForDelivery
    case delivered
    case cancelled
    
    func canTransition(to newStatus: OrderStatus) -> Bool {
        switch (self, newStatus) {
        case (.placed, .confirmed),
             (.placed, .cancelled),
             (.confirmed, .preparing),
             (.confirmed, .cancelled),
             (.preparing, .readyForPickup),
             (.readyForPickup, .outForDelivery),
             (.outForDelivery, .delivered):
            return true
        default:
            return false
        }
    }
    
    var progress: Double {
        switch self {
        case .placed: return 0.0
        case .confirmed: return 0.2
        case .preparing: return 0.4
        case .readyForPickup: return 0.6
        case .outForDelivery: return 0.8
        case .delivered: return 1.0
        case .cancelled: return 0.0
        }
    }
}
```

## State Diagram: Order Lifecycle

```
[Placed] ──confirm──▶ [Confirmed] ──start cooking──▶ [Preparing]
    │                      │                              │
    │ cancel               │ cancel                       │
    ▼                      ▼                              ▼
[Cancelled]           [Cancelled]                  [ReadyForPickup]
                                                          │
                                                          │ driver picks up
                                                          ▼
                                               [OutForDelivery]
                                                          │
                                                          │ arrive
                                                          ▼
                                                    [Delivered]
```

---

# H — Handling Concurrency

## Concurrency Challenges

| Challenge | Scenario | Solution |
|---------|----------|----------|
| Stale menu | Item sold out while ordering | Validate on checkout |
| Location updates | High frequency GPS | Throttle to 5s |
| Status sync | Push vs polling | Push with polling fallback |
| Cart persistence | Background kill | UserDefaults or CoreData |

## Thread-Safe Cart Manager

```swift
actor CartManager {
    private var activeCart: Cart?
    
    func getCart(for restaurantId: String) -> Cart {
        if let cart = activeCart, cart.restaurantId == restaurantId {
            return cart
        }
        
        // Clear cart if switching restaurants
        let newCart = Cart(restaurantId: restaurantId)
        activeCart = newCart
        return newCart
    }
    
    func addItem(_ item: CartItem, to restaurantId: String) throws {
        let cart = getCart(for: restaurantId)
        
        // Check if same item with same customizations exists
        if let existing = cart.items.firstIndex(where: { $0.matches(item) }) {
            cart.items[existing].quantity += item.quantity
        } else {
            cart.items.append(item)
        }
    }
    
    func removeItem(_ itemId: UUID) {
        activeCart?.items.removeAll { $0.id == itemId }
    }
    
    func clearCart() {
        activeCart = nil
    }
}
```

## Real-time Order Tracking

```swift
class OrderTracker: ObservableObject {
    @Published private(set) var orderStatus: OrderStatus = .placed
    @Published private(set) var driverLocation: Location?
    @Published private(set) var eta: Date?
    
    private var webSocketTask: URLSessionWebSocketTask?
    private var locationPollingTask: Task<Void, Never>?
    
    func startTracking(orderId: String) {
        // 1. Connect WebSocket for status updates
        connectWebSocket(orderId: orderId)
        
        // 2. Poll for driver location during delivery
        startLocationPolling(orderId: orderId)
    }
    
    private func connectWebSocket(orderId: String) {
        let url = URL(string: "wss://api.zomato.com/orders/\(orderId)/status")!
        webSocketTask = URLSession.shared.webSocketTask(with: url)
        webSocketTask?.resume()
        
        receiveMessage()
    }
    
    private func receiveMessage() {
        webSocketTask?.receive { [weak self] result in
            switch result {
            case .success(let message):
                if case .string(let text) = message {
                    self?.handleStatusUpdate(text)
                }
                self?.receiveMessage()  // Continue listening
            case .failure:
                // Fallback to polling
                self?.startPollingFallback()
            }
        }
    }
    
    private func startLocationPolling(orderId: String) {
        locationPollingTask = Task {
            while !Task.isCancelled {
                await fetchDriverLocation(orderId: orderId)
                try? await Task.sleep(nanoseconds: 5_000_000_000)  // 5 seconds
            }
        }
    }
    
    @MainActor
    private func handleStatusUpdate(_ json: String) {
        guard let data = json.data(using: .utf8),
              let update = try? JSONDecoder().decode(StatusUpdate.self, from: data) else {
            return
        }
        
        orderStatus = update.status
        eta = update.estimatedDelivery
        
        // Stop tracking if delivered/cancelled
        if orderStatus == .delivered || orderStatus == .cancelled {
            stopTracking()
        }
    }
    
    func stopTracking() {
        webSocketTask?.cancel()
        locationPollingTask?.cancel()
    }
}
```

---

# A — Architecture & Patterns

## Pattern 1: Observer (Order Status Updates)

### What it does:
Notifies UI of order status changes in real-time.

### Implementation:
```swift
import Combine

class OrderStatusPublisher: ObservableObject {
    @Published private(set) var status: OrderStatus = .placed
    @Published private(set) var timeline: [StatusEvent] = []
    
    private var cancellables = Set<AnyCancellable>()
    
    func subscribeToOrder(_ orderId: String) {
        // WebSocket or SSE updates
        OrderWebSocket.shared.statusUpdates
            .filter { $0.orderId == orderId }
            .receive(on: DispatchQueue.main)
            .sink { [weak self] update in
                self?.status = update.status
                self?.timeline.append(StatusEvent(
                    status: update.status,
                    timestamp: Date()
                ))
            }
            .store(in: &cancellables)
    }
}

struct StatusEvent: Identifiable {
    let id = UUID()
    let status: OrderStatus
    let timestamp: Date
}
```

### Why chosen:
1. **Real-time updates** from server
2. **Multiple UI components** need same status
3. **Decoupled** from order logic

---

## Pattern 2: Factory (Restaurant Types)

### What it does:
Creates different restaurant view models based on type.

### Implementation:
```swift
protocol RestaurantDisplayable {
    var displayName: String { get }
    var badgeText: String? { get }
    var sortPriority: Int { get }
}

class RegularRestaurant: RestaurantDisplayable {
    let restaurant: Restaurant
    
    var displayName: String { restaurant.name }
    var badgeText: String? { nil }
    var sortPriority: Int { 0 }
    
    init(restaurant: Restaurant) {
        self.restaurant = restaurant
    }
}

class PromotedRestaurant: RestaurantDisplayable {
    let restaurant: Restaurant
    let promotion: String
    
    var displayName: String { restaurant.name }
    var badgeText: String? { promotion }
    var sortPriority: Int { -10 }  // Show first
    
    init(restaurant: Restaurant, promotion: String) {
        self.restaurant = restaurant
        self.promotion = promotion
    }
}

class NewRestaurant: RestaurantDisplayable {
    let restaurant: Restaurant
    
    var displayName: String { restaurant.name }
    var badgeText: String? { "NEW" }
    var sortPriority: Int { -5 }
    
    init(restaurant: Restaurant) {
        self.restaurant = restaurant
    }
}

// Factory
class RestaurantFactory {
    func create(from restaurant: Restaurant, metadata: RestaurantMetadata?) -> RestaurantDisplayable {
        if let promotion = metadata?.currentPromotion {
            return PromotedRestaurant(restaurant: restaurant, promotion: promotion)
        }
        
        if let openDate = restaurant.openedAt, openDate > Date().addingTimeInterval(-30 * 24 * 3600) {
            return NewRestaurant(restaurant: restaurant)
        }
        
        return RegularRestaurant(restaurant: restaurant)
    }
}
```

### Why chosen:
1. **Different display types** (promoted, new, regular)
2. **Centralized creation** logic
3. **Easy to add** new types (trending, exclusive)

---

## Pattern 3: Strategy (Sorting & Filtering)

### What it does:
Swappable algorithms for sorting and filtering restaurants.

### Implementation:
```swift
// MARK: - Sorting Strategies

protocol SortingStrategy {
    func sort(_ restaurants: [Restaurant]) -> [Restaurant]
}

class DistanceSortingStrategy: SortingStrategy {
    let userLocation: Location
    
    init(userLocation: Location) {
        self.userLocation = userLocation
    }
    
    func sort(_ restaurants: [Restaurant]) -> [Restaurant] {
        restaurants.sorted { r1, r2 in
            distance(from: userLocation, to: r1.location) <
            distance(from: userLocation, to: r2.location)
        }
    }
}

class RatingSortingStrategy: SortingStrategy {
    func sort(_ restaurants: [Restaurant]) -> [Restaurant] {
        restaurants.sorted { $0.rating > $1.rating }
    }
}

class DeliveryTimeSortingStrategy: SortingStrategy {
    func sort(_ restaurants: [Restaurant]) -> [Restaurant] {
        restaurants.sorted { $0.deliveryTime.lowerBound < $1.deliveryTime.lowerBound }
    }
}

class PopularitySortingStrategy: SortingStrategy {
    func sort(_ restaurants: [Restaurant]) -> [Restaurant] {
        restaurants.sorted { $0.reviewCount > $1.reviewCount }
    }
}

// MARK: - Filtering Strategies

protocol FilteringStrategy {
    func filter(_ restaurants: [Restaurant]) -> [Restaurant]
}

class CuisineFilterStrategy: FilteringStrategy {
    let cuisines: Set<String>
    
    init(cuisines: Set<String>) {
        self.cuisines = cuisines
    }
    
    func filter(_ restaurants: [Restaurant]) -> [Restaurant] {
        restaurants.filter { restaurant in
            !Set(restaurant.cuisines).isDisjoint(with: cuisines)
        }
    }
}

class VegFilterStrategy: FilteringStrategy {
    func filter(_ restaurants: [Restaurant]) -> [Restaurant] {
        restaurants.filter { $0.hasVegOptions }
    }
}

class OpenNowFilterStrategy: FilteringStrategy {
    func filter(_ restaurants: [Restaurant]) -> [Restaurant] {
        restaurants.filter { $0.isOpen }
    }
}

class CompositeFilterStrategy: FilteringStrategy {
    private var filters: [FilteringStrategy] = []
    
    func add(_ filter: FilteringStrategy) {
        filters.append(filter)
    }
    
    func filter(_ restaurants: [Restaurant]) -> [Restaurant] {
        filters.reduce(restaurants) { result, filter in
            filter.filter(result)
        }
    }
}

// MARK: - Search Context

class RestaurantSearchContext {
    private var sortingStrategy: SortingStrategy
    private var filteringStrategy: FilteringStrategy
    
    init(sortingStrategy: SortingStrategy, filteringStrategy: FilteringStrategy) {
        self.sortingStrategy = sortingStrategy
        self.filteringStrategy = filteringStrategy
    }
    
    func setSorting(_ strategy: SortingStrategy) {
        self.sortingStrategy = strategy
    }
    
    func setFiltering(_ strategy: FilteringStrategy) {
        self.filteringStrategy = strategy
    }
    
    func search(_ restaurants: [Restaurant]) -> [Restaurant] {
        let filtered = filteringStrategy.filter(restaurants)
        return sortingStrategy.sort(filtered)
    }
}
```

### Why chosen:
1. **Multiple sort options** with same interface
2. **Composable filters**
3. **Easy to add** new sort/filter types

---

## ❌ Why Command Pattern is OVERKILL Here

### The Problem:

```swift
// Overusing Command pattern
protocol OrderCommand {
    func execute()
    func undo()
}

class AddToCartCommand: OrderCommand {
    func execute() { /* add item */ }
    func undo() { /* remove item */ }
}

class PlaceOrderCommand: OrderCommand {
    func execute() { /* place order */ }
    func undo() { /* cancel order? */ }
}
```

### Why Command Doesn't Fit:

| Issue | Explanation |
|-------|-------------|
| **Orders aren't undoable** | Can't un-deliver food |
| **No command history needed** | Not replaying actions |
| **Simple CRUD operations** | Add/remove are trivial |
| **Adds unnecessary complexity** | Extra classes without benefit |

### When Command WOULD Be Used:
- If we had order builder with multi-step undo
- If we needed macro recording (reorder entire cart)
- If we needed audit log of all actions

### What We Use Instead:

```swift
// Simple direct methods
class Cart: ObservableObject {
    @Published var items: [CartItem] = []
    
    func add(_ item: CartItem) {
        if let existing = items.firstIndex(where: { $0.matches(item) }) {
            items[existing].quantity += item.quantity
        } else {
            items.append(item)
        }
    }
    
    func remove(_ itemId: UUID) {
        items.removeAll { $0.id == itemId }
    }
    
    func updateQuantity(_ itemId: UUID, quantity: Int) {
        if let index = items.firstIndex(where: { $0.id == itemId }) {
            if quantity > 0 {
                items[index].quantity = quantity
            } else {
                items.remove(at: index)
            }
        }
    }
    
    func clear() {
        items.removeAll()
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Order Placement → Tracking

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  User  │ │  CartView  │ │  OrderSvc  │ │   Server   │ │OrderTracker│ │   MapView  │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │              │
    │ checkout   │              │              │              │              │
    │───────────▶│              │              │              │              │
    │            │              │              │              │              │
    │            │ placeOrder() │              │              │              │
    │            │─────────────▶│              │              │              │
    │            │              │              │              │              │
    │            │              │ POST /orders │              │              │
    │            │              │─────────────▶│              │              │
    │            │              │              │              │              │
    │            │              │ Order created│              │              │
    │            │              │◀─────────────│              │              │
    │            │              │              │              │              │
    │            │ Order        │              │              │              │
    │            │◀─────────────│              │              │              │
    │            │              │              │              │              │
    │ show tracking             │              │              │              │
    │◀───────────│              │              │              │              │
    │            │              │              │              │              │
    │═══════════════════════════════════════════════════════════════════════════
    │                              ORDER TRACKING PHASE                         
    │═══════════════════════════════════════════════════════════════════════════
    │            │              │              │              │              │
    │            │              │              │ connect WS   │              │
    │            │              │              │◀─────────────│              │
    │            │              │              │              │              │
    │            │              │              │ status:      │              │
    │            │              │              │ preparing    │              │
    │            │              │              │─────────────▶│              │
    │            │              │              │              │              │
    │ update UI  │              │              │              │              │
    │◀─────────────────────────────────────────────────────────              │
    │            │              │              │              │              │
    │            │              │              │ status:      │              │
    │            │              │              │ outForDelivery               │
    │            │              │              │─────────────▶│              │
    │            │              │              │              │              │
    │            │              │              │              │ show map     │
    │            │              │              │              │─────────────▶│
    │            │              │              │              │              │
    │            │              │              │              │              │
    │            │              │              │ driver location              │
    │            │              │              │─────────────▶│              │
    │            │              │              │              │              │
    │            │              │              │              │ update pin   │
    │            │              │              │              │─────────────▶│
    │            │              │              │              │              │
    │ see driver │              │              │              │              │
    │◀───────────────────────────────────────────────────────────────────────│
    │            │              │              │              │              │
```

## Step-by-Step Flow

### Flow: Order Placement

**Trigger:** User taps "Place Order" on checkout

**Steps:**
1. Validate cart (items available, restaurant open)
2. Validate delivery address
3. Create order with cart items, address, payment
4. Send to server: `POST /orders`
5. Server validates and confirms
6. Clear cart
7. Navigate to order tracking screen
8. Start WebSocket connection for status updates

**Error Handling:**
- Item unavailable: Show which item, offer alternatives
- Restaurant closed: Show next opening time
- Payment failed: Retry with different method

---

# E — Edge Cases

## Edge Case 1: Item Sold Out During Cart

**Scenario:** User adds item, item sells out before checkout

**Handling:**
```swift
func validateCart(cart: Cart) async throws -> CartValidationResult {
    var unavailableItems: [CartItem] = []
    var priceChangedItems: [(CartItem, Decimal)] = []
    
    let currentMenu = try await menuService.fetch(restaurantId: cart.restaurantId)
    
    for cartItem in cart.items {
        guard let menuItem = currentMenu.findItem(id: cartItem.menuItem.id) else {
            unavailableItems.append(cartItem)
            continue
        }
        
        if !menuItem.isAvailable {
            unavailableItems.append(cartItem)
        } else if menuItem.price != cartItem.menuItem.price {
            priceChangedItems.append((cartItem, menuItem.price))
        }
    }
    
    if !unavailableItems.isEmpty || !priceChangedItems.isEmpty {
        return .needsUpdate(
            unavailable: unavailableItems,
            priceChanged: priceChangedItems
        )
    }
    
    return .valid
}

enum CartValidationResult {
    case valid
    case needsUpdate(unavailable: [CartItem], priceChanged: [(CartItem, Decimal)])
}
```

## Edge Case 2: Restaurant Closes During Order

**Scenario:** Restaurant closes after order placed

**Impact:** Order cannot be fulfilled

**Handling:**
```swift
class OrderStatusHandler {
    func handleClosedRestaurant(order: Order) async {
        // Automatic refund
        try? await paymentService.refund(orderId: order.id, amount: order.total)
        
        // Update order status
        await orderService.updateStatus(order.id, status: .cancelled)
        
        // Notify user
        await notificationService.send(
            to: order.userId,
            title: "Order Cancelled",
            body: "Sorry, \(order.restaurantName) has closed. Full refund issued."
        )
    }
}
```

## Edge Case 3: Driver App Crashes During Delivery

**Scenario:** Driver location stops updating

**Handling:**
```swift
class DriverLocationMonitor {
    private var lastUpdateTime: Date?
    private let staleThreshold: TimeInterval = 60  // 1 minute
    
    func handleLocationUpdate(_ location: Location) {
        lastUpdateTime = Date()
        // ... update UI
    }
    
    func checkForStaleLocation() {
        guard let lastUpdate = lastUpdateTime else { return }
        
        if Date().timeIntervalSince(lastUpdate) > staleThreshold {
            // Show "Location updating..." or last known location
            showStaleLocationIndicator()
            
            // Try alternative: Call driver
            showCallDriverOption()
        }
    }
}
```

## Edge Case 4: Switching Restaurants with Items in Cart

**Scenario:** User has items from Restaurant A, browses Restaurant B

**Handling:**
```swift
class CartViewModel {
    func addItem(_ item: MenuItem, from restaurantId: String) {
        if let currentRestaurantId = cart.restaurantId,
           currentRestaurantId != restaurantId {
            // Show confirmation
            showCartReplaceAlert(
                currentRestaurant: currentRestaurantId,
                newRestaurant: restaurantId,
                newItem: item
            )
        } else {
            cart.add(CartItem(menuItem: item, quantity: 1))
        }
    }
    
    func showCartReplaceAlert(/* ... */) {
        // "Your cart has items from [Restaurant A]. 
        //  Adding this will clear your cart.
        //  [Cancel] [Clear & Add]"
    }
}
```

## Edge Case Checklist

```markdown
□ Item sold out during cart
□ Restaurant closes during order
□ Driver location stale
□ Payment timeout
□ App killed during checkout
□ Network loss during tracking
□ Switching restaurant with cart items
□ Coupon expired at checkout
□ Address out of delivery zone
□ Order cancelled by restaurant
```

---

# D — Design Trade-offs

## Trade-off 1: Real-time Tracking (WebSocket vs Polling)

| WebSocket | Polling |
|-----------|---------|
| Instant updates | 5s delay |
| Battery drain | More efficient |
| Complex reconnection | Simple |

**Decision:** WebSocket with polling fallback

**Rationale:** Real-time location is critical for delivery tracking. Use WebSocket during active order, fallback to polling if connection fails.

## Trade-off 2: Cart Persistence

| UserDefaults | CoreData | Server |
|--------------|----------|--------|
| Simple | Complex | Sync needed |
| Limited size | Full persistence | Network required |
| Fast | Slightly slower | Latency |

**Decision:** UserDefaults for single-device, server sync for logged-in users

**Rationale:** Most users order from one device. UserDefaults is sufficient for cart; server sync enables cross-device for logged-in users.

## Trade-off 3: Menu Caching

| No Cache | Aggressive Cache |
|----------|-----------------|
| Always fresh | May be stale |
| Slow | Fast |
| More data | Less data |

**Decision:** Cache with 15-minute TTL, validate on checkout

**Rationale:** Menu changes rarely, but validate freshness before placing order.

---

# iOS Implementation

## Complete ViewModel

```swift
@MainActor
class RestaurantListViewModel: ObservableObject {
    // State
    @Published private(set) var restaurants: [RestaurantDisplayable] = []
    @Published private(set) var isLoading = false
    @Published var searchQuery = ""
    @Published var selectedCuisines: Set<String> = []
    @Published var sortOption: SortOption = .distance
    @Published var showVegOnly = false
    
    // Dependencies
    private let restaurantService: RestaurantService
    private let restaurantFactory: RestaurantFactory
    private let userLocation: Location
    
    private var searchContext: RestaurantSearchContext
    private var cancellables = Set<AnyCancellable>()
    
    init(restaurantService: RestaurantService, userLocation: Location) {
        self.restaurantService = restaurantService
        self.userLocation = userLocation
        self.restaurantFactory = RestaurantFactory()
        self.searchContext = RestaurantSearchContext(
            sortingStrategy: DistanceSortingStrategy(userLocation: userLocation),
            filteringStrategy: OpenNowFilterStrategy()
        )
        
        setupSearchDebounce()
    }
    
    private func setupSearchDebounce() {
        $searchQuery
            .debounce(for: .milliseconds(300), scheduler: DispatchQueue.main)
            .sink { [weak self] query in
                Task { await self?.search(query: query) }
            }
            .store(in: &cancellables)
    }
    
    func loadRestaurants() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            let rawRestaurants = try await restaurantService.fetchNearby(location: userLocation)
            
            // Apply filters and sorting
            let filtered = searchContext.search(rawRestaurants)
            
            // Create display models
            restaurants = filtered.map { restaurant in
                restaurantFactory.create(from: restaurant, metadata: nil)
            }
        } catch {
            // Handle error
        }
    }
    
    func setSortOption(_ option: SortOption) {
        sortOption = option
        
        let strategy: SortingStrategy
        switch option {
        case .distance:
            strategy = DistanceSortingStrategy(userLocation: userLocation)
        case .rating:
            strategy = RatingSortingStrategy()
        case .deliveryTime:
            strategy = DeliveryTimeSortingStrategy()
        case .popularity:
            strategy = PopularitySortingStrategy()
        }
        
        searchContext.setSorting(strategy)
        Task { await loadRestaurants() }
    }
    
    func updateFilters() {
        let composite = CompositeFilterStrategy()
        composite.add(OpenNowFilterStrategy())
        
        if !selectedCuisines.isEmpty {
            composite.add(CuisineFilterStrategy(cuisines: selectedCuisines))
        }
        
        if showVegOnly {
            composite.add(VegFilterStrategy())
        }
        
        searchContext.setFiltering(composite)
        Task { await loadRestaurants() }
    }
}

enum SortOption: String, CaseIterable {
    case distance = "Distance"
    case rating = "Rating"
    case deliveryTime = "Delivery Time"
    case popularity = "Popularity"
}
```

## Order Tracking View

```swift
struct OrderTrackingView: View {
    @StateObject var viewModel: OrderTrackingViewModel
    
    var body: some View {
        VStack {
            // Status Timeline
            OrderStatusTimeline(
                currentStatus: viewModel.status,
                events: viewModel.timeline
            )
            
            // Map with driver location
            if viewModel.status == .outForDelivery {
                Map(coordinateRegion: $viewModel.region, annotationItems: viewModel.annotations) { annotation in
                    MapAnnotation(coordinate: annotation.coordinate) {
                        if annotation.isDriver {
                            DriverPin()
                        } else {
                            DeliveryPin()
                        }
                    }
                }
                .frame(height: 200)
                
                // ETA
                Text("Arriving in \(viewModel.etaMinutes) minutes")
                    .font(.headline)
                
                // Call driver
                Button("Call Driver") {
                    viewModel.callDriver()
                }
            }
            
            Spacer()
            
            // Cancel button (if allowed)
            if viewModel.canCancel {
                Button("Cancel Order") {
                    viewModel.cancelOrder()
                }
                .foregroundColor(.red)
            }
        }
        .onAppear {
            viewModel.startTracking()
        }
        .onDisappear {
            viewModel.stopTracking()
        }
    }
}

struct OrderStatusTimeline: View {
    let currentStatus: OrderStatus
    let events: [StatusEvent]
    
    var body: some View {
        VStack(alignment: .leading, spacing: 16) {
            ForEach(OrderStatus.allCases, id: \.self) { status in
                HStack {
                    Circle()
                        .fill(status <= currentStatus ? Color.green : Color.gray)
                        .frame(width: 20, height: 20)
                    
                    Text(status.displayName)
                        .foregroundColor(status <= currentStatus ? .primary : .secondary)
                    
                    Spacer()
                    
                    if let event = events.first(where: { $0.status == status }) {
                        Text(event.timestamp, style: .time)
                            .font(.caption)
                    }
                }
            }
        }
        .padding()
    }
}
```

---

# Interview Tips for This Problem

## What to Say

```markdown
1. "The key challenges are real-time tracking and search/filter..."
   - Shows understanding of core complexity

2. "I'll use Strategy pattern for sorting and filtering..."
   - Multiple interchangeable algorithms

3. "Observer pattern for status updates via Combine..."
   - Real-time UI updates

4. "Factory for different restaurant display types..."
   - Promoted, new, regular

5. "Command pattern is overkill here because..."
   - Proactively explain what you're NOT using

6. "For tracking, WebSocket with polling fallback..."
   - Shows pragmatic approach
```

## Red Flags to Avoid

```markdown
❌ "I'll poll every second for location updates"
   → Battery killer

❌ "Cart is just stored in a global variable"
   → No persistence, no thread safety

❌ "Command pattern for add/remove cart items"
   → Over-engineering simple operations

❌ Forgetting restaurant closing edge case
   → Incomplete order lifecycle thinking
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Food Ordering System problem!*
