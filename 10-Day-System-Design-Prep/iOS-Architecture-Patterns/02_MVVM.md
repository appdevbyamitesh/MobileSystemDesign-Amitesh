# 📊 MVVM (Model-View-ViewModel)

> **The Testable Evolution of MVC - ViewModel + Data Binding with Combine**

---

## 1️⃣ What It Is (Simple English)

MVVM adds one key player between View and Model: the **ViewModel**.

- **Model** – Data and business logic (same as MVC)
- **View** – UI elements (same as MVC, but now includes UIViewController!)
- **ViewModel** – Prepares data for display, handles user actions, NO UI code

### Real-Life Analogy: TV News Studio

| Role | News Studio | iOS |
|------|-------------|-----|
| **Model** | News reports, raw footage | Data classes, API responses |
| **View** | Camera, monitors, teleprompter | UIViewController, SwiftUI View |
| **ViewModel** | Producer (decides what to show) | ViewModel class |

The producer (ViewModel) takes raw news (Model) and decides how to present it on camera (View). The camera doesn't know about sources; it just displays what the producer prepares.

### The Key Insight

> **In MVVM, `UIViewController` IS part of the View layer, NOT a Controller.**

This is the fundamental shift from MVC. The ViewController only handles:
- UI setup
- Binding to ViewModel
- Passing user actions to ViewModel

---

## 2️⃣ Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                        MVVM Pattern                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    ┌─────────┐                           ┌─────────┐       │
│    │  Model  │◄─────── Updates ──────────│ViewModel│       │
│    └─────────┘                           └────▲────┘       │
│                                               │             │
│                              ┌────────────────┴───────────┐ │
│                              │      Data Binding          │ │
│                              │   (Combine / Observation)   │ │
│                              └────────────────┬───────────┘ │
│                                               │             │
│                                          ┌────▼────┐       │
│                                          │  View   │       │
│                                          │(VC/View)│       │
│                                          └─────────┘       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibilities | iOS Example |
|-----------|------------------|-------------|
| **Model** | Data structures, persistence, networking | `struct Post`, `APIService` |
| **View** | UI layout, display, user input capture | `UIViewController`, `SwiftUI View` |
| **ViewModel** | Transform data for display, handle actions, state management | `FeedViewModel` class |

### ViewModel Contract

```swift
// ViewModel should:
// ✅ Expose data ready for display (formatted strings, UI state)
// ✅ Accept user actions as method calls
// ✅ NOT import UIKit (no UI knowledge)
// ✅ Be testable without any UI
```

---

## 3️⃣ Data & Control Flow

### User Action Flow (Tap "Like" Button)

```
┌──────────────────────────────────────────────────────────────────┐
│ USER TAPS "LIKE" BUTTON                                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. View: Button tap sends action to ViewController              │
│  2. ViewController: Calls viewModel.toggleLike(postId: "123")    │
│  3. ViewModel: Updates internal state, calls API                 │
│  4. ViewModel: Publishes updated posts via @Published            │
│  5. ViewController: Receives update via Combine subscription     │
│  6. ViewController: Updates UI automatically                     │
│                                                                  │
│  ➡️ ViewController doesn't know HOW the like works              │
│  ➡️ ViewModel doesn't know HOW the UI updates                   │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram (Text)

```
User         View/VC        ViewModel          Model          API
 │            │                │                 │              │
 │───tap──────►│                │                 │              │
 │            │──toggleLike()──►│                 │              │
 │            │                │───toggleLike()──►│              │
 │            │                │                 │───POST /like─►│
 │            │                │                 │◄───200 OK────│
 │            │                │◄───Post────────│              │
 │            │◄──@Published───│                 │              │
 │            │    (posts)     │                 │              │
 │◄──UI update─│                │                 │              │
```

### Data Binding with Combine

```swift
// In ViewController
viewModel.$posts
    .receive(on: DispatchQueue.main)
    .sink { [weak self] posts in
        self?.updateUI(with: posts)
    }
    .store(in: &cancellables)
```

---

## 4️⃣ Strengths

✅ **Testable business logic** – ViewModel has no UI dependencies

✅ **Clear separation** – View is dumb, ViewModel is smart

✅ **Reactive updates** – Combine makes data binding elegant

✅ **SwiftUI-ready** – MVVM is the natural pattern for SwiftUI

✅ **Smaller ViewControllers** – Business logic moves to ViewModel

✅ **Reusable ViewModels** – Same ViewModel can power different Views

---

## 5️⃣ Limitations & Failure Points

### ❌ Where MVVM Breaks Down

**1. Navigation is still coupled**
```swift
// ❌ ViewModel shouldn't know about navigation
class FeedViewModel {
    func didSelectPost(_ post: Post) {
        // Where does navigation go?
        // ViewModel shouldn't import UIKit!
    }
}
```

**2. Massive ViewModels**
```swift
// ViewModels can become just as massive as ViewControllers
class FeedViewModel {
    // Data fetching (50 lines)
    // Data transformation (40 lines)
    // Pagination logic (30 lines)
    // Like/unlike logic (40 lines)
    // Error handling (30 lines)
    // Analytics (20 lines)
    // = 210+ lines in one "thin" ViewModel
}
```

**3. Complex bindings**
```swift
// When you have many @Published properties, binding becomes messy
class ProfileViewModel {
    @Published var name: String = ""
    @Published var bio: String = ""
    @Published var followerCount: String = ""
    @Published var followingCount: String = ""
    @Published var posts: [Post] = []
    @Published var isLoading: Bool = false
    @Published var errorMessage: String?
    @Published var isFollowing: Bool = false
    // ... 10 more properties
}
```

**4. No standard state management**
```swift
// Different teams implement ViewModel state differently
// Option A: Multiple @Published
// Option B: Single state enum
// Option C: Input/Output pattern
// No "one true way"
```

---

## 6️⃣ iOS-Specific Considerations

### Navigation Handling

```swift
// ❌ Bad: ViewModel handles navigation
class FeedViewModel {
    weak var viewController: UIViewController?
    
    func didSelectPost(_ post: Post) {
        let detailVC = PostDetailViewController(post: post)
        viewController?.navigationController?.pushViewController(detailVC, animated: true)
    }
}

// ✅ Better: ViewModel exposes navigation events, View handles them
class FeedViewModel {
    let navigationSubject = PassthroughSubject<FeedNavigation, Never>()
    
    enum FeedNavigation {
        case postDetail(Post)
        case userProfile(User)
    }
    
    func didSelectPost(_ post: Post) {
        navigationSubject.send(.postDetail(post))
    }
}

// In ViewController:
viewModel.navigationSubject
    .sink { [weak self] navigation in
        switch navigation {
        case .postDetail(let post):
            let detailVC = PostDetailViewController(post: post)
            self?.navigationController?.pushViewController(detailVC, animated: true)
        case .userProfile(let user):
            // ...
        }
    }
    .store(in: &cancellables)
```

**This still isn't ideal** – ViewController knows about destinations. See MVVM+C for the solution.

### Concurrency & async/await

```swift
class FeedViewModel {
    @Published private(set) var posts: [Post] = []
    @Published private(set) var isLoading = false
    @Published private(set) var error: Error?
    
    private let apiService: FeedAPIProtocol
    
    // ✅ Clean async/await in ViewModel
    @MainActor
    func loadFeed() async {
        isLoading = true
        error = nil
        
        do {
            posts = try await apiService.fetchFeed()
        } catch {
            self.error = error
        }
        
        isLoading = false
    }
}

// In ViewController:
Task {
    await viewModel.loadFeed()
}
```

### Memory Management & Retain Cycles

```swift
class FeedViewController: UIViewController {
    private var viewModel: FeedViewModel!
    private var cancellables = Set<AnyCancellable>()
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        // ⚠️ RETAIN CYCLE RISK in closures
        viewModel.$posts
            .sink { posts in
                self.tableView.reloadData()  // ❌ Strong capture
            }
            .store(in: &cancellables)
        
        // ✅ CORRECT
        viewModel.$posts
            .sink { [weak self] posts in
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
    }
}
```

### SwiftUI Compatibility

```swift
// MVVM is PERFECT for SwiftUI

class ProfileViewModel: ObservableObject {
    @Published var name: String = ""
    @Published var avatarURL: URL?
    @Published var isLoading: Bool = false
}

struct ProfileView: View {
    @StateObject var viewModel: ProfileViewModel
    
    var body: some View {
        VStack {
            if viewModel.isLoading {
                ProgressView()
            } else {
                Text(viewModel.name)
                AsyncImage(url: viewModel.avatarURL)
            }
        }
        .task {
            await viewModel.loadProfile()
        }
    }
}
```

---

## 7️⃣ Testability & Scalability

### Unit Testing: 🟢 Easy

```swift
// ViewModel testing is straightforward
class FeedViewModelTests: XCTestCase {
    var sut: FeedViewModel!
    var mockAPI: MockFeedAPI!
    var cancellables: Set<AnyCancellable>!
    
    override func setUp() {
        mockAPI = MockFeedAPI()
        sut = FeedViewModel(apiService: mockAPI)
        cancellables = []
    }
    
    func testLoadFeed_success() async {
        // Arrange
        let expectedPosts = [Post.mock()]
        mockAPI.stubbedPosts = expectedPosts
        
        // Act
        await sut.loadFeed()
        
        // Assert
        XCTAssertEqual(sut.posts, expectedPosts)
        XCTAssertFalse(sut.isLoading)
        XCTAssertNil(sut.error)
    }
    
    func testLoadFeed_failure() async {
        // Arrange
        mockAPI.stubbedError = APIError.networkError
        
        // Act
        await sut.loadFeed()
        
        // Assert
        XCTAssertTrue(sut.posts.isEmpty)
        XCTAssertNotNil(sut.error)
    }
    
    func testToggleLike_optimisticUpdate() {
        // Arrange
        sut.posts = [Post(id: "1", isLiked: false)]
        
        // Act
        sut.toggleLike(postId: "1")
        
        // Assert
        XCTAssertTrue(sut.posts[0].isLiked)  // Immediate update
    }
}
```

### Dependency Injection: 🟢 Natural

```swift
// Protocol-based dependencies
protocol FeedAPIProtocol {
    func fetchFeed() async throws -> [Post]
    func toggleLike(postId: String, isLiked: Bool) async throws -> Post
}

class FeedViewModel {
    private let apiService: FeedAPIProtocol
    private let analytics: AnalyticsProtocol
    
    // ✅ Easy to inject mocks
    init(apiService: FeedAPIProtocol, analytics: AnalyticsProtocol = Analytics.shared) {
        self.apiService = apiService
        self.analytics = analytics
    }
}
```

### Team Scalability: 🟡 Medium

| Team Size | MVVM Suitability |
|-----------|------------------|
| 1-3 developers | ✅ Great |
| 3-7 developers | ✅ Good |
| 8+ developers | ⚠️ Needs coordination on navigation |

---

## 8️⃣ Complete Swift Example: Feed Screen

### ViewModel (Input/Output Pattern)

```swift
import Combine
import Foundation

// MARK: - ViewModel Protocol (for testing)
protocol FeedViewModelProtocol: ObservableObject {
    var posts: [PostViewModel] { get }
    var isLoading: Bool { get }
    var errorMessage: String? { get }
    var hasMorePages: Bool { get }
    
    func loadFeed()
    func loadMore()
    func refresh()
    func toggleLike(postId: String)
}

// MARK: - Post ViewModel (Presentation Model)
struct PostViewModel: Identifiable, Equatable {
    let id: String
    let authorName: String
    let content: String
    let likeCountText: String
    let isLiked: Bool
    let relativeTime: String
    
    init(post: Post) {
        self.id = post.id
        self.authorName = post.authorName
        self.content = post.content
        self.likeCountText = "\(post.likeCount) likes"
        self.isLiked = post.isLiked
        self.relativeTime = Self.formatRelativeTime(post.createdAt)
    }
    
    private static func formatRelativeTime(_ date: Date) -> String {
        let formatter = RelativeDateTimeFormatter()
        formatter.unitsStyle = .short
        return formatter.localizedString(for: date, relativeTo: Date())
    }
}

// MARK: - Feed ViewModel
final class FeedViewModel: FeedViewModelProtocol {
    
    // MARK: - Published State
    @Published private(set) var posts: [PostViewModel] = []
    @Published private(set) var isLoading = false
    @Published private(set) var errorMessage: String?
    @Published private(set) var hasMorePages = true
    
    // MARK: - Private State
    private var rawPosts: [Post] = []
    private var nextCursor: String?
    private var isLoadingMore = false
    
    // MARK: - Dependencies
    private let apiService: FeedAPIProtocol
    private var cancellables = Set<AnyCancellable>()
    
    // MARK: - Init
    init(apiService: FeedAPIProtocol) {
        self.apiService = apiService
    }
    
    // MARK: - Public Methods
    func loadFeed() {
        guard !isLoading else { return }
        isLoading = true
        errorMessage = nil
        
        Task { @MainActor in
            do {
                let response = try await apiService.fetchFeed(cursor: nil)
                self.rawPosts = response.posts
                self.posts = response.posts.map(PostViewModel.init)
                self.nextCursor = response.nextCursor
                self.hasMorePages = response.nextCursor != nil
            } catch {
                self.errorMessage = error.localizedDescription
            }
            self.isLoading = false
        }
    }
    
    func loadMore() {
        guard !isLoading, !isLoadingMore, let cursor = nextCursor else { return }
        isLoadingMore = true
        
        Task { @MainActor in
            do {
                let response = try await apiService.fetchFeed(cursor: cursor)
                self.rawPosts.append(contentsOf: response.posts)
                self.posts = self.rawPosts.map(PostViewModel.init)
                self.nextCursor = response.nextCursor
                self.hasMorePages = response.nextCursor != nil
            } catch {
                self.errorMessage = error.localizedDescription
            }
            self.isLoadingMore = false
        }
    }
    
    func refresh() {
        nextCursor = nil
        loadFeed()
    }
    
    func toggleLike(postId: String) {
        guard let index = rawPosts.firstIndex(where: { $0.id == postId }) else { return }
        
        // Optimistic update
        let originalPost = rawPosts[index]
        var updatedPost = originalPost
        updatedPost.isLiked.toggle()
        rawPosts[index] = updatedPost
        posts = rawPosts.map(PostViewModel.init)
        
        Task { @MainActor in
            do {
                let serverPost = try await apiService.toggleLike(
                    postId: postId,
                    isLiked: updatedPost.isLiked
                )
                if let idx = self.rawPosts.firstIndex(where: { $0.id == postId }) {
                    self.rawPosts[idx] = serverPost
                    self.posts = self.rawPosts.map(PostViewModel.init)
                }
            } catch {
                // Revert on failure
                if let idx = self.rawPosts.firstIndex(where: { $0.id == postId }) {
                    self.rawPosts[idx] = originalPost
                    self.posts = self.rawPosts.map(PostViewModel.init)
                }
                self.errorMessage = "Failed to update like"
            }
        }
    }
}
```

### View (UIKit ViewController)

```swift
import UIKit
import Combine

final class FeedViewController: UIViewController {
    
    // MARK: - Dependencies
    private let viewModel: FeedViewModel
    private var cancellables = Set<AnyCancellable>()
    
    // MARK: - UI
    private lazy var tableView: UITableView = {
        let tv = UITableView()
        tv.translatesAutoresizingMaskIntoConstraints = false
        tv.dataSource = self
        tv.delegate = self
        tv.register(PostTableViewCell.self, forCellReuseIdentifier: PostTableViewCell.reuseId)
        tv.refreshControl = refreshControl
        return tv
    }()
    
    private lazy var refreshControl: UIRefreshControl = {
        let rc = UIRefreshControl()
        rc.addTarget(self, action: #selector(handleRefresh), for: .valueChanged)
        return rc
    }()
    
    private lazy var loadingIndicator: UIActivityIndicatorView = {
        let ai = UIActivityIndicatorView(style: .large)
        ai.translatesAutoresizingMaskIntoConstraints = false
        ai.hidesWhenStopped = true
        return ai
    }()
    
    // MARK: - Local State (UI-only)
    private var posts: [PostViewModel] = []
    
    // MARK: - Init
    init(viewModel: FeedViewModel) {
        self.viewModel = viewModel
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
        viewModel.loadFeed()
    }
    
    // MARK: - Setup
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
    
    // MARK: - Binding (The MVVM magic ✨)
    private func bindViewModel() {
        // Bind posts
        viewModel.$posts
            .receive(on: DispatchQueue.main)
            .sink { [weak self] posts in
                self?.posts = posts
                self?.tableView.reloadData()
            }
            .store(in: &cancellables)
        
        // Bind loading state
        viewModel.$isLoading
            .receive(on: DispatchQueue.main)
            .sink { [weak self] isLoading in
                if isLoading && self?.posts.isEmpty == true {
                    self?.loadingIndicator.startAnimating()
                } else {
                    self?.loadingIndicator.stopAnimating()
                    self?.refreshControl.endRefreshing()
                }
            }
            .store(in: &cancellables)
        
        // Bind error
        viewModel.$errorMessage
            .compactMap { $0 }
            .receive(on: DispatchQueue.main)
            .sink { [weak self] message in
                self?.showError(message)
            }
            .store(in: &cancellables)
    }
    
    // MARK: - Actions
    @objc private func handleRefresh() {
        viewModel.refresh()
    }
    
    private func showError(_ message: String) {
        let alert = UIAlertController(title: "Error", message: message, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .default))
        present(alert, animated: true)
    }
}

// MARK: - UITableViewDataSource
extension FeedViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return posts.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: PostTableViewCell.reuseId, for: indexPath) as! PostTableViewCell
        let post = posts[indexPath.row]
        cell.configure(with: post)
        cell.onLikeTapped = { [weak self] in
            self?.viewModel.toggleLike(postId: post.id)
        }
        return cell
    }
}

// MARK: - UITableViewDelegate
extension FeedViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        // Pagination trigger
        if indexPath.row == posts.count - 3 {
            viewModel.loadMore()
        }
    }
    
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
        // ⚠️ Navigation is still in ViewController - this is MVVM's weakness
        // See MVVM+C for the solution
    }
}
```

### SwiftUI Version

```swift
import SwiftUI

struct FeedView: View {
    @StateObject var viewModel: FeedViewModel
    
    var body: some View {
        NavigationStack {
            content
                .navigationTitle("Feed")
                .refreshable {
                    viewModel.refresh()
                }
                .task {
                    viewModel.loadFeed()
                }
        }
    }
    
    @ViewBuilder
    private var content: some View {
        if viewModel.isLoading && viewModel.posts.isEmpty {
            ProgressView()
        } else if let error = viewModel.errorMessage, viewModel.posts.isEmpty {
            ErrorView(message: error) {
                viewModel.loadFeed()
            }
        } else {
            postList
        }
    }
    
    private var postList: some View {
        List {
            ForEach(viewModel.posts) { post in
                PostRow(post: post) {
                    viewModel.toggleLike(postId: post.id)
                }
                .onAppear {
                    if post.id == viewModel.posts.last?.id {
                        viewModel.loadMore()
                    }
                }
            }
        }
        .listStyle(.plain)
    }
}

struct PostRow: View {
    let post: PostViewModel
    let onLikeTapped: () -> Void
    
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            HStack {
                Text(post.authorName)
                    .font(.headline)
                Spacer()
                Text(post.relativeTime)
                    .font(.caption)
                    .foregroundColor(.secondary)
            }
            
            Text(post.content)
                .font(.body)
            
            HStack {
                Button(action: onLikeTapped) {
                    Image(systemName: post.isLiked ? "heart.fill" : "heart")
                        .foregroundColor(post.isLiked ? .red : .gray)
                }
                Text(post.likeCountText)
                    .font(.caption)
            }
        }
        .padding(.vertical, 4)
    }
}
```

---

## 9️⃣ MVVM Deep Dive

### Role of ViewModel

1. **Data Transformation** – Convert Model to display-ready format
2. **State Management** – Track loading, error, success states
3. **Business Logic** – Handle user actions, validation
4. **API Coordination** – Call services, handle responses

### Data Binding with Combine

```swift
// Option 1: Multiple @Published properties
class ViewModelA: ObservableObject {
    @Published var data: [Item] = []
    @Published var isLoading = false
    @Published var error: Error?
}

// Option 2: Single state enum (Redux-style)
class ViewModelB: ObservableObject {
    enum State {
        case idle
        case loading
        case loaded([Item])
        case error(Error)
    }
    
    @Published var state: State = .idle
}

// Option 3: Input/Output pattern (RxSwift-inspired)
class ViewModelC {
    struct Input {
        let loadTrigger: AnyPublisher<Void, Never>
        let refreshTrigger: AnyPublisher<Void, Never>
        let itemSelected: AnyPublisher<Int, Never>
    }
    
    struct Output {
        let items: AnyPublisher<[Item], Never>
        let isLoading: AnyPublisher<Bool, Never>
        let error: AnyPublisher<Error, Never>
    }
    
    func transform(input: Input) -> Output {
        // Combine logic here
    }
}
```

### Where MVVM Becomes Hard to Manage

| Scenario | Problem |
|----------|---------|
| **Deep navigation flows** | ViewModel shouldn't know about other screens |
| **Shared state across screens** | No built-in state sharing mechanism |
| **Complex form validation** | Multiple fields = many @Published properties |
| **Feature flags + A/B tests** | Logic scattered across ViewModels |

---

## 📋 When to Use MVVM

### ✅ Good Use Cases
- Medium-sized apps (10-30 screens)
- SwiftUI projects (natural fit)
- Apps requiring good test coverage
- Teams of 2-7 developers
- When you need to separate UI from logic

### ❌ Avoid Pure MVVM When
- Complex navigation flows (use MVVM+C)
- Very large apps (50+ screens)
- Multiple feature teams (need stronger boundaries)
- Deep linking requirements

---

## 🎯 Interview Tips

### Common Questions

**Q: What's the difference between MVC and MVVM?**
> "In MVC, the Controller is the middleman between View and Model, but it often ends up doing everything. In MVVM, UIViewController becomes part of the View layer, and the ViewModel handles all the logic. The key benefit is that ViewModel has no UI dependencies, making it fully testable."

**Q: How do you handle navigation in MVVM?**
> "Pure MVVM struggles with navigation because ViewModel shouldn't know about UIKit. Common solutions include: exposing navigation events via Combine publishers, using delegate callbacks, or adopting MVVM+Coordinator where a separate Coordinator handles all navigation."

**Q: When would you choose MVVM over MVC?**
> "When testability matters, when building with SwiftUI, when the app has meaningful business logic that should be tested, or when the team is large enough that code organization matters."

---

## 📚 Next Steps

Need cleaner navigation? Continue to [MVVM + Coordinator →](./03_MVVM+C.md)
