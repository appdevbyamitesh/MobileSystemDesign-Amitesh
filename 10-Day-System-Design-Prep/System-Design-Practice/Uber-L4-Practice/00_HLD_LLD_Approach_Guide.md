# How to Approach Real-Life iOS System Design (HLD/LLD)

> [!IMPORTANT]
> This guide teaches you the exact framework used in Uber L4 iOS system design interviews. Follow this step-by-step approach for every design problem.

---

## 🎯 Interview Timeline (60 minutes)

```
0-10 min  → Requirements Clarification
10-25 min → High-Level Design (HLD)
25-45 min → Low-Level Design (LLD)
45-55 min → Deep Dives & Trade-offs
55-60 min → Questions & Wrap-up
```

---

## Phase 1: Requirements Clarification (0-10 min)

### What to Ask (Template Questions)

**Functional Requirements:**
- "What are the core features we need to support?"
- "Should we prioritize any specific user flows?"
- "Are there any features we can deprioritize for this design?"

**Non-Functional Requirements:**
- "What's our target for offline support? (read-only, full CRUD, sync?)"
- "What scale are we designing for? (10K users, 1M users?)"
- "What's the acceptable latency for data loading?"
- "Do we need to support older iOS versions?"

**Constraints & Assumptions:**
- "Can I assume we have a REST/GraphQL backend ready?"
- "What's the battery consumption tolerance?"
- "Should we optimize for memory or network?"

### 📝 Write Down Agreed Scope

Create a box on your whiteboard/doc:

```
✅ IN SCOPE:
- Display trip history with pagination
- Offline viewing of cached trips
- Pull-to-refresh
- Smooth scrolling for 1000+ trips

❌ OUT OF SCOPE:
- Trip booking
- Real-time updates
- Map integration
```

> [!TIP]
> Spending 8-10 minutes on requirements saves you from redesigning mid-interview!

---

## Phase 2: High-Level Design (HLD) (10-25 min)

### When to Draw Diagrams

**Always draw for HLD:**
1. **Architecture Layers** - Show UI → ViewModel → Repository → Network/Cache
2. **Data Flow** - Show request/response paths with arrows
3. **Component Interaction** - Show how modules communicate

### HLD Drawing Template

```
┌─────────────────────────────────────────┐
│           UI Layer (SwiftUI/UIKit)      │
│   ┌───────────────────────────────┐     │
│   │  TripListViewController       │     │
│   └───────────────────────────────┘     │
└──────────────┬──────────────────────────┘
               │ binds to
┌──────────────▼──────────────────────────┐
│         Presentation Layer              │
│   ┌───────────────────────────────┐     │
│   │  TripListViewModel            │     │
│   │  - Published properties       │     │
│   │  - Business logic             │     │
│   └───────────────────────────────┘     │
└──────────────┬──────────────────────────┘
               │ calls
┌──────────────▼──────────────────────────┐
│            Data Layer                   │
│   ┌───────────────────────────────┐     │
│   │  TripRepository               │     │
│   │  - fetchTrips(page:)          │     │
│   │  - Coordinates Network+Cache  │     │
│   └─────┬─────────────────┬───────┘     │
└─────────┼─────────────────┼─────────────┘
          │                 │
    ┌─────▼──────┐    ┌────▼──────┐
    │NetworkLayer│    │CacheLayer │
    │(URLSession)│    │(CoreData) │
    └────────────┘    └───────────┘
```

### What to Explain for HLD

**1. Architecture Pattern Choice**
```
"I'm choosing MVVM + Repository because:
✅ Separates UI from business logic
✅ Easy to unit test ViewModels
✅ Repository abstracts data sources
✅ Familiar to iOS teams at scale
"
```

**2. Key Components**
- **UI Layer**: Views, ViewControllers
- **Presentation Layer**: ViewModels, State management
- **Data Layer**: Repository, Use Cases
- **Network Layer**: API clients, Request managers
- **Cache Layer**: In-memory + Disk persistence

**3. Data Flow (Critical!)**

Draw arrows showing:
```
User Action → ViewModel → Repository → Network/Cache → ViewModel → UI Update
```

---

## Phase 3: Low-Level Design (LLD) (25-45 min)

### When to Write Code vs Draw Diagrams

**Draw Diagrams For:**
- State machines (loading → success → error)
- Sequence diagrams (pagination flow)
- Cache invalidation logic

**Write Code For:**
- Protocol definitions
- Key method signatures
- Data models
- Critical algorithms (e.g., cache eviction)

### LLD Example: Trip Repository

```swift
// Protocol-based design for testability
protocol TripRepositoryProtocol {
    func fetchTrips(page: Int) async throws -> [Trip]
    func getCachedTrips() -> [Trip]
    func clearCache() async
}

class TripRepository: TripRepositoryProtocol {
    private let networkService: NetworkServiceProtocol
    private let cacheService: CacheServiceProtocol
    private let queue = DispatchQueue(label: "com.uber.tripRepo", qos: .userInitiated)
    
    func fetchTrips(page: Int) async throws -> [Trip] {
        // Try cache first for offline support
        if page == 1, let cached = cacheService.get(key: "trips_page_1") {
            Task { await refreshInBackground(page: 1) }
            return cached
        }
        
        // Network fetch
        let trips = try await networkService.fetchTrips(page: page)
        
        // Update cache (thread-safe)
        await cacheService.save(trips, key: "trips_page_\(page)")
        
        return trips
    }
}
```

### Draw State Diagram

```
     ┌──────────┐
     │  IDLE    │
     └────┬─────┘
          │ user scrolls to bottom
          ▼
     ┌──────────┐
     │ LOADING  │──────┐
     └────┬─────┘      │
          │            │ error
          │ success    │
          ▼            ▼
     ┌──────────┐  ┌──────────┐
     │ SUCCESS  │  │  ERROR   │
     └──────────┘  └────┬─────┘
                        │ retry
                        └───────┐
                                ▼
                          (back to LOADING)
```

---

## Phase 3.5: API Design (Integrated Throughout)

### When to Discuss API Design

**You should design APIs during or after HLD, woven into your solution:**

| Interview Phase | API Discussion |
|----------------|----------------|
| **Requirements (0-10min)** | Clarify if you're designing the API too ("Should I design both iOS client AND the API?") |
| **HLD (10-25min)** | Show API endpoints briefly in architecture diagram |
| **LLD (25-45min)** | **Detail full API design here** - endpoints, pagination, errors |
| **Deep Dive (45-55min)** | Discuss API versioning, rate limiting, mobile optimizations |

> [!IMPORTANT]
> Modern iOS system design interviews expect you to design BOTH the client (iOS) and the API. This is critical for Uber L4 level!

### API Design Template

**Always cover these 4 aspects:**

**1. Endpoint Structure**
```
GET /v1/resource?param=value

Example:
GET /v1/trips?page=1&limit=20
POST /v1/trips
PUT /v1/trips/{id}
DELETE /v1/trips/{id}
```

**2. Pagination Strategy**

Show decision-making:
```
"For trip history, I'll use offset-based pagination:
  - Historical data (trips don't change order)
  - Users might want page numbers
  
But for a feed, I'd use cursor-based:
  - Real-time data (new posts appear)
  - Prevents duplicates when scrolling
"
```

**3. Error Response Format**
```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests",
    "retryAfter": 60
  }
}
```

**4. Mobile Optimizations**
- Response compression (gzip)
- Field selection (`?fields=id,title,date`)
- CDN URLs for images
- Batch endpoints for sync

### API Design Example (Trip List)

```
GET /v1/trips?page=1&limit=20

Response:
{
  "data": {
    "trips": [...],
    "pagination": {
      "currentPage": 1,
      "totalPages": 15,
      "hasMore": true
    }
  }
}

Why this design:
✅ Offset pagination works for historical trips
✅ Include both hasMore (simple) and totalPages (UX flexibility)
✅ Wrap in "data" envelope for future extensibility
```

### When Interviewer Asks: "What about the API?"

**Response Template:**

> "Let me design the API to support the iOS client:
> 
> **Endpoints:**
> - GET /v1/trips - List with pagination
> - GET /v1/trips/{id} - Detail
> 
> **Pagination Choice:**
> [Explain offset vs cursor for this use case]
> 
> **Error Handling:**
> 401 → Refresh token
> 429 → Rate limit, respect Retry-After
> 5xx → Retry with exponential backoff
> 
> **Mobile Optimizations:**
> - Compression (Accept-Encoding: gzip)
> - Conditional requests (ETag)
> - Field selection for bandwidth savings"

### What to Draw for API Design

```
┌─────────────────────────────────────┐
│  iOS Client                         │
│                                     │
│  GET /v1/trips?page=1&limit=20      │
└──────────────┬──────────────────────┘
               │
               ▼
┌─────────────────────────────────────┐
│  API Server                         │
│                                     │
│  {                                  │
│    "data": {                        │
│      "trips": [...],                │
│      "pagination": {...}            │
│    }                                │
│  }                                  │
└─────────────────────────────────────┘
```

### Quick Reference: REST Principles

| Method | Use Case | Idempotent | Cacheable |
|--------|----------|------------|-----------|
| GET | Fetch data | ✅ | ✅ |
| POST | Create resource | ❌ | ❌ |
| PUT | Replace resource | ✅ | ❌ |
| PATCH | Partial update | ❌ | ❌ |
| DELETE | Remove resource | ✅ | ❌ |

### Status Codes to Know

- **200 OK** - Success
- **201 Created** - POST succeeded
- **304 Not Modified** - Cache valid
- **400 Bad Request** - Invalid input
- **401 Unauthorized** - Auth failed
- **404 Not Found** - Resource missing
- **429 Too Many Requests** - Rate limited
- **500 Internal Error** - Server failure

> [!TIP]
> Reference the comprehensive [API Design Best Practices Guide](./05_API_Design_Best_Practices.md) for detailed patterns!

---

## Phase 4: Deep Dives (45-55 min)

### Common Deep Dive Topics

**1. Caching Strategy**

Draw this table:

| Data Type | Cache Strategy | TTL | Why |
|-----------|---------------|-----|-----|
| Trip List Page 1 | Memory + Disk | 1 hour | Fast app launch |
| Trip List Page 2+ | Disk only | 30 min | Balance memory |
| Trip Details | Memory | Session | Frequent access |
| Images | Disk (LRU) | 7 days | User scrolls back |

**2. Pagination Flow Diagram**

```
User scrolls to row 18/20
         │
         ▼
┌────────────────────┐
│ Check if loading   │───Yes──→ Return (debounce)
└────────┬───────────┘
         │ No
         ▼
┌────────────────────┐
│ Increment page     │
│ Set loading = true │
└────────┬───────────┘
         ▼
┌────────────────────┐
│  Fetch page N+1    │
└────────┬───────────┘
         ▼
┌────────────────────┐
│ Append to data     │
│ Set loading = false│
└────────────────────┘
```

**3. Thread Safety**

```swift
// Show use of actor for thread safety
actor TripCache {
    private var storage: [String: [Trip]] = [:]
    
    func save(_ trips: [Trip], key: String) {
        storage[key] = trips
    }
    
    func get(key: String) -> [Trip]? {
        storage[key]
    }
}
```

---

## What to Draw & When (Summary)

| Phase | What to Draw | Why |
|-------|-------------|-----|
| **HLD** | Architecture layers, Data flow arrows | Show separation of concerns |
| **LLD** | State diagrams, Sequence flows | Show complex logic visually |
| **Deep Dive** | Cache tables, Thread diagrams | Prove you've thought through edge cases |

---

## Tools You Need in Real Interview

### Physical Whiteboard Interview
- Use boxes for components
- Arrows for data flow
- Different colors for layers
- Label everything clearly

### Virtual Interview (CoderPad/Zoom)
- Use diagram tools (Draw.io syntax)
- Write code in main editor
- Keep diagrams simple (ASCII art works!)

---

## 🚨 Common Mistakes That Fail Uber L4 Candidates

| Mistake | Why It Fails | Fix |
|---------|-------------|-----|
| Jumping to code without HLD | Shows lack of system thinking | Always draw architecture first |
| Not asking about scale | Can't justify choices | Ask: "1K users or 1M users?" |
| Ignoring offline | Mobile-first thinking missing | Always discuss cache strategy |
| No error handling | Production readiness doubt | Show retry, fallback flows |
| Over-engineering | L4 wants pragmatic solutions | Start simple, iterate |
| Can't explain trade-offs | Memorized, not understood | For every choice, state why |

---

## Interview Answer Template

When interviewer asks: **"Why did you choose MVVM?"**

**BAD Answer:**
> "Because it's a common pattern."

**GOOD Answer (L4 Level):**
> "I chose MVVM because:
> 1. **Testability** - ViewModels are pure Swift, easy to unit test without UI
> 2. **Separation** - View only handles rendering, ViewModel handles logic
> 3. **Team familiarity** - At Uber's scale, most iOS engineers know MVVM
> 4. **Trade-off** - It's simpler than VIPER for this scope, but if we needed complex navigation coordinator, I'd reconsider"

---

## Next Steps

Practice these 3 designs using this framework:
1. [Uber Trip List Screen](./01_Uber_Trip_List_Design.md)
2. [Offline-First Architecture](./02_Offline_First_Architecture.md)
3. [Feed with Pagination](./03_Feed_Pagination_Design.md)

> [!TIP]
> Set a 60-minute timer and practice OUT LOUD. Talking through your design is 50% of the evaluation.
