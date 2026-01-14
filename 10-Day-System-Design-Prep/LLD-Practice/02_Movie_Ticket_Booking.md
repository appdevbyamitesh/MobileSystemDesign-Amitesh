# LLD Problem 2: Movie Ticket Booking System (BookMyShow)

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Concurrency**: Seat locking with timers
- **State Management**: Complex seat lifecycle states
- **Real-time Updates**: Multiple users viewing same show
- **Payment Integration**: Transaction with timeout

---

# R — Requirements

## Functional Requirements

```markdown
1. Movie & Theater Management
   - List movies by city
   - Show theaters with showtimes
   - Display seat layout

2. Seat Selection
   - Real-time seat availability
   - Multi-seat selection
   - 30-second lock timer
   - Visual seat map

3. Booking Flow
   - Lock seats → Payment → Confirmation
   - Release seats on timeout/cancel
   - Generate booking confirmation

4. Payment Processing
   - Multiple payment methods
   - Handle payment failures
   - Refund on cancellation
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Seat lock | 30 seconds | Prevent indefinite holds |
| Real-time updates | < 2s | Accurate availability |
| Booking latency | < 500ms | Good UX |
| Concurrent users | 1000+ per show | Popular movies |
| Offline | Partial | Cached movie list |

## iOS-Specific Requirements

```markdown
- Seat map with pinch-to-zoom
- Haptic feedback on selection
- Timer animation during lock
- Apple Pay integration
- Widget for upcoming bookings
```

## Clarifying Questions

1. **Concurrent selection**: Can 2 users select same seat? → No, first-come-first-locked
2. **Lock duration**: Fixed or configurable? → Fixed 30 seconds
3. **Partial booking**: If 4 seats selected, 1 already taken? → Fail entire selection
4. **Cancellation**: Refund policy? → Full refund before show

---

# E — Entities

## Core Classes

```swift
// MARK: - Movie & Theater

struct Movie {
    let id: String
    let title: String
    let posterUrl: String
    let duration: TimeInterval
    let genre: [Genre]
    let rating: Rating
    let releaseDate: Date
}

struct Theater {
    let id: String
    let name: String
    let address: Address
    let screens: [Screen]
}

struct Screen {
    let id: String
    let name: String  // "Screen 1", "IMAX"
    let seatLayout: SeatLayout
}

struct SeatLayout {
    let rows: Int
    let seatsPerRow: Int
    let seatMap: [[Seat]]  // 2D grid
}

// MARK: - Show & Seats

struct Show {
    let id: String
    let movieId: String
    let theaterId: String
    let screenId: String
    let startTime: Date
    let endTime: Date
    let pricing: ShowPricing
}

struct ShowPricing {
    let regular: Decimal
    let premium: Decimal
    let vip: Decimal
}

class Seat {
    let id: String
    let row: String    // "A", "B", "C"
    let number: Int    // 1, 2, 3
    let category: SeatCategory
    
    private(set) var state: SeatState = .available
    private(set) var lockedBy: String?  // userId
    private(set) var lockExpiresAt: Date?
    
    func lock(by userId: String, duration: TimeInterval = 30) throws {
        guard state == .available else {
            throw BookingError.seatUnavailable
        }
        state = .locked
        lockedBy = userId
        lockExpiresAt = Date().addingTimeInterval(duration)
    }
    
    func release() {
        state = .available
        lockedBy = nil
        lockExpiresAt = nil
    }
    
    func book() {
        state = .booked
        lockedBy = nil
        lockExpiresAt = nil
    }
}

// MARK: - Booking

struct Booking {
    let id: String
    let userId: String
    let showId: String
    let seats: [Seat]
    let totalAmount: Decimal
    let status: BookingStatus
    let paymentId: String?
    let createdAt: Date
    let confirmedAt: Date?
}
```

## Enums

```swift
enum SeatCategory: String {
    case regular
    case premium
    case vip
    case wheelchair
}

enum SeatState: String {
    case available
    case locked      // Temporarily held
    case booked      // Confirmed
    case blocked     // Admin blocked
    
    var validTransitions: [SeatState] {
        switch self {
        case .available:
            return [.locked, .blocked]
        case .locked:
            return [.available, .booked]  // timeout releases, payment books
        case .booked:
            return [.available]  // cancellation
        case .blocked:
            return [.available]
        }
    }
}

enum BookingStatus: String {
    case initiated
    case seatsLocked
    case paymentPending
    case confirmed
    case cancelled
    case expired
}

enum BookingError: Error {
    case seatUnavailable
    case lockExpired
    case paymentFailed
    case showStarted
    case invalidSelection
}
```

## Entity Relationships

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│    Movie    │◀─────▶│    Show     │◀─────▶│   Theater   │
└─────────────┘       └──────┬──────┘       └─────────────┘
                             │ 1:N
                             ▼
                      ┌─────────────┐
                      │    Seat     │
                      └──────┬──────┘
                             │ N:1
                             ▼
                      ┌─────────────┐
                      │   Booking   │
                      └──────┬──────┘
                             │ 1:1
                             ▼
                      ┌─────────────┐
                      │   Payment   │
                      └─────────────┘
```

---

# S — States

## Seat State Machine

```swift
enum SeatState: String {
    case available
    case locked
    case booked
    case blocked
    
    func canTransition(to newState: SeatState) -> Bool {
        switch (self, newState) {
        case (.available, .locked),
             (.available, .blocked),
             (.locked, .available),   // Timeout/cancel
             (.locked, .booked),      // Payment success
             (.booked, .available),   // Cancellation
             (.blocked, .available):
            return true
        default:
            return false
        }
    }
}
```

## State Diagram: Seat

```
                 ┌───────────────────────────────────────┐
                 │                                       │
                 ▼                                       │
           ┌───────────┐                                 │
           │ available │◀────────────┐                   │
           └─────┬─────┘             │                   │
                 │                   │ timeout/          │ cancel
                 │ select            │ cancel            │ booking
                 ▼                   │                   │
           ┌───────────┐             │             ┌─────┴─────┐
           │  locked   │─────────────┘             │  booked   │
           └─────┬─────┘                           └───────────┘
                 │                                       ▲
                 │ payment success                       │
                 └───────────────────────────────────────┘
```

## Booking State Machine

```swift
enum BookingStatus: String {
    case initiated
    case seatsLocked
    case paymentPending
    case confirmed
    case cancelled
    case expired
    
    func canTransition(to newState: BookingStatus) -> Bool {
        switch (self, newState) {
        case (.initiated, .seatsLocked),
             (.initiated, .cancelled),
             (.seatsLocked, .paymentPending),
             (.seatsLocked, .expired),
             (.seatsLocked, .cancelled),
             (.paymentPending, .confirmed),
             (.paymentPending, .seatsLocked),  // Retry
             (.paymentPending, .expired),
             (.confirmed, .cancelled):
            return true
        default:
            return false
        }
    }
}
```

## State Diagram: Booking

```
[initiated] ──lock──▶ [seatsLocked] ──pay──▶ [paymentPending] ──success──▶ [confirmed]
     │                      │                       │                           │
     │ cancel               │ timeout               │ timeout                   │ cancel
     │                      │                       │                           │
     ▼                      ▼                       ▼                           ▼
[cancelled]             [expired]              [expired]                  [cancelled]
                                                    │
                                                    │ retry
                                                    ▼
                                              [seatsLocked]
```

---

# H — Handling Concurrency

## Concurrency Challenges

| Challenge | Scenario | Solution |
|-----------|----------|----------|
| Simultaneous selection | Two users click same seat | Pessimistic locking |
| Lock expiry | User abandons without payment | Background timer |
| Stale UI | User sees outdated seat status | Real-time sync |
| Payment race | Lock expires during payment | Extend lock |

## Thread-Safe Seat Manager

```swift
actor SeatLockManager {
    private var locks: [String: SeatLock] = [:]  // seatId -> lock
    private var userLocks: [String: Set<String>] = [:]  // userId -> seatIds
    
    struct SeatLock {
        let userId: String
        let expiresAt: Date
    }
    
    // Lock multiple seats atomically
    func lockSeats(_ seatIds: [String], for userId: String, duration: TimeInterval = 30) throws -> Date {
        // Check all seats are available
        for seatId in seatIds {
            if let existing = locks[seatId] {
                if existing.expiresAt > Date() {
                    throw BookingError.seatUnavailable
                }
            }
        }
        
        // Lock all seats
        let expiresAt = Date().addingTimeInterval(duration)
        for seatId in seatIds {
            locks[seatId] = SeatLock(userId: userId, expiresAt: expiresAt)
        }
        
        // Track user's locks
        userLocks[userId, default: []].formUnion(seatIds)
        
        return expiresAt
    }
    
    // Release all locks for a user
    func releaseUserLocks(userId: String) {
        guard let seatIds = userLocks[userId] else { return }
        for seatId in seatIds {
            locks.removeValue(forKey: seatId)
        }
        userLocks.removeValue(forKey: userId)
    }
    
    // Book seats (convert lock to permanent)
    func confirmBooking(seatIds: [String], userId: String) throws {
        for seatId in seatIds {
            guard let lock = locks[seatId],
                  lock.userId == userId,
                  lock.expiresAt > Date() else {
                throw BookingError.lockExpired
            }
        }
        
        // Remove from locks (will be in bookings table)
        for seatId in seatIds {
            locks.removeValue(forKey: seatId)
        }
        userLocks.removeValue(forKey: userId)
    }
    
    // Cleanup expired locks
    func cleanupExpiredLocks() {
        let now = Date()
        for (seatId, lock) in locks {
            if lock.expiresAt <= now {
                locks.removeValue(forKey: seatId)
                userLocks[lock.userId]?.remove(seatId)
            }
        }
    }
}
```

## Lock Timer Implementation

```swift
class LockTimerManager: ObservableObject {
    @Published private(set) var remainingSeconds: Int = 30
    @Published private(set) var isExpired = false
    
    private var timer: Timer?
    private let expiryDate: Date
    private let onExpiry: () -> Void
    
    init(expiresAt: Date, onExpiry: @escaping () -> Void) {
        self.expiryDate = expiresAt
        self.onExpiry = onExpiry
        startTimer()
    }
    
    private func startTimer() {
        updateRemaining()
        
        timer = Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { [weak self] _ in
            self?.updateRemaining()
        }
    }
    
    private func updateRemaining() {
        let remaining = Int(expiryDate.timeIntervalSinceNow)
        
        if remaining <= 0 {
            remainingSeconds = 0
            isExpired = true
            timer?.invalidate()
            onExpiry()
        } else {
            remainingSeconds = remaining
        }
    }
    
    func extendLock(by seconds: TimeInterval) {
        // Called when extending during payment
    }
    
    deinit {
        timer?.invalidate()
    }
}
```

---

# A — Architecture & Patterns

## Pattern 1: Observer (Seat Updates)

### What it does:
Notifies all viewers of seat status changes in real-time.

### Implementation:
```swift
import Combine

class SeatAvailabilityPublisher {
    private let subject = PassthroughSubject<SeatUpdate, Never>()
    
    var publisher: AnyPublisher<SeatUpdate, Never> {
        subject.eraseToAnyPublisher()
    }
    
    func publish(_ update: SeatUpdate) {
        subject.send(update)
    }
}

struct SeatUpdate {
    let showId: String
    let seatId: String
    let newState: SeatState
    let lockedBy: String?
}

// Usage in ViewModel
class SeatSelectionViewModel: ObservableObject {
    @Published private(set) var seats: [[Seat]] = []
    
    private var cancellables = Set<AnyCancellable>()
    private let seatPublisher: SeatAvailabilityPublisher
    
    init(seatPublisher: SeatAvailabilityPublisher) {
        self.seatPublisher = seatPublisher
        
        seatPublisher.publisher
            .receive(on: DispatchQueue.main)
            .sink { [weak self] update in
                self?.handleSeatUpdate(update)
            }
            .store(in: &cancellables)
    }
    
    private func handleSeatUpdate(_ update: SeatUpdate) {
        // Find and update seat in 2D array
        for (rowIndex, row) in seats.enumerated() {
            for (colIndex, seat) in row.enumerated() {
                if seat.id == update.seatId {
                    seats[rowIndex][colIndex].state = update.newState
                }
            }
        }
    }
}
```

### Why chosen:
1. **Multiple viewers** need same updates
2. **Decoupled** - seat manager doesn't know about UI
3. **Real-time** - instant propagation

### Why alternatives rejected:

| Alternative | Why Rejected |
|-------------|--------------|
| Polling | Wastes resources, stale data |
| Delegate | Only 1:1, not 1:N |
| NotificationCenter | Less type-safe than Combine |

---

## Pattern 2: State Machine (Seat States)

### What it does:
Enforces valid state transitions, prevents impossible states.

### Implementation:
```swift
class SeatStateMachine {
    private(set) var state: SeatState = .available
    
    func transition(to newState: SeatState, context: TransitionContext) throws {
        guard state.canTransition(to: newState) else {
            throw BookingError.invalidTransition(from: state, to: newState)
        }
        
        // Validate transition conditions
        switch (state, newState) {
        case (.available, .locked):
            guard context.userId != nil else {
                throw BookingError.missingUserId
            }
        case (.locked, .booked):
            guard context.paymentId != nil else {
                throw BookingError.missingPayment
            }
        default:
            break
        }
        
        state = newState
    }
}

struct TransitionContext {
    let userId: String?
    let paymentId: String?
}
```

### Why chosen:
1. **Prevents invalid states** like booked→locked
2. **Self-documenting** transitions
3. **Testable** state logic

---

## Pattern 3: Strategy (Payment Methods)

### What it does:
Allows swapping payment processors without changing booking logic.

### Implementation:
```swift
protocol PaymentStrategy {
    var displayName: String { get }
    var icon: String { get }
    func processPayment(amount: Decimal, metadata: PaymentMetadata) async throws -> PaymentResult
}

struct PaymentResult {
    let transactionId: String
    let status: PaymentStatus
    let receiptUrl: String?
}

// Concrete Strategy: Card Payment
class CardPaymentStrategy: PaymentStrategy {
    let displayName = "Credit/Debit Card"
    let icon = "creditcard"
    
    func processPayment(amount: Decimal, metadata: PaymentMetadata) async throws -> PaymentResult {
        // Stripe/Razorpay integration
        let response = try await paymentGateway.chargeCard(amount: amount, card: metadata.cardDetails)
        return PaymentResult(transactionId: response.id, status: .completed, receiptUrl: response.receipt)
    }
}

// Concrete Strategy: Apple Pay
class ApplePayStrategy: PaymentStrategy {
    let displayName = "Apple Pay"
    let icon = "applelogo"
    
    func processPayment(amount: Decimal, metadata: PaymentMetadata) async throws -> PaymentResult {
        let request = PKPaymentRequest()
        request.paymentSummaryItems = [
            PKPaymentSummaryItem(label: "Movie Tickets", amount: NSDecimalNumber(decimal: amount))
        ]
        
        // Present Apple Pay sheet
        let result = try await presentApplePay(request)
        return PaymentResult(transactionId: result.transactionId, status: .completed, receiptUrl: nil)
    }
}

// Concrete Strategy: UPI
class UPIPaymentStrategy: PaymentStrategy {
    let displayName = "UPI"
    let icon = "indianrupeesign.circle"
    
    func processPayment(amount: Decimal, metadata: PaymentMetadata) async throws -> PaymentResult {
        // Open UPI app via deep link
        let upiUrl = "upi://pay?pa=\(merchantId)&am=\(amount)"
        // Wait for callback
        return try await waitForUPICallback()
    }
}

// Context: Payment Coordinator
class PaymentCoordinator {
    private var strategy: PaymentStrategy
    
    init(strategy: PaymentStrategy) {
        self.strategy = strategy
    }
    
    func setStrategy(_ strategy: PaymentStrategy) {
        self.strategy = strategy
    }
    
    func pay(amount: Decimal, metadata: PaymentMetadata) async throws -> PaymentResult {
        try await strategy.processPayment(amount: amount, metadata: metadata)
    }
}
```

### Why chosen:
1. **Multiple payment methods** with different flows
2. **Easy to add** new methods (Wallet, NetBanking)
3. **Testable** with mock strategies

---

## ❌ Why Singleton is DANGEROUS Here

### The Problem:

```swift
// BAD: Singleton for booking manager
class BookingManager {
    static let shared = BookingManager()
    
    var currentBooking: Booking?  // Shared state!
    var selectedSeats: [Seat] = []  // Shared state!
}

// User A: Starts booking for Show 1
BookingManager.shared.selectedSeats = [seatA1, seatA2]

// User A switches to User B's profile on shared device
// User B: Starts booking for Show 2
BookingManager.shared.selectedSeats = [seatB1]  // Lost User A's selection!
```

### Why Singleton Fails:

| Issue | Impact |
|-------|--------|
| Shared state across flows | Data leaks between users |
| Hard to test | Can't isolate test cases |
| Lifecycle issues | State persists incorrectly |
| Concurrent bookings | Only one at a time |

### Correct Approach: Scoped Instances

```swift
// GOOD: Scoped booking session
class BookingSession {
    let id: UUID = UUID()
    let showId: String
    let userId: String
    
    private(set) var selectedSeats: [Seat] = []
    private(set) var status: BookingStatus = .initiated
    
    init(showId: String, userId: String) {
        self.showId = showId
        self.userId = userId
    }
}

// Each booking flow creates its own session
class BookingCoordinator {
    func startNewBooking(showId: String, userId: String) -> BookingSession {
        return BookingSession(showId: showId, userId: userId)
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Seat Selection → Lock → Payment → Confirmation

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  User  │ │  SeatMap   │ │ LockManager│ │ ViewModel  │ │ PaymentSvc │ │ BookingSvc │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │              │
    │ tap seats  │              │              │              │              │
    │───────────▶│              │              │              │              │
    │            │              │              │              │              │
    │            │ selectSeats([A1, A2])       │              │              │
    │            │─────────────────────────────▶              │              │
    │            │              │              │              │              │
    │            │              │ lockSeats()  │              │              │
    │            │              │◀─────────────│              │              │
    │            │              │              │              │              │
    │            │              │──┐           │              │              │
    │            │              │  │ check     │              │              │
    │            │              │  │ available │              │              │
    │            │              │◀─┘           │              │              │
    │            │              │              │              │              │
    │            │              │ expiresAt    │              │              │
    │            │              │─────────────▶│              │              │
    │            │              │              │              │              │
    │            │ show timer (30s)            │              │              │
    │◀───────────│◀─────────────────────────────              │              │
    │            │              │              │              │              │
    │══════════════════════════════════════════════════════════════════════════════
    │                              TIMER COUNTING DOWN (30 seconds)                 
    │══════════════════════════════════════════════════════════════════════════════
    │            │              │              │              │              │
    │ proceed    │              │              │              │              │
    │───────────────────────────────────────────▶             │              │
    │            │              │              │              │              │
    │            │              │              │ pay(amount)  │              │
    │            │              │              │─────────────▶│              │
    │            │              │              │              │              │
    │            │              │              │ extendLock() │              │
    │            │              │◀─────────────│              │              │
    │            │              │              │              │              │
    │            │              │              │ result       │              │
    │            │              │              │◀─────────────│              │
    │            │              │              │              │              │
    │            │              │              │ confirmBooking()            │
    │            │              │              │─────────────────────────────▶
    │            │              │              │              │              │
    │            │              │ markBooked() │              │              │
    │            │              │◀─────────────│              │              │
    │            │              │              │              │              │
    │            │              │              │ broadcast(seatUpdate)       │
    │            │◀─────────────────────────────────────────────────────────│
    │            │              │              │              │              │
    │ confirmation              │              │              │              │
    │◀───────────│              │              │              │              │
    │            │              │              │              │              │
```

## Flow: Lock Timeout Handling

```
┌────────┐ ┌────────────┐ ┌────────────┐
│  User  │ │   Timer    │ │ LockManager│
└───┬────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │
    │            │ tick (every 1s)
    │            │─────────────▶│
    │            │              │ check expiry
    │            │              │──┐
    │            │              │◀─┘
    │            │              │
    │══════════════════════════════════════
    │         30 seconds elapse             
    │══════════════════════════════════════
    │            │              │
    │            │ expired!     │
    │            │─────────────▶│
    │            │              │
    │            │              │ releaseSeats()
    │            │              │──┐
    │            │              │◀─┘
    │            │              │
    │            │ broadcast(available)
    │◀───────────│◀─────────────│
    │            │              │
    │ show expired alert        │
    │◀───────────│              │
```

---

# E — Edge Cases

## Edge Case 1: Seat Selected by Another User While Viewing

**Scenario:** User A is viewing seat map, User B locks a seat

**Impact:** User A sees outdated availability

**Handling:**
```swift
class SeatSelectionViewModel: ObservableObject {
    @Published private(set) var seats: [[SeatViewModel]] = []
    
    private var refreshTask: Task<Void, Never>?
    
    func startAutoRefresh() {
        refreshTask = Task {
            while !Task.isCancelled {
                await refreshSeatStatus()
                try? await Task.sleep(nanoseconds: 2_000_000_000)  // 2 seconds
            }
        }
    }
    
    func handleSeatSelection(_ seatId: String) async throws {
        // Optimistic update
        updateSeatUI(seatId, state: .locked)
        
        do {
            try await seatLockManager.lockSeats([seatId], for: userId)
        } catch {
            // Rollback on failure
            updateSeatUI(seatId, state: .available)
            throw error
        }
    }
}
```

## Edge Case 2: Payment Fails After Lock

**Scenario:** Lock succeeds, payment fails, user retries

**Impact:** Seats might expire during retry

**Handling:**
```swift
class PaymentCoordinator {
    func processWithRetry(booking: BookingSession, maxRetries: Int = 3) async throws -> PaymentResult {
        var lastError: Error?
        
        for attempt in 1...maxRetries {
            do {
                // Extend lock before payment attempt
                try await seatLockManager.extendLock(
                    seatIds: booking.selectedSeats.map { $0.id },
                    duration: 30
                )
                
                return try await paymentService.process(booking: booking)
            } catch PaymentError.declined {
                throw PaymentError.declined  // Don't retry declined cards
            } catch {
                lastError = error
                
                if attempt < maxRetries {
                    try await Task.sleep(nanoseconds: 2_000_000_000)  // 2s backoff
                }
            }
        }
        
        throw lastError!
    }
}
```

## Edge Case 3: App Backgrounded During Lock

**Scenario:** User switches to another app while timer running

**Impact:** Timer might fire incorrectly, or lock expires

**Handling:**
```swift
class BookingViewController: UIViewController {
    private var backgroundTime: Date?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(appDidEnterBackground),
            name: UIApplication.didEnterBackgroundNotification,
            object: nil
        )
        
        NotificationCenter.default.addObserver(
            self,
            selector: #selector(appWillEnterForeground),
            name: UIApplication.willEnterForegroundNotification,
            object: nil
        )
    }
    
    @objc private func appDidEnterBackground() {
        backgroundTime = Date()
    }
    
    @objc private func appWillEnterForeground() {
        guard let backgroundTime = backgroundTime else { return }
        
        let elapsed = Date().timeIntervalSince(backgroundTime)
        
        if timerManager.remainingSeconds - Int(elapsed) <= 0 {
            // Lock has expired while backgrounded
            handleLockExpired()
        } else {
            // Update timer
            timerManager.adjustForBackgroundTime(elapsed)
        }
    }
}
```

## Edge Case 4: Network Loss During Seat Lock

**Scenario:** User selects seat, network fails before lock confirmed

**Handling:**
```swift
func selectSeats(_ seats: [Seat]) async {
    // Optimistic UI update
    for seat in seats {
        seat.state = .locking  // Intermediate state
    }
    
    do {
        try await withTimeout(seconds: 5) {
            try await seatLockManager.lockSeats(seats.map { $0.id }, for: userId)
        }
        
        // Confirm lock
        for seat in seats {
            seat.state = .locked
        }
    } catch {
        // Rollback
        for seat in seats {
            seat.state = .available
        }
        
        showError("Could not lock seats. Please try again.")
    }
}
```

## Edge Case Checklist

```markdown
□ Concurrent selection of same seat
□ Lock expiry during payment
□ Network timeout during lock
□ App backgrounded with active timer
□ Payment declined
□ Partial lock failure (3 of 4 seats locked)
□ Show started while user is booking
□ Device rotation during seat selection
□ Memory warning during seat map render
□ Multiple booking tabs open
```

---

# D — Design Trade-offs

## Trade-off 1: Optimistic vs Pessimistic Locking

| Optimistic | Pessimistic |
|------------|-------------|
| Better UX (instant) | Guaranteed consistency |
| May fail on confirm | Lock before showing |
| Good for low contention | Good for high demand |

**Decision:** Pessimistic locking with short timeout (30s)

**Rationale:** Movie tickets are high-demand; false availability is worse than slight delay.

## Trade-off 2: Timer on Client vs Server

| Client Timer | Server Timer |
|--------------|--------------|
| Responsive UI | Accurate |
| Can drift | Requires sync |
| Works offline | Network dependent |

**Decision:** Both - client for UI, server for enforcement

**Rationale:** Client shows countdown, but server is authoritative and releases locks.

## Trade-off 3: WebSocket vs Polling for Updates

| WebSocket | Polling |
|-----------|---------|
| Real-time | Simple |
| Complex connection | More requests |
| Battery drain | Predictable load |

**Decision:** Short polling (2s) during seat selection only

**Rationale:** Only need real-time during active selection. WebSocket complexity not justified.

---

# iOS Implementation

## Complete ViewModel

```swift
@MainActor
class SeatSelectionViewModel: ObservableObject {
    // State
    @Published private(set) var seatMap: [[SeatViewModel]] = []
    @Published private(set) var selectedSeats: [SeatViewModel] = []
    @Published private(set) var timerSeconds: Int = 0
    @Published private(set) var totalAmount: Decimal = 0
    @Published private(set) var isLocked = false
    @Published private(set) var error: BookingError?
    
    // Dependencies
    private let show: Show
    private let seatLockManager: SeatLockManager
    private let pricingService: PricingService
    private var timerManager: LockTimerManager?
    private var cancellables = Set<AnyCancellable>()
    
    init(show: Show, seatLockManager: SeatLockManager, pricingService: PricingService) {
        self.show = show
        self.seatLockManager = seatLockManager
        self.pricingService = pricingService
    }
    
    // MARK: - Seat Selection
    
    func toggleSeat(_ seat: SeatViewModel) {
        guard seat.state == .available else { return }
        
        if selectedSeats.contains(where: { $0.id == seat.id }) {
            selectedSeats.removeAll { $0.id == seat.id }
        } else {
            selectedSeats.append(seat)
        }
        
        calculateTotal()
    }
    
    private func calculateTotal() {
        totalAmount = selectedSeats.reduce(0) { total, seat in
            total + pricingService.price(for: seat.category, show: show)
        }
    }
    
    // MARK: - Lock & Payment
    
    func lockSeats() async {
        guard !selectedSeats.isEmpty else { return }
        
        do {
            let expiresAt = try await seatLockManager.lockSeats(
                selectedSeats.map { $0.id },
                for: AuthManager.shared.userId
            )
            
            isLocked = true
            startTimer(expiresAt: expiresAt)
        } catch {
            self.error = error as? BookingError
        }
    }
    
    private func startTimer(expiresAt: Date) {
        timerManager = LockTimerManager(expiresAt: expiresAt) { [weak self] in
            self?.handleLockExpired()
        }
        
        timerManager?.$remainingSeconds
            .assign(to: &$timerSeconds)
    }
    
    private func handleLockExpired() {
        isLocked = false
        selectedSeats = []
        error = .lockExpired
    }
    
    func proceedToPayment() -> PaymentRequest {
        return PaymentRequest(
            amount: totalAmount,
            showId: show.id,
            seatIds: selectedSeats.map { $0.id }
        )
    }
}
```

## SwiftUI Seat Map View

```swift
struct SeatMapView: View {
    @StateObject var viewModel: SeatSelectionViewModel
    
    var body: some View {
        VStack {
            // Timer bar when locked
            if viewModel.isLocked {
                TimerBar(seconds: viewModel.timerSeconds)
                    .transition(.move(edge: .top))
            }
            
            // Seat grid
            ScrollView([.horizontal, .vertical]) {
                VStack(spacing: 8) {
                    ForEach(viewModel.seatMap, id: \.self) { row in
                        HStack(spacing: 8) {
                            ForEach(row) { seat in
                                SeatView(seat: seat, isSelected: viewModel.selectedSeats.contains { $0.id == seat.id })
                                    .onTapGesture {
                                        viewModel.toggleSeat(seat)
                                    }
                            }
                        }
                    }
                }
                .padding()
            }
            
            // Screen indicator
            ScreenIndicator()
            
            Spacer()
            
            // Bottom bar
            if !viewModel.selectedSeats.isEmpty {
                BottomBar(
                    seatCount: viewModel.selectedSeats.count,
                    total: viewModel.totalAmount,
                    onProceed: {
                        Task { await viewModel.lockSeats() }
                    }
                )
            }
        }
        .alert(item: $viewModel.error) { error in
            Alert(title: Text("Error"), message: Text(error.localizedDescription))
        }
    }
}

struct SeatView: View {
    let seat: SeatViewModel
    let isSelected: Bool
    
    var body: some View {
        RoundedRectangle(cornerRadius: 4)
            .fill(fillColor)
            .frame(width: 30, height: 30)
            .overlay {
                Text(seat.label)
                    .font(.caption2)
            }
    }
    
    var fillColor: Color {
        switch seat.state {
        case .available:
            return isSelected ? .green : .white
        case .locked:
            return .yellow
        case .booked:
            return .gray
        case .blocked:
            return .black
        }
    }
}
```

---

# Interview Tips for This Problem

## What to Say

```markdown
1. "The key challenge here is seat locking with concurrent users..."
   - Immediately show you understand the core problem

2. "I'll use a state machine for seats to prevent invalid transitions..."
   - Shows structured thinking

3. "Singleton would be dangerous here because..."
   - Volunteer this before asked

4. "For real-time updates, I'm choosing polling over WebSocket because..."
   - Show trade-off analysis

5. "When the app is backgrounded, we need to handle timer drift..."
   - Shows iOS-specific awareness
```

## Red Flags to Avoid

```markdown
❌ "I'll just disable the seat button after selection"
   → Doesn't handle concurrent access

❌ "The timer runs on the server"
   → Shows no iOS understanding

❌ "I'll use Singleton for the booking manager"
   → Major antipattern here

❌ Not mentioning lock extension during payment
   → Incomplete edge case thinking
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Movie Ticket Booking problem!*
