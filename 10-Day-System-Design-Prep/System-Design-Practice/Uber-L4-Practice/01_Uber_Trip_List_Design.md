# Uber Trip List Screen - Complete iOS System Design

> [!NOTE]
> This is a complete 60-minute Uber L4 interview simulation. Practice this OUT LOUD with a timer.

**Problem Statement:** *Design an iOS app feature that displays a user's trip history, similar to Uber's trip list screen.*

---

## 🎯 SCALED Framework Coverage

> [!IMPORTANT]
> This design follows the **[SCALED framework](./00_SCALED_Framework_Guide.md)** - the industry-standard approach for mobile system design interviews.

| SCALED | Section in This File | What You'll Learn |
|--------|----------------------|-------------------|
| **S** - System Requirements | [0-10 min: Requirements](#0-10-min-requirements-clarification) | Functional/non-functional requirements, scope definition |
| **C** - Design Considerations | [10-25 min: HLD](#10-25-min-high-level-design-hld) | MVVM choice, offline-first strategy, pagination decision |
| **A** - Architecture | [10-25 min: HLD](#10-25-min-high-level-design-hld) | Layered architecture diagram, data flow |
| **L** - Low-Level Design | [25-45 min: LLD](#25-45-min-low-level-design-lld) | Swift code, protocols, async/await, error handling |
| **E** - Evaluating NFRs | [45-55 min: Deep Dives](#45-55-min-deep-dives) | Performance, caching, thread safety, scalability |
| **D** - API Design | [Part 4: API Design](#part-4-api-design-crucial-for-l4) | RESTful endpoints, offset pagination, mobile optimizations |
| **T** - Trade-offs | Throughout + [Deep Dives](#45-55-min-deep-dives) | MVVM vs VIPER, offset vs cursor, cache strategies |

> [!TIP]
> Look for 🎯 markers throughout this file showing where each SCALED principle is applied!

---

## 0-10 min: Requirements Clarification

### Questions to Ask Interviewer

**You:** "Let me clarify the requirements before we design. First, for functional requirements:
- Should users see all past trips or only completed trips?
- Do we need to support trip filtering (by date, type, status)?
- Should we show trip details inline or navigate to a separate screen?
- What about pagination - infinite scroll or load more button?"

**Interviewer:** "Focus on completed trips only. No filtering for now. Tap on trip navigates to details. Use infinite scroll."

**You:** "Great. For non-functional requirements:
- What's our target user base? Are we designing for 10K users or millions?
- What's the acceptable latency for loading the first page?
- Do we need offline support? If yes, how many cached trips?
- Should we optimize more for battery or data usage?"

**Interviewer:** "Design for millions of users. First load should be under 1 second. Yes, support offline viewing of at least the first 20 trips. Balance battery and data."

**You:** "Can I assume we have a RESTful API that returns paginated trip data with metadata like pickup/dropoff locations, timestamps, fare, etc.?"

**Interviewer:** "Yes, assume pagination with 20 items per page."

### Documented Scope

```
✅ IN SCOPE:
- Display completed trips in reverse chronological order
- Pagination (20 trips per page, infinite scroll)
- Offline viewing of first 20 trips (page 1)
- Pull-to-refresh
- Smooth scrolling for large datasets
- Tap to view trip details
- Loading and error states

❌ OUT OF SCOPE:
- Trip filtering/sorting
- Real-time trip updates
- Trip booking
- Map view in list
- Search functionality
```

**Non-Functional:**
- Target: 1M+ users
- First load: < 1 second
- Offline: First page cached
- Smooth 60 FPS scrolling

---

## 10-25 min: High-Level Design (HLD)

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    UI Layer                             │
│  ┌───────────────────────────────────────────────┐      │
│  │  TripListViewController / TripListView        │      │
│  │  - UITableView / List                         │      │
│  │  - Pull-to-refresh                            │      │
│  │  - Loading indicators                         │      │
│  └───────────────────────────────────────────────┘      │
└──────────────────────┬──────────────────────────────────┘
                       │ observes
┌──────────────────────▼──────────────────────────────────┐
│               Presentation Layer                        │
│  ┌───────────────────────────────────────────────┐      │
│  │  TripListViewModel                            │      │
│  │  @Published var trips: [Trip] = []            │      │
│  │  @Published var state: LoadingState           │      │
│  │  func loadInitialTrips()                      │      │
│  │  func loadMoreTrips()                         │      │
│  │  func refresh()                               │      │
│  └────────────────────┬──────────────────────────┘      │
└─────────────────────────┬───────────────────────────────┘
                          │ calls
┌─────────────────────────▼───────────────────────────────┐
│                    Data Layer                           │
│  ┌───────────────────────────────────────────────┐      │
│  │  TripRepository                               │      │
│  │  - Orchestrates Network + Cache               │      │
│  │  - Offline-first logic                        │      │
│  │  - Deduplication                              │      │
│  └─────────┬─────────────────────┬─────────────┘       │
└───────────┼─────────────────────┼───────────────────────┘
            │                      │
     ┌──────▼────────┐      ┌──────▼─────────┐
     │ NetworkService│      │  CacheService  │
     │ (URLSession)  │      │  (CoreData +   │
     │ - API calls   │      │   NSCache)     │
     └───────────────┘      └────────────────┘
```

### Why MVVM + Repository?

**You:** "I'm choosing MVVM with Repository pattern because:

1. **MVVM Benefits:**
   - View is dumb, only renders data
   - ViewModel holds business logic, easy to unit test
   - Reactive updates via Combine/Observations

2. **Repository Pattern:**
   - Single source of truth for trip data
   - Abstracts network vs cache complexity from ViewModel
   - Easy to swap implementations for testing

3. **Trade-offs:**
   - ✅ More testable than MVC
   - ✅ Clearer than VIPER for this scope
   - ❌ More files than MVC
   - ❌ If we had complex navigation, might need Coordinator"

### Data Flow (Critical!)

**Initial Load Flow:**

```
App Launch
    │
    ▼
ViewModel.loadInitialTrips()
    │
    ▼
Repository.fetchTrips(page: 1)
    │
    ├─→ Check Cache first (offline-first)
    │   ├─ If cached → return immediately
    │   └─ Start background refresh
    │
    └─→ Network call (if no cache / fresh launch)
        │
        ▼
    API: GET /trips?page=1&limit=20
        │
        ▼
    Save to Cache
        │
        ▼
    Return to ViewModel
        │
        ▼
    @Published trips updated
        │
        ▼
    UI re-renders automatically
```

**Pagination Flow:**

```
User scrolls to bottom
    │
    ▼
willDisplay cell (row 18/20)
    │
    ▼
ViewModel.loadMoreTrips()
    │
    ├─→ Check if already loading → return
    ├─→ Check if all pages loaded → return
    │
    ▼
Repository.fetchTrips(page: currentPage + 1)
    │
    ▼
Append to existing trips
    │
    ▼
@Published trips updated
    │
    ▼
tableView.reloadData()
```

---

## 25-45 min: Low-Level Design (LLD)

### Data Models

```swift
// Domain Model
struct Trip: Identifiable, Codable {
    let id: String
    let pickupLocation: Location
    let dropoffLocation: Location
    let startTime: Date
    let endTime: Date
    let fare: Decimal
    let status: TripStatus
    let driverName: String
    let vehicleType: String
    let mapSnapshotURL: URL?
}

struct Location: Codable {
    let latitude: Double
    let longitude: Double
    let address: String
}

enum TripStatus: String, Codable {
    case completed
    case cancelled
}

// API Response Model
struct TripListResponse: Codable {
    let trips: [Trip]
    let pagination: PaginationInfo
}

struct PaginationInfo: Codable {
    let currentPage: Int
    let totalPages: Int
    let hasMore: Bool
}
```

### ViewModel Implementation

```swift
enum LoadingState {
    case idle
    case loading
    case success
    case error(Error)
    case loadingMore // For pagination
}

@MainActor
class TripListViewModel: ObservableObject {
    @Published private(set) var trips: [Trip] = []
    @Published private(set) var state: LoadingState = .idle
    
    private let repository: TripRepositoryProtocol
    private var currentPage = 0
    private var hasMorePages = true
    
    init(repository: TripRepositoryProtocol) {
        self.repository = repository
    }
    
    func loadInitialTrips() async {
        guard state != .loading else { return }
        
        state = .loading
        currentPage = 1
        
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
        guard state != .loadingMore,
              hasMorePages else { return }
        
        state = .loadingMore
        
        do {
            let nextPage = currentPage + 1
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
    
    func refresh() async {
        await loadInitialTrips()
    }
}
```

### Repository (Offline-First Logic)

```swift
protocol TripRepositoryProtocol {
    func fetchTrips(page: Int) async throws -> TripListResponse
}

class TripRepository: TripRepositoryProtocol {
    private let networkService: NetworkServiceProtocol
    private let cacheService: CacheServiceProtocol
    
    init(networkService: NetworkServiceProtocol,
         cacheService: CacheServiceProtocol) {
        self.networkService = networkService
        self.cacheService = cacheService
    }
    
    func fetchTrips(page: Int) async throws -> TripListResponse {
        let cacheKey = "trips_page_\(page)"
        
        // Offline-first: Try cache first
        if let cached = await cacheService.get(key: cacheKey, type: TripListResponse.self) {
            // Return cached data immediately
            // Refresh in background if page 1
            if page == 1 {
                Task.detached {
                    try? await self.refreshFromNetwork(page: page, cacheKey: cacheKey)
                }
            }
            return cached
        }
        
        // Cache miss or not page 1 - fetch from network
        return try await fetchAndCache(page: page, cacheKey: cacheKey)
    }
    
    private func fetchAndCache(page: Int, cacheKey: String) async throws -> TripListResponse {
        let response = try await networkService.getTrips(page: page, limit: 20)
        
        // Save to cache (async, don't block)
        Task {
            await cacheService.save(response, key: cacheKey, ttl: page == 1 ? 3600 : 1800)
        }
        
        return response
    }
    
    private func refreshFromNetwork(page: Int, cacheKey: String) async throws {
        let response = try await networkService.getTrips(page: page, limit: 20)
        await cacheService.save(response, key: cacheKey, ttl: 3600)
    }
}
```

### Network Service

```swift
protocol NetworkServiceProtocol {
    func getTrips(page: Int, limit: Int) async throws -> TripListResponse
}

class NetworkService: NetworkServiceProtocol {
    private let session: URLSession
    private let baseURL = "https://api.uber.com/v1"
    
    init(session: URLSession = .shared) {
        self.session = session
    }
    
    func getTrips(page: Int, limit: Int) async throws -> TripListResponse {
        var components = URLComponents(string: "\(baseURL)/trips")!
        components.queryItems = [
            URLQueryItem(name: "page", value: "\(page)"),
            URLQueryItem(name: "limit", value: "\(limit)")
        ]
        
        var request = URLRequest(url: components.url!)
        request.setValue("Bearer \(authToken)", forHTTPHeaderField: "Authorization")
        request.cachePolicy = .reloadIgnoringLocalCacheData // Handle caching manually
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        
        guard (200...299).contains(httpResponse.statusCode) else {
            throw NetworkError.serverError(httpResponse.statusCode)
        }
        
        return try JSONDecoder().decode(TripListResponse.self, from: data)
    }
}
```

### Cache Service (Thread-Safe)

```swift
protocol CacheServiceProtocol {
    func save<T: Codable>(_ value: T, key: String, ttl: TimeInterval) async
    func get<T: Codable>(key: String, type: T.Type) async -> T?
    func clear() async
}

actor CacheService: CacheServiceProtocol {
    private let memoryCache = NSCache<NSString, CacheEntry>()
    private let diskCacheURL: URL
    private let fileManager = FileManager.default
    
    init() {
        let cacheDir = fileManager.urls(for: .cachesDirectory, in: .userDomainMask)[0]
        diskCacheURL = cacheDir.appendingPathComponent("TripCache")
        try? fileManager.createDirectory(at: diskCacheURL, withIntermediateDirectories: true)
        
        memoryCache.countLimit = 50 // Limit memory usage
    }
    
    func save<T: Codable>(_ value: T, key: String, ttl: TimeInterval) async {
        let entry = CacheEntry(value: value, expiryDate: Date().addingTimeInterval(ttl))
        
        // Memory cache
        memoryCache.setObject(entry, forKey: key as NSString)
        
        // Disk cache (for page 1 only for offline)
        if key.contains("page_1") {
            let fileURL = diskCacheURL.appendingPathComponent("\(key).json")
            if let data = try? JSONEncoder().encode(entry) {
                try? data.write(to: fileURL)
            }
        }
    }
    
    func get<T: Codable>(key: String, type: T.Type) async -> T? {
        // Try memory first
        if let entry = memoryCache.object(forKey: key as NSString) {
            guard entry.expiryDate > Date() else {
                memoryCache.removeObject(forKey: key as NSString)
                return nil
            }
            return entry.value as? T
        }
        
        // Try disk
        let fileURL = diskCacheURL.appendingPathComponent("\(key).json")
        guard let data = try? Data(contentsOf: fileURL),
              let entry = try? JSONDecoder().decode(CacheEntry.self, from: data) else {
            return nil
        }
        
        guard entry.expiryDate > Date() else {
            try? fileManager.removeItem(at: fileURL)
            return nil
        }
        
        // Promote to memory cache
        memoryCache.setObject(entry, forKey: key as NSString)
        
        return entry.value as? T
    }
    
    func clear() async {
        memoryCache.removeAllObjects()
        try? fileManager.removeItem(at: diskCacheURL)
    }
}

class CacheEntry: NSObject, Codable {
    let value: Any
    let expiryDate: Date
    
    init(value: Any, expiryDate: Date) {
        self.value = value
        self.expiryDate = expiryDate
    }
}
```

### UI Layer (UIKit Example)

```swift
class TripListViewController: UIViewController {
    private let viewModel: TripListViewModel
    private let tableView = UITableView()
    private let refreshControl = UIRefreshControl()
    private var cancellables = Set<AnyCancellable>()
    
    init(viewModel: TripListViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        bindViewModel()
        
        Task {
            await viewModel.loadInitialTrips()
        }
    }
    
    private func setupUI() {
        view.addSubview(tableView)
        tableView.frame = view.bounds
        tableView.delegate = self
        tableView.dataSource = self
        tableView.register(TripCell.self, forCellReuseIdentifier: "TripCell")
        
        refreshControl.addTarget(self, action: #selector(handleRefresh), for: .valueChanged)
        tableView.refreshControl = refreshControl
    }
    
    private func bindViewModel() {
        viewModel.$trips
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
        
        viewModel.$state
            .receive(on: DispatchQueue.main)
            .sink { [weak self] state in
                self?.handleStateChange(state)
            }
            .store(in: &cancellables)
    }
    
    private func handleStateChange(_ state: LoadingState) {
        switch state {
        case .loading:
            showLoadingView()
        case .success, .loadingMore:
            hideLoadingView()
            refreshControl.endRefreshing()
        case .error(let error):
            hideLoadingView()
            refreshControl.endRefreshing()
            showError(error)
        case .idle:
            break
        }
    }
    
    @objc private func handleRefresh() {
        Task {
            await viewModel.refresh()
        }
    }
}

extension TripListViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        viewModel.trips.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "TripCell", for: indexPath) as! TripCell
        cell.configure(with: viewModel.trips[indexPath.row])
        return cell
    }
}

extension TripListViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        // Pagination trigger
        let threshold = viewModel.trips.count - 5
        if indexPath.row >= threshold {
            Task {
                await viewModel.loadMoreTrips()
            }
        }
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        let trip = viewModel.trips[indexPath.row]
        // Navigate to details
        let detailVC = TripDetailViewController(tripId: trip.id)
        navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

---

## Part 4: API Design (Crucial for L4!)

### Trip List API Endpoints

**You:** "Let me design the RESTful API for this feature. Here's my approach:"

#### Endpoint Design

```
GET /v1/trips
  Query Params:
    - page: Int (default: 1)
    - limit: Int (default: 20, max: 100)
    - status: String? (optional: "completed", "cancelled", "all")
  
  Response: 200 OK
  {
    "data": {
      "trips": [
        {
          "id": "trip_abc123",
          "pickupLocation": {
            "latitude": 37.7749,
            "longitude": -122.4194,
            "address": "123 Main St, San Francisco, CA"
          },
          "dropoffLocation": { ... },
          "startTime": "2026-01-13T10:00:00Z",
          "endTime": "2026-01-13T10:25:00Z",
          "fare": {
            "amount": 25.50,
            "currency": "USD",
            "breakdown": {
              "baseFare": 10.00,
              "perMile": 12.50,
              "serviceFee": 3.00
            }
          },
          "status": "completed",
          "driver": {
            "id": "driver_xyz",
            "name": "John Doe",
            "rating": 4.92,
            "photoUrl": "https://cdn.uber.com/drivers/xyz.jpg"
          },
          "vehicle": {
            "type": "uberX",
            "make": "Toyota",
            "model": "Camry",
            "licensePlate": "ABC123"
          },
          "mapSnapshotUrl": "https://cdn.uber.com/maps/trip_abc123.png"
        }
      ],
      "pagination": {
        "currentPage": 1,
        "totalPages": 15,
        "hasMore": true,
        "nextPage": 2,
        "totalCount": 289
      }
    },
    "meta": {
      "requestId": "req_uuid_12345",
      "timestamp": "2026-01-13T18:00:00Z",
      "version": "1.0"
    }
  }

GET /v1/trips/{tripId}
  Response: 200 OK (detailed trip info)
  Response: 404 Not Found (trip doesn't exist)

GET /v1/trips/{tripId}/receipt
  Response: 200 OK (PDF receipt)
```

### Why This Design?

**Interviewer:** "Why did you use offset-based pagination (page/limit) instead of cursor?"

**You:** "Great question! Here's my reasoning:

| Aspect | Offset-Based | Cursor-Based |
|--------|-------------|--------------|
| **Use Case** | Historical data (trips) | Real-time feeds (posts) |
| **New Items** | Rare (completed trips don't change) | Frequent (new posts added) |
| **Jump to Page** | ✅ Easy (page numbers) | ❌ Must scroll sequentially |
| **Consistency** | ⚠️ Duplicates if data changes | ✅ Always consistent |
| **Performance** | ⚠️ Slow for large offsets | ✅ Fast (indexed WHERE) |

**For trip list:**
- Trips are completed (no new items at top)
- Users might want \"page 5\" for specific date
- Performance acceptable (user has max ~500 trips)

**Trade-off:** If we added real-time trips or expected 10K+ trips, I'd switch to cursor-based."

### Error Response Format

```json
// 400 Bad Request - Invalid parameters
{
  "error": {
    "code": "INVALID_PARAMETER",
    "message": "Page number must be positive",
    "details": [
      {
        "field": "page",
        "value": "-1",
        "constraint": "min value is 1"
      }
    ],
    "requestId": "req_uuid_12345",
    "timestamp": "2026-01-13T18:00:00Z"
  }
}

// 401 Unauthorized - Missing/invalid token
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid or expired authentication token",
    "requestId": "req_uuid_12345"
  }
}

// 429 Too Many Requests - Rate limit exceeded
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again later",
    "retryAfter": 60,
    "requestId": "req_uuid_12345"
  },
  "headers": {
    "X-RateLimit-Limit": "100",
    "X-RateLimit-Remaining": "0",
    "X-RateLimit-Reset": "1673640000",
    "Retry-After": "60"
  }
}

// 503 Service Unavailable - Server maintenance
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Service temporarily unavailable due to maintenance",
    "retryable": true,
    "estimatedDowntime": "15 minutes"
  }
}
```

### iOS Client Error Handling

```swift
extension NetworkService {
    func getTrips(page: Int, limit: Int) async throws -> TripListResponse {
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw NetworkError.invalidResponse
        }
        
        switch httpResponse.statusCode {
        case 200...299:
            return try JSONDecoder().decode(TripListResponse.self, from: data)
            
        case 400:
            let error = try JSONDecoder().decode(APIError.self, from: data)
            throw NetworkError.badRequest(error.message)
            
        case 401:
            // Token expired, trigger re-auth
            await authManager.refreshToken()
            return try await getTrips(page: page, limit: limit) // Retry once
            
        case 429:
            let error = try JSONDecoder().decode(APIError.self, from: data)
            let retryAfter = error.retryAfter ?? 60
            
            // Wait and retry
            try await Task.sleep(nanoseconds: UInt64(retryAfter * 1_000_000_000))
            return try await getTrips(page: page, limit: limit)
            
        case 500...599:
            // Server error - retry with backoff
            throw NetworkError.serverError(httpResponse.statusCode, retryable: true)
            
        default:
            throw NetworkError.unknown(httpResponse.statusCode)
        }
    }
}
```

### API Versioning Strategy

**Interviewer:** "How would you handle API versioning?"

**You:** "I'd use URL path versioning for clarity:

```swift
// Current version
let baseURL = "https://api.uber.com/v1"

// When breaking change needed
let baseURLv2 = "https://api.uber.com/v2"

// Client determines version based on app version
let apiVersion = appVersion >= "5.0" ? "v2" : "v1"
let baseURL = "https://api.uber.com/\(apiVersion)"
```

**Breaking changes that require new version:**
- Removing fields (`mapSnapshotUrl` removed)
- Changing field types (`fare` from String to Object)
- Changing response structure

**Non-breaking changes (same version):**
- Adding optional fields (`driver.photoUrl` added)
- Adding new endpoints (`GET /trips/{id}/timeline`)
- Deprecating fields (keep but mark as deprecated)

**Deprecation example:**
```json
{
  "data": {
    "trips": [...],
    "deprecated_field": "value"  // Still returned
  },
  "meta": {
    "deprecations": [
      {
        "field": "deprecated_field",
        "message": "Use new_field instead",
        "sunsetDate": "2026-06-01"
      }
    ]
  }
}
```

**iOS handling:**
```swift
struct Trip: Codable {
    @available(*, deprecated, message: "Use newField")
    let deprecatedField: String?
    let newField: String?
}
```"

### Mobile-Specific Optimizations

**1. Response Compression**

```swift
var request = URLRequest(url: url)
request.setValue("gzip, deflate", forHTTPHeaderField: "Accept-Encoding")

// Server responds with:
// Content-Encoding: gzip
// (URLSession automatically decompresses)
```

**2. Conditional Requests (Save Bandwidth)**

```swift
// First request
request.setValue("gzip", forHTTPHeaderField: "Accept-Encoding")

// Server responds
// ETag: "abc123"
// Cache-Control: max-age=3600

// Subsequent request (within 1 hour)
request.setValue("abc123", forHTTPHeaderField: "If-None-Match")

// Server responds
// 304 Not Modified (no body, use cache)
```

**3. Field Selection (Optional Optimization)**

```
GET /v1/trips?fields=id,pickupLocation,dropoffLocation,fare,status

// Returns minimal data for list view
// Full data fetched only for detail view
```

**4. Batch Requests (Advanced)**

```swift
// Instead of 3 separate requests:
// GET /trips?page=1
// GET /user/profile
// GET /promotions

// Single batch request:
POST /v1/batch
{
  "requests": [
    {"method": "GET", "path": "/trips?page=1"},
    {"method": "GET", "path": "/user/profile"},
    {"method": "GET", "path": "/promotions"}
  ]
}

// Response
{
  "responses": [
    {"status": 200, "body": {...}},
    {"status": 200, "body": {...}},
    {"status": 200, "body": {...}}
  ]
}
```

### Rate Limiting (Client Perspective)

**Server headers:**
```
X-RateLimit-Limit: 100       // Requests per hour
X-RateLimit-Remaining: 95     // Remaining this hour
X-RateLimit-Reset: 1673640000 // Unix timestamp when resets
```

**iOS tracking:**
```swift
actor RateLimitTracker {
    private var limitPerHour: Int = 100
    private var remaining: Int = 100
    private var resetTime: Date = Date()
    
    func updateFromHeaders(_ headers: [String: String]) {
        if let limit = headers["X-RateLimit-Limit"],
           let limitInt = Int(limit) {
            limitPerHour = limitInt
        }
        
        if let rem = headers["X-RateLimit-Remaining"],
           let remInt = Int(rem) {
            remaining = remInt
        }
        
        if let reset = headers["X-RateLimit-Reset"],
           let timestamp = Double(reset) {
            resetTime = Date(timeIntervalSince1970: timestamp)
        }
    }
    
    func canMakeRequest() -> Bool {
        if Date() > resetTime {
            remaining = limitPerHour  // Reset
        }
        return remaining > 0
    }
}
```

### Authentication Headers

```swift
// Standard Bearer token
request.setValue("Bearer \(accessToken)", forHTTPHeaderField: "Authorization")

// Additional headers for security
request.setValue(UUID().uuidString, forHTTPHeaderField: "X-Request-ID")
request.setValue("iOS/5.0", forHTTPHeaderField: "User-Agent")
request.setValue("en-US", forHTTPHeaderField: "Accept-Language")
```

### Interview Q&A: API Design

**Q:** "Why include both `totalPages` and `hasMore` in pagination?"

**A:** 
> "`totalPages` enables page-based UI (\"Page 1 of 15\"), while `hasMore` is simpler for infinite scroll (\"Load More\" button). Including both gives client flexibility. If I had to choose one, I'd keep `hasMore` for mobile (simpler, less UI chrome)."

**Q:** "Should we include driver details in list endpoint or separate call?"

**A:**
> "Trade-off decision:
> 
> **Include driver in list (my choice):**
> - ✅ Single request (faster, less battery)
> - ✅ Can show driver name/photo in list
> - ❌ Larger response payload
> 
> **Separate endpoint:**
> - ✅ Smaller list payload
> - ✅ Can cache driver separately
> - ❌ N+1 problem (20 trips = 20 driver requests)
> 
> For Uber, driver info is essential for trip list, so I'd include it. Alternative: Use field selection for minimal driver data (`id`, `name`, `photoUrl` only)."

**Q:** "How would you design API for pull-to-refresh vs pagination?"

**A:**
> "Same endpoint, different usage:
> 
> **Pull-to-refresh:**
> ```swift
> // Always fetch page 1
> let response = try await api.getTrips(page: 1)
> viewModel.trips = response.trips  // Replace all
> ```
> 
> **Pagination:**
> ```swift
> // Fetch next page
> let response = try await api.getTrips(page: currentPage + 1)
> viewModel.trips.append(contentsOf: response.trips)  // Append
> ```
> 
> **Optimization:** Could add `since` timestamp for pull-to-refresh to only fetch new trips:
> ```
> GET /trips?since=2026-01-13T10:00:00Z
> ```
> Returns only trips after that time (more efficient)."

---

## 45-55 min: Deep Dives & Trade-offs

### 1. Caching Strategy

**Interviewer:** "Why did you cache only page 1 to disk?"

**You:** "Great question. Here's my reasoning:

| Data | Memory Cache | Disk Cache | TTL | Reasoning |
|------|--------------|------------|-----|-----------|
| Page 1 | ✅ | ✅ | 1 hour | Most accessed, offline requirement |
| Page 2+ | ✅ | ❌ | 30 min | Less likely revisited, save disk space |
| Images | ❌ | ✅ (LRU) | 7 days | Large files, separate cache |

**Trade-offs:**
- ✅ Balances offline UX with storage constraints
- ✅ Page 1 available offline (covers 80% of use cases)
- ❌ Older trips require network
- **Alternative:** Could cache all pages but implement size-based LRU eviction"

### 2. Thread Safety

**Interviewer:** "How do you ensure thread safety in your cache?"

**You:** "I used Swift's `actor` for the CacheService:

```swift
actor CacheService {
    // All methods are async and serialized automatically
}
```

**Why:**
- ✅ Actor ensures serial access, no race conditions
- ✅ Compiler-enforced thread safety
- ✅ No manual locks needed
- ❌ Slight async overhead, but negligible for caching

**Alternative:** Could use `DispatchQueue` with barriers, but actor is more modern and safer."

### 3. Pagination Race Conditions

**Interviewer:** "What if user scrolls fast and triggers multiple page loads?"

**You:** "I prevent this with a guard check:

```swift
func loadMoreTrips() async {
    guard state != .loadingMore else { return } // Debounce
    state = .loadingMore
    // ...
}
```

**Additional safeguards:**
1. **Debouncing:** Only trigger at row count - 5, not every scroll
2. **State check:** `state != .loadingMore` prevents concurrent calls
3. **Deduplication:** Filter out duplicate trips by ID before appending

**Trade-off:**
- ✅ Prevents duplicate network calls
- ✅ Avoids corrupted state
- ❌ Very fast scrollers might perceive slight delay (acceptable)"

### 4. Error Handling

**You:** "For production, I'd add:

**Network Errors:**
```swift
catch {
    if error is URLError {
        // Retry with exponential backoff
        retry(after: 2 ** retryCount)
    } else {
        // Show user-friendly error
        state = .error(UserFacingError.networkFailed)
    }
}
```

**Fallback Strategy:**
- No internet → Show cached data with banner "You're offline"
- Server error (5xx) → Retry 3 times with backoff
- Client error (4xx) → Show error, don't retry
- Timeout → Retry once, then fail gracefully"

### 5. Performance - Memory Management

**Interviewer:** "What if user has 10,000 trips?"

**You:** "Several optimizations:

**1. Limit in-memory trips:**
```swift
// Only keep latest 200 in memory
if trips.count > 200 {
    trips = Array(trips.suffix(200))
    currentPage = max(1, currentPage - 8)
}
```

**2. Cell reusing** (already using `dequeueReusableCell`)

**3. NSCache auto-eviction** (set `countLimit = 50`)

**4. Lazy image loading:**
```swift
class TripCell {
    func configure(with trip: Trip) {
        // Load image only when cell appears
        ImageLoader.shared.load(trip.mapSnapshotURL) { image in
            self.imageView.image = image
        }
    }
}
```

**Trade-offs:**
- ✅ Constant memory usage
- ❌ User can't scroll to trip #5000 without re-fetching
- **Better Alternative:** Virtual scrolling or on-demand fetch by date range"

### 6. Testability

**You:** "Every layer is protocol-based for mocking:

```swift
// ViewModel tests
func testLoadTrips() async {
    let mockRepo = MockTripRepository()
    mockRepo.tripsToReturn = [Trip.mock1, Trip.mock2]
    
    let vm = TripListViewModel(repository: mockRepo)
    await vm.loadInitialTrips()
    
    XCTAssertEqual(vm.trips.count, 2)
    XCTAssertTrue(mockRepo.fetchTripsCalled)
}

// Repository tests
func testOfflineFirst() async {
    let mockCache = MockCacheService()
    mockCache.cachedResponse = TripListResponse.mock
    
    let repo = TripRepository(network: mockNetwork, cache: mockCache)
    let response = try await repo.fetchTrips(page: 1)
    
    // Should return cached data without network call
    XCTAssertEqual(response.trips.count, mockCache.cachedResponse.trips.count)
    XCTAssertFalse(mockNetwork.getTripsWasCalled)
}
```

**Coverage targets:**
- ViewModels: 90%+ (pure logic)
- Repository: 85%+ (offline/online paths)
- Network: 70%+ (mock URLSession)
- UI: Snapshot tests for visual regressions"

---

## Common Uber L4 Follow-Up Questions

### Q1: "How would you handle real-time trip status updates?"

**Answer:**
"I'd introduce a WebSocket connection for active trips:

```swift
class TripRepository {
    private let webSocketService: WebSocketService
    
    func subscribeToTripUpdates(tripId: String) -> AsyncStream<TripUpdate> {
        webSocketService.subscribe(topic: "trips/\(tripId)")
    }
}

// In ViewModel
for await update in repository.subscribeToTripUpdates(tripId: activeTrip.id) {
    updateTrip(update)
}
```

**For completed trips list:**
- Use polling with 30s interval (less critical than active)
- Or push notifications to trigger refresh
- **Trade-off:** WebSocket keeps connection alive (battery drain) vs real-time UX"

### Q2: "How would you optimize for slow networks (2G)?"

**Answer:**
"Several strategies:

1. **Request prioritization:**
   - Fetch trip metadata first (small payload)
   - Lazy-load images and map snapshots

2. **Response compression:**
   - Request gzip encoding
   - Trim unnecessary API fields

3. **Adaptive pagination:**
   ```swift
   let pageSize = NetworkMonitor.shared.isSlowNetwork ? 10 : 20
   ```

4. **Prefetching:**
   - Preload page 2 when user opens page 1

5. **Image quality:**
   - Request low-res thumbnails for list
   - High-res only for detail view

**Trade-off:** Complexity vs UX on slow networks (important for Uber's global users)"

### Q3: "How do you handle data consistency across app restarts?"

**Answer:**
"Data freshness strategy:

```swift
// In Repository
func fetchTrips(page: Int) async throws -> TripListResponse {
    let cached = await cacheService.get(key: cacheKey)
    
    // Check cache age
    if let cached = cached, cached.age < 5.minutes {
        return cached.data
    }
    
    // Cache stale or missing → fetch fresh
    return try await fetchAndCache(page: page)
}
```

**On app restart:**
1. Load cached data immediately (fast launch)
2. Silently refresh in background
3. Update UI only if data changed (avoid flicker)

**Trade-off:** Stale data for 5 min vs fewer server calls"

---

## 🚨 L4 Mistakes to Avoid

| Mistake | Impact | How to Avoid |
|---------|--------|--------------|
| Not discussing offline first | Shows lack of mobile thinking | Always mention cache strategy |
| Ignoring pagination edge cases | Production bugs evident | Walk through "last page" & "duplicate triggers" |
| No error handling | Not production-ready | Discuss retry, fallback, user messaging |
| Over-engineering | Senior flag (L5/L6 territory) | Start simple, add complexity only if asked |
| Can't explain trade-offs | Memorized solutions | For every choice, state alternatives |
| Skipping HLD diagram | Unclear communication | ALWAYS draw layers first |

---

## Interview Success Checklist

✅ Drew architecture diagram in first 15 minutes
✅ Explained data flow with arrows
✅ Discussed offline-first approach
✅ Showed thread-safe cache with actor
✅ Handled pagination race conditions
✅ Mentioned error handling & retry
✅ Compared alternatives for key decisions
✅ Wrote testable, protocol-based code
✅ Discussed performance for large datasets
✅ Kept code production-quality (not pseudocode)

---

**Time spent: ~50 minutes. Use remaining 10 for questions.**

> [!TIP]
> Practice this design 3 times OUT LOUD before your real interview. Time yourself!
