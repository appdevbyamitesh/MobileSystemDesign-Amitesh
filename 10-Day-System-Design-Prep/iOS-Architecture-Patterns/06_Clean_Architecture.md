# 🧅 Clean Architecture

> **Uncle Bob's Layered Approach - Dependency Inversion at Scale**

---

## 1️⃣ What It Is (Simple English)

Clean Architecture organizes code into **concentric layers** where dependencies only point **inward**. The core business logic is protected from external concerns.

Three main layers:
- **Presentation Layer** – UI, ViewModels, Views (outermost)
- **Domain Layer** – Business logic, Use Cases, Entities (center)
- **Data Layer** – Repositories, API, Database (outermost)

### Real-Life Analogy: A Company

| Layer | Company | iOS |
|-------|---------|-----|
| **Presentation** | Front desk (talks to customers) | UIViewController, SwiftUI Views |
| **Domain** | Core business (makes decisions) | Use Cases, Entities |
| **Data** | Back office (stores records) | Repositories, API, CoreData |

The front desk doesn't know how records are stored. The back office doesn't know how to talk to customers. The core business doesn't care about either—it just processes requests.

### The Key Insight

> **The Domain layer knows NOTHING about UIKit, networking, or databases.**
> 
> **It only defines what the app DOES, not HOW it does it.**

---

## 2️⃣ Core Components

```
┌─────────────────────────────────────────────────────────────────┐
│                      Clean Architecture                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    ┌───────────────────────────────────────────────────────┐   │
│    │                  PRESENTATION LAYER                   │   │
│    │    ┌─────────┐    ┌───────────┐    ┌──────────┐      │   │
│    │    │  View   │◄───│ ViewModel │◄───│Coordinator│      │   │
│    │    └─────────┘    └─────┬─────┘    └──────────┘      │   │
│    └────────────────────────┼──────────────────────────────┘   │
│                             │ uses                              │
│    ┌────────────────────────▼──────────────────────────────┐   │
│    │                    DOMAIN LAYER                       │   │
│    │    ┌───────────┐    ┌──────────┐    ┌──────────┐     │   │
│    │    │ Use Cases │────│ Entities │    │Interfaces│     │   │
│    │    └─────┬─────┘    └──────────┘    │(protocols)│     │   │
│    │          │                          └──────────┘     │   │
│    └──────────┼────────────────────────────────────────────┘   │
│               │ uses                                            │
│    ┌──────────▼────────────────────────────────────────────┐   │
│    │                     DATA LAYER                        │   │
│    │    ┌────────────┐    ┌─────────┐    ┌───────────┐    │   │
│    │    │ Repository │────│   API   │    │  Database │    │   │
│    │    │   Impl     │    │ Service │    │  Service  │    │   │
│    │    └────────────┘    └─────────┘    └───────────┘    │   │
│    └───────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### The Dependency Rule

```
Dependencies point INWARD only:

Presentation → Domain ✅
Data → Domain ✅
Domain → Presentation ❌
Domain → Data ❌
```

### Component Responsibilities

| Layer | Component | Responsibilities |
|-------|-----------|------------------|
| **Presentation** | View | Display UI, capture input |
| | ViewModel | UI state, transform data for display |
| | Coordinator | Navigation |
| **Domain** | Use Case | One business action (GetFeedUseCase) |
| | Entity | Business objects (Post, User) |
| | Repository Protocol | Interface for data access |
| **Data** | Repository Impl | Actual data fetching/caching |
| | API Service | Network calls |
| | Database Service | Local persistence |

---

## 3️⃣ Data & Control Flow

### User Action Flow (Load Feed)

```
┌──────────────────────────────────────────────────────────────────┐
│ USER TAPS "LOAD FEED"                                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  PRESENTATION:                                                   │
│  1. View: Calls viewModel.loadFeed()                             │
│  2. ViewModel: Calls getFeedUseCase.execute()                    │
│                                                                  │
│  DOMAIN:                                                         │
│  3. UseCase: Calls repository.fetchFeed() (protocol)             │
│                                                                  │
│  DATA:                                                           │
│  4. Repository: Checks cache, calls API if needed                │
│  5. API: Makes network request                                   │
│  6. Repository: Returns domain entities                          │
│                                                                  │
│  Back up the stack:                                              │
│  7. UseCase: Returns Result<[Post], Error>                       │
│  8. ViewModel: Transforms to [PostViewModel]                     │
│  9. View: Updates UI                                             │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Sequence Diagram (Text)

```
View      ViewModel     UseCase      Repository     API
 │           │            │              │           │
 │──load()───►│            │              │           │
 │           │──execute()─►│              │           │
 │           │            │──fetchFeed()─►│           │
 │           │            │              │──GET------►│
 │           │            │              │◄─200 OK───│
 │           │            │◄──[Post]─────│           │
 │           │◄─Result────│              │           │
 │◄─update───│            │              │           │
```

### Dependency Inversion in Action

```swift
// Domain layer defines the INTERFACE
protocol PostRepositoryProtocol {
    func fetchFeed() async throws -> [Post]
    func likePost(_ id: String) async throws -> Post
}

// Data layer IMPLEMENTS the interface
class PostRepository: PostRepositoryProtocol {
    private let apiService: APIServiceProtocol
    private let cache: CacheServiceProtocol
    
    func fetchFeed() async throws -> [Post] {
        // Check cache first
        if let cached = cache.get("feed") as? [Post] {
            return cached
        }
        // Fetch from API
        let response = try await apiService.fetch(FeedEndpoint.list)
        let posts = try JSONDecoder().decode([Post].self, from: response)
        cache.set("feed", value: posts)
        return posts
    }
}
```

---

## 4️⃣ Strengths

✅ **Framework agnostic** – Domain layer has no dependencies on UIKit, SwiftUI, or any framework

✅ **Highly testable** – Each layer can be tested in isolation

✅ **Scalable** – Clear boundaries prevent code sprawl

✅ **Swappable** – Change database from CoreData to Realm without touching Domain

✅ **Team-friendly** – Clear ownership (UI team, business logic team, data team)

✅ **Long-lived** – Business logic survives framework migrations

---

## 5️⃣ Limitations & Failure Points

### ❌ Verbose for Simple Features

```swift
// To add a simple "get posts" feature, you need:
// 1. Post entity
// 2. PostRepositoryProtocol
// 3. GetFeedUseCase
// 4. FeedViewModel
// 5. FeedView
// 6. PostRepository implementation
// 7. PostAPIService
// = 7+ files for one feature
```

### ❌ Use Case Explosion

```swift
// Every action becomes a use case
class GetFeedUseCase {}
class RefreshFeedUseCase {}
class LikePostUseCase {}
class UnlikePostUseCase {}
class SavePostUseCase {}
class SharePostUseCase {}
class ReportPostUseCase {}
// Dozens of small classes
```

### ❌ Over-abstraction Risk

```swift
// Sometimes abstraction goes too far
protocol PostRepositoryProtocol {}
protocol NetworkPostDataSourceProtocol {}
protocol LocalPostDataSourceProtocol {}
protocol PostCacheProtocol {}
protocol PostMapperProtocol {}
// 5 protocols for one data source
```

### Why It Can Feel Heavy for Small Teams

| Team Size | Pain Point |
|-----------|-----------|
| 1-2 devs | Too many files to navigate |
| 3-5 devs | Some layers feel redundant |
| 5+ devs | Benefits start to outweigh costs |

---

## 6️⃣ iOS-Specific Considerations

### Navigation Handling

```swift
// Navigation typically lives in Presentation layer with Coordinator

// MARK: - Coordinator
protocol FeedCoordinatorProtocol {
    func showPostDetail(_ post: Post)
    func showUserProfile(_ userId: String)
}

final class FeedCoordinator: FeedCoordinatorProtocol {
    private weak var navigationController: UINavigationController?
    private let postDetailFactory: PostDetailViewControllerFactory
    
    init(
        navigationController: UINavigationController,
        postDetailFactory: PostDetailViewControllerFactory
    ) {
        self.navigationController = navigationController
        self.postDetailFactory = postDetailFactory
    }
    
    func showPostDetail(_ post: Post) {
        let vc = postDetailFactory.make(post: post)
        navigationController?.pushViewController(vc, animated: true)
    }
}
```

### Concurrency & async/await

```swift
// Use Cases handle async operations cleanly
final class GetFeedUseCase {
    private let repository: PostRepositoryProtocol
    
    init(repository: PostRepositoryProtocol) {
        self.repository = repository
    }
    
    func execute() async -> Result<[Post], DomainError> {
        do {
            let posts = try await repository.fetchFeed()
            return .success(posts)
        } catch {
            return .failure(.networkError(error))
        }
    }
}

// ViewModel uses it naturally
final class FeedViewModel: ObservableObject {
    @Published var posts: [PostViewModel] = []
    @Published var state: ViewState = .idle
    
    private let getFeedUseCase: GetFeedUseCase
    
    @MainActor
    func loadFeed() async {
        state = .loading
        
        let result = await getFeedUseCase.execute()
        
        switch result {
        case .success(let posts):
            self.posts = posts.map(PostViewModel.init)
            state = .loaded
        case .failure(let error):
            state = .error(error.localizedDescription)
        }
    }
}
```

### Memory Management

```swift
// Clean Architecture naturally avoids retain cycles
// because dependencies flow one direction

// ViewModel owns UseCase (strong)
// UseCase owns Repository (strong, via protocol)
// No cycles!

class FeedViewModel {
    private let useCase: GetFeedUseCase  // Strong
}

class GetFeedUseCase {
    private let repository: PostRepositoryProtocol  // Strong
}

class PostRepository: PostRepositoryProtocol {
    // Doesn't reference ViewModel or UseCase
}
```

### SwiftUI Compatibility

```swift
// Clean Architecture works perfectly with SwiftUI

// View
struct FeedView: View {
    @StateObject var viewModel: FeedViewModel
    
    var body: some View {
        Group {
            switch viewModel.state {
            case .idle:
                EmptyView()
            case .loading:
                ProgressView()
            case .loaded:
                postList
            case .error(let message):
                ErrorView(message: message)
            }
        }
        .task { await viewModel.loadFeed() }
    }
    
    var postList: some View {
        List(viewModel.posts) { post in
            PostRow(post: post)
        }
    }
}

// ViewModel conforms to ObservableObject
class FeedViewModel: ObservableObject {
    @Published var posts: [PostViewModel] = []
    @Published var state: ViewState = .idle
}
```

---

## 7️⃣ Testability & Scalability

### Unit Testing: 🟢 Excellent

```swift
// Test Use Case (Domain layer)
class GetFeedUseCaseTests: XCTestCase {
    var sut: GetFeedUseCase!
    var mockRepository: MockPostRepository!
    
    override func setUp() {
        mockRepository = MockPostRepository()
        sut = GetFeedUseCase(repository: mockRepository)
    }
    
    func testExecute_success() async {
        mockRepository.stubbedPosts = [Post.mock()]
        
        let result = await sut.execute()
        
        switch result {
        case .success(let posts):
            XCTAssertEqual(posts.count, 1)
        case .failure:
            XCTFail("Expected success")
        }
    }
    
    func testExecute_networkError_returnsDomainError() async {
        mockRepository.stubbedError = NetworkError.noConnection
        
        let result = await sut.execute()
        
        switch result {
        case .success:
            XCTFail("Expected failure")
        case .failure(let error):
            XCTAssertEqual(error, .networkError)
        }
    }
}

// Test Repository (Data layer)
class PostRepositoryTests: XCTestCase {
    var sut: PostRepository!
    var mockAPI: MockAPIService!
    var mockCache: MockCacheService!
    
    func testFetchFeed_cacheHit_returnsCache() async throws {
        mockCache.stubbedValue = [Post.mock()]
        
        let result = try await sut.fetchFeed()
        
        XCTAssertFalse(mockAPI.fetchCalled)
        XCTAssertEqual(result.count, 1)
    }
    
    func testFetchFeed_cacheMiss_callsAPI() async throws {
        mockCache.stubbedValue = nil
        mockAPI.stubbedResponse = [Post.mock()]
        
        let result = try await sut.fetchFeed()
        
        XCTAssertTrue(mockAPI.fetchCalled)
        XCTAssertEqual(result.count, 1)
    }
}
```

### Dependency Injection: 🟢 Natural

```swift
// DI Container / Factory
final class DependencyContainer {
    // Data Layer
    lazy var apiService: APIServiceProtocol = APIService()
    lazy var cacheService: CacheServiceProtocol = CacheService()
    
    lazy var postRepository: PostRepositoryProtocol = PostRepository(
        apiService: apiService,
        cacheService: cacheService
    )
    
    // Domain Layer
    func makeGetFeedUseCase() -> GetFeedUseCase {
        GetFeedUseCase(repository: postRepository)
    }
    
    func makeLikePostUseCase() -> LikePostUseCase {
        LikePostUseCase(repository: postRepository)
    }
    
    // Presentation Layer
    func makeFeedViewModel() -> FeedViewModel {
        FeedViewModel(
            getFeedUseCase: makeGetFeedUseCase(),
            likePostUseCase: makeLikePostUseCase()
        )
    }
    
    func makeFeedViewController() -> FeedViewController {
        let viewModel = makeFeedViewModel()
        return FeedViewController(viewModel: viewModel)
    }
}
```

### Team Scalability: 🟢 Excellent

| Team Size | Clean Architecture Suitability |
|-----------|-------------------------------|
| 1-3 developers | ⚠️ Good, but verbose |
| 5-10 developers | ✅ Excellent |
| 10-20 developers | ✅ Great |
| 20+ developers | ✅ Great (consider RIBs for 50+) |

---

## 8️⃣ Complete Swift Example: Feed Feature

### Folder Structure

```
Features/
└── Feed/
    ├── Presentation/
    │   ├── FeedView.swift
    │   ├── FeedViewModel.swift
    │   ├── FeedCoordinator.swift
    │   └── ViewModels/
    │       └── PostViewModel.swift
    ├── Domain/
    │   ├── Entities/
    │   │   └── Post.swift
    │   ├── UseCases/
    │   │   ├── GetFeedUseCase.swift
    │   │   └── LikePostUseCase.swift
    │   └── Interfaces/
    │       └── PostRepositoryProtocol.swift
    └── Data/
        ├── Repositories/
        │   └── PostRepository.swift
        ├── Network/
        │   └── PostAPIMapper.swift
        └── DTOs/
            └── PostDTO.swift
```

### Domain Layer - Entity

```swift
// MARK: - Domain Entity
// Pure Swift struct, no framework dependencies
struct Post: Identifiable, Equatable {
    let id: String
    let authorId: String
    let authorName: String
    let content: String
    let likeCount: Int
    let isLiked: Bool
    let createdAt: Date
}

// Domain-level error
enum DomainError: Error, Equatable {
    case networkError
    case notFound
    case unauthorized
    case unknown
}
```

### Domain Layer - Repository Protocol

```swift
// MARK: - Repository Protocol (Domain Layer)
// Domain defines the interface, Data implements it
protocol PostRepositoryProtocol {
    func fetchFeed(page: Int) async throws -> [Post]
    func likePost(id: String) async throws -> Post
    func unlikePost(id: String) async throws -> Post
}
```

### Domain Layer - Use Cases

```swift
// MARK: - Get Feed Use Case
final class GetFeedUseCase {
    private let repository: PostRepositoryProtocol
    
    init(repository: PostRepositoryProtocol) {
        self.repository = repository
    }
    
    func execute(page: Int = 0) async -> Result<[Post], DomainError> {
        do {
            let posts = try await repository.fetchFeed(page: page)
            return .success(posts)
        } catch {
            return .failure(mapError(error))
        }
    }
    
    private func mapError(_ error: Error) -> DomainError {
        switch error {
        case is URLError:
            return .networkError
        default:
            return .unknown
        }
    }
}

// MARK: - Like Post Use Case
final class LikePostUseCase {
    private let repository: PostRepositoryProtocol
    
    init(repository: PostRepositoryProtocol) {
        self.repository = repository
    }
    
    func execute(postId: String, isCurrentlyLiked: Bool) async -> Result<Post, DomainError> {
        do {
            let post: Post
            if isCurrentlyLiked {
                post = try await repository.unlikePost(id: postId)
            } else {
                post = try await repository.likePost(id: postId)
            }
            return .success(post)
        } catch {
            return .failure(.networkError)
        }
    }
}
```

### Data Layer - DTO

```swift
// MARK: - Data Transfer Object
// Decoupled from domain entity
struct PostDTO: Decodable {
    let id: String
    let author_id: String
    let author_name: String
    let content: String
    let like_count: Int
    let is_liked: Bool
    let created_at: String
}

// MARK: - Mapper
extension PostDTO {
    func toDomain() -> Post {
        Post(
            id: id,
            authorId: author_id,
            authorName: author_name,
            content: content,
            likeCount: like_count,
            isLiked: is_liked,
            createdAt: ISO8601DateFormatter().date(from: created_at) ?? Date()
        )
    }
}
```

### Data Layer - Repository Implementation

```swift
// MARK: - Repository Implementation
final class PostRepository: PostRepositoryProtocol {
    private let apiService: APIServiceProtocol
    private let cacheService: CacheServiceProtocol
    
    init(apiService: APIServiceProtocol, cacheService: CacheServiceProtocol) {
        self.apiService = apiService
        self.cacheService = cacheService
    }
    
    func fetchFeed(page: Int) async throws -> [Post] {
        // Check cache for first page
        if page == 0, let cached: [Post] = cacheService.get(key: "feed_page_0") {
            // Return cache and refresh in background
            Task {
                try? await refreshFeed(page: page)
            }
            return cached
        }
        
        return try await refreshFeed(page: page)
    }
    
    private func refreshFeed(page: Int) async throws -> [Post] {
        let endpoint = FeedEndpoint.list(page: page)
        let data = try await apiService.request(endpoint)
        let dtos = try JSONDecoder().decode([PostDTO].self, from: data)
        let posts = dtos.map { $0.toDomain() }
        
        if page == 0 {
            cacheService.set(key: "feed_page_0", value: posts)
        }
        
        return posts
    }
    
    func likePost(id: String) async throws -> Post {
        let endpoint = FeedEndpoint.like(postId: id)
        let data = try await apiService.request(endpoint)
        let dto = try JSONDecoder().decode(PostDTO.self, from: data)
        return dto.toDomain()
    }
    
    func unlikePost(id: String) async throws -> Post {
        let endpoint = FeedEndpoint.unlike(postId: id)
        let data = try await apiService.request(endpoint)
        let dto = try JSONDecoder().decode(PostDTO.self, from: data)
        return dto.toDomain()
    }
}
```

### Presentation Layer - ViewModel

```swift
// MARK: - View State
enum ViewState: Equatable {
    case idle
    case loading
    case loaded
    case refreshing
    case error(String)
}

// MARK: - Post ViewModel (for display)
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
        
        let formatter = RelativeDateTimeFormatter()
        self.relativeTime = formatter.localizedString(for: post.createdAt, relativeTo: Date())
    }
}

// MARK: - Feed ViewModel
@MainActor
final class FeedViewModel: ObservableObject {
    
    // MARK: - Published State
    @Published private(set) var posts: [PostViewModel] = []
    @Published private(set) var state: ViewState = .idle
    
    // MARK: - Dependencies
    private let getFeedUseCase: GetFeedUseCase
    private let likePostUseCase: LikePostUseCase
    private let coordinator: FeedCoordinatorProtocol
    
    // MARK: - Private State
    private var rawPosts: [Post] = []
    private var currentPage = 0
    
    init(
        getFeedUseCase: GetFeedUseCase,
        likePostUseCase: LikePostUseCase,
        coordinator: FeedCoordinatorProtocol
    ) {
        self.getFeedUseCase = getFeedUseCase
        self.likePostUseCase = likePostUseCase
        self.coordinator = coordinator
    }
    
    // MARK: - Actions
    func loadFeed() async {
        guard state != .loading else { return }
        state = .loading
        
        let result = await getFeedUseCase.execute(page: 0)
        handleFeedResult(result, isRefresh: true)
    }
    
    func refresh() async {
        state = .refreshing
        currentPage = 0
        
        let result = await getFeedUseCase.execute(page: 0)
        handleFeedResult(result, isRefresh: true)
    }
    
    func loadMore() async {
        guard state == .loaded else { return }
        
        let result = await getFeedUseCase.execute(page: currentPage + 1)
        if case .success = result {
            currentPage += 1
        }
        handleFeedResult(result, isRefresh: false)
    }
    
    func toggleLike(postId: String) async {
        guard let index = rawPosts.firstIndex(where: { $0.id == postId }) else { return }
        let post = rawPosts[index]
        
        // Optimistic update
        let optimisticPost = Post(
            id: post.id,
            authorId: post.authorId,
            authorName: post.authorName,
            content: post.content,
            likeCount: post.isLiked ? post.likeCount - 1 : post.likeCount + 1,
            isLiked: !post.isLiked,
            createdAt: post.createdAt
        )
        rawPosts[index] = optimisticPost
        posts = rawPosts.map(PostViewModel.init)
        
        // Server update
        let result = await likePostUseCase.execute(postId: postId, isCurrentlyLiked: post.isLiked)
        
        switch result {
        case .success(let updatedPost):
            rawPosts[index] = updatedPost
            posts = rawPosts.map(PostViewModel.init)
        case .failure:
            // Revert on failure
            rawPosts[index] = post
            posts = rawPosts.map(PostViewModel.init)
        }
    }
    
    func didSelectPost(at index: Int) {
        guard index < rawPosts.count else { return }
        coordinator.showPostDetail(rawPosts[index])
    }
    
    // MARK: - Helpers
    private func handleFeedResult(_ result: Result<[Post], DomainError>, isRefresh: Bool) {
        switch result {
        case .success(let newPosts):
            if isRefresh {
                rawPosts = newPosts
            } else {
                rawPosts.append(contentsOf: newPosts)
            }
            posts = rawPosts.map(PostViewModel.init)
            state = .loaded
        case .failure(let error):
            state = .error(error.localizedDescription)
        }
    }
}
```

### Presentation Layer - SwiftUI View

```swift
// MARK: - Feed View
struct FeedView: View {
    @StateObject var viewModel: FeedViewModel
    
    var body: some View {
        NavigationStack {
            content
                .navigationTitle("Feed")
                .refreshable { await viewModel.refresh() }
                .task { await viewModel.loadFeed() }
        }
    }
    
    @ViewBuilder
    private var content: some View {
        switch viewModel.state {
        case .idle:
            Color.clear
        case .loading:
            ProgressView()
        case .loaded, .refreshing:
            postList
        case .error(let message):
            ErrorView(message: message) {
                Task { await viewModel.loadFeed() }
            }
        }
    }
    
    private var postList: some View {
        List {
            ForEach(Array(viewModel.posts.enumerated()), id: \.element.id) { index, post in
                PostRow(post: post) {
                    Task { await viewModel.toggleLike(postId: post.id) }
                }
                .onTapGesture {
                    viewModel.didSelectPost(at: index)
                }
                .onAppear {
                    if index == viewModel.posts.count - 3 {
                        Task { await viewModel.loadMore() }
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
                Text(post.authorName).font(.headline)
                Spacer()
                Text(post.relativeTime).font(.caption).foregroundStyle(.secondary)
            }
            
            Text(post.content)
            
            HStack {
                Button(action: onLikeTapped) {
                    Image(systemName: post.isLiked ? "heart.fill" : "heart")
                        .foregroundColor(post.isLiked ? .red : .gray)
                }
                .buttonStyle(.plain)
                
                Text(post.likeCountText).font(.caption)
            }
        }
        .padding(.vertical, 4)
    }
}
```

---

## 9️⃣ Clean Architecture Deep Dive

### Presentation, Domain, Data Layers

| Layer | Knows About | Example Classes |
|-------|-------------|-----------------|
| **Presentation** | Domain | ViewModel, View, Coordinator |
| **Domain** | Nothing | UseCase, Entity, RepositoryProtocol |
| **Data** | Domain | Repository, APIService, Database |

### Dependency Inversion

```swift
// ❌ Wrong: UseCase depends on concrete Repository
class GetFeedUseCase {
    let repository = PostRepository()  // Concrete dependency
}

// ✅ Correct: UseCase depends on protocol
class GetFeedUseCase {
    let repository: PostRepositoryProtocol  // Abstract dependency
    
    init(repository: PostRepositoryProtocol) {
        self.repository = repository
    }
}
```

### Why Clean Architecture Scales Well

1. **Replace database** → Only change Data layer
2. **Replace UI** (UIKit → SwiftUI) → Only change Presentation layer
3. **Test business logic** → Test Domain layer without UI or network
4. **Team boundaries** → Teams own layers, not features

### Why It Can Feel Heavy for Small Teams

| Concern | Small Team Reality |
|---------|-------------------|
| **Many files** | 1-2 devs must maintain all layers |
| **Abstract protocols** | May feel unnecessary for simple apps |
| **Use case explosion** | Every action becomes a class |
| **Folder navigation** | Deep folder structures slow development |

---

## 📋 When to Use Clean Architecture

### ✅ Good Use Cases
- Apps expected to live 5+ years
- Teams of 5+ developers
- High test coverage requirements
- Potential UI framework changes (UIKit → SwiftUI)
- Multiple data sources (API + local DB)

### ❌ Avoid Clean Architecture When
- Building MVPs or prototypes
- Very small team (1-2 developers)
- Simple CRUD apps
- Tight deadlines

---

## 🎯 Interview Tips

### Common Questions

**Q: What's the difference between Clean Architecture and VIPER?**
> "Clean Architecture is a set of principles, VIPER is an implementation. Clean Architecture focuses on layer separation (Presentation/Domain/Data) with dependency inversion. VIPER is per-module with V-I-P-E-R components. Clean Architecture is more flexible; VIPER is more prescriptive."

**Q: What's the role of Use Cases?**
> "Use Cases represent one business action. They orchestrate repository calls, apply business rules, and return domain results. Use Cases are the API of your Domain layer—everything flows through them."

**Q: How do you handle shared state in Clean Architecture?**
> "Shared state typically lives in repositories that act as single sources of truth. ViewModels observe repository state changes. Alternatively, you can use a dedicated state management layer (like Redux) within the Domain layer."

**Q: Would you use Clean Architecture with SwiftUI?**
> "Absolutely. Clean Architecture's ViewModel works perfectly as an @ObservableObject. SwiftUI views observe the ViewModel, and the layers below are unchanged. It's actually easier with SwiftUI because data binding is built-in."

---

## 📚 Next Steps

Ready to compare architectures? Continue to [Architecture Comparison →](./07_Architecture_Comparison.md)
