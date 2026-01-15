# iOS Low-Level Design (LLD) Practice — Amazon Interview Prep

> **Comprehensive LLD problems using the RESHADED framework for iOS engineers**
> 
> **NEW: Interview Scripts Guide with word-by-word explanations of what to say!**

---

## 📂 Folder Structure

| File | Problem | Pattern Focus | Difficulty |
|------|---------|---------------|------------|
| `00_RESHADED_Template.md` | **Template** | All patterns | Framework |
| `01_Parking_Lot_System.md` | Parking Lot | Factory, Strategy, Singleton | ⭐⭐⭐ |
| `02_Movie_Ticket_Booking.md` | BookMyShow | Observer, State, Strategy | ⭐⭐⭐⭐ |
| `03_Chess_Game.md` | Chess Game | Factory, Strategy, Command | ⭐⭐⭐⭐ |
| `04_Expense_Sharing_App.md` | Splitwise | Strategy, Observer | ⭐⭐⭐ |
| `05_Food_Ordering_System.md` | Zomato/Swiggy | Observer, Factory, Strategy | ⭐⭐⭐⭐ |
| `06_Rate_Limiter.md` | Rate Limiting | Strategy, Singleton | ⭐⭐⭐ |
| `07_LRU_Cache.md` | LRU Cache | Data Structures | ⭐⭐⭐ |
| `08_Notification_System.md` | Notifications | Observer, Strategy, Factory | ⭐⭐⭐ |
| `09_Meeting_Room_Booking.md` | Room Booking | Facade, Strategy | ⭐⭐⭐ |
| `10_Social_Media_Feed.md` | Feed System | Strategy, Observer, Facade | ⭐⭐⭐⭐ |
| `11_Feed_Image_Loader.md` | Image Loader | Facade, Strategy, Caching | ⭐⭐⭐⭐⭐ |
| `12_Interview_Scripts_Guide.md` | **🎤 Interview Scripts** | Word-by-word explanations | Essential |


---

## 🎯 RESHADED Framework

Every problem follows the **RESHADED** approach:

```
R - Requirements       → Functional + Non-functional
E - Entities           → Core classes + relationships
S - States             → State machines + transitions
H - Handling           → Concurrency + thread safety
A - Architecture       → Patterns + WHY chosen + WHY rejected
D - Data Flow          → Sequence diagrams
E - Edge Cases         → Failures + race conditions
D - Design Trade-offs  → Explicit comparisons
```

---

## 🏆 High Priority Problems (Amazon Favorites)

### 1️⃣ Parking Lot System
- **Patterns**: Factory, Strategy, Singleton
- **iOS Angle**: Real-time spot availability, map integration
- **Key Challenge**: Concurrent spot reservation

### 2️⃣ Movie Ticket Booking (BookMyShow)
- **Patterns**: Observer, State Machine, Strategy
- **iOS Angle**: 30-second seat lock timer, optimistic updates
- **Key Challenge**: Seat locking with timeout

### 3️⃣ Chess Game
- **Patterns**: Factory, Strategy, Command
- **iOS Angle**: Move animations, undo/redo
- **Key Challenge**: Polymorphic piece movement

### 4️⃣ Expense Sharing (Splitwise)
- **Patterns**: Strategy, Observer
- **iOS Angle**: Real-time balance updates
- **Key Challenge**: Split algorithms + debt simplification

### 5️⃣ Food Ordering (Zomato/Swiggy)
- **Patterns**: Observer, Factory, Strategy
- **iOS Angle**: Real-time tracking, WebSocket
- **Key Challenge**: Order status tracking

---

## 📊 Medium Priority Problems

### 6️⃣ Rate Limiter
- Token bucket vs sliding window
- Client-side request throttling

### 7️⃣ LRU Cache
- HashMap + Doubly Linked List
- iOS image caching

### 8️⃣ Notification System
- UNUserNotificationCenter
- Push, local, in-app channels

### 9️⃣ Meeting Room Booking
- EventKit integration
- Conflict detection

### 🔟 Social Media Feed
- Infinite scroll + prefetching
- Optimistic UI updates

---

## 🛠 iOS-Specific Discussion Points

### Concurrency

| Tool | Use When |
|------|----------|
| `async/await` | Modern async code (iOS 15+) |
| `Actor` | Shared mutable state |
| `OperationQueue` | Cancellable, dependencies |
| `GCD` | Quick fire-and-forget |

### Memory Management

```swift
// Closures - always weak self
service.fetch { [weak self] result in
    guard let self else { return }
    self.update(result)
}

// Unowned - when guaranteed lifetime
class Child {
    unowned let parent: Parent
}
```

### Protocol-Oriented Design

```swift
protocol DataFetching {
    func fetch() async throws -> Data
}

// Dependency injection for testing
class ViewModel {
    private let service: DataFetching
    
    init(service: DataFetching = RealService()) {
        self.service = service
    }
}
```

---

## 🎤 Interview Tips

### What to Say

```markdown
1. "Let me start by clarifying the requirements..."
2. "I'll use [Pattern] because [Reason 1] and [Reason 2]..."
3. "The alternative would be [Other Pattern], but I'm not using it because..."
4. "For thread safety, I'll use Actor because..."
5. "An edge case to handle is..."
```

### Red Flags to Avoid

| Don't | Do |
|-------|-----|
| Jump to code immediately | Start with RESHADED |
| Overuse Singleton | Prefer dependency injection |
| Ignore concurrency | Discuss thread safety |
| Skip error handling | Handle edge cases |
| Use patterns unnecessarily | Keep it simple when appropriate |

---

## 📚 How to Use These Materials

1. **Start with the Template** (`00_RESHADED_Template.md`)
   - Understand the framework
   - Learn pattern selection guide

2. **Practice High Priority First**
   - Parking Lot → Movie Booking → Chess
   - These are Amazon favorites

3. **Time Yourself**
   - 45 minutes per problem
   - Follow the time allocation in template

4. **Practice Explaining Out Loud**
   - Interviewers evaluate communication
   - Use the "magic phrases" from template

5. **Draw Diagrams**
   - Practice sequence diagrams
   - Entity relationship diagrams

---

## ✅ Quick Reference Checklist

For every LLD problem, ensure you cover:

```markdown
□ Clarified requirements (functional + non-functional)
□ Identified core entities and relationships
□ Defined state machine for key entities
□ Addressed thread safety / concurrency
□ Selected patterns WITH justification
□ Explained WHY alternatives were rejected
□ Drew sequence diagram for main flow
□ Listed key edge cases
□ Discussed trade-offs
□ Mentioned iOS-specific considerations
```

---

*Good luck with your Amazon iOS LLD interviews! 🍀*
