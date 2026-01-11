# Advanced iOS Developer Interview - 10 Day Preparation Plan

## 📋 Interview Structure (Standard Top-Tier Process)

Top-tier tech company iOS interviews typically have **4-5 rounds** over 1-2 days:

1. **Phone Screen (45-60 min)** - Coding + iOS fundamentals
2. **Onsite Round 1: System Design (HLD) (60 min)** - High-level architecture
3. **Onsite Round 2: Low-Level Design (LLD) (60 min)** - Detailed implementation
4. **Onsite Round 3: Coding (60 min)** - Data structures & algorithms
5. **Onsite Round 4: Behavioral (60 min)** - Leadership principles & cultural fit

***

## 🎯 10-Day Master Plan

### **Days 1-2: High-Level Design (HLD) Foundation**

#### What Top Companies Test
- System architecture for mobile apps
- Scalability and performance
- Offline support and caching
- API design and networking
- Data persistence strategies

#### Study Topics

**Day 1 Morning (3 hours): Architecture Patterns**
```
✅ MVC vs MVVM vs VIPER
✅ Clean Architecture principles
✅ Coordinator pattern
✅ Repository pattern
✅ Dependency Injection
```

**Day 1 Afternoon (3 hours): Networking & Caching**
```
✅ REST API design
✅ GraphQL basics
✅ Caching strategies (memory, disk, CDN)
✅ Offline-first architecture
✅ Data synchronization
```

**Day 2 Morning (3 hours): Data Layer**
```
✅ Core Data
✅ Realm vs SQLite vs UserDefaults
✅ Data migration strategies
✅ Local database design
✅ Encryption & security
```

> 📘 **Deep Dive Guide:**
> - [Local Data Storage & Persistence](Local_Data_Storage_Persistence_Guide.md)

**Day 2 Afternoon (3 hours): Practice HLD Questions**
```
Practice designing:
1. Instagram feed
2. WhatsApp chat app
3. Twitter timeline
4. Uber ride booking
5. E-commerce shopping app
```

#### Key HLD Template

```swift
// Always structure your answer like this:

1. REQUIREMENTS CLARIFICATION (5 min)
   - Functional requirements
   - Non-functional requirements (scale, performance)
   - Out of scope

2. HIGH-LEVEL ARCHITECTURE (10 min)
   - Client-Server architecture
   - Main components (UI, Business Logic, Data, Network)
   - Draw boxes and arrows

3. DATA MODEL (10 min)
   - Core entities
   - Relationships
   - Storage choices

4. API DESIGN (10 min)
   - Key endpoints
   - Request/Response format
   - Error handling

5. KEY CHALLENGES & SOLUTIONS (15 min)
   - Offline support
   - Performance optimization
   - Security
   - Scalability

6. TRADEOFFS & ALTERNATIVES (10 min)
   - Why you chose X over Y
   - Pros and cons
```

***

### **Days 3-4: Low-Level Design (LLD) Deep Dive**

#### What Top Companies Test
- Clean code implementation
- Design patterns usage
- Protocol-oriented programming
- Memory management
- Threading and concurrency
- Error handling

#### Study Topics

**Day 3 Morning (3 hours): Design Patterns**
```
✅ Singleton (when to use/avoid)
✅ Factory pattern
✅ Builder pattern
✅ Observer pattern (NotificationCenter, Combine)
✅ Adapter pattern
✅ Decorator pattern
✅ Strategy pattern
```

**Day 3 Afternoon (3 hours): Protocol-Oriented Programming**
```
✅ Protocols vs inheritance
✅ Protocol extensions
✅ Associated types
✅ Type erasure
✅ Dependency injection with protocols
```

**Day 4 Morning (3 hours): Concurrency & Memory**
```
✅ GCD vs OperationQueue vs async/await
✅ Actors and thread safety
✅ Memory management (ARC, weak, unowned)
✅ Retain cycles prevention
✅ Memory leaks detection
```

**Day 4 Afternoon (3 hours): Practice LLD Questions**
```
Implement:
1. Image caching system
2. Network request manager
3. Thread-safe data store
4. Download manager with priority
5. Observable pattern implementation
```

#### Key LLD Template

```swift
// Structure your implementation:

1. CLARIFY REQUIREMENTS (3 min)
   - What classes/protocols needed?
   - What functionality exactly?
   - Any constraints?

2. DEFINE INTERFACES (5 min)
   protocol ImageLoader {
       func load(url: URL, completion: @escaping (UIImage?) -> Void)
   }

3. DISCUSS ARCHITECTURE (5 min)
   - Which patterns will you use?
   - How components interact?

4. IMPLEMENT STEP BY STEP (35 min)
   - Start with core functionality
   - Add error handling
   - Add thread safety
   - Discuss edge cases

5. TEST & OPTIMIZE (10 min)
   - Unit test approach
   - Performance considerations
   - Memory management
```

***

### **Days 5-6: Practice HLD Questions**

#### Common HLD Questions

**Day 5: Practice 3 Complete Designs**

**Question 1: Design Instagram Feed (3 hours)**
```
Requirements:
- Infinite scroll feed
- Image loading with caching
- Like/comment functionality
- Offline support
- Real-time updates

Focus on:
✅ Pagination strategy
✅ Image loading architecture
✅ Cache invalidation
✅ Memory management
✅ Network optimization
```

**Question 2: Design Chat Application (3 hours)**
```
Requirements:
- Send/receive messages
- Online/offline status
- Message persistence
- Push notifications
- Media attachments

Focus on:
✅ WebSocket vs polling
✅ Local database design
✅ Message synchronization
✅ Encryption
✅ Offline message queue
```

**Day 6: Practice 2 Complete Designs**

**Question 3: Design Video Streaming App (3 hours)**
```
Requirements:
- Video player with controls
- Download for offline viewing
- Quality adaptation
- Continue watching
- Recommendations

Focus on:
✅ Adaptive bitrate streaming
✅ Background downloads
✅ Storage management
✅ Analytics tracking
✅ Prefetching strategy
```

**Question 4: Design E-commerce App (3 hours)**
```
Requirements:
- Product catalog with search
- Shopping cart
- Order history
- Payment integration
- Push notifications

Focus on:
✅ Search architecture
✅ Cart persistence
✅ Payment flow security
✅ Order sync strategy
✅ Performance optimization
```

***

### **Days 7-8: Practice LLD Questions**

#### Common LLD Questions

**Day 7: Implement 4 Systems**

**LLD 1: Image Download & Cache Manager (1.5 hours)**
```swift
// Requirements:
- Download images from URLs
- Two-level cache (memory + disk)
- Priority-based downloading
- Cancellation support
- Thread-safe

// Key interfaces to design:
protocol ImageLoader { }
protocol ImageCache { }
protocol ImageDownloader { }

// Implementation focus:
✅ NSCache for memory
✅ FileManager for disk
✅ OperationQueue for downloads
✅ Priority queue logic
✅ Cancellation handling
```

**LLD 2: Network Request Manager (1.5 hours)**
```swift
// Requirements:
- RESTful API calls
- Retry logic
- Request/response logging
- Authentication handling
- Error handling

// Key interfaces:
protocol NetworkService { }
protocol RequestBuilder { }
protocol ResponseParser { }

// Implementation focus:
✅ URLSession configuration
✅ Request interceptors
✅ Retry with exponential backoff
✅ Token refresh flow
✅ Error type design
```

**LLD 3: Local Database Manager (1.5 hours)**
```swift
// Requirements:
- CRUD operations
- Thread-safe access
- Migration support
- Batch operations
- Query interface

// Key interfaces:
protocol DatabaseManager { }
protocol Entity { }
protocol Query { }

// Implementation focus:
✅ Core Data stack setup
✅ Background context handling
✅ Fetch request optimization
✅ Migration versions
✅ Error propagation
```

**LLD 4: Observable Store (1.5 hours)**
```swift
// Requirements:
- State management
- Observer pattern
- Thread-safe updates
- History/undo support
- Persistence

// Key interfaces:
protocol Store { }
protocol Reducer { }
protocol Subscriber { }

// Implementation focus:
✅ Generic state container
✅ Subscriber management
✅ Immutable updates
✅ Notification patterns
✅ Persistence layer
```

**Day 8: Advanced Implementations**

**LLD 5: Download Manager with Priority (2 hours)**
```swift
// Full implementation with:
- Priority queue
- Concurrent downloads limit
- Pause/resume functionality
- Progress tracking
- Network condition awareness

// Test cases:
✅ Multiple downloads
✅ Priority changes
✅ Cancellation
✅ Network failure recovery
✅ Storage full scenario
```

**LLD 6: Thread-Safe Analytics Tracker (2 hours)**
```swift
// Requirements:
- Track app events
- Batch send to server
- Offline queueing
- Deduplication
- Privacy compliance

// Implementation:
✅ Serial queue for writes
✅ Batch upload scheduler
✅ Local persistence
✅ Network retry logic
✅ Event validation
```

**LLD 7: Memory Cache with LRU Eviction (2 hours)**
```swift
// From scratch implementation:
- Generic cache
- LRU eviction policy
- Size/count limits
- Thread-safe
- Performance optimized

// Data structures:
✅ Dictionary + Doubly linked list
✅ Lock-free reads if possible
✅ Size calculation
✅ Memory warning handling
```

***

### **Day 9: Mock Interviews & System Review**

#### Morning Session (3 hours): HLD Mock

**Practice Question: Design Generic E-Commerce App**

**Your 60-minute timeline:**
```
0-5 min: Requirements clarification
- Browse products
- Search
- Add to cart
- Checkout
- Order tracking
- Recommendations

5-15 min: High-level architecture
[Draw on paper/whiteboard]

┌─────────────────────────────────────┐
│           iOS App                   │
│  ┌──────────────────────────────┐  │
│  │   Presentation Layer         │  │
│  │   (UIKit/SwiftUI)            │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │   Business Logic             │  │
│  │   (ViewModels/Coordinators)  │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │   Data Layer                 │  │
│  │   (Repositories/Cache)       │  │
│  └──────────────────────────────┘  │
│  ┌──────────────────────────────┐  │
│  │   Network Layer              │  │
│  │   (API Client)               │  │
│  └──────────────────────────────┘  │
└─────────────────────────────────────┘
         ↓ ↑ REST/GraphQL
┌─────────────────────────────────────┐
│      Backend Services               │
│  - Product Service                  │
│  - Cart Service                     │
│  - Order Service                    │
│  - User Service                     │
│  - Search Service                   │
└─────────────────────────────────────┘

15-25 min: Data model
struct Product {
    let id: String
    let name: String
    let price: Decimal
    let images: [URL]
    let rating: Double
    let reviews: Int
}

struct CartItem {
    let product: Product
    var quantity: Int
}

struct Order {
    let id: String
    let items: [CartItem]
    let totalPrice: Decimal
    let status: OrderStatus
    let timestamp: Date
}

25-40 min: Key components deep dive
1. Product Catalog
   - Search with pagination
   - Filters and sorting
   - Image lazy loading
   - Prefetching

2. Shopping Cart
   - Local persistence
   - Server sync strategy
   - Conflict resolution
   - Price updates

3. Checkout Flow
   - Payment integration (Stripe/Apple Pay)
   - Order validation
   - Error handling
   - Receipt generation

40-55 min: Challenges & Solutions
1. Offline Support
   - Cache product data
   - Queue cart changes
   - Sync when online

2. Performance
   - Image CDN
   - API response caching
   - Lazy loading
   - Background prefetch

3. Security
   - Token-based auth
   - Encrypted payment data
   - SSL pinning

55-60 min: Q&A and alternatives
```

#### Afternoon Session (3 hours): LLD Mock

**Practice Question: Implement Image Loading System**

**Complete implementation with test cases**

***

### **Day 10: Final Revision & Behavioral Prep**

#### Morning (2 hours): Quick Revision

**Concurrency Cheat Sheet**
```swift
// GCD
DispatchQueue.main.async { }
DispatchQueue.global(qos: .userInitiated).async { }

// async/await
Task {
    let result = try await fetchData()
}

// Actors
actor DataStore {
    private var data: [String: Any] = [:]
    func set(_ value: Any, forKey key: String) {
        data[key] = value
    }
}

// Common patterns
- Serial queue for thread-safety
- Barriers for read-write locks
- DispatchGroup for coordination
- Semaphores for resource limiting
```

**Memory Management Cheat Sheet**
```swift
// Strong (default)
var object = MyClass()

// Weak (optional, becomes nil)
weak var delegate: MyDelegate?

// Unowned (non-optional, crashes if nil)
unowned let parent: Parent

// Capture lists in closures
{ [weak self] in
    self?.doSomething()
}

// Common retain cycles
- Closures capturing self
- Delegates (use weak)
- Timer targets (use weak)
```

**Design Patterns Cheat Sheet**
```swift
// Singleton
class Manager {
    static let shared = Manager()
    private init() {}
}

// Factory
protocol Vehicle { }
class VehicleFactory {
    static func make(type: VehicleType) -> Vehicle { }
}

// Observer
NotificationCenter.default.addObserver(...)

// Builder
class RequestBuilder {
    func setURL(_ url: URL) -> Self { }
    func setMethod(_ method: HTTPMethod) -> Self { }
    func build() -> URLRequest { }
}
```

#### Afternoon (4 hours): Behavioral Preparation

**Behavioral Competencies & Leadership Principles - Prepare STAR Stories**

You MUST prepare 2-3 stories for each core competency:

**1. Customer/User Focus**
```
Story: Improved app performance based on user feedback
- Situation: Users complained about slow loading
- Task: Reduce app launch time
- Action: Profiled app, optimized image loading, lazy initialization
- Result: 40% faster launch, 5-star reviews increased
```

**2. Ownership & Accountability**
```
Story: Took ownership of critical bug fix
- Situation: Production crash affecting 10% of users
- Task: Fix before weekend
- Action: Debugged, found memory leak, implemented fix, tested thoroughly
- Result: Zero crashes, deployed hotfix in 4 hours
```

**3. Innovation & Simplification**
```
Story: Simplified complex checkout flow
- Situation: 60% cart abandonment rate
- Task: Reduce checkout steps
- Action: Redesigned to 2-step process, added Apple Pay
- Result: Cart abandonment reduced to 30%
```

**4. Good Judgment / Decision Making**
```
Story: Chose Core Data over Realm
- Situation: Needed local database solution
- Task: Make best technical decision
- Action: Evaluated both, ran performance tests, assessed long-term support
- Result: Core Data proved better for our use case, no regrets
```

**5. Learning & Curiosity**
```
Story: Adopted SwiftUI despite team using UIKit
- Situation: New feature needed modern UI
- Task: Learn SwiftUI in 2 weeks
- Action: Online courses, sample projects, experimented
- Result: Delivered feature using SwiftUI, taught team
```

**6. Hiring & Team Development**
```
Story: Mentored junior developer
- Situation: New hire struggling with iOS concepts
- Task: Help them become productive
- Action: Weekly 1-on-1s, code reviews, pair programming
- Result: They became top contributor in 3 months
```

**7. High Standards & Quality**
```
Story: Refactored messy codebase
- Situation: Technical debt slowing development
- Task: Improve code quality
- Action: Introduced code review process, unit tests, linting
- Result: Bug rate decreased 50%, faster feature delivery
```

**8. Strategic Thinking / Vision**
```
Story: Proposed offline-first architecture
- Situation: Poor network reliability in target market
- Task: Make app work offline
- Action: Designed sync engine, local-first approach
- Result: 90% feature usability offline, user retention up 25%
```

**9. Bias for Action / Speed**
```
Story: Quick fix for critical production issue
- Situation: Payment flow breaking for iOS 15 users
- Task: Fix immediately
- Action: Quickly isolated issue, implemented workaround, tested on device
- Result: Fixed in 2 hours, minimal user impact
```

**10. Frugality / Resourcefulness**
```
Story: Reduced cloud costs by optimizing API calls
- Situation: High cloud costs from excessive API polling
- Task: Reduce costs without affecting UX
- Action: Implemented smart caching, reduced polling frequency, used webhooks
- Result: 60% cost reduction, better performance
```

**11. Trust & Collaboration**
```
Story: Admitted mistake and fixed it proactively
- Situation: My code caused memory leak
- Task: Own up and resolve
- Action: Immediately informed team, found root cause, implemented fix
- Result: Team trusted me more for honesty
```

**12. Deep Dive / Problem Solving**
```
Story: Debugged complex crash only happening on specific devices
- Situation: Crash reports from iPhone X only
- Task: Find root cause
- Action: Got iPhone X, reproduced, used Instruments, found 64-bit issue
- Result: Fixed architecture-specific bug
```

**13. Backbone / Conflict Resolution**
```
Story: Disagreed with technical decision but supported it
- Situation: Team wanted to use React Native
- Task: Voice concerns but support decision
- Action: Expressed concerns about performance, but committed fully once decided
- Result: Made it work, proved myself wrong on some points
```

**14. Delivering Results**
```
Story: Delivered major feature under tight deadline
- Situation: Feature needed for product launch in 3 weeks
- Task: Complete on time with quality
- Action: Broke down tasks, worked extra hours, daily standups
- Result: Delivered 2 days early, zero critical bugs
```

**Format each story:**
```
1. SITUATION (20%): Set the context
2. TASK (10%): What was your responsibility
3. ACTION (50%): What YOU did (not "we")
4. RESULT (20%): Quantifiable outcome
```

***

## 📚 Key Resources to Study

> 📘 **Advanced Interview Cheat Sheet:**
> - [Advanced Interview Cheat Sheet](Advanced_Interview_Cheat_Sheet.md) - Comprehensive guide for Uber, Amazon, Google interviews with HLD patterns, LLD implementations, and company-specific focus areas

**Documentation**
- Swift Language Guide
- iOS Concurrency Guide
- Core Data Programming Guide
- URLSession documentation

**Practice Platforms**
- LeetCode iOS-specific questions
- System Design Primer (GitHub)
- Grokking System Design course

**Books (Quick Read)**
- "iOS Interview Guide" (specific scenarios)
- "Cracking the Coding Interview" (behavioral section)

***

## 🎯 Day-Before Checklist

**Technical Review (2 hours)**
```
✅ Review concurrency patterns
✅ Review common iOS architectures
✅ Practice one HLD question
✅ Practice one LLD question
✅ Review your prepared code samples
```

**Behavioral Review (1 hour)**
```
✅ Read through all STAR stories
✅ Practice articulating them out loud
✅ Prepare questions to ask interviewer
```

**Logistics**
```
✅ Test video setup (if remote)
✅ Prepare whiteboard/paper/markers
✅ Have laptop with Xcode ready
✅ Charge devices
✅ Get good sleep (8 hours)
```

***

## 💡 Interview Day Tips

**For HLD Round:**
1. **Clarify requirements first** - Don't assume
2. **Think out loud** - Show your thought process
3. **Start high-level** - Don't jump into code
4. **Draw diagrams** - Visual communication is key
5. **Discuss tradeoffs** - Show you understand pros/cons
6. **Ask about scale** - "How many users? How much data?"

**For LLD Round:**
1. **Define interfaces first** - Protocol before implementation
2. **Write clean code** - Proper naming, spacing
3. **Handle errors** - Show you think about edge cases
4. **Discuss thread safety** - Top companies love this
5. **Write tests mentally** - Talk through test cases
6. **Optimize if asked** - Time/space complexity

**For Behavioral:**
1. **Use STAR format strictly**
2. **Be specific with numbers** - "Reduced by 40%"
3. **Focus on YOUR actions** - Not team's
4. **Be honest** - Don't exaggerate
5. **Show learning** - "What I learned was..."

***

## 🚨 Common Mistakes to Avoid

**Technical:**
- ❌ Jumping to code before clarifying
- ❌ Ignoring edge cases
- ❌ Not discussing time/space complexity
- ❌ Forgetting thread safety
- ❌ Not handling errors
- ❌ Poor variable naming

**Behavioral:**
- ❌ Vague answers without specifics
- ❌ Saying "we" instead of "I"
- ❌ No quantifiable results
- ❌ Blaming others
- ❌ Not showing what you learned

***

## 📊 Daily Time Commitment

**Days 1-8:** 6 hours/day
- 3 hours morning
- 3 hours afternoon/evening

**Day 9:** 8 hours (full mock interviews)

**Day 10:** 4 hours (final review)

**Total:** ~54 hours of focused preparation

***

## ✅ Final Confidence Checklist

By Day 10, you should be able to:

**HLD:**
- ✅ Design Instagram feed in 45 minutes
- ✅ Explain caching strategies
- ✅ Discuss offline-first architecture
- ✅ Draw architecture diagrams confidently

**LLD:**
- ✅ Implement image cache from scratch in 30 minutes
- ✅ Design thread-safe data structures
- ✅ Use proper design patterns
- ✅ Write clean, production-quality code

**Behavioral:**
- ✅ Have 2-3 stories per leadership principle
- ✅ Deliver STAR format smoothly
- ✅ Handle follow-up questions confidently

***

**You've got this! Top companies value builders who dive deep and deliver results. Show them you're that person! 🚀**
