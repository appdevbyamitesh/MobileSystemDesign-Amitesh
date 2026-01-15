# 🏗️ Real-World Architecture Application

> **Designing Features with Multiple Architectures - A Practical Comparison**

---

## 📱 Case Study: Uber Trip List Screen

Let's design the same feature using two different architectures and compare complexity, scalability, and testability.

### Feature Requirements

- Display list of past trips
- Pull-to-refresh
- Pagination (infinite scroll)
- Tap to view trip details
- Filter trips by date range
- Loading/error/empty states

---

## 🔵 Implementation 1: MVVM+C

### File Structure (4 files)

```
TripList/
├── TripListCoordinator.swift
├── TripListViewModel.swift
├── TripListViewController.swift
└── TripViewModel.swift (display model)
```

### Coordinator

```swift
protocol TripListCoordinatorProtocol: AnyObject {
    func showTripDetail(_ trip: Trip)
    func showDateFilter(selected: DateRange?, completion: @escaping (DateRange?) -> Void)
}

final class TripListCoordinator: TripListCoordinatorProtocol {
    private weak var navigationController: UINavigationController?
    private let tripService: TripServiceProtocol
    
    init(navigationController: UINavigationController, tripService: TripServiceProtocol) {
        self.navigationController = navigationController
        self.tripService = tripService
    }
    
    func start() {
        let viewModel = TripListViewModel(tripService: tripService, coordinator: self)
        let viewController = TripListViewController(viewModel: viewModel)
        navigationController?.pushViewController(viewController, animated: true)
    }
    
    func showTripDetail(_ trip: Trip) {
        let detailCoordinator = TripDetailCoordinator(
            navigationController: navigationController!,
            trip: trip
        )
        detailCoordinator.start()
    }
    
    func showDateFilter(selected: DateRange?, completion: @escaping (DateRange?) -> Void) {
        let filterVC = DateFilterViewController(selected: selected, completion: completion)
        let nav = UINavigationController(rootViewController: filterVC)
        navigationController?.present(nav, animated: true)
    }
}
```

### ViewModel

```swift
@MainActor
final class TripListViewModel: ObservableObject {
    
    // MARK: - State
    enum State: Equatable {
        case idle, loading, loaded, error(String), empty
    }
    
    @Published private(set) var trips: [TripViewModel] = []
    @Published private(set) var state: State = .idle
    @Published private(set) var selectedFilter: DateRange?
    
    // MARK: - Dependencies
    private let tripService: TripServiceProtocol
    private weak var coordinator: TripListCoordinatorProtocol?
    
    // MARK: - Pagination
    private var rawTrips: [Trip] = []
    private var currentPage = 0
    private var hasMore = true
    
    init(tripService: TripServiceProtocol, coordinator: TripListCoordinatorProtocol) {
        self.tripService = tripService
        self.coordinator = coordinator
    }
    
    // MARK: - Actions
    func loadTrips() async {
        guard state != .loading else { return }
        state = .loading
        currentPage = 0
        
        do {
            let result = try await tripService.getTrips(page: 0, filter: selectedFilter)
            rawTrips = result.trips
            hasMore = result.hasMore
            trips = rawTrips.map(TripViewModel.init)
            state = trips.isEmpty ? .empty : .loaded
        } catch {
            state = .error(error.localizedDescription)
        }
    }
    
    func loadMore() async {
        guard state == .loaded, hasMore else { return }
        
        do {
            let result = try await tripService.getTrips(page: currentPage + 1, filter: selectedFilter)
            rawTrips.append(contentsOf: result.trips)
            hasMore = result.hasMore
            currentPage += 1
            trips = rawTrips.map(TripViewModel.init)
        } catch {
            // Silently fail pagination
        }
    }
    
    func refresh() async {
        await loadTrips()
    }
    
    func didSelectTrip(at index: Int) {
        guard index < rawTrips.count else { return }
        coordinator?.showTripDetail(rawTrips[index])
    }
    
    func didTapFilter() {
        coordinator?.showDateFilter(selected: selectedFilter) { [weak self] newFilter in
            self?.selectedFilter = newFilter
            Task { await self?.loadTrips() }
        }
    }
}
```

### ViewController (abbreviated)

```swift
final class TripListViewController: UIViewController {
    private let viewModel: TripListViewModel
    private var cancellables = Set<AnyCancellable>()
    
    init(viewModel: TripListViewModel) {
        self.viewModel = viewModel
        super.init(nibName: nil, bundle: nil)
    }
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        bindViewModel()
        Task { await viewModel.loadTrips() }
    }
    
    private func bindViewModel() {
        viewModel.$trips
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in self?.tableView.reloadData() }
            .store(in: &cancellables)
        
        viewModel.$state
            .receive(on: DispatchQueue.main)
            .sink { [weak self] state in self?.handleState(state) }
            .store(in: &cancellables)
    }
}
```

### MVVM+C Summary

| Metric | Value |
|--------|-------|
| **Files** | 4 files |
| **Lines of Code** | ~250 lines |
| **Testable Components** | ViewModel (fully), Coordinator (navigation only) |
| **Boilerplate** | Low |

---

## 🔴 Implementation 2: Clean Architecture

### File Structure (7 files)

```
TripList/
├── Presentation/
│   ├── TripListCoordinator.swift
│   ├── TripListViewModel.swift
│   ├── TripListViewController.swift
│   └── TripCellViewModel.swift
├── Domain/
│   ├── GetTripsUseCase.swift
│   └── TripRepositoryProtocol.swift (in shared Domain)
└── Data/
    └── TripRepository.swift (in shared Data)
```

### Domain Layer - Use Case

```swift
// Domain/GetTripsUseCase.swift
final class GetTripsUseCase {
    private let repository: TripRepositoryProtocol
    
    init(repository: TripRepositoryProtocol) {
        self.repository = repository
    }
    
    func execute(page: Int, filter: DateRange?) async -> Result<PaginatedTrips, DomainError> {
        do {
            let trips = try await repository.getTrips(page: page, filter: filter)
            return .success(trips)
        } catch {
            return .failure(.networkError)
        }
    }
}

// Domain/TripRepositoryProtocol.swift
protocol TripRepositoryProtocol {
    func getTrips(page: Int, filter: DateRange?) async throws -> PaginatedTrips
}

struct PaginatedTrips {
    let trips: [Trip]
    let hasMore: Bool
}
```

### Data Layer - Repository

```swift
// Data/TripRepository.swift
final class TripRepository: TripRepositoryProtocol {
    private let apiService: APIServiceProtocol
    private let cache: CacheServiceProtocol
    
    init(apiService: APIServiceProtocol, cache: CacheServiceProtocol) {
        self.apiService = apiService
        self.cache = cache
    }
    
    func getTrips(page: Int, filter: DateRange?) async throws -> PaginatedTrips {
        // Check cache for first page with no filter
        if page == 0 && filter == nil, let cached: PaginatedTrips = cache.get(key: "trips") {
            return cached
        }
        
        let endpoint = TripEndpoint.list(page: page, startDate: filter?.start, endDate: filter?.end)
        let data = try await apiService.request(endpoint)
        let response = try JSONDecoder().decode(TripListResponse.self, from: data)
        
        let result = PaginatedTrips(
            trips: response.trips.map { $0.toDomain() },
            hasMore: response.nextPage != nil
        )
        
        if page == 0 && filter == nil {
            cache.set(key: "trips", value: result)
        }
        
        return result
    }
}
```

### Presentation Layer - ViewModel

```swift
// Presentation/TripListViewModel.swift
@MainActor
final class TripListViewModel: ObservableObject {
    
    enum State: Equatable {
        case idle, loading, loaded, error(String), empty
    }
    
    @Published private(set) var trips: [TripCellViewModel] = []
    @Published private(set) var state: State = .idle
    @Published private(set) var selectedFilter: DateRange?
    
    private let getTripsUseCase: GetTripsUseCase
    private weak var coordinator: TripListCoordinatorProtocol?
    
    private var rawTrips: [Trip] = []
    private var currentPage = 0
    private var hasMore = true
    
    init(getTripsUseCase: GetTripsUseCase, coordinator: TripListCoordinatorProtocol) {
        self.getTripsUseCase = getTripsUseCase
        self.coordinator = coordinator
    }
    
    func loadTrips() async {
        guard state != .loading else { return }
        state = .loading
        currentPage = 0
        
        let result = await getTripsUseCase.execute(page: 0, filter: selectedFilter)
        
        switch result {
        case .success(let paginated):
            rawTrips = paginated.trips
            hasMore = paginated.hasMore
            trips = rawTrips.map(TripCellViewModel.init)
            state = trips.isEmpty ? .empty : .loaded
        case .failure(let error):
            state = .error(error.localizedDescription)
        }
    }
    
    func loadMore() async {
        guard state == .loaded, hasMore else { return }
        
        let result = await getTripsUseCase.execute(page: currentPage + 1, filter: selectedFilter)
        
        if case .success(let paginated) = result {
            rawTrips.append(contentsOf: paginated.trips)
            hasMore = paginated.hasMore
            currentPage += 1
            trips = rawTrips.map(TripCellViewModel.init)
        }
    }
    
    func didSelectTrip(at index: Int) {
        guard index < rawTrips.count else { return }
        coordinator?.showTripDetail(rawTrips[index])
    }
}
```

### Clean Architecture Summary

| Metric | Value |
|--------|-------|
| **Files** | 7 files |
| **Lines of Code** | ~350 lines |
| **Testable Components** | ViewModel, UseCase, Repository (all isolated) |
| **Boilerplate** | Medium |

---

## ⚖️ Comparison: MVVM+C vs Clean Architecture

### Complexity

| Aspect | MVVM+C | Clean Architecture |
|--------|--------|-------------------|
| **File count** | 4 | 7 |
| **Lines of code** | ~250 | ~350 |
| **Learning curve** | Medium | High |
| **Setup time** | Faster | Slower |

### Scalability

| Aspect | MVVM+C | Clean Architecture |
|--------|--------|-------------------|
| **Adding features** | Moderate | Easy (clear boundaries) |
| **Sharing logic** | Via services | Via Use Cases/Repositories |
| **Changing data source** | Medium effort | Swap Repository only |
| **Team growth** | Good (5-10 devs) | Excellent (10-20 devs) |

### Testability

| Component | MVVM+C | Clean Architecture |
|-----------|--------|-------------------|
| **Business logic** | ViewModel tests | UseCase tests (pure) |
| **Data layer** | Mock services | Mock Repository |
| **UI logic** | ViewModel tests | ViewModel tests |
| **Integration** | Possible | Difficult (more mocks) |

---

## 🎯 Architecture Recommendations

### For a Startup App

```
Recommended: MVVM+C

Why:
- Faster time to market
- Lower complexity
- Easier to iterate
- Sufficient testability for small team
- Can migrate to Clean Arch later if needed
```

### For an Amazon-Scale App

```
Recommended: Clean Architecture

Why:
- Clear boundaries for 15-30 devs
- Swappable data sources
- Highly testable domain layer
- Long-term maintainability
- Supports complex business logic
```

### For an Uber-Scale App

```
Recommended: RIBs

Why:
- 50+ developers need maximum isolation
- Tree-based state management
- Viewless RIBs for state machines
- Cross-platform alignment with Android
- Team ownership per RIB
```

---

## 📱 Case Study 2: Social Media Feed

### Feature Requirements

- Infinite scrolling feed
- Like/comment/share actions
- Story carousel at top
- Pull-to-refresh
- Optimistic updates for likes
- Multiple post types (text, image, video, poll)

### Architecture Selection

| Scenario | Architecture | Justification |
|----------|--------------|---------------|
| **2-3 person startup** | MVVM+C | Fast iteration, simple posts |
| **10-person team, Series A** | Clean Arch | Need testable domain, scaling |
| **50+ engineers, mature product** | RIBs | Team isolation, complex feed logic |

### Key Design Decisions

**Feed Pagination:**
```swift
// Cursor-based pagination in ViewModel
func loadMore() async {
    guard !isLoadingMore, let cursor = nextCursor else { return }
    isLoadingMore = true
    
    let result = await getPosts(after: cursor)
    // Append to existing posts
}
```

**Optimistic Like Update:**
```swift
// Immediately update UI, revert on failure
func toggleLike(postId: String) async {
    // 1. Optimistic update
    updatePostLocally(postId, isLiked: true)
    
    // 2. Server call
    let result = await likePostUseCase.execute(postId)
    
    // 3. Handle failure
    if case .failure = result {
        updatePostLocally(postId, isLiked: false)
        showError("Failed to like post")
    }
}
```

**Multiple Post Types:**
```swift
// Use protocol + enum for flexibility
enum PostContent {
    case text(String)
    case image(URL)
    case video(URL, thumbnailURL: URL)
    case poll(PollData)
}

struct Post {
    let id: String
    let author: User
    let content: PostContent
    let metrics: PostMetrics
}
```

---

## 📋 Case Study 3: Booking/Checkout Flow

### Feature Requirements

- Multi-step wizard (dates → room → extras → payment → confirm)
- State persistence across back navigation
- Validation at each step
- Order summary sidebar
- Payment integration
- Confirmation with deep link sharing

### Architecture Considerations

**MVVM+C Approach:**
- One Coordinator owns entire flow
- ViewModels communicate via shared BookingState

```swift
class BookingFlowCoordinator {
    private var bookingState = BookingState()
    
    func showDateSelection() {
        let vm = DateSelectionViewModel(state: bookingState) { [weak self] in
            self?.showRoomSelection()
        }
        // Present
    }
    
    func showRoomSelection() {
        let vm = RoomSelectionViewModel(state: bookingState) { [weak self] in
            self?.showExtras()
        }
        // Present
    }
}
```

**Clean Architecture Approach:**
- BookingState in Domain layer
- Each step has its own UseCase
- Repository persists draft booking

```swift
// Domain
class SelectDatesUseCase {
    func execute(dates: DateRange) -> Result<BookingDraft, Error>
}

class SelectRoomUseCase {
    func execute(room: Room, draft: BookingDraft) -> Result<BookingDraft, Error>
}

class ConfirmBookingUseCase {
    func execute(draft: BookingDraft, payment: PaymentInfo) -> Result<Booking, Error>
}
```

**RIBs Approach (if Uber-scale):**
- BookingRIB as parent
- Child RIBs for each step
- State flows via Listener protocols

```
BookingRIB
├── DateSelectionRIB
├── RoomSelectionRIB
├── ExtrasRIB
├── PaymentRIB (viewless for payment state)
└── ConfirmationRIB
```

---

## 🎓 Key Takeaways

1. **No "best" architecture** — Context determines the right choice
2. **Start simple, evolve** — MVVM+C → Clean Arch → RIBs as you scale
3. **Team size matters most** — Architecture must match team coordination needs
4. **Testability has a cost** — More isolation = more files = more setup
5. **Navigation complexity drives decisions** — Simple nav? Simple arch.

### Interview Answer Template

> "For [FEATURE] with [TEAM SIZE] developers, I would choose [ARCHITECTURE] because:
> 
> 1. **Testability**: [How it meets testing needs]
> 2. **Scalability**: [How it supports team growth]
> 3. **Complexity fit**: [Why it matches the feature's complexity]
> 4. **Alternative considered**: I considered [OTHER] but [REASON NOT TO USE]"
