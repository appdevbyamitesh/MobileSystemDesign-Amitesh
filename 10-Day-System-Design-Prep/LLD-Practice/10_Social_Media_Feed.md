# LLD Problem 10: Social Media Feed

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Performance**: Infinite scroll, prefetching
- **Optimistic UI**: Like/comment feedback
- **Real-time Updates**: Live content
- **Caching Strategy**: Feed persistence

---

# R — Requirements

## Functional Requirements

```markdown
1. Feed Display
   - Infinite scroll
   - Multiple content types (posts, ads, stories)
   - Pull to refresh

2. Interactions
   - Like/unlike
   - Comment
   - Share
   - Save

3. Real-time Updates
   - New posts indicator
   - Like count updates
   - Live comments
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Scroll FPS | 60 | Smooth experience |
| Image load | < 300ms | Perceived speed |
| Prefetch | 5 items ahead | No loading gaps |
| Offline | Cached feed | Subway usage |

---

# E — Entities

```swift
// MARK: - Feed Items

protocol FeedItem: Identifiable {
    var id: String { get }
    var timestamp: Date { get }
}

struct Post: FeedItem, Codable {
    let id: String
    let author: User
    let content: PostContent
    let timestamp: Date
    var likeCount: Int
    var commentCount: Int
    var isLiked: Bool
    var isSaved: Bool
}

enum PostContent: Codable {
    case text(String)
    case image(ImagePost)
    case video(VideoPost)
    case carousel([MediaItem])
}

struct ImagePost: Codable {
    let imageUrl: String
    let aspectRatio: CGFloat
    let caption: String?
}

struct VideoPost: Codable {
    let videoUrl: String
    let thumbnailUrl: String
    let duration: TimeInterval
    let caption: String?
}

struct Ad: FeedItem, Codable {
    let id: String
    let advertiser: String
    let content: AdContent
    let timestamp: Date
    let actionUrl: String
}

struct Story: FeedItem, Codable {
    let id: String
    let author: User
    let items: [StoryItem]
    let timestamp: Date
    var isViewed: Bool
}

// MARK: - Pagination

struct FeedPage: Codable {
    let items: [AnyFeedItem]
    let nextCursor: String?
    let hasMore: Bool
}

// Type-erased FeedItem for mixed content
struct AnyFeedItem: Codable {
    let type: FeedItemType
    let post: Post?
    let ad: Ad?
    let story: Story?
    
    var asItem: any FeedItem {
        switch type {
        case .post: return post!
        case .ad: return ad!
        case .story: return story!
        }
    }
}

enum FeedItemType: String, Codable {
    case post, ad, story
}
```

---

# A — Architecture & Patterns

## Pattern 1: Strategy (Feed Ranking)

```swift
protocol FeedRankingStrategy {
    func rank(_ items: [any FeedItem]) -> [any FeedItem]
}

// Chronological ranking
class ChronologicalRankingStrategy: FeedRankingStrategy {
    func rank(_ items: [any FeedItem]) -> [any FeedItem] {
        items.sorted { $0.timestamp > $1.timestamp }
    }
}

// Engagement-based ranking
class EngagementRankingStrategy: FeedRankingStrategy {
    func rank(_ items: [any FeedItem]) -> [any FeedItem] {
        items.sorted { item1, item2 in
            engagementScore(item1) > engagementScore(item2)
        }
    }
    
    private func engagementScore(_ item: any FeedItem) -> Double {
        guard let post = item as? Post else { return 0 }
        
        let recencyScore = 1.0 / max(1, Date().timeIntervalSince(post.timestamp) / 3600)
        let engagementScore = Double(post.likeCount + post.commentCount * 2)
        
        return recencyScore * 0.4 + engagementScore * 0.6
    }
}

// Friend activity prioritized
class FriendsPriorityRankingStrategy: FeedRankingStrategy {
    private let currentUserId: String
    private let friendIds: Set<String>
    
    func rank(_ items: [any FeedItem]) -> [any FeedItem] {
        items.sorted { item1, item2 in
            let score1 = friendScore(item1)
            let score2 = friendScore(item2)
            
            if score1 != score2 {
                return score1 > score2
            }
            return item1.timestamp > item2.timestamp
        }
    }
    
    private func friendScore(_ item: any FeedItem) -> Int {
        guard let post = item as? Post else { return 0 }
        return friendIds.contains(post.author.id) ? 1 : 0
    }
}
```

---

## Pattern 2: Observer (Real-time Updates)

```swift
import Combine

class FeedUpdatePublisher: ObservableObject {
    static let shared = FeedUpdatePublisher()
    
    private let newPostsSubject = PassthroughSubject<Int, Never>()
    private let likeUpdateSubject = PassthroughSubject<LikeUpdate, Never>()
    
    var newPostsPublisher: AnyPublisher<Int, Never> {
        newPostsSubject.eraseToAnyPublisher()
    }
    
    var likeUpdatePublisher: AnyPublisher<LikeUpdate, Never> {
        likeUpdateSubject.eraseToAnyPublisher()
    }
    
    func publishNewPostsAvailable(count: Int) {
        newPostsSubject.send(count)
    }
    
    func publishLikeUpdate(_ update: LikeUpdate) {
        likeUpdateSubject.send(update)
    }
}

struct LikeUpdate {
    let postId: String
    let newLikeCount: Int
    let isLiked: Bool
}
```

---

## Pattern 3: Facade (Feed API Layer)

```swift
class FeedFacade {
    private let apiClient: APIClient
    private let cacheManager: FeedCacheManager
    private let rankingStrategy: FeedRankingStrategy
    private let prefetcher: ImagePrefetcher
    
    func loadFeed(cursor: String? = nil) async throws -> FeedPage {
        // 1. Try cache for initial load
        if cursor == nil, let cached = try? await cacheManager.getCachedFeed() {
            // Trigger background refresh
            Task { try? await refreshFromNetwork() }
            return cached
        }
        
        // 2. Fetch from network
        let page = try await apiClient.fetchFeed(cursor: cursor)
        
        // 3. Apply ranking
        let rankedItems = rankingStrategy.rank(page.items.map { $0.asItem })
        
        // 4. Cache if first page
        if cursor == nil {
            try? await cacheManager.cache(page)
        }
        
        // 5. Prefetch images
        prefetchImages(from: page.items)
        
        return page
    }
    
    private func prefetchImages(from items: [AnyFeedItem]) {
        let imageUrls = items.compactMap { item -> URL? in
            guard let post = item.post else { return nil }
            if case .image(let imagePost) = post.content {
                return URL(string: imagePost.imageUrl)
            }
            return nil
        }
        
        prefetcher.prefetch(urls: imageUrls)
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Feed Load → Pagination → Update

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  View  │ │ ViewModel  │ │FeedFacade  │ │   Cache    │ │  Server    │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │
    │ onAppear   │              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ loadFeed()   │              │              │
    │            │─────────────▶│              │              │
    │            │              │              │              │
    │            │              │ getCached()  │              │
    │            │              │─────────────▶│              │
    │            │              │              │              │
    │            │              │◀─────────────│ cached feed  │
    │            │              │              │              │
    │            │◀─────────────│ (show cached)│              │
    │            │              │              │              │
    │ show feed  │              │              │              │
    │◀───────────│              │              │              │
    │            │              │              │              │
    │            │              │ (background) │              │
    │            │              │ fetchFeed()  │              │
    │            │              │────────────────────────────▶│
    │            │              │              │              │
    │            │              │◀────────────────────────────│
    │            │              │              │              │
    │            │◀─────────────│ updated feed │              │
    │            │              │              │              │
    │ update UI  │              │              │              │
    │◀───────────│              │              │              │
    │            │              │              │              │
─────────────────────────────────────────────────────────────────
    │            │              │              │              │
    │ scroll near end           │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ loadMore(cursor)            │              │
    │            │─────────────▶│              │              │
    │            │              │              │              │
    │            │              │ fetchFeed(cursor)           │
    │            │              │────────────────────────────▶│
    │            │              │              │              │
    │ append items              │              │              │
    │◀───────────│◀─────────────│◀────────────────────────────│
    │            │              │              │              │
```

---

# H — Handling Concurrency

## Optimistic UI for Likes

```swift
@MainActor
class FeedViewModel: ObservableObject {
    @Published private(set) var posts: [Post] = []
    @Published private(set) var isLoading = false
    @Published private(set) var newPostsAvailable = 0
    
    private var likeOperations: [String: Task<Void, Never>] = [:]
    
    func toggleLike(for postId: String) {
        // Cancel any pending operation
        likeOperations[postId]?.cancel()
        
        // Optimistic update
        guard let index = posts.firstIndex(where: { $0.id == postId }) else { return }
        
        let wasLiked = posts[index].isLiked
        posts[index].isLiked = !wasLiked
        posts[index].likeCount += wasLiked ? -1 : 1
        
        // Haptic feedback
        UIImpactFeedbackGenerator(style: .light).impactOccurred()
        
        // Sync with server
        likeOperations[postId] = Task {
            do {
                if wasLiked {
                    try await apiClient.unlike(postId: postId)
                } else {
                    try await apiClient.like(postId: postId)
                }
            } catch {
                // Rollback on failure
                await MainActor.run {
                    if let index = posts.firstIndex(where: { $0.id == postId }) {
                        posts[index].isLiked = wasLiked
                        posts[index].likeCount += wasLiked ? 1 : -1
                    }
                }
            }
            
            likeOperations.removeValue(forKey: postId)
        }
    }
}
```

## Prefetching for Smooth Scroll

```swift
class FeedViewController: UIViewController, UICollectionViewDataSourcePrefetching {
    let viewModel: FeedViewModel
    let imagePrefetcher = ImagePrefetcher()
    
    func collectionView(
        _ collectionView: UICollectionView,
        prefetchItemsAt indexPaths: [IndexPath]
    ) {
        let posts = indexPaths.compactMap { viewModel.posts[safe: $0.item] }
        let imageUrls = posts.compactMap { post -> URL? in
            if case .image(let imagePost) = post.content {
                return URL(string: imagePost.imageUrl)
            }
            return nil
        }
        
        imagePrefetcher.prefetch(urls: imageUrls)
        
        // Trigger pagination if near end
        if let maxIndex = indexPaths.map({ $0.item }).max(),
           maxIndex >= viewModel.posts.count - 5 {
            Task { await viewModel.loadMore() }
        }
    }
    
    func collectionView(
        _ collectionView: UICollectionView,
        cancelPrefetchingForItemsAt indexPaths: [IndexPath]
    ) {
        let posts = indexPaths.compactMap { viewModel.posts[safe: $0.item] }
        let imageUrls = posts.compactMap { post -> URL? in
            if case .image(let imagePost) = post.content {
                return URL(string: imagePost.imageUrl)
            }
            return nil
        }
        
        imagePrefetcher.cancelPrefetch(urls: imageUrls)
    }
}
```

---

# E — Edge Cases

## Edge Case 1: Rapid Like/Unlike

```swift
// Already handled with task cancellation
func toggleLike(for postId: String) {
    likeOperations[postId]?.cancel()  // Cancel previous
    // ... rest of implementation
}
```

## Edge Case 2: New Posts While Scrolling

```swift
class FeedViewModel {
    @Published private(set) var newPostsAvailable = 0
    
    func handleNewPostsNotification(count: Int) {
        // Don't disrupt scroll, just show indicator
        newPostsAvailable = count
    }
    
    func loadNewPosts() async {
        // Insert at top
        let newPosts = try? await apiClient.fetchNewPosts()
        posts.insert(contentsOf: newPosts ?? [], at: 0)
        newPostsAvailable = 0
    }
}
```

## Edge Case 3: Image Load Failure

```swift
struct FeedImageView: View {
    let imageUrl: URL
    @State private var loadFailed = false
    
    var body: some View {
        AsyncImage(url: imageUrl) { phase in
            switch phase {
            case .empty:
                ShimmerPlaceholder()
            case .success(let image):
                image.resizable()
            case .failure:
                FailedImageView()
                    .onTapGesture {
                        // Retry logic
                        loadFailed.toggle()
                    }
            @unknown default:
                EmptyView()
            }
        }
    }
}
```

---

# D — Design Trade-offs

| Pull to Refresh | Auto Refresh |
|-----------------|--------------|
| User controlled | May interrupt |
| Less data usage | Always current |
| Clear mental model | Can be jarring |

**Decision:** Pull to refresh + "New posts" banner

---

# iOS Implementation

```swift
struct FeedView: View {
    @StateObject var viewModel: FeedViewModel
    
    var body: some View {
        ZStack(alignment: .top) {
            ScrollView {
                LazyVStack(spacing: 0) {
                    ForEach(viewModel.posts) { post in
                        PostView(post: post) {
                            viewModel.toggleLike(for: post.id)
                        }
                        .onAppear {
                            if post.id == viewModel.posts.last?.id {
                                Task { await viewModel.loadMore() }
                            }
                        }
                    }
                    
                    if viewModel.isLoading {
                        ProgressView()
                            .padding()
                    }
                }
            }
            .refreshable {
                await viewModel.refresh()
            }
            
            // New posts banner
            if viewModel.newPostsAvailable > 0 {
                NewPostsBanner(count: viewModel.newPostsAvailable) {
                    Task { await viewModel.loadNewPosts() }
                }
                .transition(.move(edge: .top))
            }
        }
    }
}

struct PostView: View {
    let post: Post
    let onLike: () -> Void
    
    var body: some View {
        VStack(alignment: .leading, spacing: 8) {
            // Author header
            AuthorRow(author: post.author)
            
            // Content
            switch post.content {
            case .text(let text):
                Text(text)
            case .image(let imagePost):
                CachedAsyncImage(url: URL(string: imagePost.imageUrl)!)
                    .aspectRatio(imagePost.aspectRatio, contentMode: .fill)
            case .video(let videoPost):
                VideoPlayerView(url: URL(string: videoPost.videoUrl)!)
            case .carousel(let items):
                CarouselView(items: items)
            }
            
            // Actions
            HStack {
                Button(action: onLike) {
                    Image(systemName: post.isLiked ? "heart.fill" : "heart")
                        .foregroundColor(post.isLiked ? .red : .primary)
                }
                
                Text("\(post.likeCount)")
                
                Spacer()
            }
        }
        .padding()
    }
}
```

---

# Interview Tips

## What to Say

```markdown
1. "Strategy pattern for feed ranking algorithms..."
2. "Optimistic UI for like interactions with rollback..."
3. "Prefetching for smooth infinite scroll..."
4. "Observer pattern for real-time updates..."
5. "Facade to hide API complexity..."
```

## Red Flags to Avoid

```markdown
❌ "I'll reload the whole feed on like"
   → Terrible UX

❌ No mention of prefetching
   → Shows lack of iOS perf knowledge

❌ "Just use a simple list"
   → Missing infinite scroll complexity

❌ No caching strategy
   → Ignores offline/performance needs
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Social Media Feed problem!*
