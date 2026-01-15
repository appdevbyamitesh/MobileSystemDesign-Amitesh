# 🚗 RIBs (Router-Interactor-Builder)

> **Uber's Architecture for Massive, Multi-Team Mobile Applications**

---

## 1️⃣ What It Is (Simple English)

RIBs is Uber's architecture designed for **very large apps** with **many independent teams**. It focuses on:

- **R**outer – Attaches/detaches child RIBs, manages the RIB tree
- **I**nteractor – Business logic, the brain of the RIB
- **B**uilder – Creates the RIB and injects dependencies

Optional: **Presenter** and **View** (only when UI is needed)

### Real-Life Analogy: Large Corporation

| Role | Corporation | iOS |
|------|-------------|-----|
| **Router** | HR (hires/fires departments) | Attaches/detaches child RIBs |
| **Interactor** | CEO (makes business decisions) | Business logic |
| **Builder** | Recruiter (assembles teams) | Creates & wires components |
| **View** | Customer-facing office | UI (optional) |

Key insight: Some departments (RIBs) don't need a customer-facing office (View). An analytics RIB or authentication state RIB might be "viewless."

### The Key Insights

> **1. Business logic drives the app, not views.**
> 
> **2. RIBs form a tree structure, similar to UIKit's view hierarchy, but for business logic.**
> 
> **3. Not every RIB needs a View—viewless RIBs manage state and logic.**

---

## 2️⃣ Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                          RIB Structure                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌───────────┐                                               │
│    │  Builder  │────creates────┐                               │
│    └───────────┘               │                               │
│                                ▼                               │
│    ┌─────────────────────────────────────────┐                 │
│    │                 RIB                      │                 │
│    │  ┌──────────┐    ┌───────────┐          │                 │
│    │  │ Router   │◄──►│ Interactor│          │                 │
│    │  └────┬─────┘    └─────┬─────┘          │                 │
│    │       │                │                 │                 │
│    │  child RIBs      ┌─────▼─────┐          │                 │
│    │                  │ Presenter │ (optional)│                 │
│    │                  └─────┬─────┘          │                 │
│    │                        │                 │                 │
│    │                  ┌─────▼─────┐          │                 │
│    │                  │   View    │ (optional)│                 │
│    │                  └───────────┘          │                 │
│    └─────────────────────────────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### RIB Tree Structure

```
RootRIB (LoggedOut)
│
├── LoggedOut RIB
│   ├── Login RIB
│   └── SignUp RIB
│
└── LoggedIn RIB (after auth)
    ├── Home RIB
    │   ├── Feed RIB
    │   ├── Search RIB
    │   └── Promotions RIB (viewless)
    │
    ├── Ride RIB
    │   ├── RequestRide RIB
    │   ├── Matching RIB (viewless)
    │   ├── OnTrip RIB
    │   └── Rating RIB
    │
    └── Profile RIB
        ├── Settings RIB
        └── PaymentMethods RIB
```

### Component Responsibilities

| Component | Responsibilities |
|-----------|------------------|
| **Builder** | Creates all components, injects dependencies, returns Router |
| **Router** | Manages RIB lifecycle (attach/detach children), holds child Routers |
| **Interactor** | Business logic, reacts to events, tells Router when to navigate |
| **Presenter** | Transforms Interactor output to View input (optional) |
| **View** | UI elements, captures user input (optional) |

---

## 3️⃣ Data & Control Flow

### Attaching a Child RIB (User Requests Ride)

```
┌──────────────────────────────────────────────────────────────────┐
│ USER TAPS "REQUEST RIDE"                                         │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. View: Button tap calls Interactor                            │
│  2. Interactor: Business validation, API call                    │
│  3. Interactor: Tells Router to attach RequestRide child         │
│  4. Router: Calls RequestRideBuilder.build(listener:)            │
│  5. Builder: Creates child RIB, injects dependencies             │
│  6. Router: Attaches child Router, adds child View               │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram (Attaching Child RIB)

```
User     View      Interactor     Router        Builder      ChildRIB
 │        │           │            │              │            │
 │──tap───►│           │            │              │            │
 │        │─rideReq()─►│            │              │            │
 │        │           │──attach()──►│              │            │
 │        │           │            │───build()────►│            │
 │        │           │            │◄─────Router───│            │
 │        │           │            │─attachChild()─┼───────────►│
 │        │◄──────────│────────────│───presentVC───│            │
 │◄───────│           │            │              │            │
```

### Parent-Child Communication via Listener

```swift
// Child RIB exposes a Listener protocol
protocol RequestRideListener: AnyObject {
    func requestRideDidComplete(trip: Trip)
    func requestRideDidCancel()
}

// Parent Interactor implements the Listener
class RideInteractor: RideInteractable, RequestRideListener {
    func requestRideDidComplete(trip: Trip) {
        // Detach RequestRide, attach OnTrip
        router?.detachRequestRide()
        router?.attachOnTrip(trip: trip)
    }
    
    func requestRideDidCancel() {
        router?.detachRequestRide()
    }
}
```

---

## 4️⃣ Strengths

✅ **Viewless RIBs** – Business logic can exist without UI

✅ **True isolation** – Each RIB is a self-contained unit

✅ **Multi-team scalability** – Teams own RIBs, not screens

✅ **Predictable state** – RIB tree is the source of truth for app state

✅ **Deep linking** – Navigate to any RIB state programmatically

✅ **Cross-platform** – Same architecture for iOS and Android

✅ **Dependency scoping** – Dependencies injected per-RIB, no singletons

---

## 5️⃣ Limitations & Failure Points

### ❌ Very Steep Learning Curve

```swift
// Understanding RIBs requires learning:
// 1. The RIB tree mental model
// 2. Rx/Combine streams everywhere
// 3. DI framework (NILs, Needle, custom)
// 4. Build-time code generation
// 5. Parent-child Listener patterns
// 6. Viewless RIB concept
```

### ❌ Overkill for Small Apps

| App Type | RIBs Fit |
|----------|----------|
| Startup MVP | ❌ Massive overkill |
| 10-screen app | ❌ Overkill |
| 30-screen app | ⚠️ Maybe |
| 100+ screen enterprise app | ✅ Perfect |

### ❌ Complex Testing Setup

```swift
// Testing requires:
// - Mock builders
// - Mock routers
// - Mock interactors
// - Mock listeners
// - Much more infrastructure than MVVM
```

### ❌ Debugging Challenges

- RIB tree is invisible in Xcode's view hierarchy
- State is spread across many Interactors
- Stack traces span multiple RIBs

---

## 6️⃣ iOS-Specific Considerations

### Navigation Handling

```swift
// Router manages UIViewController presentation
final class RideRouter: ViewableRouter<RideInteractable, RideViewControllable>, RideRouting {
    
    private var requestRideRouter: ViewableRouting?
    private var onTripRouter: ViewableRouting?
    
    private let requestRideBuilder: RequestRideBuildable
    private let onTripBuilder: OnTripBuildable
    
    init(
        interactor: RideInteractable,
        viewController: RideViewControllable,
        requestRideBuilder: RequestRideBuildable,
        onTripBuilder: OnTripBuildable
    ) {
        self.requestRideBuilder = requestRideBuilder
        self.onTripBuilder = onTripBuilder
        super.init(interactor: interactor, viewController: viewController)
    }
    
    func attachRequestRide() {
        guard requestRideRouter == nil else { return }
        
        let router = requestRideBuilder.build(withListener: interactor)
        requestRideRouter = router
        attachChild(router)
        viewController.present(router.viewControllable)
    }
    
    func detachRequestRide() {
        guard let router = requestRideRouter else { return }
        
        viewController.dismiss(router.viewControllable)
        detachChild(router)
        requestRideRouter = nil
    }
}
```

### Viewless RIBs (Unique to RIBs)

```swift
// A RIB that manages state without any UI
final class AnalyticsRouter: Router<AnalyticsInteractable>, AnalyticsRouting {
    // No viewController property
    // Just manages analytics state
}

final class AnalyticsInteractor: Interactor, AnalyticsInteractable {
    // Tracks user journey across the app
    // Sends events to analytics backend
    // No UI involvement
    
    func trackRideStarted(_ ride: Ride) {
        analyticsService.track(.rideStarted(ride))
    }
    
    func trackPaymentCompleted(_ payment: Payment) {
        analyticsService.track(.paymentCompleted(payment))
    }
}
```

### Concurrency with Combine/RxSwift

```swift
// RIBs often use reactive streams heavily
final class FeedInteractor: PresentableInteractor<FeedPresentable>, FeedInteractable {
    
    private let feedService: FeedServiceProtocol
    private var cancellables = Set<AnyCancellable>()
    
    override func didBecomeActive() {
        super.didBecomeActive()
        
        // Subscribe to feed updates
        feedService.feedPublisher
            .receive(on: DispatchQueue.main)
            .sink { [weak self] posts in
                self?.presenter.displayPosts(posts)
            }
            .store(in: &cancellables)
        
        // React to app state changes
        appStateService.statePublisher
            .filter { $0 == .foreground }
            .sink { [weak self] _ in
                self?.refreshFeed()
            }
            .store(in: &cancellables)
    }
    
    override func willResignActive() {
        super.willResignActive()
        cancellables.removeAll()
    }
}
```

### Memory Management

```swift
// RIBs have strict lifecycle management
class Router<InteractorType>: Routing {
    var children: [Routing] = []
    
    func attachChild(_ child: Routing) {
        children.append(child)
        child.didLoad()  // Lifecycle hook
        child.interactable.activate()
    }
    
    func detachChild(_ child: Routing) {
        child.interactable.deactivate()
        children.removeAll { $0 === child }
        // Child is deallocated when removed
    }
}

// ⚠️ Common mistake: Forgetting to detach
// This causes memory leaks because the child stays in the array
```

### SwiftUI Compatibility

```swift
// RIBs Framework provides SwiftUI adapters (recent updates)
final class FeedRouter: ViewableRouter<FeedInteractable, FeedViewControllable> {
    // ViewControllable can wrap either UIKit or SwiftUI
}

// SwiftUI View wrapper
struct FeedView: View {
    @ObservedObject var presenter: FeedPresenter
    
    var body: some View {
        List(presenter.posts) { post in
            PostRow(post: post)
                .onTapGesture {
                    presenter.didSelectPost(post)
                }
        }
    }
}

// Bridge to ViewControllable
class FeedViewController: UIHostingController<FeedView>, FeedViewControllable {
    // ...
}
```

---

## 7️⃣ Testability & Scalability

### Unit Testing: 🟢 Excellent (with infrastructure)

```swift
// Testing Interactor
class FeedInteractorTests: XCTestCase {
    var sut: FeedInteractor!
    var mockPresenter: MockFeedPresentable!
    var mockRouter: MockFeedRouting!
    var mockFeedService: MockFeedService!
    
    override func setUp() {
        mockPresenter = MockFeedPresentable()
        mockRouter = MockFeedRouting()
        mockFeedService = MockFeedService()
        
        sut = FeedInteractor(
            feedService: mockFeedService
        )
        sut.router = mockRouter
        sut.presenter = mockPresenter
    }
    
    func testDidBecomeActive_loadsFeed() {
        sut.activate()
        
        XCTAssertTrue(mockFeedService.fetchFeedCalled)
    }
    
    func testDidSelectPost_routesToDetail() {
        let post = Post.mock()
        
        sut.didSelectPost(post)
        
        XCTAssertTrue(mockRouter.attachPostDetailCalled)
        XCTAssertEqual(mockRouter.attachedPost?.id, post.id)
    }
}

// Testing Router
class FeedRouterTests: XCTestCase {
    var sut: FeedRouter!
    var mockInteractor: MockFeedInteractable!
    var mockViewController: MockFeedViewControllable!
    var mockPostDetailBuilder: MockPostDetailBuildable!
    
    func testAttachPostDetail_attachesChild() {
        sut.attachPostDetail(post: Post.mock())
        
        XCTAssertTrue(mockPostDetailBuilder.buildCalled)
        XCTAssertEqual(sut.children.count, 1)
        XCTAssertTrue(mockViewController.presentCalled)
    }
}
```

### Dependency Injection: 🟡 Custom DI Required

```swift
// RIBs uses Builder-based DI
protocol FeedDependency: Dependency {
    var feedService: FeedServiceProtocol { get }
    var analyticsService: AnalyticsProtocol { get }
}

final class FeedComponent: Component<FeedDependency>, PostDetailDependency {
    // Expose dependencies to child builders
    var feedService: FeedServiceProtocol {
        dependency.feedService
    }
    
    // Create dependencies for this scope
    var postRepository: PostRepositoryProtocol {
        PostRepository(apiService: apiService)
    }
}

final class FeedBuilder: Builder<FeedDependency>, FeedBuildable {
    func build(withListener listener: FeedListener) -> FeedRouting {
        let component = FeedComponent(dependency: dependency)
        let viewController = FeedViewController()
        let interactor = FeedInteractor(
            feedService: component.feedService,
            presenter: viewController
        )
        interactor.listener = listener
        
        return FeedRouter(
            interactor: interactor,
            viewController: viewController,
            postDetailBuilder: PostDetailBuilder(dependency: component)
        )
    }
}
```

### Team Scalability: 🟢 Excellent

| Team Size | RIBs Suitability |
|-----------|------------------|
| 1-5 developers | ❌ Overkill |
| 5-15 developers | ⚠️ Consider carefully |
| 15-50 developers | ✅ Great |
| 50+ developers | ✅ Perfect (Uber's use case) |

---

## 8️⃣ Complete Swift Example: Ride Request Flow

### Protocols

```swift
// MARK: - Feed RIB Protocols
protocol FeedRouting: ViewableRouting {
    func attachPostDetail(post: Post)
    func detachPostDetail()
}

protocol FeedPresentable: Presentable {
    var listener: FeedPresentableListener? { get set }
    func displayPosts(_ posts: [PostViewModel])
    func displayLoading(_ isLoading: Bool)
}

protocol FeedListener: AnyObject {
    // Parent listens for events from this RIB
}

protocol FeedInteractable: Interactable, PostDetailListener {
    var router: FeedRouting? { get set }
    var listener: FeedListener? { get set }
}

protocol FeedBuildable: Buildable {
    func build(withListener listener: FeedListener) -> FeedRouting
}
```

### Builder

```swift
// MARK: - Builder
final class FeedBuilder: Builder<FeedDependency>, FeedBuildable {
    
    override init(dependency: FeedDependency) {
        super.init(dependency: dependency)
    }
    
    func build(withListener listener: FeedListener) -> FeedRouting {
        let component = FeedComponent(dependency: dependency)
        let viewController = FeedViewController()
        let interactor = FeedInteractor(
            presenter: viewController,
            feedService: component.feedService
        )
        interactor.listener = listener
        
        let router = FeedRouter(
            interactor: interactor,
            viewController: viewController,
            postDetailBuilder: PostDetailBuilder(dependency: component)
        )
        
        return router
    }
}

// MARK: - Component (DI Scope)
final class FeedComponent: Component<FeedDependency>, PostDetailDependency {
    var feedService: FeedServiceProtocol {
        dependency.feedService
    }
    
    var postService: PostServiceProtocol {
        PostService(apiClient: dependency.apiClient)
    }
}
```

### Interactor

```swift
// MARK: - Interactor
final class FeedInteractor: PresentableInteractor<FeedPresentable>, FeedInteractable {
    
    weak var router: FeedRouting?
    weak var listener: FeedListener?
    
    private let feedService: FeedServiceProtocol
    private var posts: [Post] = []
    private var cancellables = Set<AnyCancellable>()
    
    init(presenter: FeedPresentable, feedService: FeedServiceProtocol) {
        self.feedService = feedService
        super.init(presenter: presenter)
        presenter.listener = self
    }
    
    override func didBecomeActive() {
        super.didBecomeActive()
        loadFeed()
    }
    
    private func loadFeed() {
        presenter.displayLoading(true)
        
        Task {
            do {
                let posts = try await feedService.fetchFeed()
                await MainActor.run {
                    self.posts = posts
                    self.presenter.displayPosts(posts.map(PostViewModel.init))
                    self.presenter.displayLoading(false)
                }
            } catch {
                await MainActor.run {
                    self.presenter.displayLoading(false)
                    // Handle error
                }
            }
        }
    }
}

// MARK: - PresentableListener (View → Interactor)
extension FeedInteractor: FeedPresentableListener {
    func didSelectPost(at index: Int) {
        guard index < posts.count else { return }
        router?.attachPostDetail(post: posts[index])
    }
    
    func didPullToRefresh() {
        loadFeed()
    }
}

// MARK: - Child Listener
extension FeedInteractor: PostDetailListener {
    func postDetailDidClose() {
        router?.detachPostDetail()
    }
}
```

### Router

```swift
// MARK: - Router
final class FeedRouter: ViewableRouter<FeedInteractable, FeedViewControllable>, FeedRouting {
    
    private let postDetailBuilder: PostDetailBuildable
    private var postDetailRouter: ViewableRouting?
    
    init(
        interactor: FeedInteractable,
        viewController: FeedViewControllable,
        postDetailBuilder: PostDetailBuildable
    ) {
        self.postDetailBuilder = postDetailBuilder
        super.init(interactor: interactor, viewController: viewController)
        interactor.router = self
    }
    
    func attachPostDetail(post: Post) {
        guard postDetailRouter == nil else { return }
        
        let router = postDetailBuilder.build(withListener: interactor, post: post)
        postDetailRouter = router
        attachChild(router)
        viewController.push(router.viewControllable)
    }
    
    func detachPostDetail() {
        guard let router = postDetailRouter else { return }
        
        viewController.pop(router.viewControllable)
        detachChild(router)
        postDetailRouter = nil
    }
}
```

### View

```swift
// MARK: - View Protocol
protocol FeedViewControllable: ViewControllable {
    func push(_ viewControllable: ViewControllable)
    func pop(_ viewControllable: ViewControllable)
}

protocol FeedPresentableListener: AnyObject {
    func didSelectPost(at index: Int)
    func didPullToRefresh()
}

// MARK: - ViewController
final class FeedViewController: UIViewController, FeedPresentable, FeedViewControllable {
    
    weak var listener: FeedPresentableListener?
    
    private var posts: [PostViewModel] = []
    private lazy var tableView = UITableView()
    private lazy var refreshControl = UIRefreshControl()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
    }
    
    private func setupUI() {
        title = "Feed"
        tableView.dataSource = self
        tableView.delegate = self
        tableView.refreshControl = refreshControl
        refreshControl.addTarget(self, action: #selector(pullToRefresh), for: .valueChanged)
        
        // Layout...
    }
    
    @objc private func pullToRefresh() {
        listener?.didPullToRefresh()
    }
    
    // MARK: - FeedPresentable
    func displayPosts(_ posts: [PostViewModel]) {
        self.posts = posts
        tableView.reloadData()
        refreshControl.endRefreshing()
    }
    
    func displayLoading(_ isLoading: Bool) {
        // Show/hide loading indicator
    }
    
    // MARK: - FeedViewControllable
    func push(_ viewControllable: ViewControllable) {
        navigationController?.pushViewController(viewControllable.uiviewController, animated: true)
    }
    
    func pop(_ viewControllable: ViewControllable) {
        navigationController?.popViewController(animated: true)
    }
}

extension FeedViewController: UITableViewDataSource, UITableViewDelegate {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        posts.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        // Configure cell
        let cell = UITableViewCell()
        cell.textLabel?.text = posts[indexPath.row].title
        return cell
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        listener?.didSelectPost(at: indexPath.row)
    }
}
```

---

## 9️⃣ RIBs Deep Dive

### Why Uber Created RIBs

| Problem | RIBs Solution |
|---------|---------------|
| **100+ iOS engineers** | Team ownership of RIBs |
| **Shared business logic iOS/Android** | Same architecture, same tree |
| **Complex state management** | RIB tree represents state |
| **Deep linking at scale** | Navigate to any RIB state |
| **Testability** | Every component is mockable |

### Tree-Based Feature Isolation

```
// App state is the RIB tree shape
// Different tree = different app state

LoggedIn └── Riding └── OnTrip     = User is on a trip
LoggedIn └── Home └── Feed         = User is browsing
LoggedOut └── Login                = User logging in
```

### Routing, Interactor, and Dependency Management

```swift
// Router: ONLY manages child RIBs
// Never contains business logic
router.attachChild(router)   // Yes
router.checkUserEligibility() // No!

// Interactor: ONLY contains business logic
// Never directly manipulates UI
interactor.processPayment()   // Yes
interactor.tableView.reload() // No!

// Builder: ONLY creates and wires components
// Never called after initial build
builder.build(withListener:)  // Yes
builder.addNewView()          // No!
```

### Why RIBs Fits Very Large, Complex Apps

1. **Team boundaries** = RIB boundaries
2. **Feature flags** can attach/detach entire RIBs
3. **A/B tests** can swap RIB implementations
4. **Analytics** via viewless RIBs
5. **Cross-platform consistency** with Android

---

## 📋 When to Use RIBs

### ✅ Good Use Cases
- 50+ iOS engineers
- 100+ screens with complex state
- Shared architecture with Android team
- Enterprise apps with long lifecycle
- Apps requiring viewless business logic

### ❌ Avoid RIBs When
- Team < 15 developers
- App < 30 screens
- Tight deadlines
- Team unfamiliar with reactive programming
- SwiftUI-first project

---

## 🎯 Interview Tips

### Common Questions

**Q: Why would you choose RIBs over VIPER?**
> "RIBs is designed for massive scale with 50+ developers. VIPER is per-screen, while RIBs forms a tree where business logic can exist without views. RIBs also aligns iOS and Android architectures, which VIPER doesn't address."

**Q: What's a viewless RIB? Give an example.**
> "A viewless RIB has Interactor and Router but no View. Examples include: auth state management, analytics tracking, feature flags, background sync. These RIBs manage app-wide state without rendering anything."

**Q: How do child RIBs communicate with parent RIBs?**
> "Through the Listener protocol. Parent Interactor implements the child's Listener protocol. Child Interactor calls `listener?.someEvent()` to notify the parent. This inverts control—parent decides how to react."

**Q: Would you recommend RIBs for a startup?**
> "No. RIBs has significant infrastructure cost. For startups, I'd recommend MVVM+C or Clean Architecture until the team grows past 15-20 developers and the app exceeds 50+ screens."

---

## 📚 Next Steps

Need a simpler alternative for large apps? Continue to [Clean Architecture →](./06_Clean_Architecture.md)
