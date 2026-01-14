# LLD Problem 4: Expense Sharing App (Splitwise)

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Business Logic**: Split algorithms (equal, exact, percentage)
- **Graph Algorithms**: Debt simplification
- **Real-time Updates**: Balance synchronization
- **Financial Accuracy**: Decimal precision handling

---

# R — Requirements

## Functional Requirements

```markdown
1. User & Group Management
   - Create/join groups
   - Add friends
   - Group member management

2. Expense Management
   - Add expenses (single/group)
   - Multiple split types (equal, exact, percentage)
   - Multi-payer support
   - Expense editing/deletion

3. Balance Tracking
   - Real-time balance updates
   - Who owes whom
   - Settle up functionality

4. Transaction History
   - Full expense history
   - Settlement history
   - Export functionality
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Balance accuracy | Exact cents | Financial data |
| Sync latency | < 2s | Real-time feel |
| Offline support | Full | Trip scenarios |
| Calculation time | < 100ms | Instant feedback |

## iOS-Specific Requirements

```markdown
- Widget showing balances
- Quick expense entry
- Receipt scanning (Vision)
- Haptic on settlement
- Shared expense via AirDrop
```

## Clarifying Questions

1. **Currency**: Multiple currencies? → Single for now (simplify)
2. **Simplification**: Auto-minimize transactions? → Yes
3. **Recurring**: Recurring expenses? → Out of scope
4. **Sync**: Real-time or manual? → Near real-time

---

# E — Entities

## Core Classes

```swift
// MARK: - User & Group

struct User: Identifiable, Codable {
    let id: String
    let name: String
    let email: String
    let phone: String?
    let avatarUrl: String?
}

struct Group: Identifiable, Codable {
    let id: String
    let name: String
    let members: [String]  // User IDs
    let createdAt: Date
    var expenses: [String]  // Expense IDs
    var imageUrl: String?
}

// MARK: - Expense

struct Expense: Identifiable, Codable {
    let id: String
    let groupId: String?
    let description: String
    let totalAmount: Decimal
    let currency: String
    let paidBy: [Payment]       // Who paid
    let splitAmong: [Split]     // Who owes
    let createdBy: String
    let createdAt: Date
    var receiptUrl: String?
    var notes: String?
}

struct Payment: Codable {
    let userId: String
    let amount: Decimal
}

struct Split: Codable {
    let userId: String
    let amount: Decimal
    let isPaid: Bool
}

// MARK: - Balance

struct Balance: Codable {
    let fromUserId: String
    let toUserId: String
    let amount: Decimal
    
    var isSettled: Bool { amount == 0 }
}

// MARK: - Settlement

struct Settlement: Identifiable, Codable {
    let id: String
    let fromUserId: String
    let toUserId: String
    let amount: Decimal
    let groupId: String?
    let settledAt: Date
    let notes: String?
}
```

## Split Types

```swift
enum SplitType {
    case equal
    case exact(amounts: [String: Decimal])  // userId -> amount
    case percentage(percentages: [String: Decimal])  // userId -> percentage
    case shares(shares: [String: Int])  // userId -> share count
}

protocol SplitStrategy {
    func calculateSplits(total: Decimal, among users: [String]) -> [Split]
}
```

## Entity Relationships

```
┌─────────────┐       ┌─────────────┐
│    User     │◀─────▶│    Group    │
└──────┬──────┘       └──────┬──────┘
       │ N:M                 │ 1:N
       │                     ▼
       │              ┌─────────────┐
       └─────────────▶│   Expense   │
                      └──────┬──────┘
                             │ 1:N
                      ┌──────┴──────┐
                      ▼             ▼
               ┌──────────┐  ┌──────────┐
               │ Payment  │  │  Split   │
               └──────────┘  └──────────┘
                             
       ┌─────────────┐       ┌─────────────┐
       │   Balance   │       │ Settlement  │
       └─────────────┘       └─────────────┘
```

---

# S — States

## Expense States

```swift
enum ExpenseState {
    case draft
    case active
    case partiallySettled
    case fullySettled
    case deleted
    
    func canTransition(to newState: ExpenseState) -> Bool {
        switch (self, newState) {
        case (.draft, .active),
             (.draft, .deleted),
             (.active, .partiallySettled),
             (.active, .fullySettled),
             (.active, .deleted),
             (.partiallySettled, .fullySettled),
             (.partiallySettled, .active):  // Undo partial settlement
            return true
        default:
            return false
        }
    }
}
```

## State Diagram: Expense Lifecycle

```
[Draft] ──save──▶ [Active] ──partial pay──▶ [PartiallySettled]
   │                  │                            │
   │ discard          │ settle all                 │ settle rest
   ▼                  ▼                            ▼
[Deleted]        [FullySettled] ◀─────────────────┘
```

---

# H — Handling Concurrency

## Concurrency Challenges

| Challenge | Scenario | Solution |
|---------|----------|----------|
| Stale balances | User adds expense, another settles | Sync on action |
| Race on settlement | Both users settle simultaneously | Idempotent settlement |
| Offline conflicts | Two offline edits | Last-write-wins |

## Thread-Safe Balance Manager

```swift
actor BalanceManager {
    private var balances: [String: [String: Decimal]] = [:]  // from -> to -> amount
    
    // Get balance between two users
    func balance(from: String, to: String) -> Decimal {
        return balances[from]?[to] ?? 0
    }
    
    // Get all balances for a user
    func allBalances(for userId: String) -> [Balance] {
        var result: [Balance] = []
        
        // Amounts user owes
        if let owes = balances[userId] {
            for (toUser, amount) in owes {
                result.append(Balance(fromUserId: userId, toUserId: toUser, amount: amount))
            }
        }
        
        // Amounts owed to user
        for (fromUser, toUsers) in balances {
            if let amount = toUsers[userId], amount > 0 {
                result.append(Balance(fromUserId: fromUser, toUserId: userId, amount: amount))
            }
        }
        
        return result
    }
    
    // Add expense affects balances
    func addExpense(_ expense: Expense) {
        // For each payer
        for payment in expense.paidBy {
            // For each person who owes
            for split in expense.splitAmong where split.userId != payment.userId {
                let portionPaid = payment.amount / expense.totalAmount
                let amountOwed = split.amount * portionPaid
                
                // Add to balance: split.userId owes payment.userId
                add(amount: amountOwed, from: split.userId, to: payment.userId)
            }
        }
    }
    
    // Settle reduces balance
    func settle(from: String, to: String, amount: Decimal) throws {
        let currentBalance = balance(from: from, to: to)
        guard currentBalance >= amount else {
            throw SplitError.insufficientBalance
        }
        
        subtract(amount: amount, from: from, to: to)
    }
    
    private func add(amount: Decimal, from: String, to: String) {
        if balances[from] == nil {
            balances[from] = [:]
        }
        balances[from]![to, default: 0] += amount
    }
    
    private func subtract(amount: Decimal, from: String, to: String) {
        if var fromBalances = balances[from] {
            fromBalances[to, default: 0] -= amount
            if fromBalances[to] == 0 {
                fromBalances.removeValue(forKey: to)
            }
            balances[from] = fromBalances
        }
    }
}
```

## Balance Publisher for Real-time Updates

```swift
import Combine

class BalancePublisher: ObservableObject {
    @Published private(set) var balances: [Balance] = []
    
    private let balanceManager: BalanceManager
    private var syncTask: Task<Void, Never>?
    
    init(balanceManager: BalanceManager) {
        self.balanceManager = balanceManager
        startSync()
    }
    
    func startSync() {
        syncTask = Task {
            // Poll for updates every 5 seconds
            while !Task.isCancelled {
                await refreshBalances()
                try? await Task.sleep(nanoseconds: 5_000_000_000)
            }
        }
    }
    
    @MainActor
    private func refreshBalances() async {
        let currentUserId = AuthManager.shared.userId
        balances = await balanceManager.allBalances(for: currentUserId)
    }
}
```

---

# A — Architecture & Patterns

## Pattern 1: Strategy (Split Algorithms)

### What it does:
Encapsulates different ways to split an expense.

### Implementation:
```swift
protocol SplitStrategy {
    func calculateSplits(total: Decimal, among userIds: [String], context: SplitContext) -> [Split]
}

struct SplitContext {
    var exactAmounts: [String: Decimal]?
    var percentages: [String: Decimal]?
    var shares: [String: Int]?
}

// Equal Split
class EqualSplitStrategy: SplitStrategy {
    func calculateSplits(total: Decimal, among userIds: [String], context: SplitContext) -> [Split] {
        let count = Decimal(userIds.count)
        let perPerson = total / count
        
        // Handle rounding: last person gets remainder
        var splits: [Split] = []
        var runningTotal: Decimal = 0
        
        for (index, userId) in userIds.enumerated() {
            let amount: Decimal
            if index == userIds.count - 1 {
                amount = total - runningTotal  // Handles rounding
            } else {
                amount = perPerson.rounded(scale: 2)
            }
            runningTotal += amount
            splits.append(Split(userId: userId, amount: amount, isPaid: false))
        }
        
        return splits
    }
}

// Exact Amount Split
class ExactSplitStrategy: SplitStrategy {
    func calculateSplits(total: Decimal, among userIds: [String], context: SplitContext) -> [Split] {
        guard let exactAmounts = context.exactAmounts else {
            fatalError("Exact amounts required")
        }
        
        // Validate total matches
        let sum = exactAmounts.values.reduce(0, +)
        guard sum == total else {
            // Return error or adjust
            return []
        }
        
        return userIds.map { userId in
            Split(userId: userId, amount: exactAmounts[userId] ?? 0, isPaid: false)
        }
    }
}

// Percentage Split
class PercentageSplitStrategy: SplitStrategy {
    func calculateSplits(total: Decimal, among userIds: [String], context: SplitContext) -> [Split] {
        guard let percentages = context.percentages else {
            fatalError("Percentages required")
        }
        
        // Validate percentages sum to 100
        let sum = percentages.values.reduce(0, +)
        guard sum == 100 else {
            return []
        }
        
        var splits: [Split] = []
        var runningTotal: Decimal = 0
        
        for (index, userId) in userIds.enumerated() {
            let percentage = percentages[userId] ?? 0
            let amount: Decimal
            
            if index == userIds.count - 1 {
                amount = total - runningTotal
            } else {
                amount = (total * percentage / 100).rounded(scale: 2)
            }
            
            runningTotal += amount
            splits.append(Split(userId: userId, amount: amount, isPaid: false))
        }
        
        return splits
    }
}

// Share-based Split (e.g., 2 shares for parent, 1 for child)
class ShareSplitStrategy: SplitStrategy {
    func calculateSplits(total: Decimal, among userIds: [String], context: SplitContext) -> [Split] {
        guard let shares = context.shares else {
            fatalError("Shares required")
        }
        
        let totalShares = Decimal(shares.values.reduce(0, +))
        let perShare = total / totalShares
        
        return userIds.map { userId in
            let userShares = Decimal(shares[userId] ?? 1)
            let amount = (perShare * userShares).rounded(scale: 2)
            return Split(userId: userId, amount: amount, isPaid: false)
        }
    }
}

// Split Calculator using Strategy
class SplitCalculator {
    private var strategy: SplitStrategy
    
    init(strategy: SplitStrategy = EqualSplitStrategy()) {
        self.strategy = strategy
    }
    
    func setStrategy(_ strategy: SplitStrategy) {
        self.strategy = strategy
    }
    
    func calculate(total: Decimal, among userIds: [String], context: SplitContext = SplitContext()) -> [Split] {
        strategy.calculateSplits(total: total, among: userIds, context: context)
    }
}
```

### Why chosen:
1. **Multiple algorithms** that are interchangeable
2. **Clean extension** for new split types
3. **Testable** in isolation

---

## Pattern 2: Observer (Balance Updates)

### What it does:
Notifies UI when balances change.

### Implementation:
```swift
import Combine

// Observable Balance Store
class BalanceStore: ObservableObject {
    @Published private(set) var userBalances: [UserBalance] = []
    @Published private(set) var simplified: [Settlement] = []
    
    private let balanceManager: BalanceManager
    private var cancellables = Set<AnyCancellable>()
    
    init(balanceManager: BalanceManager) {
        self.balanceManager = balanceManager
    }
    
    func subscribeToUpdates(for userId: String) {
        // Using Combine for reactive updates
        NotificationCenter.default
            .publisher(for: .expenseAdded)
            .sink { [weak self] _ in
                Task { await self?.refreshBalances(for: userId) }
            }
            .store(in: &cancellables)
        
        NotificationCenter.default
            .publisher(for: .settlementMade)
            .sink { [weak self] _ in
                Task { await self?.refreshBalances(for: userId) }
            }
            .store(in: &cancellables)
    }
    
    @MainActor
    func refreshBalances(for userId: String) async {
        let balances = await balanceManager.allBalances(for: userId)
        
        // Aggregate balances per user
        var perUser: [String: Decimal] = [:]
        for balance in balances {
            if balance.fromUserId == userId {
                perUser[balance.toUserId, default: 0] -= balance.amount  // I owe them
            } else {
                perUser[balance.fromUserId, default: 0] += balance.amount  // They owe me
            }
        }
        
        userBalances = perUser.map { UserBalance(userId: $0.key, netAmount: $0.value) }
        simplified = simplifyDebts(balances: balances)
    }
    
    private func simplifyDebts(balances: [Balance]) -> [Settlement] {
        // Debt simplification algorithm
        // Reduce number of transactions needed
        return DebtSimplifier.simplify(balances)
    }
}

struct UserBalance: Identifiable {
    let id = UUID()
    let userId: String
    let netAmount: Decimal  // Positive = they owe me, Negative = I owe them
    
    var description: String {
        if netAmount > 0 {
            return "owes you \(netAmount.formatted(.currency(code: "USD")))"
        } else if netAmount < 0 {
            return "you owe \((-netAmount).formatted(.currency(code: "USD")))"
        } else {
            return "settled up"
        }
    }
}
```

### Why chosen:
1. **Real-time UI updates** when expenses added
2. **Decoupled** from expense logic
3. **Multiple subscribers** possible

---

## ❌ Why Factory Pattern is UNNECESSARY Here

### The Problem:

```swift
// Overuse of Factory
protocol EntityFactory {
    func createExpense(...) -> Expense
    func createGroup(...) -> Group
    func createSettlement(...) -> Settlement
}

class ConcreteEntityFactory: EntityFactory {
    func createExpense(...) -> Expense {
        return Expense(...)  // Just a simple init
    }
}
```

### Why Factory Doesn't Fit:

| Issue | Explanation |
|-------|-------------|
| **No polymorphism** | All expenses are of same type |
| **No complex creation** | Init is straightforward |
| **Adds indirection** | Extra layer without benefit |
| **Not swappable** | No need to swap expense types |

### When Factory WOULD Be Used:
- If we had `GroupExpense`, `IndividualExpense`, `RecurringExpense` as subclasses
- If creation required complex validation or network calls

### What We Use Instead:

```swift
// Simple static factory methods on the type itself
extension Expense {
    static func create(
        description: String,
        amount: Decimal,
        paidBy: User,
        splitAmong: [User],
        splitType: SplitType
    ) -> Expense {
        let calculator = SplitCalculator(strategy: splitType.strategy)
        let splits = calculator.calculate(
            total: amount,
            among: splitAmong.map { $0.id }
        )
        
        return Expense(
            id: UUID().uuidString,
            groupId: nil,
            description: description,
            totalAmount: amount,
            currency: "USD",
            paidBy: [Payment(userId: paidBy.id, amount: amount)],
            splitAmong: splits,
            createdBy: paidBy.id,
            createdAt: Date()
        )
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Adding Expense → Recalculating Balances

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  User  │ │ ViewModel  │ │ SplitCalc  │ │ BalanceMgr │ │BalanceStore│
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │
    │ add expense│              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ enter details│              │              │
    │            │──┐           │              │              │
    │            │◀─┘           │              │              │
    │            │              │              │              │
    │ tap save   │              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ calculate splits             │              │
    │            │─────────────▶│              │              │
    │            │              │              │              │
    │            │              │──┐           │              │
    │            │              │  │ apply     │              │
    │            │              │  │ strategy  │              │
    │            │              │◀─┘           │              │
    │            │              │              │              │
    │            │ [Splits]     │              │              │
    │            │◀─────────────│              │              │
    │            │              │              │              │
    │            │ create Expense               │              │
    │            │──┐           │              │              │
    │            │◀─┘           │              │              │
    │            │              │              │              │
    │            │ addExpense   │              │              │
    │            │────────────────────────────▶│              │
    │            │              │              │              │
    │            │              │              │ update       │
    │            │              │              │ balances     │
    │            │              │              │──┐           │
    │            │              │              │◀─┘           │
    │            │              │              │              │
    │            │              │              │ notify       │
    │            │              │              │─────────────▶│
    │            │              │              │              │
    │            │              │              │              │ refresh
    │            │              │              │              │──┐
    │            │              │              │              │◀─┘
    │            │              │              │              │
    │            │◀────────────────────────────────────────────│ @Published
    │            │              │              │              │
    │ show updated              │              │              │
    │ balances   │              │              │              │
    │◀───────────│              │              │              │
    │            │              │              │              │
```

## Step-by-Step Flow

### Flow: Add Expense

**Trigger:** User fills expense form and taps save

**Steps:**
1. User enters description, amount, payer, participants
2. User selects split type (equal, exact, percentage)
3. SplitCalculator uses appropriate Strategy
4. Strategy returns Split array with amounts
5. Create Expense object with splits
6. BalanceManager updates all affected balances
7. BalanceStore publishes update via @Published
8. UI refreshes showing new balances

**Error Handling:**
- Decimal precision: Round to 2 decimal places
- Sum mismatch: Adjust last split to match total

---

# E — Edge Cases

## Edge Case 1: Rounding Errors in Equal Split

**Scenario:** $100 split among 3 people = $33.33... each

**Impact:** Pennies are lost or gained

**Handling:**
```swift
func calculateEqualSplit(total: Decimal, count: Int) -> [Decimal] {
    let perPerson = (total / Decimal(count)).rounded(scale: 2, mode: .down)
    var amounts = Array(repeating: perPerson, count: count)
    
    // Give remainder to last person
    let distributed = perPerson * Decimal(count)
    let remainder = total - distributed
    amounts[count - 1] += remainder
    
    return amounts
}

// $100 / 3:
// [0] = $33.33
// [1] = $33.33
// [2] = $33.34  (gets the extra cent)
```

## Edge Case 2: Circular Debts

**Scenario:** A owes B $10, B owes C $10, C owes A $10

**Impact:** Net zero but appears as $30 debt

**Handling:**
```swift
class DebtSimplifier {
    static func simplify(_ balances: [Balance]) -> [Settlement] {
        // Calculate net balances
        var netBalances: [String: Decimal] = [:]
        
        for balance in balances {
            netBalances[balance.fromUserId, default: 0] -= balance.amount
            netBalances[balance.toUserId, default: 0] += balance.amount
        }
        
        // Separate creditors (positive) and debtors (negative)
        var creditors: [(String, Decimal)] = []
        var debtors: [(String, Decimal)] = []
        
        for (userId, amount) in netBalances {
            if amount > 0 {
                creditors.append((userId, amount))
            } else if amount < 0 {
                debtors.append((userId, -amount))
            }
        }
        
        // Match debtors to creditors
        var settlements: [Settlement] = []
        var i = 0, j = 0
        
        while i < debtors.count && j < creditors.count {
            let debtor = debtors[i]
            let creditor = creditors[j]
            
            let amount = min(debtor.1, creditor.1)
            settlements.append(Settlement(
                id: UUID().uuidString,
                fromUserId: debtor.0,
                toUserId: creditor.0,
                amount: amount,
                groupId: nil,
                settledAt: Date(),
                notes: nil
            ))
            
            debtors[i].1 -= amount
            creditors[j].1 -= amount
            
            if debtors[i].1 == 0 { i += 1 }
            if creditors[j].1 == 0 { j += 1 }
        }
        
        return settlements
    }
}

// Circular debt example:
// Before: A→B $10, B→C $10, C→A $10
// Net: A=0, B=0, C=0
// After simplification: No settlements needed!
```

## Edge Case 3: Settlement Mid-Expense

**Scenario:** User settles while expense being added

**Handling:**
```swift
actor BalanceManager {
    // Use transaction-like approach
    func addExpenseAndSettle(expense: Expense, settlement: Settlement) {
        // Both operations happen atomically
        addExpense(expense)
        
        do {
            try settle(from: settlement.fromUserId, to: settlement.toUserId, amount: settlement.amount)
        } catch {
            // Rollback expense if settlement fails
            removeExpense(expense)
            throw error
        }
    }
}
```

## Edge Case 4: Floating Point Currency Issues

**Scenario:** $19.99 + $0.01 ≠ $20.00 with Float

**Handling:**
```swift
// ALWAYS use Decimal for money
struct Money {
    let amount: Decimal
    let currency: String
    
    init(_ amount: Decimal, currency: String = "USD") {
        self.amount = amount.rounded(scale: 2)
        self.currency = currency
    }
    
    static func + (lhs: Money, rhs: Money) -> Money {
        precondition(lhs.currency == rhs.currency)
        return Money(lhs.amount + rhs.amount, currency: lhs.currency)
    }
}

extension Decimal {
    func rounded(scale: Int, mode: NSDecimalNumber.RoundingMode = .bankers) -> Decimal {
        var result = Decimal()
        var value = self
        NSDecimalRound(&result, &value, scale, mode)
        return result
    }
}
```

## Edge Case Checklist

```markdown
□ Rounding in equal splits
□ Percentages not summing to 100%
□ Zero amount expenses
□ Self-payment (payer in split)
□ Circular debt simplification
□ Currency precision
□ Negative balances
□ Empty group expenses
□ Offline expense sync conflicts
□ Deleted user in group
```

---

# D — Design Trade-offs

## Trade-off 1: Balance Calculation (Real-time vs On-demand)

| Real-time | On-demand |
|-----------|-----------|
| Always up to date | Calculated when viewed |
| Storage overhead | Computation overhead |
| Fast reads | Slower reads |

**Decision:** Real-time with caching

**Rationale:** Balance dashboard is viewed frequently. Pre-computed balances are worth the storage.

## Trade-off 2: Debt Simplification (Auto vs Manual)

| Automatic | Manual |
|-----------|--------|
| Fewer transactions | Preserves who-owes-whom |
| May confuse users | Intuitive |
| Efficient | More settlements |

**Decision:** Show both views

**Rationale:** Show detailed balances by default, offer "simplify" option for settlement.

## Trade-off 3: Offline Support

| Full Offline | Online Required |
|--------------|-----------------|
| Complex sync | Simple |
| Conflict handling | No conflicts |
| Better UX for trips | May lose data |

**Decision:** Full offline with last-write-wins

**Rationale:** Expense apps are often used on trips without connectivity.

---

# iOS Implementation

## Complete ViewModel

```swift
@MainActor
class ExpenseViewModel: ObservableObject {
    // State
    @Published private(set) var expenses: [Expense] = []
    @Published private(set) var balances: [UserBalance] = []
    @Published private(set) var suggestedSettlements: [Settlement] = []
    @Published private(set) var isLoading = false
    
    // Form State
    @Published var description: String = ""
    @Published var amount: String = ""
    @Published var payer: User?
    @Published var selectedParticipants: Set<String> = []
    @Published var splitType: SplitTypeOption = .equal
    
    // Dependencies
    private let balanceManager: BalanceManager
    private let splitCalculator: SplitCalculator
    private let expenseRepository: ExpenseRepository
    
    init(balanceManager: BalanceManager, expenseRepository: ExpenseRepository) {
        self.balanceManager = balanceManager
        self.expenseRepository = expenseRepository
        self.splitCalculator = SplitCalculator()
    }
    
    // MARK: - Add Expense
    
    func addExpense() async throws {
        guard let amountDecimal = Decimal(string: amount),
              let payer = payer,
              !selectedParticipants.isEmpty else {
            throw ExpenseError.invalidInput
        }
        
        isLoading = true
        defer { isLoading = false }
        
        // Set strategy based on split type
        splitCalculator.setStrategy(splitType.strategy)
        
        // Calculate splits
        let splits = splitCalculator.calculate(
            total: amountDecimal,
            among: Array(selectedParticipants),
            context: splitType.context
        )
        
        // Create expense
        let expense = Expense(
            id: UUID().uuidString,
            groupId: nil,
            description: description,
            totalAmount: amountDecimal,
            currency: "USD",
            paidBy: [Payment(userId: payer.id, amount: amountDecimal)],
            splitAmong: splits,
            createdBy: AuthManager.shared.userId,
            createdAt: Date()
        )
        
        // Save and update balances
        try await expenseRepository.save(expense)
        await balanceManager.addExpense(expense)
        
        // Refresh UI
        await refreshData()
        resetForm()
    }
    
    // MARK: - Settlement
    
    func settleUp(with userId: String, amount: Decimal) async throws {
        let currentUserId = AuthManager.shared.userId
        
        try await balanceManager.settle(
            from: currentUserId,
            to: userId,
            amount: amount
        )
        
        await refreshData()
        
        // Haptic feedback
        UINotificationFeedbackGenerator().notificationOccurred(.success)
    }
    
    // MARK: - Data Loading
    
    func refreshData() async {
        let userId = AuthManager.shared.userId
        
        async let expensesTask = expenseRepository.fetchAll()
        async let balancesTask = balanceManager.allBalances(for: userId)
        
        let (fetchedExpenses, fetchedBalances) = await (
            try? expensesTask,
            balancesTask
        )
        
        expenses = fetchedExpenses ?? []
        
        // Aggregate balances per user
        var perUser: [String: Decimal] = [:]
        for balance in fetchedBalances {
            if balance.fromUserId == userId {
                perUser[balance.toUserId, default: 0] -= balance.amount
            } else {
                perUser[balance.fromUserId, default: 0] += balance.amount
            }
        }
        
        balances = perUser.map { UserBalance(userId: $0.key, netAmount: $0.value) }
        suggestedSettlements = DebtSimplifier.simplify(fetchedBalances)
    }
    
    private func resetForm() {
        description = ""
        amount = ""
        payer = nil
        selectedParticipants = []
        splitType = .equal
    }
}

enum SplitTypeOption {
    case equal
    case exact
    case percentage
    case shares
    
    var strategy: SplitStrategy {
        switch self {
        case .equal: return EqualSplitStrategy()
        case .exact: return ExactSplitStrategy()
        case .percentage: return PercentageSplitStrategy()
        case .shares: return ShareSplitStrategy()
        }
    }
}
```

## SwiftUI Balance View

```swift
struct BalancesView: View {
    @StateObject var viewModel: ExpenseViewModel
    
    var body: some View {
        NavigationStack {
            List {
                Section("Your Balances") {
                    ForEach(viewModel.balances) { balance in
                        BalanceRow(balance: balance) {
                            // Settle up action
                            Task {
                                try await viewModel.settleUp(
                                    with: balance.userId,
                                    amount: abs(balance.netAmount)
                                )
                            }
                        }
                    }
                }
                
                if !viewModel.suggestedSettlements.isEmpty {
                    Section("Suggested Settlements") {
                        ForEach(viewModel.suggestedSettlements) { settlement in
                            SettlementRow(settlement: settlement)
                        }
                    }
                }
            }
            .navigationTitle("Balances")
            .refreshable {
                await viewModel.refreshData()
            }
        }
    }
}

struct BalanceRow: View {
    let balance: UserBalance
    let onSettleUp: () -> Void
    
    var body: some View {
        HStack {
            UserAvatar(userId: balance.userId)
            
            VStack(alignment: .leading) {
                Text(balance.userName)
                    .font(.headline)
                Text(balance.description)
                    .font(.subheadline)
                    .foregroundColor(balance.netAmount > 0 ? .green : .red)
            }
            
            Spacer()
            
            if balance.netAmount != 0 {
                Button("Settle") {
                    onSettleUp()
                }
                .buttonStyle(.borderedProminent)
            }
        }
    }
}
```

---

# Interview Tips for This Problem

## What to Say

```markdown
1. "The core challenge is accurate split calculation with rounding..."
   - Shows financial domain awareness

2. "I'll use Strategy pattern for different split algorithms..."
   - Equal, exact, percentage are interchangeable strategies

3. "Factory pattern isn't needed because all expenses are the same type..."
   - Shows understanding of when NOT to use patterns

4. "For real-time balance updates, Observer via Combine works well..."
   - @Published for reactive UI

5. "Debt simplification reduces N transactions to minimum necessary..."
   - Shows algorithmic thinking
```

## Red Flags to Avoid

```markdown
❌ "I'll use Double for money amounts"
   → Shows lack of financial precision awareness

❌ "Factory pattern to create different expense types"
   → Overengineering for homogeneous entities

❌ "Each expense recalculates all balances from scratch"
   → Performance nightmare

❌ Forgetting rounding edge cases
   → Pennies matter in financial apps
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Expense Sharing problem!*
