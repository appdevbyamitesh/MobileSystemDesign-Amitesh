# LLD Problem 1: Design a Parking Lot System

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Object Modeling**: Multiple entity types with clear relationships
- **Design Patterns**: Factory, Strategy, and Singleton usage
- **Business Logic**: Fee calculation algorithms
- **Scalability**: Real-time spot availability

---

# R — Requirements

## Functional Requirements

```markdown
1. Parking Lot Management
   - Support multiple floors
   - Different spot types (Bike, Car, Truck)
   - Entry/Exit gates with ticketing

2. Vehicle Parking
   - Assign nearest available spot
   - Support different vehicle types
   - Issue ticket on entry

3. Fee Calculation
   - Time-based pricing
   - Different rates for vehicle types
   - Support hourly/daily rates

4. Exit Processing
   - Calculate fee based on duration
   - Process payment
   - Free up the spot
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Spot lookup | < 100ms | Quick entry experience |
| Real-time updates | Yes | Show live availability |
| Offline support | Partial | Cached spot data |
| Thread safety | Required | Concurrent entry/exit |
| Testability | High | CI/CD pipeline |

## iOS-Specific Requirements

```markdown
- Map integration for navigation to spot
- Push notifications for parking expiry
- Widget for quick spot check
- Apple Pay for payment
```

## Clarifying Questions

1. **Scale**: How many spots? → 500 spots across 5 floors
2. **Payment**: Multiple payment methods? → Yes (Card, UPI, Cash)
3. **Reservation**: Can users reserve spots? → Out of scope for now
4. **Real-time**: How to update availability? → Polling every 30s

---

# E — Entities

## Core Classes

```swift
// MARK: - Core Entities

/// Singleton for the parking lot
final class ParkingLot {
    static let shared = ParkingLot()
    
    private(set) var floors: [Floor] = []
    private(set) var entryGates: [Gate] = []
    private(set) var exitGates: [Gate] = []
    
    private init() {}
    
    func addFloor(_ floor: Floor) {
        floors.append(floor)
    }
}

/// Floor in the parking lot
class Floor {
    let id: String
    let level: Int
    private(set) var spots: [ParkingSpot]
    
    init(id: String, level: Int, spots: [ParkingSpot]) {
        self.id = id
        self.level = level
        self.spots = spots
    }
    
    func getAvailableSpots(for vehicleType: VehicleType) -> [ParkingSpot] {
        spots.filter { $0.isAvailable && $0.canFit(vehicleType) }
    }
}

/// Individual parking spot
class ParkingSpot {
    let id: String
    let spotType: SpotType
    let floorLevel: Int
    let spotNumber: String
    
    private(set) var isAvailable: Bool = true
    private(set) var parkedVehicle: Vehicle?
    
    func canFit(_ vehicleType: VehicleType) -> Bool {
        switch (spotType, vehicleType) {
        case (.bike, .bike):
            return true
        case (.compact, .bike), (.compact, .car):
            return true
        case (.regular, .bike), (.regular, .car):
            return true
        case (.large, _):
            return true
        default:
            return false
        }
    }
    
    func park(_ vehicle: Vehicle) throws {
        guard isAvailable else { throw ParkingError.spotOccupied }
        guard canFit(vehicle.type) else { throw ParkingError.vehicleTooLarge }
        
        parkedVehicle = vehicle
        isAvailable = false
    }
    
    func vacate() -> Vehicle? {
        let vehicle = parkedVehicle
        parkedVehicle = nil
        isAvailable = true
        return vehicle
    }
}

/// Vehicle information
struct Vehicle {
    let licensePlate: String
    let type: VehicleType
    let color: String?
}

/// Ticket issued on entry
struct ParkingTicket {
    let id: String
    let vehicle: Vehicle
    let spot: ParkingSpot
    let entryTime: Date
    var exitTime: Date?
    var fee: Decimal?
    var status: TicketStatus
}
```

## Enums and Types

```swift
enum VehicleType: String, CaseIterable {
    case bike
    case car
    case truck
}

enum SpotType: String {
    case bike       // Only bikes
    case compact    // Small cars
    case regular    // Regular cars
    case large      // Trucks, SUVs
}

enum TicketStatus {
    case active
    case paid
    case lost
}

enum ParkingError: Error {
    case noAvailableSpot
    case spotOccupied
    case vehicleTooLarge
    case ticketNotFound
    case paymentFailed
}
```

## Entity Relationships

```
┌─────────────────┐
│   ParkingLot    │ (Singleton)
│   (1 instance)  │
└────────┬────────┘
         │ 1:N
         ▼
┌─────────────────┐
│     Floor       │
│ (multiple)      │
└────────┬────────┘
         │ 1:N
         ▼
┌─────────────────┐       ┌─────────────────┐
│  ParkingSpot    │◀─────▶│    Vehicle      │
│ (many per floor)│ 0..1  │ (when parked)   │
└────────┬────────┘       └─────────────────┘
         │
         │ 1:1
         ▼
┌─────────────────┐
│  ParkingTicket  │
│ (active ticket) │
└─────────────────┘
```

---

# S — States

## Parking Spot States

```swift
enum SpotState {
    case available
    case occupied
    case reserved      // Future: for reservations
    case maintenance
    
    var validTransitions: [SpotState] {
        switch self {
        case .available:
            return [.occupied, .reserved, .maintenance]
        case .occupied:
            return [.available]
        case .reserved:
            return [.occupied, .available]
        case .maintenance:
            return [.available]
        }
    }
}
```

## State Diagram: Parking Spot

```
                    ┌─────────────┐
                    │ maintenance │
                    └──────┬──────┘
                           │ fixed
                           ▼
┌───────────┐  park   ┌───────────┐
│ available │────────▶│ occupied  │
└───────────┘         └─────┬─────┘
      ▲                     │
      │      vacate         │
      └─────────────────────┘
```

## Ticket States

```swift
enum TicketState {
    case issued
    case active
    case paymentPending
    case paid
    case exited
    
    var validTransitions: [TicketState] {
        switch self {
        case .issued:
            return [.active]
        case .active:
            return [.paymentPending]
        case .paymentPending:
            return [.paid, .active]  // Can fail and retry
        case .paid:
            return [.exited]
        case .exited:
            return []  // Terminal state
        }
    }
}
```

## State Diagram: Ticket Lifecycle

```
[issued] ──scan──▶ [active] ──exit request──▶ [paymentPending]
                                                     │
                                     ┌───────────────┼───────────────┐
                                     │ success       │ failure       │
                                     ▼               ▼               │
                                  [paid] ──gate──▶ [exited]     [active]
```

---

# H — Handling Concurrency

## Concurrency Challenges

| Challenge | Example | Solution |
|-----------|---------|----------|
| Double booking | Two cars claim same spot | Actor isolation |
| Stale data | UI shows wrong availability | Reactive updates |
| Race on exit | Payment and gate conflict | Transaction-like handling |

## Thread-Safe Spot Manager

```swift
actor SpotManager {
    private var spots: [String: ParkingSpot] = [:]
    private var availableSpots: [SpotType: Set<String>] = [:]
    
    func registerSpot(_ spot: ParkingSpot) {
        spots[spot.id] = spot
        if spot.isAvailable {
            availableSpots[spot.spotType, default: []].insert(spot.id)
        }
    }
    
    func findAndReserveSpot(for vehicleType: VehicleType) -> ParkingSpot? {
        // Find compatible spot type
        let compatibleTypes = SpotType.allCases.filter { spotType in
            canFit(vehicleType: vehicleType, in: spotType)
        }
        
        // Try to get available spot
        for spotType in compatibleTypes {
            if let spotId = availableSpots[spotType]?.first,
               let spot = spots[spotId] {
                // Atomically reserve
                availableSpots[spotType]?.remove(spotId)
                return spot
            }
        }
        
        return nil
    }
    
    func releaseSpot(_ spotId: String) {
        guard let spot = spots[spotId] else { return }
        availableSpots[spot.spotType, default: []].insert(spotId)
    }
    
    private func canFit(vehicleType: VehicleType, in spotType: SpotType) -> Bool {
        // Size compatibility logic
        switch (vehicleType, spotType) {
        case (.bike, _): return true
        case (.car, .compact), (.car, .regular), (.car, .large): return true
        case (.truck, .large): return true
        default: return false
        }
    }
}
```

## Why Actor over Other Options

| Option | Pros | Cons | Decision |
|--------|------|------|----------|
| **Actor** ✓ | Built-in safety, clean syntax | iOS 15+ | ✅ Chosen |
| DispatchQueue | Works on older iOS | Boilerplate | ❌ |
| NSLock | Fine-grained control | Error-prone | ❌ |

**Rationale**: Actor provides compile-time safety for mutable state. Since parking lot is new development, we can target iOS 15+.

---

# A — Architecture & Patterns

## Pattern 1: Singleton (ParkingLot)

### What it does:
Ensures only one ParkingLot instance exists in the app.

### Implementation:
```swift
final class ParkingLot {
    static let shared = ParkingLot()
    
    private init() {}
    
    // Configuration
    private(set) var config: ParkingLotConfig?
    
    func configure(with config: ParkingLotConfig) {
        self.config = config
    }
}
```

### Why chosen:
1. **Single source of truth** for parking lot state
2. **Global access** from any part of app
3. **Lazy initialization** - created only when needed

### Why alternatives rejected:

| Alternative | Why Rejected |
|-------------|--------------|
| Dependency Injection | Overkill for single parking lot |
| Global variable | No control over initialization |
| Service Locator | More complexity without benefit |

### ⚠️ Singleton Caution for Testing:
```swift
// Make testable with protocol
protocol ParkingLotProtocol {
    var floors: [Floor] { get }
    func findSpot(for vehicle: VehicleType) async -> ParkingSpot?
}

extension ParkingLot: ParkingLotProtocol {}

// In tests
class MockParkingLot: ParkingLotProtocol {
    var floors: [Floor] = []
    func findSpot(for vehicle: VehicleType) async -> ParkingSpot? {
        return mockSpot
    }
}
```

---

## Pattern 2: Factory (Spot Creation)

### What it does:
Creates different types of parking spots without exposing creation logic.

### Implementation:
```swift
protocol ParkingSpotFactory {
    func createSpot(type: SpotType, floor: Int, number: String) -> ParkingSpot
}

class ConcreteParkingSpotFactory: ParkingSpotFactory {
    private var spotCounter = 0
    
    func createSpot(type: SpotType, floor: Int, number: String) -> ParkingSpot {
        spotCounter += 1
        let id = "SPOT-\(floor)-\(spotCounter)"
        
        return ParkingSpot(
            id: id,
            spotType: type,
            floorLevel: floor,
            spotNumber: number
        )
    }
}

// Floor creation using factory
class FloorBuilder {
    private let spotFactory: ParkingSpotFactory
    
    init(spotFactory: ParkingSpotFactory) {
        self.spotFactory = spotFactory
    }
    
    func buildFloor(level: Int, bikeSpots: Int, carSpots: Int, truckSpots: Int) -> Floor {
        var spots: [ParkingSpot] = []
        
        // Create bike spots
        for i in 1...bikeSpots {
            spots.append(spotFactory.createSpot(type: .bike, floor: level, number: "B\(i)"))
        }
        
        // Create car spots
        for i in 1...carSpots {
            spots.append(spotFactory.createSpot(type: .regular, floor: level, number: "C\(i)"))
        }
        
        // Create truck spots
        for i in 1...truckSpots {
            spots.append(spotFactory.createSpot(type: .large, floor: level, number: "T\(i)"))
        }
        
        return Floor(id: "FLOOR-\(level)", level: level, spots: spots)
    }
}
```

### Why chosen:
1. **Encapsulates** spot creation logic
2. **Consistent** ID generation
3. **Easy to extend** new spot types

### Why alternatives rejected:

| Alternative | Why Rejected |
|-------------|--------------|
| Direct instantiation | Duplicated logic, inconsistent IDs |
| Builder alone | Factory better for type-based creation |
| Abstract Factory | Only one factory type needed |

---

## Pattern 3: Strategy (Fee Calculation)

### What it does:
Allows swapping fee calculation algorithms without changing client code.

### Implementation:
```swift
// Strategy Protocol
protocol FeeCalculationStrategy {
    func calculateFee(entryTime: Date, exitTime: Date, vehicleType: VehicleType) -> Decimal
}

// Concrete Strategy 1: Hourly Rate
class HourlyFeeStrategy: FeeCalculationStrategy {
    private let rates: [VehicleType: Decimal] = [
        .bike: 10,
        .car: 20,
        .truck: 30
    ]
    
    func calculateFee(entryTime: Date, exitTime: Date, vehicleType: VehicleType) -> Decimal {
        let hours = ceil(exitTime.timeIntervalSince(entryTime) / 3600)
        let rate = rates[vehicleType] ?? 20
        return Decimal(hours) * rate
    }
}

// Concrete Strategy 2: Time Slot Based
class SlotBasedFeeStrategy: FeeCalculationStrategy {
    func calculateFee(entryTime: Date, exitTime: Date, vehicleType: VehicleType) -> Decimal {
        let duration = exitTime.timeIntervalSince(entryTime)
        let hours = duration / 3600
        
        var fee: Decimal = 0
        let baseRate: Decimal = vehicleType == .bike ? 10 : (vehicleType == .car ? 20 : 30)
        
        switch hours {
        case ..<1:
            fee = baseRate  // First hour flat
        case 1..<4:
            fee = baseRate + Decimal(hours - 1) * (baseRate * 0.5)  // 50% for 1-4 hours
        case 4..<8:
            fee = baseRate * 3  // 4-8 hours = 3x base
        default:
            fee = baseRate * 5  // Full day
        }
        
        return fee
    }
}

// Concrete Strategy 3: Weekend Premium
class WeekendFeeStrategy: FeeCalculationStrategy {
    private let baseStrategy: FeeCalculationStrategy
    private let weekendMultiplier: Decimal = 1.5
    
    init(baseStrategy: FeeCalculationStrategy) {
        self.baseStrategy = baseStrategy
    }
    
    func calculateFee(entryTime: Date, exitTime: Date, vehicleType: VehicleType) -> Decimal {
        let baseFee = baseStrategy.calculateFee(entryTime: entryTime, exitTime: exitTime, vehicleType: vehicleType)
        
        if isWeekend(entryTime) {
            return baseFee * weekendMultiplier
        }
        return baseFee
    }
    
    private func isWeekend(_ date: Date) -> Bool {
        let weekday = Calendar.current.component(.weekday, from: date)
        return weekday == 1 || weekday == 7
    }
}

// Context: Fee Calculator
class FeeCalculator {
    private var strategy: FeeCalculationStrategy
    
    init(strategy: FeeCalculationStrategy = HourlyFeeStrategy()) {
        self.strategy = strategy
    }
    
    func setStrategy(_ strategy: FeeCalculationStrategy) {
        self.strategy = strategy
    }
    
    func calculate(ticket: ParkingTicket) -> Decimal {
        let exitTime = ticket.exitTime ?? Date()
        return strategy.calculateFee(
            entryTime: ticket.entryTime,
            exitTime: exitTime,
            vehicleType: ticket.vehicle.type
        )
    }
}
```

### Why chosen:
1. **Multiple algorithms** for fee calculation
2. **Runtime switching** (weekday vs weekend)
3. **Easy to add** new pricing models

### Why alternatives rejected:

| Alternative | Why Rejected |
|-------------|--------------|
| if-else chain | Not extensible, violates OCP |
| Template Method | Fee calc doesn't have fixed steps |
| Simple function | Can't swap at runtime |

---

## Why Other Patterns Are NOT Suitable

### ❌ Observer Pattern

**Why not used:**
- Parking spot updates are **pull-based** (refresh button)
- No need for push notifications to multiple subscribers
- Would add complexity without benefit

**When it WOULD be used:**
- If we had real-time dashboard showing live updates
- If multiple apps needed spot notifications

### ❌ Command Pattern

**Why not used:**
- Parking operations are **not undoable** (can't un-park)
- No need for command queueing
- Operations are simple request-response

**When it WOULD be used:**
- If we needed undo/redo for reservations
- If we needed command history/audit log

### ❌ Decorator Pattern

**Why not used:**
- Fee strategies are **mutually exclusive**, not stackable
- No need to add behavior dynamically

**When it WOULD be used:**
- If we needed to stack: base fee + weekend + holiday + loyalty discount

---

# D — Data Flow

## Sequence Diagram: Vehicle Entry → Parking → Exit → Payment

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│ Driver │ │ Entry Gate │ │ SpotManager│ │ TicketSvc  │ │ FeeCalc    │ │ PaymentSvc │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │              │
    │ arrive     │              │              │              │              │
    │───────────▶│              │              │              │              │
    │            │              │              │              │              │
    │            │ findSpot(car)│              │              │              │
    │            │─────────────▶│              │              │              │
    │            │              │              │              │              │
    │            │ Spot F2-C15  │              │              │              │
    │            │◀─────────────│              │              │              │
    │            │              │              │              │              │
    │            │ reserveSpot()│              │              │              │
    │            │─────────────▶│              │              │              │
    │            │              │              │              │              │
    │            │ createTicket()              │              │              │
    │            │────────────────────────────▶│              │              │
    │            │              │              │              │              │
    │ Ticket #123│              │              │              │              │
    │◀───────────│              │              │              │              │
    │            │              │              │              │              │
    │══════════════════════════════════════════════════════════════════════════════
    │                           PARKING DURATION (2 hours)                         
    │══════════════════════════════════════════════════════════════════════════════
    │            │              │              │              │              │
    │ exit       │              │              │              │              │
    │──────────────────────────────────────────▶              │              │
    │            │              │              │              │              │
    │            │              │              │ calculateFee │              │
    │            │              │              │─────────────▶│              │
    │            │              │              │              │              │
    │            │              │              │    ₹40       │              │
    │            │              │              │◀─────────────│              │
    │            │              │              │              │              │
    │ Show ₹40   │              │              │              │              │
    │◀────────────────────────────────────────│              │              │
    │            │              │              │              │              │
    │ pay (card) │              │              │              │              │
    │─────────────────────────────────────────────────────────────────────▶│
    │            │              │              │              │              │
    │            │              │              │              │   success    │
    │◀─────────────────────────────────────────────────────────────────────│
    │            │              │              │              │              │
    │            │              │ releaseSpot()│              │              │
    │            │              │◀─────────────│              │              │
    │            │              │              │              │              │
    │ Gate opens │              │              │              │              │
    │◀───────────│              │              │              │              │
    │            │              │              │              │              │
```

## Step-by-Step Data Flow

### Flow: Vehicle Entry

**Trigger:** Driver arrives at entry gate

**Steps:**
1. Driver scans/enters vehicle details
2. Entry Gate calls `SpotManager.findSpot(vehicleType)`
3. SpotManager atomically finds and reserves nearest spot
4. TicketService creates ticket with entry time
5. Ticket printed/sent to driver's app
6. Gate opens

**Error Handling:**
- No available spot: Show "Parking Full", suggest nearby lots
- System error: Manual ticketing as fallback

### Flow: Vehicle Exit

**Trigger:** Driver arrives at exit gate

**Steps:**
1. Driver scans ticket
2. TicketService retrieves ticket, sets exit time
3. FeeCalculator calculates fee using current strategy
4. PaymentService processes payment
5. On success: SpotManager releases spot
6. Gate opens

**Error Handling:**
- Payment failed: Offer retry or alternate payment
- Lost ticket: Manual lookup by license plate + flat fee

---

# E — Edge Cases

## Edge Case 1: Double Booking Race Condition

**Scenario:** Two cars at different entrances claim same spot simultaneously

**Impact:** Both get same spot, one can't park

**Detection:** Actor isolation prevents this at compile time

**Handling:**
```swift
actor SpotManager {
    // Only one caller can execute at a time
    func findAndReserveSpot(for vehicleType: VehicleType) -> ParkingSpot? {
        // Atomic find + reserve
        guard let spot = findAvailableSpot(for: vehicleType) else {
            return nil
        }
        
        // Immediately mark as taken
        markSpotAsOccupied(spot.id)
        return spot
    }
}
```

## Edge Case 2: App Crash During Payment

**Scenario:** User's app crashes while payment is processing

**Impact:** Payment may complete but spot not released

**Handling:**
```swift
class PaymentCoordinator {
    func processPayment(ticket: ParkingTicket) async throws {
        // 1. Mark ticket as "payment_initiated"
        await ticketService.updateStatus(ticket.id, status: .paymentInitiated)
        
        // 2. Process payment
        let result = try await paymentService.charge(ticket.fee)
        
        // 3. On success, update atomically
        await ticketService.completePayment(ticket.id, transactionId: result.id)
        await spotManager.releaseSpot(ticket.spot.id)
    }
    
    // Background job to reconcile
    func reconcileStuckPayments() async {
        let stuckTickets = await ticketService.findStuck(olderThan: .minutes(10))
        for ticket in stuckTickets {
            let paymentStatus = await paymentService.checkStatus(ticket.id)
            if paymentStatus == .completed {
                await spotManager.releaseSpot(ticket.spot.id)
            }
        }
    }
}
```

## Edge Case 3: Lost Ticket

**Scenario:** Driver loses ticket, cannot prove entry time

**Handling:**
```swift
class TicketService {
    func handleLostTicket(licensePlate: String) async throws -> RecoveredTicket {
        // Try to find by license plate
        if let ticket = await findByLicensePlate(licensePlate) {
            return RecoveredTicket(original: ticket, fee: calculateFee(ticket))
        }
        
        // Fallback: Flat fee for lost ticket
        return RecoveredTicket(
            original: nil,
            fee: config.lostTicketFee  // e.g., ₹500
        )
    }
}
```

## Edge Case 4: Memory Warning on Spot List

**Scenario:** Low memory while loading all spots for map

**Handling:**
```swift
class SpotMapViewController: UIViewController {
    private var spots: [ParkingSpot] = []
    
    override func didReceiveMemoryWarning() {
        super.didReceiveMemoryWarning()
        
        // Keep only visible floor's spots
        let currentFloor = mapView.currentFloor
        spots = spots.filter { $0.floorLevel == currentFloor }
        
        // Clear image cache
        ImageCache.shared.clearMemory()
    }
}
```

## Edge Case Checklist

```markdown
□ Concurrent spot reservation
□ Payment failure mid-transaction
□ Network timeout during entry
□ Lost/damaged ticket
□ Power outage (gate stuck)
□ Vehicle mismatch (car in bike spot)
□ Overstay beyond max duration
□ Memory warning while loading spots
□ App killed during navigation
□ Invalid spot state (available but occupied)
```

---

# D — Design Trade-offs

## Trade-off 1: Singleton vs Dependency Injection

| Singleton | Dependency Injection |
|-----------|---------------------|
| Simple access | Better testability |
| Global state | Explicit dependencies |
| Thread-safe by design | More boilerplate |

**Decision:** Singleton with protocol extraction for testability

**Rationale:** Only one parking lot exists, but we extract protocol for mocking in tests.

## Trade-off 2: Real-time Updates vs Polling

| Real-time (WebSocket) | Polling |
|----------------------|---------|
| Instant updates | 30s delay acceptable |
| Battery drain | Battery efficient |
| Complex connection management | Simple implementation |

**Decision:** Polling every 30 seconds

**Rationale:** Parking availability doesn't change rapidly enough to justify WebSocket complexity and battery impact.

## Trade-off 3: In-Memory vs Persistent State

| In-Memory | Core Data/SQLite |
|-----------|------------------|
| Fast access | Survives app kill |
| Lost on crash | Slower queries |
| Simple | Complex setup |

**Decision:** In-memory with server sync

**Rationale:** Server is source of truth. Local state is for UI responsiveness only.

---

# iOS Implementation

## Complete ViewModel

```swift
@MainActor
class ParkingLotViewModel: ObservableObject {
    // Published state
    @Published private(set) var floors: [Floor] = []
    @Published private(set) var currentTicket: ParkingTicket?
    @Published private(set) var isLoading = false
    @Published private(set) var error: ParkingError?
    
    // Dependencies
    private let spotManager: SpotManager
    private let ticketService: TicketService
    private let feeCalculator: FeeCalculator
    private let paymentService: PaymentService
    
    init(
        spotManager: SpotManager,
        ticketService: TicketService,
        feeCalculator: FeeCalculator,
        paymentService: PaymentService
    ) {
        self.spotManager = spotManager
        self.ticketService = ticketService
        self.feeCalculator = feeCalculator
        self.paymentService = paymentService
    }
    
    // MARK: - Entry Flow
    
    func enterParking(vehicle: Vehicle) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            // Find and reserve spot
            guard let spot = await spotManager.findAndReserveSpot(for: vehicle.type) else {
                throw ParkingError.noAvailableSpot
            }
            
            // Create ticket
            let ticket = try await ticketService.createTicket(vehicle: vehicle, spot: spot)
            currentTicket = ticket
            
            // Navigate to spot
            await navigateToSpot(spot)
        } catch {
            self.error = error as? ParkingError ?? .unknown
        }
    }
    
    // MARK: - Exit Flow
    
    func exitParking() async {
        guard let ticket = currentTicket else { return }
        isLoading = true
        defer { isLoading = false }
        
        do {
            // Calculate fee
            let fee = feeCalculator.calculate(ticket: ticket)
            
            // Process payment
            try await paymentService.charge(amount: fee)
            
            // Release spot
            await spotManager.releaseSpot(ticket.spot.id)
            
            // Clear ticket
            currentTicket = nil
        } catch {
            self.error = error as? ParkingError ?? .paymentFailed
        }
    }
    
    private func navigateToSpot(_ spot: ParkingSpot) async {
        // Integrate with MapKit for navigation
    }
}
```

## SwiftUI View

```swift
struct ParkingLotView: View {
    @StateObject private var viewModel: ParkingLotViewModel
    @State private var showingVehicleInput = false
    
    var body: some View {
        NavigationStack {
            VStack {
                if let ticket = viewModel.currentTicket {
                    ActiveTicketView(ticket: ticket) {
                        Task { await viewModel.exitParking() }
                    }
                } else {
                    AvailabilityGridView(floors: viewModel.floors)
                    
                    Button("Enter Parking") {
                        showingVehicleInput = true
                    }
                    .buttonStyle(.borderedProminent)
                }
            }
            .overlay {
                if viewModel.isLoading {
                    ProgressView()
                }
            }
            .alert(item: $viewModel.error) { error in
                Alert(title: Text("Error"), message: Text(error.localizedDescription))
            }
            .sheet(isPresented: $showingVehicleInput) {
                VehicleInputView { vehicle in
                    Task { await viewModel.enterParking(vehicle: vehicle) }
                }
            }
        }
    }
}
```

---

# Interview Tips for This Problem

## What to Say

```markdown
1. "Let me start by clarifying the requirements..."
   - Ask about scale, payment methods, real-time needs

2. "For entities, I see ParkingLot, Floor, Spot, Vehicle, Ticket..."
   - Draw relationships

3. "I'll use Singleton for ParkingLot because there's only one lot..."
   - Mention testability concern and protocol extraction

4. "Factory pattern for spot creation gives us consistent IDs..."
   - Show the factory implementation

5. "For fee calculation, Strategy pattern allows swapping algorithms..."
   - Give weekday vs weekend example

6. "Thread safety is critical here. I'll use Actor..."
   - Explain why Actor over DispatchQueue
```

## Red Flags to Avoid

```markdown
❌ "I'll just use a global dictionary for spots"
   → Shows no understanding of concurrency

❌ "Singleton is bad, I'll inject everything"
   → Over-engineering for simple case

❌ "Observer pattern for spot updates"
   → Overkill when polling suffices

❌ Forgetting lost ticket edge case
   → Shows incomplete thinking
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Parking Lot problem!*
