# SCALED Framework for iOS System Design Interviews

![SCALED Framework](file:///Users/amiteshmanitiwari/.gemini/antigravity/brain/efb3d0a0-f732-4adb-8035-5c253d57aaec/uploaded_image_1768330233184.png)

> [!IMPORTANT]
> **SCALED** is the comprehensive framework used in all practice files. This guide shows you EXACTLY how to apply each principle in a real Uber L4 interview.

---

## What is SCALED?

**SCALED** is an acronym for the 6 critical components of mobile system design:

| Letter | Component | Interview Phase | Time |
|--------|-----------|----------------|------|
| **S** | **System Requirements** | Requirements Clarification | 0-10 min |
| **C** | **Design Considerations** | High-Level Design (HLD) | 10-15 min |
| **A** | **Architecture** | High-Level Design (HLD) | 15-25 min |
| **L** | **Low-Level Design (LLD)** | Implementation Details | 25-40 min |
| **E** | **Evaluating NFRs** | Deep Dives | 40-50 min |
| **D** | **API Design** | LLD + Deep Dives | 30-50 min |
| **T** | **Trade-offs** | Throughout (esp. Deep Dives) | 45-55 min |

---

## Real Interview Script: Uber Trip List (SCALED in Action)

### S - System Requirements (0-10 min)

**Interview Script:**

```
INTERVIEWER: "Design an iOS app to show a user's trip history, like Uber's trip list."

YOU: "Great! Let me clarify the system requirements first."

📋 SCALED Principle #1: SYSTEM REQUIREMENTS
├─ Functional Requirements
├─ Non-Functional Requirements  
├─ Scale & Constraints
└─ Documented Scope

YOU: "For FUNCTIONAL requirements:
- What features are in scope? (display trips, pagination, details view?)
- What trip statuses should we show? (completed only, or all?)
- Should users be able to filter or search trips?
- Any actions needed? (view receipt, contact driver, rebook?)"

INTERVIEWER: "Focus on completed trips, allow viewing trip details. No filtering for now."

YOU: "For NON-FUNCTIONAL requirements:
- Scale: How many users? How many trips per user typically?
- Performance: What's acceptable latency for first load?
- Offline: Should cached trips be available offline? How many?
- Reliability: Any retry or error handling expectations?"

INTERVIEWER: "1M+ users, typical user has 50-500 trips. First load under 1 second. 
Cache first page for offline. Yes, handle errors gracefully."

YOU: "Let me document our agreed scope:"

✅ IN SCOPE:
- Display completed trips chronologically (newest first)
- Pagination (20 trips/page, infinite scroll)
- Offline viewing of first page
- Pull-to-refresh
- Tap trip → navigate to detail screen

❌ OUT OF SCOPE:
- Trip booking
- Real-time status updates
- Filtering/sorting
- Search

YOU: "This covers our System Requirements. Ready to move to design?"
```

**✅ SCALED Checkpoint S:**
- [ ] Asked functional requirements
- [ ] Asked non-functional requirements
- [ ] Clarified scale & constraints
- [ ] Documented scope clearly

---

### C - Design Considerations (10-15 min)

**Interview Script:**

```
YOU: "Before drawing the architecture, let me think through key DESIGN CONSIDERATIONS."

📋 SCALED Principle #2: DESIGN CONSIDERATIONS
├─ What patterns fit this problem?
├─ What are the technical challenges?
├─ What are mobile-specific constraints?
└─ What alternatives exist?

YOU: "Key considerations for this design:

1. OFFLINE-FIRST
   - Mobile users have intermittent connectivity
   - Need cache-first strategy
   - Decision: Repository pattern with cache layer

2. PERFORMANCE
   - 500 trips = smooth scrolling required
   - Decision: Pagination + cell reuse + prefetching

3. DATA CONSISTENCY
   - Trips don't change after completion
   - Decision: Simple cache invalidation (TTL-based)

4. ARCHITECTURE PATTERN
   - Need testability, separation of concerns
   - Decision: MVVM + Repository
   - Alternative considered: VIPER (too complex for scope)

5. PAGINATION STRATEGY
   - Historical data (not real-time)
   - Users might want page numbers
   - Decision: Offset-based pagination
   - Alternative: Cursor-based (better for feeds)

Should I proceed to architecture diagram?"
```

**✅ SCALED Checkpoint C:**
- [ ] Identified key technical challenges
- [ ] Considered mobile constraints
- [ ] Stated design decisions with rationale
- [ ] Mentioned alternatives

---

### A - Architecture (15-25 min)

**Interview Script:**

```
YOU: "Let me draw the ARCHITECTURE showing all layers and data flow."

📋 SCALED Principle #3: ARCHITECTURE
├─ Draw layered architecture
├─ Show component interactions
├─ Explain data flow with arrows
└─ Justify pattern choice

YOU: "I'm using MVVM + Repository pattern. Let me draw this:"

[DRAWS ON WHITEBOARD]

┌─────────────────────────────────────┐
│         UI Layer                    │
│  ┌───────────────────────────┐      │
│  │ TripListViewController    │      │
│  │ - UITableView             │      │
│  │ - PrefetchDataSource      │      │
│  └───────────────────────────┘      │
└──────────────┬──────────────────────┘
               │ binds to (@Published)
┌──────────────▼──────────────────────┐
│      Presentation Layer             │
│  ┌───────────────────────────┐      │
│  │ TripListViewModel         │      │
│  │ - @Published trips        │      │
│  │ - loadTrips()             │      │
│  │ - loadMore()              │      │
│  └───────────────────────────┘      │
└──────────────┬──────────────────────┘
               │ calls
┌──────────────▼──────────────────────┐
│         Data Layer                  │
│  ┌───────────────────────────┐      │
│  │ TripRepository            │      │
│  │ - Coordinates cache+net   │      │
│  └─────┬─────────────┬───────┘      │
└────────┼─────────────┼──────────────┘
         │             │
    ┌────▼──────┐ ┌───▼────────┐
    │ Network   │ │ Cache      │
    │ Service   │ │ Service    │
    │ (API)     │ │ (CoreData) │
    └───────────┘ └────────────┘

YOU: "Why MVVM + Repository?

✅ MVVM Benefits:
- ViewModel is pure Swift → easy unit testing
- View is dumb → just renders @Published data
- Reactive updates via Combine

✅ Repository Benefits:
- Single source of truth for trip data
- Abstracts cache vs network complexity
- Easy to mock for tests

❌ Trade-off:
- More files than MVC
- If we had complex navigation, might add Coordinator

DATA FLOW for initial load:

App Launch
    │
    ▼
ViewModel.loadTrips()
    │
    ▼
Repository.fetchTrips(page: 1)
    │
    ├─→ Check Cache (offline-first)
    │   ├─ If cached → return immediately
    │   └─ Refresh in background
    │
    └─→ Network: GET /v1/trips?page=1&limit=20
        │
        ▼
    Cache response (disk + memory)
        │
        ▼
    Update @Published trips
        │
        ▼
    UI automatically re-renders

Does this architecture make sense?"
```

**✅ SCALED Checkpoint A:**
- [ ] Drew layered architecture diagram
- [ ] Showed component separation
- [ ] Explained data flow with arrows
- [ ] Justified pattern choice with trade-offs

---

### L - Low-Level Design (25-40 min)

**Interview Script:**

```
YOU: "Now for LOW-LEVEL DESIGN - the actual Swift implementation."

📋 SCALED Principle #4: LOW-LEVEL DESIGN (LLD)
├─ Define data models
├─ Write protocol interfaces
├─ Show key method implementations
├─ Address concurrency & thread safety
└─ Handle errors & edge cases

YOU: "Let me start with data models:"

[WRITES CODE]

// Domain Model
struct Trip: Identifiable, Codable {
    let id: String
    let pickupLocation: Location
    let dropoffLocation: Location
    let startTime: Date
    let endTime: Date
    let fare: Decimal
    let driverName: String
    let vehicleType: String
}

struct Location: Codable {
    let latitude: Double
    let longitude: Double
    let address: String
}

// API Response
struct TripListResponse: Codable {
    let trips: [Trip]
    let pagination: PaginationInfo
}

struct PaginationInfo: Codable {
    let currentPage: Int
    let totalPages: Int
    let hasMore: Bool
}

YOU: "Now the ViewModel with state management:"

[WRITES CODE]

@MainActor
class TripListViewModel: ObservableObject {
    @Published private(set) var trips: [Trip] = []
    @Published private(set) var state: LoadingState = .idle
    
    private let repository: TripRepositoryProtocol
    private var currentPage = 0
    private var hasMorePages = true
    
    func loadInitialTrips() async {
        guard state != .loading else { return }
        state = .loading
        
        do {
            let response = try await repository.fetchTrips(page: 1)
            trips = response.trips
            hasMorePages = response.pagination.hasMore
            state = .success
        } catch {
            state = .error(error)
        }
    }
    
    func loadMoreTrips() async {
        guard state != .loadingMore, hasMorePages else { return }
        
        state = .loadingMore
        let nextPage = currentPage + 1
        
        do {
            let response = try await repository.fetchTrips(page: nextPage)
            
            // Deduplicate before appending
            let newTrips = response.trips.filter { newTrip in
                !trips.contains(where: { $0.id == newTrip.id })
            }
            
            trips.append(contentsOf: newTrips)
            currentPage = nextPage
            hasMorePages = response.pagination.hasMore
            state = .success
        } catch {
            state = .error(error)
        }
    }
}

YOU: "Key LLD decisions:
- async/await for concurrency
- @MainActor to ensure UI updates on main thread
- Guard checks to prevent duplicate loads
- Deduplication to handle race conditions

Should I continue with Repository and Cache implementation?"
```

**✅ SCALED Checkpoint L:**
- [ ] Defined clear data models
- [ ] Used protocols for testability
- [ ] Showed async/await concurrency
- [ ] Added guard checks for edge cases
- [ ] Included error handling

---

### E - Evaluating NFRs (40-50 min)

**Interview Script:**

```
INTERVIEWER: "How do you handle the performance requirement of <1 second first load 
with potentially 500 cached trips?"

YOU: "Great question! Let me evaluate our NON-FUNCTIONAL REQUIREMENTS."

📋 SCALED Principle #5: EVALUATING NFRs
├─ Performance (latency, throughput)
├─ Scalability (large datasets)
├─ Reliability (errors, retries)
├─ Maintainability (testing, monitoring)
└─ Mobile-specific (battery, bandwidth)

YOU: "For PERFORMANCE optimization:

1. CACHING STRATEGY

| Data | Memory Cache | Disk Cache | TTL | Why |
|------|--------------|------------|-----|-----|
| Page 1 | ✅ NSCache | ✅ CoreData | 1 hour | Offline requirement |
| Page 2+ | ✅ NSCache | ❌ | 30 min | Save disk space |
| Images | ❌ | ✅ LRU | 7 days | Large files |

2. FIRST LOAD OPTIMIZATION

[DRAWS DIAGRAM]

App Launch
    │
    ├─ Immediately: Return cached page 1 from disk (~50ms)
    │  └─ Show data to user FAST
    │
    └─ Background: Refresh from network
       └─ Update UI only if changed

This achieves <1 second even on cold start!

3. MEMORY MANAGEMENT for 500+ trips

- Limit in-memory trips to 200
- NSCache auto-evicts based on memory pressure
- Cell reusing in UITableView
- Lazy image loading

4. BATTERY OPTIMIZATION

- Batch multiple page loads into one request if possible
- Prefetch only when on WiFi
- Use exponential backoff for retries (not aggressive polling)

5. RELIABILITY

[WRITES CODE]

func fetchWithRetry(maxRetries: Int = 3) async throws -> TripListResponse {
    var lastError: Error?
    
    for attempt in 0..<maxRetries {
        do {
            return try await networkService.getTrips()
        } catch let error as URLError where error.code == .networkConnectionLost {
            lastError = error
            try await Task.sleep(nanoseconds: UInt64(pow(2.0, Double(attempt)) * 1_000_000_000))
        } catch {
            throw error
        }
    }
    throw lastError!
}

This covers our NFR evaluation. Questions?"
```

**✅ SCALED Checkpoint E:**
- [ ] Addressed performance requirements
- [ ] Showed scalability considerations
- [ ] Demonstrated reliability (retry logic)
- [ ] Covered mobile-specific concerns (battery, memory)
- [ ] Provided concrete metrics or strategies

---

### D - API Design (30-50 min)

**Interview Script:**

```
INTERVIEWER: "What would the backend API look like?"

YOU: "Let me design the API to support this iOS client."

📋 SCALED Principle #6: API DESIGN
├─ RESTful endpoints
├─ Pagination strategy
├─ Error responses
├─ Mobile optimizations
└─ Versioning

YOU: "Here's my API design:

ENDPOINT:
GET /v1/trips?page=1&limit=20

RESPONSE (200 OK):
{
  "data": {
    "trips": [
      {
        "id": "trip_abc123",
        "pickupLocation": {
          "latitude": 37.7749,
          "longitude": -122.4194,
          "address" "123 Main St, SF"
        },
        "dropoffLocation": {...},
        "startTime": "2026-01-13T10:00:00Z",
        "endTime": "2026-01-13T10:25:00Z",
        "fare": {
          "amount": 25.50,
          "currency": "USD"
        },
        "driverName": "John Doe",
        "vehicleType": "uberX"
      }
    ],
    "pagination": {
      "currentPage": 1,
      "totalPages": 15,
      "hasMore": true
    }
  }
}

PAGINATION CHOICE: Offset-based

| Consideration | Offset-based | Cursor-based |
|---------------|--------------|--------------|
| Use case | Historical trips | Real-time feed |
| New items | Rare (completed trips) | Frequent (new posts) |
| Jump to page | ✅ Easy | ❌ Can't |
| Performance | ⚠️ Slower for large offsets | ✅ Fast |

For trip history, offset works because:
- Trips are completed (won't have new items appearing)
- Users might want \"page 5\" for specific date
- Typical user has <500 trips (acceptable performance)

If this were a real-time feature, I'd use cursor-based.

ERROR HANDLING:

400 Bad Request:
{
  \"error\": {
    \"code\": \"INVALID_PAGE\",
    \"message\": \"Page must be >= 1\"
  }
}

401 Unauthorized → iOS auto-refreshes token
429 Rate Limited → iOS respects Retry-After header
5xx Server Error → iOS retries with exponential backoff

MOBILE OPTIMIZATIONS:
- Compression: Accept-Encoding: gzip (70% size reduction)
- Conditional requests: If-None-Match (ETag) for 304 responses
- Field selection: ?fields=id,pickup,dropoff,fare (exclude heavy fields)

API VERSIONING:
- URL path: /v1/trips (clear, easy to route)
- When breaking changes needed: /v2/trips
- Support N-1 versions for 6 months

This API design complete?"
```

**✅ SCALED Checkpoint D:**
- [ ] Designed RESTful endpoints
- [ ] Chose pagination strategy with justification
- [ ] Defined error response format
- [ ] Included mobile optimizations
- [ ] Addressed versioning

---

### T - Trade-offs (Throughout, esp. 45-55 min)

**Interview Script:**

```
INTERVIEWER: "Why MVVM instead of VIPER or Clean Architecture?"

YOU: "Great question about TRADE-OFFS. Let me compare:"

📋 SCALED Principle #7: TRADE-OFFS
├─ For every major decision, state alternatives
├─ Clearly list pros AND cons
├─ Justify choice with context
└─ Show production awareness

YOU: "Architecture Pattern Trade-offs:

| Pattern | Pros | Cons | When to Use |
|---------|------|------|-------------|
| **MVVM** ✅ | ✅ Testable ViewModels<br>✅ Reactive (Combine)<br>✅ Team familiarity | ❌ Can get bloated<br>❌ No navigation flow | Simple-moderate apps |
| **VIPER** | ✅ Maximum separation<br>✅ Clear responsibilities | ❌ Lots of files<br>❌ Overkill for simple features | Complex, multi-module apps |
| **Clean Architecture** | ✅ Domain-centric<br>✅ Framework-independent | ❌ Steep learning curve<br>❌ Over-engineering risk | Enterprise apps |

MY CHOICE: MVVM because:
- Trip list is moderate complexity (not simple, not complex)
- Uber iOS team likely familiar with MVVM
- Combine integration is straightforward
- Can refactor to VIPER later if navigation complexity grows

PAGINATION Trade-offs:

| Strategy | Our Choice | Why |
|----------|------------|-----|
| **Offset** | ✅ Chosen | Historical data, page numbers useful |
| **Cursor** | ❌ Not needed | No real-time updates, no duplicate risk |
| **Keyset** | ❌ Overkill | Performance fine with offset for <500 trips |

CACHING Trade-offs:

| Approach | Our Choice | Why |
|----------|------------|-----|
| **Memory only** | ❌ | Loses data on app kill |
| **Disk only** | ❌ | Slow first load |
| **Memory + Disk** ✅ | ✅ Chosen | Fast access + offline support |

PRODUCTION CONSIDERATIONS:

If I were deploying this at Uber:
- Add analytics: Track page load times, cache hit rates
- Add monitoring: Crash reporting for network failures
- Add feature flags: A/B test pagination size (20 vs 50)
- Add logging: Debug sync issues in production

My design balances:
- ✅ Simplicity (easy to maintain)
- ✅ Performance (<1s first load)
- ✅ Offline support (cache first page)
- ❌ Complexity is moderate (worth it for scale)

Any concerns with these trade-offs?"
```

**✅ SCALED Checkpoint T:**
- [ ] Compared 2-3 alternatives for each major decision
- [ ] Listed pros AND cons clearly
- [ ] Justified choices with context (team, scale, constraints)
- [ ] Showed production awareness

---

## SCALED Checklist for Every Interview

Use this during practice to ensure you cover all SCALED principles:

### ✅ S - System Requirements
- [ ] Asked functional requirements
- [ ] Asked non-functional requirements (scale, performance, offline)
- [ ] Clarified constraints (iOS version, battery, network)
- [ ] Documented scope (in/out)

### ✅ C - Design Considerations
- [ ] Identified key technical challenges
- [ ] Considered mobile-specific constraints
- [ ] Stated major design decisions upfront
- [ ] Mentioned alternative approaches

### ✅ A - Architecture
- [ ] Drew layered architecture diagram
- [ ] Showed component separation clearly
- [ ] Explained data flow with arrows
- [ ] Justified architecture pattern choice

### ✅ L - Low-Level Design
- [ ] Defined clear data models (structs/classes)
- [ ] Wrote protocol interfaces
- [ ] Showed key implementations (not pseudocode)
- [ ] Used async/await or proper concurrency
- [ ] Handled errors and edge cases

### ✅ E - Evaluating NFRs
- [ ] Addressed performance requirements
- [ ] Showed scalability strategies
- [ ] Demonstrated reliability (retry, fallback)
- [ ] Covered mobile concerns (battery, bandwidth, memory)

### ✅ D - API Design
- [ ] Designed RESTful endpoints
- [ ] Chose pagination strategy with justification
- [ ] Defined error response format
- [ ] Included mobile optimizations
- [ ] Addressed versioning

### ✅ T - Trade-offs
- [ ] Compared alternatives for major decisions
- [ ] Listed pros AND cons explicitly
- [ ] Justified with context
- [ ] Showed production awareness

---

## How SCALED Maps to Interview Phases

| Interview Minute | SCALED Focus | What You're Doing |
|------------------|--------------|-------------------|
| **0-10** | **S** | Clarifying requirements |
| **10-15** | **C** | Stating design considerations |
| **15-25** | **A** | Drawing architecture |
| **25-30** | **D** (start) | Showing API endpoints |
| **30-40** | **L** | Writing Swift code |
| **40-45** | **D** (detail) | Detailing API design |
| **45-50** | **E** | Evaluating NFRs |
| **50-55** | **T** | Discussing trade-offs |

**Note:** Trade-offs (T) should be woven throughout, not just at the end!

---

## Practice Files Using SCALED

Every practice file in this folder follows SCALED:

| File | SCALED Coverage |
|------|-----------------|
| [01_Uber_Trip_List_Design.md](./01_Uber_Trip_List_Design.md) | Complete SCALED walkthrough |
| [02_Offline_First_Architecture.md](./02_Offline_First_Architecture.md) | SCALED for sync/offline apps |
| [03_Feed_Pagination_Design.md](./03_Feed_Pagination_Design.md) | SCALED for real-time feeds |

Each file has:
- ✅ Requirements section (S)
- ✅ Design considerations (C)
- ✅ Architecture diagram (A)
- ✅ LLD with code (L)
- ✅ Performance deep dives (E)
- ✅ API design section (D)
- ✅ Trade-off discussions (T)

---

## Common Mistakes That Violate SCALED

| Mistake | SCALED Violation | Fix |
|---------|------------------|-----|
| Jumping to code | Skipping S, C, A | Always clarify first |
| No architecture diagram | Skipping A | Draw layers before coding |
| Pseudocode instead of Swift | Weak L | Write production-quality code |
| Ignoring offline | Weak E | Always discuss mobile NFRs |
| No API design | Skipping D | Design endpoints + pagination |
| "It's just better" | Weak T | Always compare alternatives |

---

## Next Steps

1. **Study this SCALED guide** (30 min)
2. **Open any practice file** (e.g., Trip List)
3. **Identify where each SCALED principle appears** in the file
4. **Practice out loud** using the interview scripts above
5. **Use SCALED checklist** to self-evaluate

> [!TIP]
> In your real interview, you don't need to say "Now I'm doing SCALED principle S". Just naturally follow the flow! The framework ensures completeness.

---

**You're now equipped with the SCALED framework used by top iOS engineers at Uber, Meta, and Google. Practice makes perfect!** 🚀
