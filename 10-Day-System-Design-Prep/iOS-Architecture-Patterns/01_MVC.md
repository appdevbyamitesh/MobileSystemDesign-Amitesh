# 🏛️ MVC (Model-View-Controller)

> **Apple's Default Pattern - The Foundation of iOS Development**

---

## 1️⃣ What It Is (Simple English)

MVC divides your app into three roles:
- **Model** – The data and business logic (what your app knows)
- **View** – The UI elements (what users see)
- **Controller** – The coordinator that connects Model and View (the traffic cop)

### Real-Life Analogy: Restaurant

| Role | Restaurant | iOS |
|------|-----------|-----|
| **Model** | Kitchen (prepares food) | Data classes, API responses |
| **View** | Dining area (tables, plates) | UILabel, UIButton, UITableView |
| **Controller** | Waiter (takes orders, serves food) | UIViewController |

The waiter doesn't cook (no business logic in Controller ideally), and the kitchen doesn't see customers (Model doesn't know about UI).

---

## 2️⃣ Core Components

```
┌─────────────────────────────────────────────────────────────┐
│                         MVC Pattern                         │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│    ┌─────────┐    Updates    ┌─────────┐                   │
│    │  Model  │──────────────►│  View   │                   │
│    └────▲────┘               └────▲────┘                   │
│         │                         │                         │
│         │ Updates                 │ User Actions           │
│         │                         │                         │
│    ┌────┴─────────────────────────┴────┐                   │
│    │          Controller               │                   │
│    │       (UIViewController)          │                   │
│    └───────────────────────────────────┘                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Component Responsibilities

| Component | Responsibilities |
|-----------|------------------|
| **Model** | Data storage, business rules, API calls, persistence |
| **View** | Display data, capture user input, animations |
| **Controller** | Receive user actions, update Model, refresh View, handle navigation |

---

## 3️⃣ Data & Control Flow

### User Action Flow (Tap "Like" Button)

```
┌──────────────────────────────────────────────────────────────────┐
│ USER TAPS "LIKE" BUTTON                                          │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. View: UIButton sends action to ViewController                │
│  2. Controller: Receives @IBAction likeButtonTapped()            │
│  3. Controller: Calls model.toggleLike()                         │
│  4. Model: Updates isLiked = true, likeCount += 1                │
│  5. Model: Posts notification or calls completion handler        │
│  6. Controller: Receives update, calls view.updateLikeButton()   │
│  7. View: Updates button image and label                         │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram (Text)

```
User         View          Controller         Model          API
 │            │                │                │              │
 │───tap──────►│                │                │              │
 │            │────action()────►│                │              │
 │            │                │────toggleLike()─►│              │
 │            │                │                │───POST /like──►│
 │            │                │                │◄──200 OK──────│
 │            │                │◄───callback()──│              │
 │            │◄──updateUI()───│                │              │
 │◄───visual feedback──────────│                │              │
```

---

## 4️⃣ Strengths

✅ **Apple's recommended pattern** – All documentation, tutorials, and Xcode templates use MVC

✅ **Low learning curve** – Easy for beginners, quick to get started

✅ **Fast prototyping** – Fewer files, less boilerplate, quick iterations

✅ **Built-in framework support** – UIViewController is designed for MVC

✅ **Good for small apps** – 5-10 screens with simple logic work great

---

## 5️⃣ Limitations & Failure Points

### ❌ Massive View Controller (The Famous Problem)

```swift
// This is what happens in production MVC apps:
class FeedViewController: UIViewController {
    // 1. View setup (50 lines)
    // 2. TableView delegate/datasource (100 lines)
    // 3. Networking code (80 lines)
    // 4. Data transformation (60 lines)
    // 5. Navigation (40 lines)
    // 6. Error handling (50 lines)
    // 7. Analytics (30 lines)
    // 8. Caching logic (40 lines)
    // = 450+ lines in ONE file
}
```

### Why Massive View Controller Happens

| Problem | Why It Occurs |
|---------|---------------|
| **Unclear boundaries** | "Where does this code go?" → Controller! |
| **View lifecycle coupling** | Business logic mixed with viewDidLoad |
| **Tight View-Controller coupling** | Can't test Controller without View |
| **No clear networking layer** | API calls directly in Controller |
| **Navigation in Controller** | Hard to reuse across flows |

### Real Production Pain Points

1. **Testing is nearly impossible** – ViewControllers require full UI setup
2. **Code reviews are painful** – 500+ line files with mixed concerns
3. **Feature isolation fails** – Change one thing, break another
4. **Onboarding is slow** – New devs can't understand the "blob"
5. **Merge conflicts are constant** – Everyone touches the same files

---

## 6️⃣ iOS-Specific Considerations

### Navigation Handling

```swift
// In MVC, navigation lives in the Controller
class ProductListViewController: UIViewController {
    func didSelectProduct(_ product: Product) {
        // ❌ Tight coupling - Controller knows about all destinations
        let detailVC = ProductDetailViewController(product: product)
        navigationController?.pushViewController(detailVC, animated: true)
    }
}
```

**Problem**: Controller knows about other Controllers → Hard to test, hard to reuse.

### Concurrency & async/await

```swift
class FeedViewController: UIViewController {
    func loadFeed() {
        Task {
            do {
                let posts = try await api.fetchFeed()
                // Must dispatch to main thread
                await MainActor.run {
                    self.posts = posts
                    self.tableView.reloadData()
                }
            } catch {
                await MainActor.run {
                    self.showError(error)
                }
            }
        }
    }
}
```

**Issue**: Mixing async code with UI updates in one file becomes messy.

### Memory Management & Retain Cycles

```swift
class FeedViewController: UIViewController {
    var posts: [Post] = []
    
    func loadData() {
        // ⚠️ RETAIN CYCLE RISK
        api.fetchFeed { posts in
            self.posts = posts  // Strong reference to self
            self.tableView.reloadData()
        }
        
        // ✅ CORRECT
        api.fetchFeed { [weak self] posts in
            self?.posts = posts
            self?.tableView.reloadData()
        }
    }
}
```

### SwiftUI Compatibility

```swift
// MVC doesn't translate well to SwiftUI
// SwiftUI is declarative - no "Controller" exists

// In UIKit MVC:
class ProfileViewController: UIViewController {
    func updateUI() {
        nameLabel.text = user.name
        avatarImageView.image = user.avatar
    }
}

// In SwiftUI (naturally fits MVVM):
struct ProfileView: View {
    @ObservedObject var viewModel: ProfileViewModel
    
    var body: some View {
        VStack {
            Text(viewModel.name)
            Image(viewModel.avatar)
        }
    }
}
```

**Verdict**: MVC doesn't work well with SwiftUI. Migrate to MVVM for SwiftUI projects.

---

## 7️⃣ Testability & Scalability

### Unit Testing Difficulty: 🔴 Hard

```swift
// Testing a ViewController in MVC
func testFeedLoading() {
    // ❌ You need to:
    // 1. Instantiate the ViewController
    // 2. Trigger viewDidLoad (load the View)
    // 3. Mock the network layer (if it's even injectable)
    // 4. Wait for async callbacks
    // 5. Assert on UI elements
    
    let vc = FeedViewController()
    _ = vc.view  // Force viewDidLoad
    // ... complex setup, hard to maintain
}
```

**Problem**: Can't test business logic without loading the entire View.

### Dependency Injection Support: 🟡 Possible but Awkward

```swift
// DI in MVC requires manual constructor injection
class FeedViewController: UIViewController {
    private let apiService: APIServiceProtocol
    private let cacheService: CacheServiceProtocol
    
    // ✅ Better: Inject dependencies
    init(apiService: APIServiceProtocol, cacheService: CacheServiceProtocol) {
        self.apiService = apiService
        self.cacheService = cacheService
        super.init(nibName: nil, bundle: nil)
    }
    
    // ❌ Worse: Storyboard segues don't support custom init
    required init?(coder: NSCoder) {
        fatalError("Use init(apiService:cacheService:)")
    }
}
```

### Team Scalability: 🔴 Poor for Large Teams

| Team Size | MVC Suitability |
|-----------|-----------------|
| 1-2 developers | ✅ Great |
| 3-5 developers | ⚠️ Starting to hurt |
| 5+ developers | ❌ Constant conflicts |

---

## 8️⃣ Complete Swift Example: Feed Screen

### Model

```swift
// MARK: - Models
struct Post: Codable, Identifiable {
    let id: String
    let authorName: String
    let content: String
    let likeCount: Int
    var isLiked: Bool
    let createdAt: Date
}

struct FeedResponse: Codable {
    let posts: [Post]
    let nextCursor: String?
}
```

### API Service

```swift
// MARK: - Networking (Often lives in Controller in pure MVC 😬)
protocol FeedAPIProtocol {
    func fetchFeed(cursor: String?) async throws -> FeedResponse
    func toggleLike(postId: String, isLiked: Bool) async throws -> Post
}

class FeedAPIService: FeedAPIProtocol {
    private let session: URLSession
    private let baseURL: URL
    
    init(session: URLSession = .shared, baseURL: URL) {
        self.session = session
        self.baseURL = baseURL
    }
    
    func fetchFeed(cursor: String?) async throws -> FeedResponse {
        var components = URLComponents(url: baseURL.appendingPathComponent("feed"), resolvingAgainstBaseURL: false)!
        if let cursor = cursor {
            components.queryItems = [URLQueryItem(name: "cursor", value: cursor)]
        }
        
        let (data, _) = try await session.data(from: components.url!)
        return try JSONDecoder().decode(FeedResponse.self, from: data)
    }
    
    func toggleLike(postId: String, isLiked: Bool) async throws -> Post {
        var request = URLRequest(url: baseURL.appendingPathComponent("posts/\(postId)/like"))
        request.httpMethod = isLiked ? "POST" : "DELETE"
        
        let (data, _) = try await session.data(for: request)
        return try JSONDecoder().decode(Post.self, from: data)
    }
}
```

### View (Custom Cell)

```swift
// MARK: - View
protocol PostCellDelegate: AnyObject {
    func postCell(_ cell: PostCell, didTapLikeFor postId: String)
}

class PostCell: UITableViewCell {
    static let reuseIdentifier = "PostCell"
    
    weak var delegate: PostCellDelegate?
    private var postId: String?
    
    private let authorLabel = UILabel()
    private let contentLabel = UILabel()
    private let likeButton = UIButton(type: .system)
    private let likeCountLabel = UILabel()
    
    override init(style: UITableViewCell.CellStyle, reuseIdentifier: String?) {
        super.init(style: style, reuseIdentifier: reuseIdentifier)
        setupUI()
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    private func setupUI() {
        let stack = UIStackView(arrangedSubviews: [authorLabel, contentLabel, likeButton, likeCountLabel])
        stack.axis = .vertical
        stack.spacing = 8
        stack.translatesAutoresizingMaskIntoConstraints = false
        
        contentView.addSubview(stack)
        NSLayoutConstraint.activate([
            stack.topAnchor.constraint(equalTo: contentView.topAnchor, constant: 12),
            stack.leadingAnchor.constraint(equalTo: contentView.leadingAnchor, constant: 16),
            stack.trailingAnchor.constraint(equalTo: contentView.trailingAnchor, constant: -16),
            stack.bottomAnchor.constraint(equalTo: contentView.bottomAnchor, constant: -12)
        ])
        
        likeButton.addTarget(self, action: #selector(likeButtonTapped), for: .touchUpInside)
    }
    
    func configure(with post: Post) {
        self.postId = post.id
        authorLabel.text = post.authorName
        contentLabel.text = post.content
        likeCountLabel.text = "\(post.likeCount) likes"
        
        let imageName = post.isLiked ? "heart.fill" : "heart"
        likeButton.setImage(UIImage(systemName: imageName), for: .normal)
        likeButton.tintColor = post.isLiked ? .systemRed : .systemGray
    }
    
    @objc private func likeButtonTapped() {
        guard let postId = postId else { return }
        delegate?.postCell(self, didTapLikeFor: postId)
    }
}
```

### Controller (The "Massive" One)

```swift
// MARK: - Controller (This is where MVC gets messy)
class FeedViewController: UIViewController {
    
    // MARK: - Dependencies
    private let apiService: FeedAPIProtocol
    
    // MARK: - State
    private var posts: [Post] = []
    private var nextCursor: String?
    private var isLoading = false
    
    // MARK: - UI
    private let tableView = UITableView()
    private let refreshControl = UIRefreshControl()
    private let loadingIndicator = UIActivityIndicatorView(style: .large)
    
    // MARK: - Init
    init(apiService: FeedAPIProtocol) {
        self.apiService = apiService
        super.init(nibName: nil, bundle: nil)
    }
    
    required init?(coder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    // MARK: - Lifecycle
    override func viewDidLoad() {
        super.viewDidLoad()
        setupUI()
        loadFeed()
    }
    
    // MARK: - UI Setup
    private func setupUI() {
        title = "Feed"
        view.backgroundColor = .systemBackground
        
        // TableView
        tableView.translatesAutoresizingMaskIntoConstraints = false
        tableView.dataSource = self
        tableView.delegate = self
        tableView.register(PostCell.self, forCellReuseIdentifier: PostCell.reuseIdentifier)
        tableView.refreshControl = refreshControl
        view.addSubview(tableView)
        
        // Refresh Control
        refreshControl.addTarget(self, action: #selector(handleRefresh), for: .valueChanged)
        
        // Loading Indicator
        loadingIndicator.translatesAutoresizingMaskIntoConstraints = false
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
    
    // MARK: - Data Loading (Business logic in Controller 😬)
    private func loadFeed(refresh: Bool = false) {
        guard !isLoading else { return }
        isLoading = true
        
        if posts.isEmpty {
            loadingIndicator.startAnimating()
        }
        
        let cursor = refresh ? nil : nextCursor
        
        Task {
            do {
                let response = try await apiService.fetchFeed(cursor: cursor)
                
                await MainActor.run {
                    if refresh {
                        self.posts = response.posts
                    } else {
                        self.posts.append(contentsOf: response.posts)
                    }
                    self.nextCursor = response.nextCursor
                    self.tableView.reloadData()
                    self.loadingIndicator.stopAnimating()
                    self.refreshControl.endRefreshing()
                    self.isLoading = false
                }
            } catch {
                await MainActor.run {
                    self.showError(error)
                    self.loadingIndicator.stopAnimating()
                    self.refreshControl.endRefreshing()
                    self.isLoading = false
                }
            }
        }
    }
    
    @objc private func handleRefresh() {
        loadFeed(refresh: true)
    }
    
    private func showError(_ error: Error) {
        let alert = UIAlertController(title: "Error", message: error.localizedDescription, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "Retry", style: .default) { [weak self] _ in
            self?.loadFeed()
        })
        alert.addAction(UIAlertAction(title: "Cancel", style: .cancel))
        present(alert, animated: true)
    }
    
    // MARK: - Like Action (More business logic in Controller 😬)
    private func toggleLike(for postId: String) {
        guard let index = posts.firstIndex(where: { $0.id == postId }) else { return }
        let post = posts[index]
        
        // Optimistic update
        var updatedPost = post
        updatedPost.isLiked.toggle()
        posts[index] = updatedPost
        tableView.reloadRows(at: [IndexPath(row: index, section: 0)], with: .none)
        
        Task {
            do {
                let serverPost = try await apiService.toggleLike(postId: postId, isLiked: updatedPost.isLiked)
                await MainActor.run {
                    if let idx = self.posts.firstIndex(where: { $0.id == postId }) {
                        self.posts[idx] = serverPost
                        self.tableView.reloadRows(at: [IndexPath(row: idx, section: 0)], with: .none)
                    }
                }
            } catch {
                // Revert on failure
                await MainActor.run {
                    if let idx = self.posts.firstIndex(where: { $0.id == postId }) {
                        self.posts[idx] = post
                        self.tableView.reloadRows(at: [IndexPath(row: idx, section: 0)], with: .none)
                    }
                    self.showError(error)
                }
            }
        }
    }
}

// MARK: - UITableViewDataSource
extension FeedViewController: UITableViewDataSource {
    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return posts.count
    }
    
    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let cell = tableView.dequeueReusableCell(withIdentifier: PostCell.reuseIdentifier, for: indexPath) as! PostCell
        cell.configure(with: posts[indexPath.row])
        cell.delegate = self
        return cell
    }
}

// MARK: - UITableViewDelegate
extension FeedViewController: UITableViewDelegate {
    func tableView(_ tableView: UITableView, didSelectRowAt indexPath: IndexPath) {
        tableView.deselectRow(at: indexPath, animated: true)
        
        // ❌ Navigation tightly coupled to Controller
        let post = posts[indexPath.row]
        let detailVC = PostDetailViewController(post: post)
        navigationController?.pushViewController(detailVC, animated: true)
    }
    
    func tableView(_ tableView: UITableView, willDisplay cell: UITableViewCell, forRowAt indexPath: IndexPath) {
        // Pagination
        if indexPath.row == posts.count - 3 && nextCursor != nil {
            loadFeed()
        }
    }
}

// MARK: - PostCellDelegate
extension FeedViewController: PostCellDelegate {
    func postCell(_ cell: PostCell, didTapLikeFor postId: String) {
        toggleLike(for: postId)
    }
}
```

---

## 9️⃣ Why Apple Chose MVC

1. **Historical reasons** – MVC was the dominant pattern in Cocoa (macOS) before iOS
2. **Simple mental model** – Three clear roles are easy to explain
3. **Framework integration** – UIViewController naturally fits the Controller role
4. **Low barrier to entry** – New developers can ship apps quickly

---

## 🔟 Why MVC Struggles at Scale

| Scale Factor | Problem |
|--------------|---------|
| **More features** | Controllers grow to 1000+ lines |
| **More developers** | Everyone edits the same Controller files |
| **More tests** | Can't unit test Controllers without UI |
| **Complex navigation** | Push/present logic scattered everywhere |
| **Reusable components** | Business logic trapped in Controllers |

---

## 📋 When to Use MVC

### ✅ Good Use Cases
- Prototypes and MVPs
- Simple apps with 5-10 screens
- Apps with minimal business logic
- Learning iOS development
- Quick internal tools

### ❌ Avoid MVC When
- Building a large production app
- Working with a team of 3+ developers
- Requiring high test coverage
- Complex navigation flows
- Long-term maintenance expected

---

## 🎯 Interview Tips

### Common Questions

**Q: Why does MVC lead to Massive View Controllers?**
> "MVC doesn't have a clear place for business logic, networking, or data transformation. Since the View is passive and the Model should be pure data, developers end up putting everything in the Controller by default."

**Q: Can you make MVC testable?**
> "Partially. You can extract business logic into separate Service classes and inject them into Controllers. But you still can't unit test the Controller itself without instantiating the View. That's why patterns like MVVM exist."

**Q: When would you still choose MVC?**
> "For prototypes, internal tools, or very simple apps where testability and scalability aren't priorities. Also for onboarding junior developers who need to learn iOS fundamentals first."

---

## 📚 Next Steps

Ready to solve MVC's problems? Continue to [MVVM →](./02_MVVM.md)
