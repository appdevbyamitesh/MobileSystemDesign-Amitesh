# 🏗️ VIPER

> **View-Interactor-Presenter-Entity-Router: Maximum Separation of Concerns**

---

## 1️⃣ What It Is (Simple English)

VIPER is an architecture that **strictly separates** every responsibility into its own component. Each letter represents a role:

- **V**iew – Displays data, captures user input (dumb)
- **I**nteractor – Business logic, use cases (the brain)
- **P**resenter – Formats data for display, orchestrates actions
- **E**ntity – Plain data models (structs)
- **R**outer – Navigation between modules

### Real-Life Analogy: Hospital

| Role | Hospital | iOS |
|------|----------|-----|
| **View** | Patient registration screen | UIViewController |
| **Interactor** | Doctor (diagnoses, prescribes) | Business logic class |
| **Presenter** | Nurse (explains diagnosis to patient) | Data formatting class |
| **Entity** | Medical records | Data models |
| **Router** | Hospital navigation/transport | Navigation handler |

The patient (View) talks to the nurse (Presenter). The nurse consults the doctor (Interactor). The doctor updates records (Entity). If the patient needs surgery, transport (Router) takes them there.

### The Key Insight

> **Each component has ONE job and communicates through protocols.**

---

## 2️⃣ Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                          VIPER Module                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌────────┐    User action   ┌───────────┐                   │
│    │  View  │─────────────────►│ Presenter │                   │
│    │        │◄─────────────────│           │                   │
│    └────────┘    UI updates    └─────┬─────┘                   │
│                                      │                          │
│                        Business      │      Navigation          │
│                        request       │      request             │
│                                      ▼                          │
│    ┌────────────┐            ┌───────────┐                     │
│    │ Interactor │◄───────────│  Presenter │                    │
│    │            │────────────►│           │                    │
│    └─────┬──────┘  Result    └─────┬─────┘                     │
│          │                         │                            │
│          ▼                         ▼                            │
│    ┌────────────┐            ┌───────────┐                     │
│    │   Entity   │            │  Router   │                     │
│    │   (Data)   │            │           │                     │
│    └────────────┘            └───────────┘                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibilities | Protocol Direction |
|-----------|------------------|-------------------|
| **View** | UI layout, display, forward user actions | Implements `PresenterToViewProtocol` |
| **Interactor** | Business logic, API calls, data operations | Implements `PresenterToInteractorProtocol` |
| **Presenter** | Transform data, orchestrate View & Interactor | Implements `ViewToPresenterProtocol`, `InteractorToPresenterProtocol` |
| **Entity** | Data structures (just structs) | N/A |
| **Router** | Create module, handle navigation | Implements `PresenterToRouterProtocol` |

---

## 3️⃣ Data & Control Flow

### User Action Flow (Tap "Load Feed")

```
┌──────────────────────────────────────────────────────────────────┐
│ USER TAPS "LOAD FEED"                                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. View: Calls presenter.loadFeed()                             │
│  2. Presenter: Shows loading, calls interactor.fetchFeed()       │
│  3. Interactor: Makes API call                                   │
│  4. Interactor: Calls presenter.feedFetched(entities)            │
│  5. Presenter: Transforms entities to view models                │
│  6. Presenter: Calls view.displayFeed(viewModels)                │
│  7. View: Updates UITableView                                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram (Text)

```
User      View       Presenter     Interactor      API
 │         │            │              │            │
 │──tap────►│            │              │            │
 │         │──loadFeed()─►│              │            │
 │         │◄─showLoading│              │            │
 │         │            │──fetchFeed()──►│            │
 │         │            │              │───GET /feed─►│
 │         │            │              │◄──200 OK────│
 │         │            │◄─feedFetched─│            │
 │         │◄displayFeed│              │            │
 │◄────────│────────────│              │            │
```

### Navigation Flow (Tap on Post)

```
User      View       Presenter      Router       DetailModule
 │         │            │             │              │
 │──tap────►│            │             │              │
 │         │─didSelect()►│             │              │
 │         │            │─showDetail()─►│              │
 │         │            │             │──createModule─►│
 │         │            │             │◄──────────────│
 │         │            │◄─push VC────│              │
 │◄────────│────────────│─────────────│──────────────│
```

---

## 4️⃣ Strengths

✅ **Maximum separation** – Each component has exactly one responsibility

✅ **Highly testable** – Every component can be unit tested in isolation

✅ **Parallel development** – Different devs can work on View, Interactor, Presenter

✅ **Clear boundaries** – No ambiguity about where code belongs

✅ **Enterprise-ready** – Scales to very large codebases

✅ **Protocol-driven** – Easy to mock for testing

---

## 5️⃣ Limitations & Failure Points

### ❌ Boilerplate Explosion

```swift
// For ONE screen, you need 6+ files:
// 1. FeedView.swift (ViewController)
// 2. FeedPresenter.swift
// 3. FeedInteractor.swift
// 4. FeedRouter.swift
// 5. FeedProtocols.swift
// 6. FeedEntity.swift (often part of shared models)

// Plus 6+ protocols!
```

### ❌ Over-engineering for Simple Features

```swift
// A simple "About" screen with static text
// still needs View, Presenter, Interactor, Router
// when a single ViewController would suffice
```

### ❌ Learning Curve

| Component | Questions New Devs Ask |
|-----------|----------------------|
| Presenter vs ViewModel | "Isn't this the same thing?" |
| Interactor vs Service | "Why not just call the service?" |
| Router | "Why not push directly?" |

### ❌ Why Some Teams Abandon VIPER

1. **Too much ceremony** – Simple changes require touching 4+ files
2. **Slow onboarding** – New developers struggle with the mental model
3. **Refactoring pain** – Moving logic between components is tedious
4. **Template dependency** – Teams need Xcode templates or code generators

---

## 6️⃣ iOS-Specific Considerations

### Navigation Handling

```swift
// Router owns the UINavigationController
protocol FeedRouterProtocol: AnyObject {
    static func createModule() -> UIViewController
    func navigateToPostDetail(post: PostEntity)
    func navigateToUserProfile(userId: String)
}

final class FeedRouter: FeedRouterProtocol {
    weak var viewController: UIViewController?
    
    static func createModule() -> UIViewController {
        let view = FeedViewController()
        let presenter = FeedPresenter()
        let interactor = FeedInteractor()
        let router = FeedRouter()
        
        view.presenter = presenter
        presenter.view = view
        presenter.interactor = interactor
        presenter.router = router
        interactor.presenter = presenter
        router.viewController = view
        
        return view
    }
    
    func navigateToPostDetail(post: PostEntity) {
        let detailVC = PostDetailRouter.createModule(post: post)
        viewController?.navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

### Concurrency & async/await

```swift
// Interactor handles async operations
protocol FeedInteractorProtocol: AnyObject {
    func fetchFeed()
    func toggleLike(postId: String)
}

final class FeedInteractor: FeedInteractorProtocol {
    weak var presenter: FeedInteractorOutputProtocol?
    private let apiService: FeedAPIProtocol
    
    init(apiService: FeedAPIProtocol = FeedAPIService()) {
        self.apiService = apiService
    }
    
    func fetchFeed() {
        Task {
            do {
                let posts = try await apiService.fetchFeed(cursor: nil)
                await MainActor.run {
                    presenter?.feedFetchedSuccess(posts: posts.posts)
                }
            } catch {
                await MainActor.run {
                    presenter?.feedFetchedFailure(error: error)
                }
            }
        }
    }
}
```

### Memory Management & Retain Cycles

```swift
// ⚠️ VIPER has many references - easy to create cycles

// ❌ BAD: Strong references everywhere
class FeedPresenter {
    var view: FeedViewProtocol?        // Strong
    var interactor: FeedInteractorProtocol?  // Strong
    var router: FeedRouterProtocol?    // Strong
}

class FeedInteractor {
    var presenter: FeedPresenterOutputProtocol?  // Strong → CYCLE!
}

// ✅ CORRECT: Weak backwards references
class FeedPresenter {
    weak var view: FeedViewProtocol?   // Weak (View owns Presenter)
    var interactor: FeedInteractorProtocol?
    var router: FeedRouterProtocol?
}

class FeedInteractor {
    weak var presenter: FeedPresenterOutputProtocol?  // Weak
}

class FeedRouter {
    weak var viewController: UIViewController?  // Weak
}
```

### SwiftUI Compatibility

```swift
// VIPER doesn't fit SwiftUI naturally
// Option 1: Use Presenter as ObservableObject
// Option 2: Skip VIPER for SwiftUI, use Clean Architecture

// If you must use VIPER with SwiftUI:
final class FeedPresenter: ObservableObject, FeedPresenterProtocol {
    @Published var posts: [PostViewModel] = []
    @Published var isLoading = false
    
    weak var interactor: FeedInteractorProtocol?
    
    func loadFeed() {
        isLoading = true
        interactor?.fetchFeed()
    }
    
    func feedFetchedSuccess(posts: [PostEntity]) {
        self.posts = posts.map(PostViewModel.init)
        isLoading = false
    }
}

struct FeedView: View {
    @StateObject var presenter: FeedPresenter
    
    var body: some View {
        List(presenter.posts) { post in
            PostRow(post: post)
        }
        .task { presenter.loadFeed() }
    }
}
```

---

## 7️⃣ Testability & Scalability

### Unit Testing: 🟢 Excellent

```swift
// Test Presenter in isolation
class FeedPresenterTests: XCTestCase {
    var sut: FeedPresenter!
    var mockView: MockFeedView!
    var mockInteractor: MockFeedInteractor!
    var mockRouter: MockFeedRouter!
    
    override func setUp() {
        mockView = MockFeedView()
        mockInteractor = MockFeedInteractor()
        mockRouter = MockFeedRouter()
        
        sut = FeedPresenter()
        sut.view = mockView
        sut.interactor = mockInteractor
        sut.router = mockRouter
    }
    
    func testLoadFeed_callsInteractor() {
        sut.loadFeed()
        XCTAssertTrue(mockInteractor.fetchFeedCalled)
    }
    
    func testFeedFetchedSuccess_updatesView() {
        let posts = [PostEntity.mock()]
        sut.feedFetchedSuccess(posts: posts)
        
        XCTAssertTrue(mockView.displayFeedCalled)
        XCTAssertEqual(mockView.displayedPosts.count, 1)
    }
    
    func testDidSelectPost_callsRouter() {
        let post = PostEntity.mock()
        sut.didSelectPost(post)
        
        XCTAssertTrue(mockRouter.navigateToPostDetailCalled)
        XCTAssertEqual(mockRouter.navigatedPost?.id, post.id)
    }
}

// Test Interactor in isolation
class FeedInteractorTests: XCTestCase {
    var sut: FeedInteractor!
    var mockPresenter: MockFeedPresenterOutput!
    var mockAPI: MockFeedAPI!
    
    func testFetchFeed_success() async {
        mockAPI.stubbedPosts = [PostEntity.mock()]
        
        sut.fetchFeed()
        
        // Wait for async
        try? await Task.sleep(nanoseconds: 100_000_000)
        
        XCTAssertTrue(mockPresenter.feedFetchedSuccessCalled)
    }
}
```

### Dependency Injection: 🟢 Natural via Protocols

```swift
// Every dependency is protocol-based
protocol FeedAPIProtocol {
    func fetchFeed(cursor: String?) async throws -> FeedResponse
}

protocol AnalyticsProtocol {
    func trackFeedViewed()
    func trackPostLiked(postId: String)
}

final class FeedInteractor: FeedInteractorProtocol {
    weak var presenter: FeedPresenterOutputProtocol?
    
    private let apiService: FeedAPIProtocol
    private let analytics: AnalyticsProtocol
    
    init(
        apiService: FeedAPIProtocol = FeedAPIService(),
        analytics: AnalyticsProtocol = Analytics.shared
    ) {
        self.apiService = apiService
        self.analytics = analytics
    }
}
```

### Team Scalability: 🟢 Excellent

| Team Size | VIPER Suitability |
|-----------|-------------------|
| 1-3 developers | ⚠️ Overkill |
| 5-10 developers | ✅ Excellent |
| 10-20 developers | ✅ Great |
| 20+ developers | ✅ Good (consider RIBs) |

---

## 8️⃣ Complete Swift Example: Feed Module

### Protocols File

```swift
import UIKit

// MARK: - View Protocol
protocol FeedViewProtocol: AnyObject {
    var presenter: FeedPresenterProtocol? { get set }
    
    func showLoading()
    func hideLoading()
    func displayFeed(_ posts: [PostViewModel])
    func displayError(_ message: String)
}

// MARK: - Presenter Protocol (View → Presenter)
protocol FeedPresenterProtocol: AnyObject {
    var view: FeedViewProtocol? { get set }
    var interactor: FeedInteractorInputProtocol? { get set }
    var router: FeedRouterProtocol? { get set }
    
    func viewDidLoad()
    func loadFeed()
    func didSelectPost(at index: Int)
    func didTapLike(postId: String)
}

// MARK: - Presenter Output Protocol (Interactor → Presenter)
protocol FeedPresenterOutputProtocol: AnyObject {
    func feedFetchedSuccess(posts: [PostEntity])
    func feedFetchedFailure(error: Error)
    func likeToogledSuccess(post: PostEntity)
}

// MARK: - Interactor Protocol (Presenter → Interactor)
protocol FeedInteractorInputProtocol: AnyObject {
    var presenter: FeedPresenterOutputProtocol? { get set }
    
    func fetchFeed()
    func toggleLike(postId: String, isLiked: Bool)
}

// MARK: - Router Protocol
protocol FeedRouterProtocol: AnyObject {
    static func createModule() -> UIViewController
    func navigateToPostDetail(_ post: PostEntity)
    func navigateToUserProfile(_ userId: String)
}
```

### Entity

```swift
// MARK: - Entity
struct PostEntity: Identifiable {
    let id: String
    let authorId: String
    let authorName: String
    let content: String
    let likeCount: Int
    var isLiked: Bool
    let createdAt: Date
}

struct FeedResponse {
    let posts: [PostEntity]
    let nextCursor: String?
}
```

### View

```swift
import UIKit

// MARK: - View
final class FeedViewController: UIViewController, FeedViewProtocol {
    
    var presenter: FeedPresenterProtocol?
    
    private var posts: [PostViewModel] = []
    
    private lazy var tableView: UITableView = {
        let tv = UITableView()
        tv.translatesAutoresizingMaskIntoConstraints = false
        tv.dataSource = self
        tv.delegate = self
        tv.register(PostCell.self, forCellReuseIdentifier: "PostCell")
        return tv
    }()
    
    private lazy var loadingIndicator: UIActivityIndicatorView = {
        let ai = UIActivityIndicatorView(style: .large)
        ai.translatesAutoresizingMaskIntoConstraints = false
        ai.hidesWhenStopped = true
        return ai
    }()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        presenter?.viewDidLoad()
    }
    
    private func setupUI() {
        title = "Feed"
        view.backgroundColor = .systemBackground
        
        view.addSubview(tableView)
        view.addSubview(loadingIndicator)
        
        NSLayoutConstraint.activate([
            tableView.topAnchor.constraint(equalTo: view.topAnchor),
            tableView.leadingAnchor.constraint(equalTo: view.leadingAnchor),
            tableView.trailingAnchor.constraint(equalTo: view.trailingAnchor),
            tableView.bottomAnchor.constraint(equalTo: view.bottomAnchor),
            
            loadingIndicator.centerXAnchor.constraint(equalTo: view.centerXAnchor),
            loadingIndicator.centerYAnchor.constraint(equalTo: view.centerYAnchor)
        ])
    }
    
    // MARK: - FeedViewProtocol
    func showLoading() {
        loadingIndicator.startAnimating()
    }
    
    func hideLoading() {
        loadingIndicator.stopAnimating()
    }
    
    func displayFeed(_ posts: [PostViewModel]) {
        self.posts = posts
        tableView.reloadData()
    }
    
    func displayError(_ message: String) {
        let alert = UIAlertController(title: "Error", message: message, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "Retry", style: .default) { [weak self] _ in
            self?.presenter?.loadFeed()
        })
        alert.addAction(UIAlertAction(title: "Cancel", style: .cancel))
        present(alert, animated: true)
    }
}

extension FeedViewController: UITableViewDataSource, UITableViewDelegate {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        posts.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: "PostCell", for: indexPath) as! PostCell
        let post = posts[indexPath.row]
        cell.configure(with: post)
        cell.onLikeTapped = { [weak self] in
            self?.presenter?.didTapLike(postId: post.id)
        }
        return cell
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
        presenter?.didSelectPost(at: indexPath.row)
    }
}
```

### Presenter

```swift
// MARK: - Presenter
final class FeedPresenter: FeedPresenterProtocol {
    
    weak var view: FeedViewProtocol?
    var interactor: FeedInteractorInputProtocol?
    var router: FeedRouterProtocol?
    
    private var entities: [PostEntity] = []
    
    func viewDidLoad() {
        loadFeed()
    }
    
    func loadFeed() {
        view?.showLoading()
        interactor?.fetchFeed()
    }
    
    func didSelectPost(at index: Int) {
        guard index < entities.count else { return }
        router?.navigateToPostDetail(entities[index])
    }
    
    func didTapLike(postId: String) {
        guard let post = entities.first(where: { $0.id == postId }) else { return }
        interactor?.toggleLike(postId: postId, isLiked: !post.isLiked)
    }
}

extension FeedPresenter: FeedPresenterOutputProtocol {
    func feedFetchedSuccess(posts: [PostEntity]) {
        view?.hideLoading()
        entities = posts
        let viewModels = posts.map { PostViewModel(entity: $0) }
        view?.displayFeed(viewModels)
    }
    
    func feedFetchedFailure(error: Error) {
        view?.hideLoading()
        view?.displayError(error.localizedDescription)
    }
    
    func likeToogledSuccess(post: PostEntity) {
        if let index = entities.firstIndex(where: { $0.id == post.id }) {
            entities[index] = post
            let viewModels = entities.map { PostViewModel(entity: $0) }
            view?.displayFeed(viewModels)
        }
    }
}
```

### Interactor

```swift
// MARK: - Interactor
final class FeedInteractor: FeedInteractorInputProtocol {
    
    weak var presenter: FeedPresenterOutputProtocol?
    private let apiService: FeedAPIProtocol
    
    init(apiService: FeedAPIProtocol = FeedAPIService()) {
        self.apiService = apiService
    }
    
    func fetchFeed() {
        Task {
            do {
                let response = try await apiService.fetchFeed(cursor: nil)
                await MainActor.run {
                    presenter?.feedFetchedSuccess(posts: response.posts)
                }
            } catch {
                await MainActor.run {
                    presenter?.feedFetchedFailure(error: error)
                }
            }
        }
    }
    
    func toggleLike(postId: String, isLiked: Bool) {
        Task {
            do {
                let post = try await apiService.toggleLike(postId: postId, isLiked: isLiked)
                await MainActor.run {
                    presenter?.likeToogledSuccess(post: post)
                }
            } catch {
                // Handle error
            }
        }
    }
}
```

### Router

```swift
// MARK: - Router
final class FeedRouter: FeedRouterProtocol {
    
    weak var viewController: UIViewController?
    
    static func createModule() -> UIViewController {
        let view = FeedViewController()
        let presenter = FeedPresenter()
        let interactor = FeedInteractor()
        let router = FeedRouter()
        
        // Wiring
        view.presenter = presenter
        presenter.view = view
        presenter.interactor = interactor
        presenter.router = router
        interactor.presenter = presenter
        router.viewController = view
        
        return view
    }
    
    func navigateToPostDetail(_ post: PostEntity) {
        let detailVC = PostDetailRouter.createModule(post: post)
        viewController?.navigationController?.pushViewController(detailVC, animated: true)
    }
    
    func navigateToUserProfile(_ userId: String) {
        let profileVC = ProfileRouter.createModule(userId: userId)
        viewController?.navigationController?.pushViewController(profileVC, animated: true)
    }
}
```

### ViewModel (For Display)

```swift
// MARK: - ViewModel (Display Model)
struct PostViewModel: Identifiable {
    let id: String
    let authorName: String
    let content: String
    let likeCountText: String
    let isLiked: Bool
    let relativeTime: String
    
    init(entity: PostEntity) {
        self.id = entity.id
        self.authorName = entity.authorName
        self.content = entity.content
        self.likeCountText = "\(entity.likeCount) likes"
        self.isLiked = entity.isLiked
        
        let formatter = RelativeDateTimeFormatter()
        self.relativeTime = formatter.localizedString(for: entity.createdAt, relativeTo: Date())
    }
}
```

---

## 9️⃣ VIPER Deep Dive

### Clear Separation of Concerns

| Layer | Never Does |
|-------|-----------|
| **View** | Never calls API, never formats data, never navigates |
| **Presenter** | Never calls API directly, never handles UI layout |
| **Interactor** | Never touches UI, never knows about ViewModels |
| **Router** | Never contains business logic |

### Boilerplate Cost vs Clarity

| Aspect | Cost | Benefit |
|--------|------|---------|
| **Files per feature** | 6+ files | Clear file-level organization |
| **Protocols** | 6+ protocols | Mockable, testable |
| **Wiring** | Manual setup | Dependency graph is explicit |
| **Onboarding** | Steep curve | Consistent patterns |

### Why Some Teams Abandon VIPER

1. **Template dependency** – Without templates, setup is tedious
2. **Small features suffer** – A simple screen still needs full setup
3. **Protocol fatigue** – Too many protocols to maintain
4. **Framework evolution** – SwiftUI makes VIPER feel outdated

---

## 📋 When to Use VIPER

### ✅ Good Use Cases
- Large enterprise apps (30+ screens)
- Teams of 5-20 developers
- When maximum testability is required
- Long-lived codebases (5+ years)
- Regulatory requirements for code audits

### ❌ Avoid VIPER When
- Building prototypes or MVPs
- Small team (1-4 developers)
- SwiftUI-first projects
- Simple apps with minimal business logic
- Tight deadlines

---

## 🎯 Interview Tips

### Common Questions

**Q: What's the difference between VIPER and MVVM+C?**
> "VIPER separates business logic (Interactor) from orchestration (Presenter), while MVVM+C combines both in the ViewModel. VIPER has stricter boundaries but more boilerplate. MVVM+C is more pragmatic for medium-sized apps."

**Q: Why does VIPER have so many protocols?**
> "Protocols define the contract between components. This makes mocking trivial for tests and enforces that each component only knows about its immediate neighbors, not the entire system."

**Q: How do you handle shared state in VIPER?**
> "Shared state typically lives in a Service or Store that the Interactor depends on. Multiple Interactors can read/write to the same Store. Alternatively, you can use a Redux-like pattern with a central state container."

**Q: Would you use VIPER for a new project today?**
> "It depends on context. For a large UIKit app with a big team, VIPER is still valid. For a new SwiftUI project, I'd prefer Clean Architecture or The Composable Architecture. VIPER's mental model doesn't map naturally to SwiftUI's declarative approach."

---

## 📚 Next Steps

Need Uber-scale architecture? Continue to [RIBs →](./05_RIBs.md)
