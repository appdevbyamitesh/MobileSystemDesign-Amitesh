# iOS Low-Level Design (LLD) Template — RESHADED Framework

> **Universal template for Amazon / Big Tech iOS LLD interviews**

---

## Table of Contents

1. [RESHADED Framework Overview](#reshaded-framework-overview)
2. [Template Structure](#template-structure)
3. [Pattern Selection Guide](#pattern-selection-guide)
4. [iOS-Specific Considerations](#ios-specific-considerations)
5. [Interview Walkthrough Strategy](#interview-walkthrough-strategy)
6. [Common Mistakes to Avoid](#common-mistakes-to-avoid)

---

# RESHADED Framework Overview

RESHADED is a systematic approach to solve LLD problems in iOS interviews. Each letter represents a critical design aspect:

```
R - Requirements       → What to build (functional + non-functional)
E - Entities           → Core classes and relationships  
S - States             → State machines and lifecycles
H - Handling           → Concurrency and thread safety
A - Architecture       → Patterns with justification
D - Data Flow          → Step-by-step + sequence diagrams
E - Edge Cases         → Failures, race conditions, limits
D - Design Trade-offs  → Explicit comparisons
```

---

## Why RESHADED Works for Amazon Interviews

| Aspect | Why It Matters |
|--------|----------------|
| **Structured** | Interviewers know you have a method |
| **Complete** | Covers all angles they evaluate |
| **Defensible** | Forces you to justify decisions |
| **Time-efficient** | Prevents going off-track |

---

# Template Structure

## R — Requirements

### Functional Requirements (What the system DOES)

```markdown
1. Core Feature 1
   - Sub-requirement a
   - Sub-requirement b

2. Core Feature 2
   - Sub-requirement a
   - Sub-requirement b

3. Core Feature 3
   ...
```

### Non-Functional Requirements (How the system BEHAVES)

```markdown
| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Latency | < X ms | User experience |
| Offline Support | Yes/No | Mobile-first |
| Memory | < X MB | Device limits |
| Thread Safety | Required | Concurrent access |
| Testability | Unit + UI | CI/CD pipeline |
```

### Clarifying Questions to Ask

```markdown
1. Scale: How many users / items?
2. Offline: Required?
3. Real-time: WebSocket or polling?
4. Platform: iOS only or cross-platform?
5. Persistence: Local DB needed?
```

---

## E — Entities

### Entity Identification Template

```swift
// Core Domain Entity
struct EntityName {
    let id: String
    let property1: Type
    let property2: Type
    let relationship: RelatedEntity?
}

// Value Objects (immutable, no identity)
struct ValueObject {
    let attribute1: Type
    let attribute2: Type
}

// Enums for Finite States
enum EntityState: String {
    case state1
    case state2
    case state3
}
```

### Relationship Diagram

```
┌──────────────┐       ┌──────────────┐
│   Entity A   │──────▶│   Entity B   │
└──────────────┘       └──────────────┘
       │                      │
       │ 1:N                  │ 1:1
       ▼                      ▼
┌──────────────┐       ┌──────────────┐
│   Entity C   │       │   Entity D   │
└──────────────┘       └──────────────┘
```

### Relationship Types

| Type | Example | Swift Implementation |
|------|---------|---------------------|
| One-to-One | User ↔ Profile | `var profile: Profile?` |
| One-to-Many | User → Orders | `var orders: [Order]` |
| Many-to-Many | User ↔ Groups | `var groups: [Group]` |

---

## S — States

### State Machine Template

```swift
enum EntityState: String {
    case initial
    case inProgress
    case completed
    case failed
    
    var validTransitions: [EntityState] {
        switch self {
        case .initial:
            return [.inProgress]
        case .inProgress:
            return [.completed, .failed]
        case .completed:
            return []  // Terminal state
        case .failed:
            return [.initial]  // Allow retry
        }
    }
    
    func canTransition(to newState: EntityState) -> Bool {
        validTransitions.contains(newState)
    }
}
```

### State Diagram Format

```
[Initial] ──trigger──▶ [InProgress] ──success──▶ [Completed]
                              │
                              │ failure
                              ▼
                          [Failed] ──retry──▶ [Initial]
```

### State Manager Implementation

```swift
@MainActor
class StateManager<State: Equatable>: ObservableObject {
    @Published private(set) var state: State
    private let validator: (State, State) -> Bool
    
    init(initial: State, validator: @escaping (State, State) -> Bool) {
        self.state = initial
        self.validator = validator
    }
    
    func transition(to newState: State) throws {
        guard validator(state, newState) else {
            throw StateError.invalidTransition(from: state, to: newState)
        }
        state = newState
    }
}
```

---

## H — Handling Concurrency

### Concurrency Decision Matrix

| Scenario | Tool | Why |
|----------|------|-----|
| Simple async | async/await | Modern, readable |
| Dependency chain | OperationQueue | Dependencies supported |
| Quick fire-forget | GCD | Lightweight |
| Actor isolation | Actor | Built-in safety |
| Combine streams | Combine | Reactive pipelines |

### Thread Safety Patterns

```swift
// OPTION 1: Actor (Preferred for iOS 15+)
actor ThreadSafeStore<T> {
    private var storage: [String: T] = [:]
    
    func get(_ key: String) -> T? {
        storage[key]
    }
    
    func set(_ key: String, value: T) {
        storage[key] = value
    }
}

// OPTION 2: Serial Queue (Pre-iOS 15)
class QueueSafeStore<T> {
    private var storage: [String: T] = [:]
    private let queue = DispatchQueue(label: "store.queue")
    
    func get(_ key: String) -> T? {
        queue.sync { storage[key] }
    }
    
    func set(_ key: String, value: T) {
        queue.async { [weak self] in
            self?.storage[key] = value
        }
    }
}

// OPTION 3: NSLock (Fine-grained control)
class LockSafeStore<T> {
    private var storage: [String: T] = [:]
    private let lock = NSLock()
    
    func get(_ key: String) -> T? {
        lock.lock()
        defer { lock.unlock() }
        return storage[key]
    }
}
```

### When to Use What

| Lock Type | Best For | Avoid When |
|-----------|----------|------------|
| Actor | Most cases in iOS 15+ | Need synchronous access |
| DispatchQueue | Pre-iOS 15, simple sync | Complex dependencies |
| NSLock | Performance-critical | Holding across await |
| OperationQueue | Cancellable, dependencies | Simple one-off tasks |

---

## A — Architecture & Patterns

### Pattern Analysis Template

```markdown
### Selected Pattern: [Pattern Name]

**What it does:**
Brief explanation in simple terms

**Why chosen:**
1. Reason 1 aligned with requirements
2. Reason 2 aligned with constraints
3. Reason 3 specific to iOS

**Why alternatives rejected:**

| Alternative | Why Rejected |
|-------------|--------------|
| Pattern A | Reason |
| Pattern B | Reason |
| Pattern C | Reason |
```

### Common Patterns Quick Reference

| Pattern | Use When | iOS Example |
|---------|----------|-------------|
| **Factory** | Multiple types, same interface | Creating different cell types |
| **Strategy** | Interchangeable algorithms | Sorting, payment methods |
| **Observer** | 1:N notifications | NotificationCenter, Combine |
| **Singleton** | Single shared instance | NetworkManager (careful!) |
| **Builder** | Complex object construction | URLRequest builder |
| **Decorator** | Add behavior dynamically | Logging wrapper |
| **Facade** | Simplify complex subsystem | API client hiding URLSession |
| **Command** | Encapsulate actions | Undo/Redo |
| **State** | State-dependent behavior | UI states |

### Pattern Implementation Templates

```swift
// FACTORY Pattern
protocol EntityFactory {
    associatedtype Product
    func create(type: String) -> Product
}

class ConcreteFactory: EntityFactory {
    func create(type: String) -> Entity {
        switch type {
        case "typeA": return EntityA()
        case "typeB": return EntityB()
        default: fatalError("Unknown type")
        }
    }
}

// STRATEGY Pattern
protocol Strategy {
    func execute(input: Input) -> Output
}

class Context {
    private var strategy: Strategy
    
    func setStrategy(_ strategy: Strategy) {
        self.strategy = strategy
    }
    
    func performAction(input: Input) -> Output {
        strategy.execute(input: input)
    }
}

// OBSERVER Pattern (Combine)
class Observable<T> {
    private let subject = PassthroughSubject<T, Never>()
    var publisher: AnyPublisher<T, Never> { subject.eraseToAnyPublisher() }
    
    func emit(_ value: T) {
        subject.send(value)
    }
}
```

---

## D — Data Flow

### Sequence Diagram Format

```
┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐
│  User   │    │   View  │    │ViewModel│    │ Service │
└────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘
     │              │              │              │
     │ tap button   │              │              │
     │─────────────▶│              │              │
     │              │ action()     │              │
     │              │─────────────▶│              │
     │              │              │ fetchData()  │
     │              │              │─────────────▶│
     │              │              │              │ API call
     │              │              │              │─────────▶
     │              │              │              │
     │              │              │◀─────────────│ response
     │              │ update state │              │
     │              │◀─────────────│              │
     │ UI updates   │              │              │
     │◀─────────────│              │              │
     │              │              │              │
```

### Data Flow Description Template

```markdown
### Flow: [Name of Flow]

**Trigger:** What initiates this flow

**Steps:**
1. [Actor] does [Action]
2. [Component A] calls [Component B] with [Data]
3. [Component B] processes and returns [Result]
4. [Component A] updates [State]
5. UI reflects changes

**Error Handling:**
- If step X fails: [Recovery action]
- If step Y fails: [Recovery action]
```

---

## E — Edge Cases

### Edge Case Categories

| Category | Examples |
|----------|----------|
| **Network** | No connection, timeout, partial failure |
| **Memory** | Low memory warning, cache eviction |
| **State** | Invalid transitions, race conditions |
| **User** | Double-tap, back during operation |
| **Data** | Empty state, malformed response |
| **Device** | Background/foreground, rotation |

### Edge Case Template

```markdown
### Edge Case: [Name]

**Scenario:** Describe when this happens

**Impact:** What breaks if unhandled

**Detection:** How to identify this situation

**Handling:**
```swift
// Code showing the fix
```

**Testing:** How to verify the fix works
```

### Common Edge Cases Checklist

```markdown
□ Network unavailable
□ Request timeout
□ Response parsing failure
□ Empty data sets
□ Memory warning received
□ App backgrounded during operation
□ Concurrent modification
□ Stale data on UI
□ User cancels mid-operation
□ Token expired during request
```

---

## D — Design Trade-offs

### Trade-off Analysis Template

```markdown
### Trade-off: [Decision Point]

| Option A | Option B |
|----------|----------|
| Pro 1 | Pro 1 |
| Pro 2 | Pro 2 |
| Con 1 | Con 1 |
| Con 2 | Con 2 |

**Decision:** Option [X]

**Rationale:** 
Why this option is better for this specific context
```

### Common iOS Trade-offs

| Trade-off | Option A | Option B | Guidance |
|-----------|----------|----------|----------|
| Caching | Memory | Disk | Memory for speed, Disk for size |
| Updates | Polling | WebSocket | Polling simpler, WS for real-time |
| Navigation | Coordinator | Router | Coordinator for complex flows |
| State | @State | ViewModel | ViewModel for testability |
| Parsing | Codable | Manual | Codable unless performance-critical |

---

# Pattern Selection Guide

## Decision Tree for Pattern Selection

```
Need to create objects?
├── YES → Multiple types with same interface?
│         ├── YES → FACTORY ✓
│         └── NO → Simple init is fine
│
├── Need interchangeable algorithms?
│   ├── YES → STRATEGY ✓
│   └── NO → Continue...
│
├── Need to notify multiple observers?
│   ├── YES → OBSERVER ✓
│   └── NO → Continue...
│
├── Need single global instance?
│   ├── YES → Is it stateless?
│   │         ├── YES → SINGLETON ✓ (with caution)
│   │         └── NO → Consider Dependency Injection instead
│   └── NO → Continue...
│
├── Need undo/redo?
│   ├── YES → COMMAND ✓
│   └── NO → Continue...
│
├── Complex state-dependent behavior?
│   ├── YES → STATE ✓
│   └── NO → Simple conditionals are fine
│
└── Need to simplify complex subsystem?
    ├── YES → FACADE ✓
    └── NO → Keep it simple
```

---

# iOS-Specific Considerations

## Concurrency Discussion Points

```markdown
### GCD vs OperationQueue vs async/await

| Criteria | GCD | OperationQueue | async/await |
|----------|-----|----------------|-------------|
| Simplicity | ★★★ | ★★ | ★★★★ |
| Cancellation | ❌ | ✓ | ✓ |
| Dependencies | ❌ | ✓ | Manual |
| iOS version | iOS 4+ | iOS 4+ | iOS 15+ |
| Debugging | Hard | Medium | Easy |

**Recommendation:** 
- Use async/await for new code (iOS 15+)
- Use OperationQueue when you need cancellation/dependencies
- Use GCD for quick fire-and-forget tasks
```

## Memory Management Patterns

```swift
// WEAK reference - use in closures to avoid retain cycles
class ViewModel {
    func fetchData() {
        service.fetch { [weak self] result in
            guard let self = self else { return }
            self.updateUI(result)
        }
    }
}

// UNOWNED reference - use when reference will never be nil during closure lifetime
class Parent {
    var child: Child?
    
    init() {
        child = Child(parent: self)
    }
}

class Child {
    unowned let parent: Parent  // Parent always exists while Child exists
}
```

## Protocol-Oriented Design

```swift
// Protocol first approach
protocol DataFetching {
    func fetch() async throws -> Data
}

protocol DataCaching {
    func cache(_ data: Data)
    func retrieve() -> Data?
}

// Composition over inheritance
class Repository: DataFetching, DataCaching {
    // Implement both protocols
}

// Easy to mock for testing
class MockRepository: DataFetching {
    func fetch() async throws -> Data {
        return testData
    }
}
```

## Dependency Injection for Testability

```swift
// Constructor injection (preferred)
class ViewModel {
    private let service: ServiceProtocol
    
    init(service: ServiceProtocol = RealService()) {
        self.service = service
    }
}

// Testing
let viewModel = ViewModel(service: MockService())

// Property injection (for optional dependencies)
class ViewController: UIViewController {
    var analyticsService: AnalyticsProtocol = RealAnalytics()
}
```

---

# Interview Walkthrough Strategy

## Time Allocation (45 min interview)

| Phase | Time | Activity |
|-------|------|----------|
| Clarify | 5 min | Ask questions, confirm scope |
| Requirements | 5 min | List functional + non-functional |
| Entities | 7 min | Core classes + relationships |
| States | 5 min | State machines if applicable |
| Architecture | 8 min | Patterns + justification |
| Data Flow | 7 min | Sequence diagram |
| Edge Cases | 5 min | Key failure scenarios |
| Trade-offs | 3 min | Key decisions explained |

## Magic Phrases to Use

```markdown
"Before I start designing, let me clarify a few requirements..."

"For entities, I'm thinking of these core classes..."

"I'm choosing [Pattern] because [Reason 1] and [Reason 2]..."

"The alternative would be [Other Pattern], but I'm not using it because..."

"For thread safety, I'll use [Actor/GCD] because..."

"An edge case to consider is [Scenario]. I'll handle it by..."

"The trade-off here is [Option A] vs [Option B]. I chose [X] because..."
```

---

# Common Mistakes to Avoid

## ❌ DON'T Do This

| Mistake | Why It's Bad | Do This Instead |
|---------|--------------|-----------------|
| Jump to code immediately | Miss requirements | Start with RESHADED |
| Overuse Singleton | Testing nightmare | Dependency injection |
| Ignore concurrency | Race conditions | Discuss thread safety |
| Skip error handling | Crashes in production | Handle edge cases |
| Use patterns unnecessarily | Over-engineering | Keep it simple |
| Forget mobile constraints | Not practical | Discuss memory/battery |

## ✅ What Interviewers Want to Hear

1. **Requirements understanding** before solution
2. **Trade-off discussions** showing maturity
3. **iOS-specific considerations** (memory, threads)
4. **Pattern justification** not just pattern names
5. **Error handling** as part of design
6. **Testing considerations** built-in

---

# Quick Reference Card

```
┌─────────────────────────────────────────────────────────────┐
│                    RESHADED QUICK REFERENCE                  │
├─────────────────────────────────────────────────────────────┤
│ R - Requirements   → Functional + Non-functional            │
│ E - Entities       → Classes + Relationships                │
│ S - States         → State machine + Transitions            │
│ H - Handling       → Concurrency + Thread safety            │
│ A - Architecture   → Patterns + WHY chosen + WHY rejected   │
│ D - Data Flow      → Sequence diagram                       │
│ E - Edge Cases     → Failures + Race conditions             │
│ D - Design         → Trade-offs + Comparisons               │
├─────────────────────────────────────────────────────────────┤
│ PATTERNS: Factory | Strategy | Observer | Singleton |       │
│           Builder | Decorator | Facade | Command | State    │
├─────────────────────────────────────────────────────────────┤
│ CONCURRENCY: Actor > async/await > OperationQueue > GCD     │
├─────────────────────────────────────────────────────────────┤
│ MEMORY: weak in closures | unowned when guaranteed lifetime │
└─────────────────────────────────────────────────────────────┘
```

---

*Use this template as a starting point for any iOS LLD problem!*
