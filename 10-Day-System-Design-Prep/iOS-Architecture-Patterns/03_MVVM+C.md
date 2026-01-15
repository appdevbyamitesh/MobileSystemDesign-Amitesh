# 🧭 MVVM + Coordinator (MVVM+C)

> **Solving MVVM's Navigation Problem - Decoupled, Testable Navigation Flows**

---

## 1️⃣ What It Is (Simple English)

MVVM+C adds a **Coordinator** to handle all navigation, removing this responsibility from both ViewModel and ViewController.

- **Model** – Data and business logic (same as MVVM)
- **View** – UI elements, UIViewController (same as MVVM)
- **ViewModel** – Logic, state, no navigation knowledge (same as MVVM)
- **Coordinator** – ONLY handles navigation between screens

### Real-Life Analogy: Airport

| Role | Airport | iOS |
|------|---------|-----|
| **Model** | Flight data, passenger info | Data classes, API responses |
| **View** | Screens, signage, gates | UIViewController, Views |
| **ViewModel** | Check-in agent (processes your data) | ViewModel class |
| **Coordinator** | Airport guide (directs you gate to gate) | Coordinator class |

The check-in agent doesn't walk you to your gate. The airport guide doesn't check you in. Each has one job.

### The Key Insight

> **Navigation is a first-class citizen that deserves its own abstraction.**

---

## 2️⃣ Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                      MVVM + Coordinator                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌─────────────┐         ┌─────────────┐                     │
│    │ Coordinator │─creates─►│ ViewController│                   │
│    │  (Parent)   │         │   + ViewModel  │                   │
│    └──────┬──────┘         └───────┬───────┘                   │
│           │                        │                            │
│           │ child                  │ navigation event           │
│           ▼                        ▼                            │
│    ┌─────────────┐         ┌─────────────┐                     │
│    │ Coordinator │◄────────│ Coordinator │                     │
│    │  (Child)    │ delegate│  (Parent)   │                     │
│    └─────────────┘         └─────────────┘                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibilities |
|-----------|------------------|
| **Coordinator** | Create ViewControllers, push/present/dismiss, manage child coordinators |
| **ViewModel** | Business logic, publish navigation *events*, no UIKit imports |
| **ViewController** | UI setup, binding, forward events to Coordinator |
| **Model** | Same as MVVM |

### Coordinator Hierarchy

```
AppCoordinator (root)
├── AuthCoordinator
│   ├── LoginViewController
│   └── SignUpViewController
├── MainTabCoordinator
│   ├── FeedCoordinator
│   │   ├── FeedViewController
│   │   └── PostDetailViewController
│   ├── SearchCoordinator
│   └── ProfileCoordinator
│       ├── ProfileViewController
│       └── SettingsCoordinator
```

---

## 3️⃣ Data & Control Flow

### Navigation Flow (Tap on Post → Detail)

```
┌──────────────────────────────────────────────────────────────────┐
│ USER TAPS ON A POST                                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. View: Cell receives tap, passes to ViewController            │
│  2. ViewController: Tells ViewModel user selected post           │
│  3. ViewModel: Publishes navigation event via delegate/Combine   │
│  4. ViewController: Receives event, forwards to Coordinator      │
│  5. Coordinator: Creates PostDetailViewController + ViewModel    │
│  6. Coordinator: Pushes onto navigationController                │
│                                                                  │
│  ➡️ ViewModel doesn't know about UINavigationController         │
│  ➡️ ViewController doesn't create other ViewControllers         │
│  ➡️ Coordinator owns the navigation stack                       │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram (Text)

```
User     View/VC      ViewModel    Coordinator    NewVC
 │         │             │             │            │
 │──tap────►│             │             │            │
 │         │──didSelect──►│             │            │
 │         │             │──publish─────►│            │
 │         │             │  navigation  │            │
 │         │             │   event      │            │
 │         │             │             │──create────►│
 │         │             │             │◄───────────│
 │         │             │◄─────push───│            │
 │◄────────│─────────────│─────────────│────────────│
```

---

## 4️⃣ Strengths

✅ **Fully decoupled navigation** – ViewModels don't import UIKit

✅ **Testable navigation flows** – Test Coordinator without UI

✅ **Reusable ViewControllers** – Same VC in different flows

✅ **Deep linking support** – Coordinator can navigate to any state

✅ **Flow encapsulation** – Auth flow, onboarding flow, checkout flow as units

✅ **Clean ViewModel** – No navigation pollution

---

## 5️⃣ Limitations & Failure Points

### ❌ Where MVVM+C Breaks Down

**1. Coordinator bloat**
```swift
// Large apps can end up with 50+ Coordinators
// Each feature = 1 Coordinator
// Coordinator management becomes its own problem
```

**2. Parent-child communication complexity**
```swift
// Who dismisses a modal? Parent or child Coordinator?
// How do we pass results back up the tree?
// Delegate chains can get deep
```

**3. No standard pattern**
```swift
// Multiple ways to implement Coordinators:
// - Protocol-based
// - Closure-based
// - Combine-based
// Team debates ensue
```

**4. Memory management hazards**
```swift
// Coordinators must be stored somewhere
// Easy to create retain cycles
// Child removal is error-prone
```

---

## 6️⃣ iOS-Specific Considerations

### Navigation Handling

```swift
// ✅ Coordinator owns the UINavigationController
final class FeedCoordinator: Coordinator {
    var navigationController: UINavigationController
    var childCoordinators: [Coordinator] = []
    
    init(navigationController: UINavigationController) {
        self.navigationController = navigationController
    }
    
    func start() {
        let viewModel = FeedViewModel(apiService: APIService())
        let viewController = FeedViewController(viewModel: viewModel)
        viewController.coordinator = self
        navigationController.pushViewController(viewController, animated: true)
    }
    
    func showPostDetail(_ post: Post) {
        let viewModel = PostDetailViewModel(post: post)
        let viewController = PostDetailViewController(viewModel: viewModel)
        viewController.coordinator = self
        navigationController.pushViewController(viewController, animated: true)
    }
    
    func showUserProfile(_ userId: String) {
        // Start a child coordinator for the profile flow
        let profileCoordinator = ProfileCoordinator(
            navigationController: navigationController,
            userId: userId
        )
        profileCoordinator.parentCoordinator = self
        childCoordinators.append(profileCoordinator)
        profileCoordinator.start()
    }
}
```

### Concurrency & async/await

```swift
// Coordinators can handle async flows
final class CheckoutCoordinator: Coordinator {
    
    func showPayment() {
        let viewModel = PaymentViewModel(paymentService: paymentService)
        let viewController = PaymentViewController(viewModel: viewModel)
        
        // Listen for completion
        viewModel.paymentCompleted
            .sink { [weak self] result in
                self?.handlePaymentResult(result)
            }
            .store(in: &cancellables)
        
        navigationController.pushViewController(viewController, animated: true)
    }
    
    func handlePaymentResult(_ result: PaymentResult) {
        switch result {
        case .success:
            showConfirmation()
        case .failure(let error):
            showError(error)
        case .cancelled:
            navigationController.popViewController(animated: true)
        }
    }
}
```

### Memory Management & Retain Cycles

```swift
// ⚠️ COMMON RETAIN CYCLE
class FeedViewController: UIViewController {
    var coordinator: FeedCoordinator?  // Strong reference
}

class FeedCoordinator {
    var viewController: FeedViewController?  // Strong reference
    // ❌ CYCLE: Coordinator → VC → Coordinator
}

// ✅ CORRECT PATTERN
class FeedViewController: UIViewController {
    weak var coordinator: FeedCoordinator?  // Weak reference
}

class FeedCoordinator {
    // Don't store VC reference, or use weak
    private weak var currentViewController: UIViewController?
}
```

### Child Coordinator Cleanup

```swift
// ✅ PROPER CHILD REMOVAL
protocol Coordinator: AnyObject {
    var childCoordinators: [Coordinator] { get set }
    func childDidFinish(_ child: Coordinator)
}

extension Coordinator {
    func childDidFinish(_ child: Coordinator) {
        childCoordinators.removeAll { $0 === child }
    }
}

// In child coordinator:
final class ProfileCoordinator: Coordinator {
    weak var parentCoordinator: Coordinator?
    
    func finish() {
        parentCoordinator?.childDidFinish(self)
    }
}
```

### SwiftUI Compatibility

```swift
// MVVM+C works with SwiftUI using NavigationStack + NavigationPath

final class FeedCoordinator: ObservableObject {
    @Published var navigationPath = NavigationPath()
    
    enum Destination: Hashable {
        case postDetail(Post)
        case userProfile(String)
    }
    
    func showPostDetail(_ post: Post) {
        navigationPath.append(Destination.postDetail(post))
    }
    
    func showUserProfile(_ userId: String) {
        navigationPath.append(Destination.userProfile(userId))
    }
    
    func pop() {
        navigationPath.removeLast()
    }
}

struct FeedFlowView: View {
    @StateObject var coordinator = FeedCoordinator()
    @StateObject var viewModel = FeedViewModel()
    
    var body: some View {
        NavigationStack(path: $coordinator.navigationPath) {
            FeedView(viewModel: viewModel, coordinator: coordinator)
                .navigationDestination(for: FeedCoordinator.Destination.self) { destination in
                    switch destination {
                    case .postDetail(let post):
                        PostDetailView(post: post)
                    case .userProfile(let userId):
                        ProfileView(userId: userId)
                    }
                }
        }
    }
}
```

---

## 7️⃣ Testability & Scalability

### Unit Testing: 🟢 Excellent

```swift
// Testing ViewModel (same as MVVM)
func testPostSelection_publishesNavigationEvent() {
    let viewModel = FeedViewModel(apiService: mockAPI)
    var navigationEvents: [FeedViewModel.Navigation] = []
    
    viewModel.navigationPublisher
        .sink { navigationEvents.append($0) }
        .store(in: &cancellables)
    
    viewModel.didSelectPost(mockPost)
    
    XCTAssertEqual(navigationEvents, [.showPostDetail(mockPost)])
}

// Testing Coordinator
func testShowPostDetail_pushesViewController() {
    let navController = MockNavigationController()
    let coordinator = FeedCoordinator(navigationController: navController)
    
    coordinator.showPostDetail(mockPost)
    
    XCTAssertTrue(navController.pushedViewController is PostDetailViewController)
}
```

### Dependency Injection: 🟢 Natural

```swift
// Coordinators can inject dependencies into ViewModels
final class FeedCoordinator: Coordinator {
    private let apiService: FeedAPIProtocol
    private let analytics: AnalyticsProtocol
    private let imageLoader: ImageLoaderProtocol
    
    init(
        navigationController: UINavigationController,
        apiService: FeedAPIProtocol = FeedAPIService(),
        analytics: AnalyticsProtocol = Analytics.shared,
        imageLoader: ImageLoaderProtocol = ImageLoader.shared
    ) {
        self.navigationController = navigationController
        self.apiService = apiService
        self.analytics = analytics
        self.imageLoader = imageLoader
    }
    
    func start() {
        let viewModel = FeedViewModel(
            apiService: apiService,
            analytics: analytics
        )
        // ...
    }
}
```

### Team Scalability: 🟢 Good

| Team Size | MVVM+C Suitability |
|-----------|-------------------|
| 1-3 developers | ✅ Great |
| 3-10 developers | ✅ Excellent |
| 10+ developers | ⚠️ Needs coordination patterns |

---

## 8️⃣ Complete Swift Example: Feed Flow

### Coordinator Protocol

```swift
import UIKit

// MARK: - Coordinator Protocol
protocol Coordinator: AnyObject {
    var navigationController: UINavigationController { get set }
    var childCoordinators: [Coordinator] { get set }
    var parentCoordinator: Coordinator? { get set }
    
    func start()
    func finish()
}

extension Coordinator {
    func finish() {
        parentCoordinator?.childDidFinish(self)
    }
    
    func childDidFinish(_ child: Coordinator) {
        childCoordinators.removeAll { $0 === child }
    }
    
    func addChild(_ coordinator: Coordinator) {
        childCoordinators.append(coordinator)
        coordinator.parentCoordinator = self
    }
}
```

### App Coordinator (Root)

```swift
// MARK: - App Coordinator
final class AppCoordinator: Coordinator {
    var navigationController: UINavigationController
    var childCoordinators: [Coordinator] = []
    weak var parentCoordinator: Coordinator?
    
    private let window: UIWindow
    private let authService: AuthServiceProtocol
    
    init(window: UIWindow, authService: AuthServiceProtocol = AuthService.shared) {
        self.window = window
        self.authService = authService
        self.navigationController = UINavigationController()
    }
    
    func start() {
        window.rootViewController = navigationController
        window.makeKeyAndVisible()
        
        if authService.isLoggedIn {
            showMainFlow()
        } else {
            showAuthFlow()
        }
    }
    
    private func showAuthFlow() {
        let authCoordinator = AuthCoordinator(navigationController: navigationController)
        authCoordinator.onLoginSuccess = { [weak self] in
            self?.childDidFinish(authCoordinator)
            self?.showMainFlow()
        }
        addChild(authCoordinator)
        authCoordinator.start()
    }
    
    private func showMainFlow() {
        navigationController.viewControllers = []
        let feedCoordinator = FeedCoordinator(navigationController: navigationController)
        addChild(feedCoordinator)
        feedCoordinator.start()
    }
}
```

### Feed Coordinator

```swift
// MARK: - Feed Coordinator
protocol FeedCoordinatorProtocol: AnyObject {
    func showPostDetail(_ post: Post)
    func showUserProfile(_ userId: String)
    func showCompose()
}

final class FeedCoordinator: Coordinator, FeedCoordinatorProtocol {
    var navigationController: UINavigationController
    var childCoordinators: [Coordinator] = []
    weak var parentCoordinator: Coordinator?
    
    private let apiService: FeedAPIProtocol
    
    init(
        navigationController: UINavigationController,
        apiService: FeedAPIProtocol = FeedAPIService()
    ) {
        self.navigationController = navigationController
        self.apiService = apiService
    }
    
    func start() {
        let viewModel = FeedViewModel(apiService: apiService)
        let viewController = FeedViewController(viewModel: viewModel, coordinator: self)
        navigationController.pushViewController(viewController, animated: false)
    }
    
    // MARK: - Navigation Methods
    func showPostDetail(_ post: Post) {
        let viewModel = PostDetailViewModel(post: post, apiService: apiService)
        let viewController = PostDetailViewController(viewModel: viewModel, coordinator: self)
        navigationController.pushViewController(viewController, animated: true)
    }
    
    func showUserProfile(_ userId: String) {
        let profileCoordinator = ProfileCoordinator(
            navigationController: navigationController,
            userId: userId
        )
        addChild(profileCoordinator)
        profileCoordinator.start()
    }
    
    func showCompose() {
        let composeCoordinator = ComposeCoordinator(
            presentingViewController: navigationController
        )
        composeCoordinator.onPostCreated = { [weak self] post in
            self?.handleNewPost(post)
        }
        composeCoordinator.onDismiss = { [weak self] in
            self?.childDidFinish(composeCoordinator)
        }
        addChild(composeCoordinator)
        composeCoordinator.start()
    }
    
    private func handleNewPost(_ post: Post) {
        // Refresh feed, scroll to top, etc.
    }
}
```

### Feed ViewModel (Navigation Events)

```swift
import Combine

// MARK: - Feed ViewModel
final class FeedViewModel: ObservableObject {
    
    // MARK: - Navigation Events (NOT actions)
    enum Navigation {
        case postDetail(Post)
        case userProfile(String)
        case compose
    }
    
    // MARK: - Published State
    @Published private(set) var posts: [PostViewModel] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?
    
    // MARK: - Navigation Publisher
    private let navigationSubject = PassthroughSubject<Navigation, Never>()
    var navigationPublisher: AnyPublisher<Navigation, Never> {
        navigationSubject.eraseToAnyPublisher()
    }
    
    // MARK: - Dependencies
    private let apiService: FeedAPIProtocol
    private var rawPosts: [Post] = []
    
    init(apiService: FeedAPIProtocol) {
        self.apiService = apiService
    }
    
    // MARK: - User Actions
    func didSelectPost(at index: Int) {
        guard index < rawPosts.count else { return }
        navigationSubject.send(.postDetail(rawPosts[index]))
    }
    
    func didTapUserAvatar(userId: String) {
        navigationSubject.send(.userProfile(userId))
    }
    
    func didTapCompose() {
        navigationSubject.send(.compose)
    }
    
    // MARK: - Data Loading
    func loadFeed() {
        guard !isLoading else { return }
        isLoading = true
        
        Task { @MainActor in
            do {
                let response = try await apiService.fetchFeed(cursor: nil)
                self.rawPosts = response.posts
                self.posts = response.posts.map(PostViewModel.init)
            } catch {
                self.errorMessage = error.localizedDescription
            }
            self.isLoading = false
        }
    }
    
    func toggleLike(postId: String) {
        // Same as MVVM implementation
    }
}
```

### Feed ViewController (Binds to Coordinator)

```swift
import UIKit
import Combine

// MARK: - Feed ViewController
final class FeedViewController: UIViewController {
    
    // MARK: - Dependencies
    private let viewModel: FeedViewModel
    private weak var coordinator: FeedCoordinatorProtocol?
    private var cancellables = Set<AnyCancellable>()
    
    // MARK: - UI
    private lazy var tableView = UITableView()
    private lazy var composeButton = UIBarButtonItem(
        barButtonSystemItem: .compose,
        target: self,
        action: #selector(composeTapped)
    )
    
    // MARK: - Init
    init(viewModel: FeedViewModel, coordinator: FeedCoordinatorProtocol) {
        self.viewModel = viewModel
        self.coordinator = coordinator
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    // MARK: - Lifecycle
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        bindViewModel()
        bindNavigation()
        viewModel.loadFeed()
    }
    
    // MARK: - Setup
    private func setupUI() {
        title = "Feed"
        navigationItem.rightBarButtonItem = composeButton
        
        tableView.translatesAutoresizingMaskIntoConstraints = false
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(PostTableViewCell.self, forCellReuseIdentifier: PostTableViewCell.reuseId)
        view.addSubview(tableView)
        
        NSLayoutConstraint.activate([
            tableView.topAnchor.constraint(equalTo: view.topAnchor),
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor)
        ])
    }
    
    private func bindViewModel() {
        viewModel.$posts
            .receive(on: DispatchQueue.main)
            .sink { [weak self] _ in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
    }
    
    // MARK: - Navigation Binding (The MVVM+C magic ✨)
    private func bindNavigation() {
        viewModel.navigationPublisher
            .receive(on: DispatchQueue.main)
            .sink { [weak self] navigation in
                switch navigation {
                case .postDetail(let post):
                    self?.coordinator?.showPostDetail(post)
                case .userProfile(let userId):
                    self?.coordinator?.showUserProfile(userId)
                case .compose:
                    self?.coordinator?.showCompose()
                }
            }
            .store(in: &cancellables)
    }
    
    @objc private func composeTapped() {
        viewModel.didTapCompose()
    }
}

// MARK: - UITableViewDataSource & Delegate
extension FeedViewController: UITableViewDataSource, UITableViewDelegate {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return viewModel.posts.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: PostTableViewCell.reuseId, for: indexPath) as! PostTableViewCell
        cell.configure(with: viewModel.posts[indexPath.row])
        return cell
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
        viewModel.didSelectPost(at: indexPath.row)
    }
}
```

---

## 9️⃣ MVVM+C Deep Dive

### Why Navigation Doesn't Belong in ViewModels

| Problem | Explanation |
|---------|-------------|
| **UIKit coupling** | Navigation needs `UINavigationController` |
| **Testability** | Can't unit test navigation logic |
| **Reusability** | Same ViewModel, different navigation flows |
| **Deep linking** | External URLs need direct navigation access |

### Coordinator Responsibilities

1. **Create** ViewControllers and inject ViewModels
2. **Push/Present** screens onto the navigation stack
3. **Manage** child flows (sub-coordinators)
4. **Handle** flow completion and cleanup
5. **Support** deep linking from any state

### How MVVM+C Improves Testability

```swift
// Test navigation without any UI
func testFeedCoordinator_showPostDetail() {
    let mockNavController = MockNavigationController()
    let coordinator = FeedCoordinator(navigationController: mockNavController)
    
    let post = Post.mock()
    coordinator.showPostDetail(post)
    
    XCTAssertEqual(mockNavController.pushCallCount, 1)
    XCTAssertTrue(mockNavController.lastPushedVC is PostDetailViewController)
}

// Test deep linking
func testAppCoordinator_deepLink_opensPostDetail() {
    let coordinator = AppCoordinator(window: mockWindow)
    coordinator.start()
    
    coordinator.handleDeepLink(URL(string: "myapp://post/123")!)
    
    // Assert correct navigation occurred
}
```

---

## 📋 When to Use MVVM+C

### ✅ Good Use Cases
- Apps with complex navigation (multiple flows)
- Deep linking requirements
- Reusable ViewControllers in different contexts
- Teams that value testability
- Medium to large apps (15-50 screens)

### ❌ Avoid MVVM+C When
- Very simple apps (5-10 screens)
- Prototypes or MVPs
- When team is unfamiliar with the pattern
- Linear, simple navigation

---

## 🎯 Interview Tips

### Common Questions

**Q: Why separate navigation into Coordinators?**
> "Navigation is a cross-cutting concern that impacts testability and reusability. With Coordinators, ViewModels don't need UIKit, making them unit-testable. ViewControllers become reusable in different flows. And deep linking becomes straightforward because Coordinators can navigate directly to any state."

**Q: How do you handle passing data back from a modal?**
> "Child Coordinators expose completion callbacks or Combine publishers. When the modal finishes, it tells its Coordinator, which publishes the result to the parent Coordinator via a delegate or closure. The parent then decides what to do next."

**Q: What's the difference between MVVM and MVVM+C?**
> "MVVM+C is MVVM with one critical addition: the Coordinator. In pure MVVM, navigation is either in the ViewController or leaks into the ViewModel. MVVM+C extracts navigation into a dedicated layer, making flows testable and reusable."

**Q: How do Coordinators handle memory management?**
> "Parent Coordinators hold strong references to child Coordinators in an array. Child Coordinators hold weak references back to parents. When a flow completes, the child calls `parentCoordinator?.childDidFinish(self)`, which removes it from the array, allowing deallocation."

---

## 📚 Next Steps

Need even stricter separation? Continue to [VIPER →](./04_VIPER.md)
