# GraphQL for iOS: Complete Mobile System Design Guide

## 📚 Table of Contents

1. [Beginner: GraphQL Fundamentals](#beginner-graphql-fundamentals)
2. [Intermediate: Production GraphQL iOS](#intermediate-production-graphql-ios)
3. [Advanced: Enterprise GraphQL](#advanced-enterprise-graphql)
4. [Interview Questions](#interview-questions)

---

## 🎈 BEGINNER: GraphQL Fundamentals

### What is GraphQL? (Explained Like You're 5)

**Restaurant Analogy:**

Imagine you're at two different restaurants:

**REST Restaurant (Traditional):**
- You: "I want a burger"
- Waiter brings: Burger + Fries + Drink + Napkins + Ketchup (even though you only wanted the burger)
- You: "Now I want dessert"
- You have to call the waiter again for a second trip

**GraphQL Restaurant (Smart):**
- You: "I want exactly: 1 burger (no pickles), 1 small fries, apple pie"
- Waiter brings: Exactly what you asked for, nothing more, nothing less, all in ONE trip

**In iOS terms:**

```swift
// REST - Multiple network calls, lots of extra data

// Call 1: Get user
GET /users/123
Response: {
    id: 123,
    name: "John",
    email: "john@example.com",
    address: {...},           // Don't need this
    payment_methods: {...},   // Don't need this
    preferences: {...},       // Don't need this
    created_at: "...",        // Don't need this
}

// Call 2: Get user's posts
GET /users/123/posts
Response: [...posts...]

// Call 3: Get comments on first post
GET /posts/456/comments
Response: [...comments...]

// Result: 3 network calls, lots of wasted data


// GraphQL - One call, exact data you need

query {
    user(id: 123) {
        name              // Only what we need
        posts {           // Nested in one call!
            title
            comments {    // Even deeper nesting!
                text
                author { name }
            }
        }
    }
}

// Result: 1 network call, zero waste! 🎯
```

### Why Mobile Apps Love GraphQL

**Problem 1: Over-fetching (Getting too much data)**

```swift
// REST endpoint returns EVERYTHING
struct User: Codable {
    let id: String
    let name: String
    let email: String
    let address: Address           // ❌ Don't need for profile screen
    let paymentMethods: [Payment]  // ❌ Don't need for profile screen
    let orderHistory: [Order]      // ❌ Don't need for profile screen
    let preferences: Preferences   // ❌ Don't need for profile screen
    // ... 20 more fields
}

// Your profile screen only needs name and email!
// But you download 500KB when you only need 2KB
// = Wasted bandwidth 💸
// = Slower app 🐌
// = Battery drain 🔋
```

**GraphQL Solution:**

```swift
query ProfileScreen {
    user(id: "123") {
        name   // Only these 2 fields
        email  // That's it!
    }
}

// Downloads: 0.5KB instead of 500KB
// = 1000x less data! 🚀
```

**Problem 2: Under-fetching (Not getting enough data)**

```swift
// Instagram feed screen needs:
// 1. Post image
// 2. Author name
// 3. Author avatar
// 4. Like count
// 5. Recent comments

// REST requires multiple calls:
GET /posts/123           // Get post
GET /users/456           // Get author info
GET /posts/123/likes     // Get likes
GET /posts/123/comments  // Get comments

// 4 network calls = slow! 🐌


// GraphQL - One call:
query FeedPost {
    post(id: "123") {
        image
        author {
            name
            avatar
        }
        likeCount
        comments(limit: 3) {
            text
            author { name }
        }
    }
}

// 1 network call = fast! ⚡
```

### GraphQL Core Concepts

**1. Query (Reading data - like GET in REST)**

```swift
// Think: "Ask for data"

query {
    restaurant(id: "123") {
        name
        rating
        menu {
            itemName
            price
        }
    }
}

// iOS Usage: Loading a restaurant detail screen
struct RestaurantDetailView: View {
    func loadRestaurant() async {
        let query = """
        query {
            restaurant(id: "\(restaurantId)") {
                name
                rating
                cuisine
            }
        }
        """
        
        let data = try await graphQLClient.query(query)
        // Update UI with exact data needed
    }
}
```

**2. Mutation (Writing data - like POST/PUT/DELETE in REST)**

```swift
// Think: "Change something"

mutation {
    placeOrder(
        restaurantId: "123",
        items: ["burger", "fries"]
    ) {
        orderId
        status
        estimatedTime
    }
}

// iOS Usage: User places food order
func placeOrder() async {
    let mutation = """
    mutation {
        placeOrder(restaurantId: "123", items: ["burger"]) {
            orderId
            status
        }
    }
    """
    
    let result = try await graphQLClient.mutate(mutation)
    // Show order confirmation
}
```

**3. Schema (Contract between app and server)**

```swift
// Like a menu at a restaurant - tells you what's available

type Restaurant {
    id: ID!              // ! means required
    name: String!
    rating: Float
    menu: [MenuItem]     // [] means array
}

type MenuItem {
    name: String!
    price: Float!
    isAvailable: Boolean
}

// In iOS, you create Swift models matching the schema
struct Restaurant: Codable {
    let id: String
    let name: String
    let rating: Double?
    let menu: [MenuItem]?
}
```

### Real iOS Screen → GraphQL Query Examples

**Example 1: Uber Home Screen**

```swift
// Screen needs:
// - User name
// - User rating
// - Recent ride history
// - Saved addresses

query UberHomeScreen {
    currentUser {
        name
        rating
        recentRides(limit: 3) {
            destination
            date
            fare
        }
        savedAddresses {
            label    // "Home", "Work"
            address
        }
    }
}

// iOS ViewModel
class HomeViewModel: ObservableObject {
    @Published var userName: String = ""
    @Published var userRating: Double = 0.0
    @Published var recentRides: [Ride] = []
    
    func loadHomeScreen() async {
        let query = """
        query {
            currentUser {
                name
                rating
                recentRides(limit: 3) {
                    destination
                    date
                    fare
                }
            }
        }
        """
        
        let response = try await graphQL.query(query)
        
        await MainActor.run {
            self.userName = response.currentUser.name
            self.userRating = response.currentUser.rating
            self.recentRides = response.currentUser.recentRides
        }
    }
}
```

**Example 2: Instagram Post Detail**

```swift
query PostDetail($postId: ID!) {
    post(id: $postId) {
        imageUrl
        caption
        likeCount
        isLikedByMe
        createdAt
        author {
            username
            profilePicture
            isFollowing
        }
        comments(first: 20) {
            text
            createdAt
            author {
                username
                profilePicture
            }
        }
    }
}

// iOS Model
struct Post: Codable {
    let imageUrl: String
    let caption: String
    let likeCount: Int
    let isLikedByMe: Bool
    let author: Author
    let comments: [Comment]
}

struct Author: Codable {
    let username: String
    let profilePicture: String
    let isFollowing: Bool
}
```

**Example 3: E-commerce Product Page**

```swift
query ProductPage($productId: ID!) {
    product(id: $productId) {
        name
        price
        images
        description
        inStock
        rating
        reviews(first: 5) {
            rating
            comment
            reviewer { name }
        }
        relatedProducts {
            id
            name
            price
            thumbnail
        }
    }
}
```

### GraphQL vs REST: Side-by-Side

```swift
// SCENARIO: Load user profile with posts

// ═══════════════════════════════════════════
// REST APPROACH
// ═══════════════════════════════════════════

// Request 1
GET /api/users/123
// Downloads: 50KB (includes address, payment info, etc. we don't need)

// Request 2  
GET /api/users/123/posts?limit=10
// Downloads: 200KB

// Total: 2 requests, 250KB, ~800ms


// ═══════════════════════════════════════════
// GRAPHQL APPROACH
// ═══════════════════════════════════════════

query {
    user(id: "123") {
        name
        avatar
        bio
        posts(limit: 10) {
            title
            image
            likeCount
        }
    }
}

// Total: 1 request, 30KB, ~200ms ⚡

// Benefits:
// ✅ 87% less data
// ✅ 75% faster
// ✅ 50% less battery usage
// ✅ Works better on slow networks
```

### Simple GraphQL Client for iOS

```swift
import Foundation

// Basic GraphQL client
class SimpleGraphQLClient {
    let endpoint: URL
    
    init(endpoint: String) {
        self.endpoint = URL(string: endpoint)!
    }
    
    func query<T: Decodable>(_ query: String) async throws -> T {
        // Build request
        var request = URLRequest(url: endpoint)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        // GraphQL always sends query in body
        let body: [String: Any] = ["query": query]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)
        
        // Make request
        let (data, _) = try await URLSession.shared.data(for: request)
        
        // Parse response
        struct GraphQLResponse<T: Decodable>: Decodable {
            let data: T?
            let errors: [GraphQLError]?
        }
        
        let response = try JSONDecoder().decode(GraphQLResponse<T>.self, from: data)
        
        if let errors = response.errors {
            throw GraphQLError(message: errors.first?.message ?? "Unknown error")
        }
        
        guard let data = response.data else {
            throw GraphQLError(message: "No data in response")
        }
        
        return data
    }
}

struct GraphQLError: Error, Decodable {
    let message: String
}

// Usage
let client = SimpleGraphQLClient(endpoint: "https://api.example.com/graphql")

struct UserResponse: Decodable {
    let user: User
}

struct User: Decodable {
    let name: String
    let email: String
}

// Make query
let query = """
query {
    user(id: "123") {
        name
        email
    }
}
"""

let response: UserResponse = try await client.query(query)
print("User: \(response.user.name)")
```

---

## 🚀 INTERMEDIATE: Production GraphQL iOS

### Designing GraphQL Queries for iOS ViewModels

**Best Practice: One Query Per Screen**

```swift
// ❌ BAD: Multiple small queries
class FeedViewModel {
    func loadFeed() async {
        let posts = await graphQL.query("query { posts { id } }")
        
        for post in posts {
            // N+1 problem! Making 100 queries for 100 posts
            let details = await graphQL.query("query { post(id: \(post.id)) {...} }")
        }
    }
}

// ✅ GOOD: One query for everything
class FeedViewModel {
    func loadFeed() async {
        let query = """
        query FeedScreen {
            feed(limit: 20) {
                id
                image
                caption
                author {
                    username
                    avatar
                }
                likeCount
                commentCount
            }
        }
        """
        
        let response = await graphQL.query(query)
        // All data in one shot!
    }
}
```

**Using Fragments for Reusable Fields**

```swift
// Define reusable fragments
let UserFragment = """
fragment UserInfo on User {
    id
    name
    avatar
    isVerified
}
"""

let PostFragment = """
fragment PostInfo on Post {
    id
    image
    caption
    likeCount
    author {
        ...UserInfo
    }
}
"""

// Use in queries
let query = """
\(UserFragment)
\(PostFragment)

query Feed {
    feed {
        ...PostInfo
    }
}
"""
```

### Pagination Strategies for Mobile

**1. Offset-based Pagination (Simple but has issues)**

```swift
// Traditional approach
query Feed($offset: Int!, $limit: Int!) {
    posts(offset: $offset, limit: $limit) {
        id
        title
    }
}

// iOS Implementation
class FeedViewModel: ObservableObject {
    @Published var posts: [Post] = []
    private var offset = 0
    private let limit = 20
    
    func loadMore() async {
        let query = """
        query {
            posts(offset: \(offset), limit: \(limit)) {
                id
                title
                image
            }
        }
        """
        
        let newPosts = try await graphQL.query(query)
        
        await MainActor.run {
            self.posts.append(contentsOf: newPosts)
            self.offset += self.limit
        }
    }
}

// ❌ Problems:
// - If new posts are added, you might see duplicates
// - If posts are deleted, you might skip some
// - Not efficient for large offsets
```

**2. Cursor-based Pagination (Recommended for mobile)**

```swift
// Better approach - uses cursors
query Feed($after: String, $limit: Int!) {
    feed(after: $after, first: $limit) {
        edges {
            cursor    // Unique position marker
            node {
                id
                title
                image
            }
        }
        pageInfo {
            hasNextPage
            endCursor
        }
    }
}

// iOS Implementation
class FeedViewModel: ObservableObject {
    @Published var posts: [Post] = []
    private var endCursor: String?
    private var hasMore = true
    
    func loadMore() async {
        guard hasMore else { return }
        
        let query = """
        query {
            feed(after: "\(endCursor ?? "")", first: 20) {
                edges {
                    cursor
                    node {
                        id
                        title
                        image
                    }
                }
                pageInfo {
                    hasNextPage
                    endCursor
                }
            }
        }
        """
        
        let response = try await graphQL.query(query)
        
        await MainActor.run {
            let newPosts = response.feed.edges.map { $0.node }
            self.posts.append(contentsOf: newPosts)
            self.endCursor = response.feed.pageInfo.endCursor
            self.hasMore = response.feed.pageInfo.hasNextPage
        }
    }
}

// ✅ Benefits:
// - No duplicates even if data changes
// - Efficient at any position
// - Works with real-time updates
```

### Error Handling in GraphQL

**GraphQL can return partial data + errors!**

```swift
// Example response
{
    "data": {
        "user": {
            "name": "John",
            "posts": null    // This failed!
        }
    },
    "errors": [
        {
            "message": "Not authorized to view posts",
            "path": ["user", "posts"]
        }
    ]
}

// iOS Error Handling
class GraphQLClient {
    func query<T: Decodable>(_ query: String) async throws -> T {
        let response = try await executeQuery(query)
        
        // Check for errors
        if let errors = response.errors {
            // Log errors
            print("GraphQL Errors:")
            for error in errors {
                print("  - \(error.message)")
                if let path = error.path {
                    print("    Path: \(path.joined(separator: " → "))")
                }
            }
            
            // Decide: throw or return partial data?
            if response.data == nil {
                // No data at all - throw
                throw GraphQLError.requestFailed(errors)
            } else {
                // Partial data - log but continue
                // UI can show what's available
            }
        }
        
        guard let data = response.data else {
            throw GraphQLError.noData
        }
        
        return data
    }
}

// Error display in UI
struct ProfileView: View {
    @StateObject var viewModel = ProfileViewModel()
    
    var body: some View {
        VStack {
            Text(viewModel.user?.name ?? "Unknown")
            
            if viewModel.postsError != nil {
                // Show error for posts but keep showing name
                Text("Failed to load posts")
                    .foregroundColor(.red)
            } else {
                ForEach(viewModel.posts) { post in
                    PostRow(post: post)
                }
            }
        }
    }
}
```

### Caching GraphQL Responses

**Memory Cache for GraphQL**

```swift
actor GraphQLCache {
    // Cache complete responses by query
    private var cache: [String: CacheEntry] = [:]
    
    struct CacheEntry {
        let data: Data
        let timestamp: Date
        let ttl: TimeInterval
        
        var isExpired: Bool {
            Date().timeIntervalSince(timestamp) > ttl
        }
    }
    
    func get(query: String) -> Data? {
        guard let entry = cache[query], !entry.isExpired else {
            cache.removeValue(forKey: query)
            return nil
        }
        
        return entry.data
    }
    
    func set(query: String, data: Data, ttl: TimeInterval = 300) {
        cache[query] = CacheEntry(
            data: data,
            timestamp: Date(),
            ttl: ttl
        )
    }
}

// Usage in client
class GraphQLClient {
    private let cache = GraphQLCache()
    
    func query<T: Decodable>(_ query: String, cachePolicy: CachePolicy = .cacheFirst) async throws -> T {
        // Check cache first
        if cachePolicy == .cacheFirst || cachePolicy == .cacheOnly {
            if let cachedData = await cache.get(query: query) {
                print("🎯 Cache hit")
                return try JSONDecoder().decode(T.self, from: cachedData)
            }
            
            if cachePolicy == .cacheOnly {
                throw GraphQLError.cacheNotFound
            }
        }
        
        // Network request
        if cachePolicy != .cacheOnly {
            let data = try await executeQuery(query)
            
            // Cache the result
            await cache.set(query: query, data: data)
            
            return try JSONDecoder().decode(T.self, from: data)
        }
        
        throw GraphQLError.cacheNotFound
    }
}

enum CachePolicy {
    case networkOnly    // Skip cache, always fetch
    case cacheOnly      // Only use cache, never fetch
    case cacheFirst     // Try cache, fallback to network
    case networkFirst   // Try network, fallback to cache
}
```

### Network Performance Optimization

**1. Query Batching**

```swift
// Instead of 3 separate requests:
// query { user(id: "1") {...} }
// query { user(id: "2") {...} }
// query { user(id: "3") {...} }

// Batch into one:
query {
    user1: user(id: "1") { name avatar }
    user2: user(id: "2") { name avatar }
    user3: user(id: "3") { name avatar }
}

// iOS Implementation
class BatchedGraphQLClient {
    private var pendingQueries: [String] = []
    private var batchTimer: Timer?
    
    func query(_ query: String) async throws -> Data {
        // Add to batch
        pendingQueries.append(query)
        
        // Debounce - wait 50ms for more queries
        batchTimer?.invalidate()
        batchTimer = Timer.scheduledTimer(withTimeInterval: 0.05, repeats: false) { _ in
            Task {
                await self.executeBatch()
            }
        }
        
        // Wait for batch execution
        // ...
    }
    
    private func executeBatch() async {
        // Combine all queries into one request
        let combinedQuery = pendingQueries.joined(separator: "\n")
        // Execute...
    }
}
```

**2. Compression**

```swift
class GraphQLClient {
    func query(_ query: String) async throws -> Data {
        var request = URLRequest(url: endpoint)
        request.httpMethod = "POST"
        
        // Enable gzip compression
        request.setValue("gzip", forHTTPHeaderField: "Accept-Encoding")
        
        let body = ["query": query]
        request.httpBody = try JSONSerialization.data(withJSONObject: body)
        
        // Response will be compressed, saving bandwidth
        let (data, _) = try await URLSession.shared.data(for: request)
        
        return data
    }
}

// Savings: 70-80% smaller responses!
```

**3. Persisted Queries**

```swift
// Instead of sending full query text:
query VeryLongQueryText {
    user { ... }
    posts { ... }
    // 200 lines of query
}

// Send only hash:
{
    "extensions": {
        "persistedQuery": {
            "version": 1,
            "sha256Hash": "abc123..."
        }
    }
}

// Server recognizes hash and executes stored query
// Reduces request size by 90%!
```

### When GraphQL is a BAD Idea

```swift
/*
❌ DON'T use GraphQL for:

1. SIMPLE CRUD APPS
   - Basic form → save → list
   - RESTful patterns work fine
   - GraphQL adds unnecessary complexity

2. FILE UPLOADS
   - GraphQL isn't designed for binary data
   - Use REST multipart/form-data instead
   
3. STREAMING DATA
   - Large video/audio streams
   - Use HTTP streaming or WebSockets directly
   
4. REAL-TIME GAMING
   - Need ultra-low latency
   - Use WebSockets/WebRTC instead
   
5. SIMPLE PUBLIC APIs
   - Blog posts, weather data
   - REST is simpler for consumers

✅ DO use GraphQL for:

1. COMPLEX MOBILE UIs
   - Many interconnected screens
   - Lots of nested data (social feeds, e-commerce)
   
2. MULTIPLE CLIENTS
   - iOS, Android, Web all need different data
   - Each can query exactly what they need
   
3. RAPID ITERATION
   - UI changes frequently
   - No need to update backend for new fields
   
4. DATA-HEAVY APPS
   - Social media, analytics dashboards
   - Complex filtering and relationships
*/
```

---

## 💎 ADVANCED: Enterprise GraphQL

### Normalized Caching (Apollo-style)

**Problem: Duplicate data in cache**

```swift
// Without normalization:
cache["query1"] = {
    user(id: "123") { name: "John", email: "john@example.com" }
}

cache["query2"] = {
    user(id: "123") { name: "John", age: 30 }
}

// User "123" is stored TWICE!
// If name changes, cache is inconsistent
```

**Solution: Normalize by ID**

```swift
// Normalized cache stores objects by ID
class NormalizedCache {
    // Object store: ID → Object
    private var objects: [String: [String: Any]] = [:]
    
    // Query results: Query → [IDs]
    private var queryResults: [String: [String]] = [:]
    
    func store(response: GraphQLResponse, forQuery query: String) {
        // Extract all objects with IDs
        let objects = extractObjects(from: response.data)
        
        // Store each object by ID
        for object in objects {
            if let id = object["id"] as? String,
               let typename = object["__typename"] as? String {
                let cacheKey = "\(typename):\(id)"
                
                // Merge with existing data
                if var existing = self.objects[cacheKey] {
                    existing.merge(object) { _, new in new }
                    self.objects[cacheKey] = existing
                } else {
                    self.objects[cacheKey] = object
                }
            }
        }
        
        // Store query result references
        queryResults[query] = objects.compactMap { $0["id"] as? String }
    }
    
    func get(query: String) -> Any? {
        guard let ids = queryResults[query] else { return nil }
        
        // Reconstruct response from normalized objects
        return ids.compactMap { objects["User:\($0)"] }
    }
    
    // Update single object (updates all queries that reference it!)
    func updateObject(id: String, typename: String, fields: [String: Any]) {
        let cacheKey = "\(typename):\(id)"
        
        if var object = objects[cacheKey] {
            object.merge(fields) { _, new in new }
            objects[cacheKey] = object
            
            print("✅ Updated \(cacheKey) - all queries automatically updated!")
        }
    }
}

// Example usage
let cache = NormalizedCache()

// Query 1
let response1 = """
{
    "data": {
        "__typename": "User",
        "id": "123",
        "name": "John"
    }
}
"""
cache.store(response: response1, forQuery: "query1")

// Query 2  
let response2 = """
{
    "data": {
        "__typename": "User",
        "id": "123",
        "age": 30
    }
}
"""
cache.store(response: response2, forQuery: "query2")

// Now both queries have complete data!
// Update name once:
cache.updateObject(id: "123", typename: "User", fields: ["name": "John Updated"])

// Both query1 and query2 automatically have updated name! 🎯
```

### Real-time Updates: Subscriptions vs Polling

**1. Polling (Simple but inefficient)**

```swift
class ChatViewModel: ObservableObject {
    @Published var messages: [Message] = []
    private var pollingTimer: Timer?
    
    func startPolling() {
        // Poll every 3 seconds
        pollingTimer = Timer.scheduledTimer(withTimeInterval: 3.0, repeats: true) { _ in
            Task {
                await self.fetchNewMessages()
            }
        }
    }
    
    func fetchNewMessages() async {
        let query = """
        query {
            messages(after: "\(lastMessageId)") {
                id
                text
                sender { name }
            }
        }
        """
        
        let newMessages = try await graphQL.query(query)
        
        await MainActor.run {
            self.messages.append(contentsOf: newMessages)
        }
    }
    
    // ❌ Problems:
    // - Wastes battery (polling even when no updates)
    // - Delays (up to 3 sec before seeing new message)
    // - Wastes bandwidth (empty responses)
}
```

**2. GraphQL Subscriptions (Efficient real-time)**

```swift
// Server pushes updates via WebSocket
subscription {
    newMessage(chatId: "123") {
        id
        text
        sender {
            name
            avatar
        }
        timestamp
    }
}

// iOS Implementation
class ChatViewModel: ObservableObject {
    @Published var messages: [Message] = []
    private var subscription: Task<Void, Never>?
    
    func subscribeToMessages() {
        subscription = Task {
            let subscription = """
            subscription {
                newMessage(chatId: "123") {
                    id
                    text
                    sender { name avatar }
                }
            }
            """
            
            // WebSocket stream
            for await message in graphQL.subscribe(subscription) {
                await MainActor.run {
                    self.messages.append(message)
                }
            }
        }
    }
    
    func unsubscribe() {
        subscription?.cancel()
    }
    
    // ✅ Benefits:
    // - Instant updates (no delay)
    // - Efficient (only sends when there's data)
    // - Better battery life
}

// Subscription client implementation
class GraphQLSubscriptionClient {
    private var webSocket: URLSessionWebSocketTask?
    
    func subscribe<T: Decodable>(_ subscription: String) -> AsyncStream<T> {
        AsyncStream { continuation in
            // Connect WebSocket
            let url = URL(string: "wss://api.example.com/graphql")!
            webSocket = URLSession.shared.webSocketTask(with: url)
            webSocket?.resume()
            
            // Send subscription
            let message = [
                "type": "start",
                "payload": ["query": subscription]
            ]
            
            let data = try! JSONEncoder().encode(message)
            webSocket?.send(.data(data)) { error in
                if let error = error {
                    continuation.finish()
                }
            }
            
            // Listen for messages
            Task {
                while true {
                    guard let message = try? await webSocket?.receive() else {
                        break
                    }
                    
                    switch message {
                    case .data(let data):
                        if let decoded = try? JSONDecoder().decode(T.self, from: data) {
                            continuation.yield(decoded)
                        }
                    default:
                        break
                    }
                }
                
                continuation.finish()
            }
        }
    }
}
```

**3. Hybrid Approach (Best for mobile)**

```swift
class SmartChatViewModel: ObservableObject {
    @Published var messages: [Message] = []
    
    func connectToChat() async {
        // Initial load from cache/network
        await loadCachedMessages()
        
        // Then subscribe to real-time updates
        subscribeToNewMessages()
    }
    
    private func loadCachedMessages() async {
        // Load from cache first (instant)
        let cachedMessages = await cache.getMessages(chatId: "123")
        
        await MainActor.run {
            self.messages = cachedMessages
        }
        
        // Fetch fresh data in background
        Task {
            let fresh = try await fetchMessagesFromServer()
            await MainActor.run {
                self.messages = fresh
            }
        }
    }
    
    private func subscribeToNewMessages() {
        // Subscribe to new messages via WebSocket
        // Append in real-time
    }
}
```

### Offline-First GraphQL

```swift
actor OfflineGraphQLClient {
    private let cache: NormalizedCache
    private let syncQueue: SyncQueue
    private let network: NetworkMonitor
    
    // Execute query with offline support
    func query<T: Decodable>(_ query: String) async throws -> T {
        // 1. Always return cached data first
        if let cached = await cache.get(query: query) {
            print("🎯 Returning cached data")
            
            // 2. Refresh in background if online
            if await network.isConnected {
                Task {
                    try? await self.refreshQuery(query)
                }
            }
            
            return cached
        }
        
        // 3. No cache - must fetch (need network)
        guard await network.isConnected else {
            throw GraphQLError.offline
        }
        
        return try await fetchAndCache(query)
    }
    
    // Execute mutation with queue
    func mutate(_ mutation: String) async throws {
        // 1. Apply optimistic update to cache
        let optimisticId = UUID().uuidString
        await cache.applyOptimisticUpdate(id: optimisticId, mutation: mutation)
        
        // 2. Queue mutation for sync
        await syncQueue.enqueue(mutation: mutation, optimisticId: optimisticId)
        
        // 3. Sync immediately if online
        if await network.isConnected {
            await syncNow()
        } else {
            print("📴 Offline - queued for later sync")
        }
    }
    
    private func syncNow() async {
        let pending = await syncQueue.getPending()
        
        for mutation in pending {
            do {
                // Execute mutation
                try await executeMutation(mutation)
                
                // Remove from queue
                await syncQueue.markCompleted(mutation.id)
                
                // Remove optimistic update, use real data
                await cache.removeOptimisticUpdate(id: mutation.optimisticId)
            } catch {
                // Keep in queue for retry
                print("❌ Sync failed: \(error)")
            }
        }
    }
}

// Usage in UI
struct PostListView: View {
    @StateObject var viewModel = PostViewModel()
    
    var body: some View {
        List(viewModel.posts) { post in
            PostRow(post: post)
                .opacity(post.isOptimistic ? 0.5 : 1.0)  // Show pending posts
        }
        .refreshable {
            await viewModel.refresh()
        }
    }
}

class PostViewModel: ObservableObject {
    @Published var posts: [Post] = []
    private let client = OfflineGraphQLClient()
    
    func createPost(text: String) async {
        let mutation = """
        mutation {
            createPost(text: "\(text)") {
                id
                text
                createdAt
            }
        }
        """
        
        // Works offline! Will sync when online
        try await client.mutate(mutation)
    }
}
```

### Security & Authorization

```swift
// PROBLEM: Over-exposing data in GraphQL

// ❌ BAD: Query exposes sensitive data
query {
    user(id: "123") {
        name
        email
        socialSecurityNumber  // 😱 Should never be queryable!
        password              // 😱😱😱
    }
}

// ✅ GOOD: Field-level authorization
type User {
    id: ID!
    name: String!
    email: String!
    
    # Only user themselves can query
    privateData: PrivateData @auth(requires: OWNER)
}

type PrivateData {
    savedCards: [PaymentCard]
    address: Address
}


// iOS: Handle authorization errors
class SecureGraphQLClient {
    func query<T: Decodable>(_ query: String) async throws -> T {
        do {
            return try await executeQuery(query)
        } catch let error as GraphQLError {
            if error.code == "UNAUTHENTICATED" {
                // Token expired - refresh
                try await refreshAuthToken()
                
                // Retry query
                return try await executeQuery(query)
            } else if error.code == "FORBIDDEN" {
                // User doesn't have permission
                throw AppError.unauthorized
            }
            
            throw error
        }
    }
    
    private func refreshAuthToken() async throws {
        let mutation = """
        mutation {
            refreshToken(refreshToken: "\(currentRefreshToken)") {
                accessToken
                refreshToken
            }
        }
        """
        
        let response = try await executeQuery(mutation)
        // Update tokens
    }
}
```

### GraphQL vs REST vs gRPC

```swift
/*
┌────────────────┬─────────────┬─────────────┬─────────────┐
│ Feature        │ REST        │ GraphQL     │ gRPC        │
├────────────────┼─────────────┼─────────────┼─────────────┤
│ Data Fetching  │ Fixed       │ Flexible    │ Fixed       │
│ Over-fetching  │ ❌ Common   │ ✅ None     │ ❌ Common   │
│ Multiple Calls │ ❌ Yes      │ ✅ No       │ ✅ No       │
│ Learning Curve │ ✅ Easy     │ ⚠️ Medium   │ ❌ Hard     │
│ Tooling        │ ✅ Mature   │ ✅ Good     │ ⚠️ Limited  │
│ Performance    │ ⚠️ Good     │ ✅ Great    │ ✅ Fastest  │
│ Binary Data    │ ✅ Easy     │ ❌ Hard     │ ✅ Native   │
│ Caching        │ ✅ HTTP     │ ⚠️ Complex  │ ❌ Manual   │
│ Streaming      │ ⚠️ SSE      │ ⚠️ Subscr.  │ ✅ Native   │
└────────────────┴─────────────┴─────────────┴─────────────┘

WHEN TO USE EACH:

REST:
✅ Simple CRUD apps
✅ Public APIs
✅ File uploads/downloads
✅ Simple caching needs
❌ Complex nested data
❌ Multiple round trips

GraphQL:
✅ Complex UIs (social feeds, dashboards)
✅ Mobile apps (bandwidth optimization)
✅ Multiple clients with different needs
✅ Rapid UI iteration
❌ Simple apps
❌ File handling
❌ Real-time gaming

gRPC:
✅ Microservices communication
✅ Real-time streaming
✅ Low latency requirements
✅ Binary data
❌ Web browsers (limited support)
❌ Public APIs
❌ Simple REST patterns
*/
```

---

## 📝 INTERVIEW QUESTIONS

### Q1: Design the GraphQL schema and queries for an Uber-like ride-hailing app

**What They're Testing:** Real-world system design, understanding of mobile needs

**Expected Answer:**

```swift
/*
SCREENS TO SUPPORT:
1. Home Screen (request ride)
2. Ride Tracking (active ride)
3. Ride History
4. Driver Profile

SCHEMA DESIGN:
*/

type User {
    id: ID!
    name: String!
    phone: String!
    rating: Float
    paymentMethods: [PaymentMethod]
    savedAddresses: [SavedAddress]
}

type SavedAddress {
    id: ID!
    label: String!  # "Home", "Work"
    address: String!
    coordinates: Coordinates
}

type Ride {
    id: ID!
    status: RideStatus!
    passenger: User!
    driver: Driver
    pickup: Location!
    dropoff: Location!
    estimatedFare: Money
    actualFare: Money
    estimatedArrival: DateTime
    distance: Float
    duration: Int
}

enum RideStatus {
    REQUESTED
    DRIVER_ASSIGNED
    DRIVER_ARRIVING
    IN_PROGRESS
    COMPLETED
    CANCELLED
}

type Driver {
    id: ID!
    name: String!
    rating: Float!
    vehicle: Vehicle!
    currentLocation: Coordinates
    photo: String
}

# QUERIES

# Home Screen
query HomeScreen {
    currentUser {
        name
        savedAddresses {
            label
            address
            coordinates
        }
        recentRides(limit: 3) {
            dropoff { address }
            date
        }
    }
    
    nearbyDrivers(location: $userLocation, radius: 2000) {
        id
        currentLocation
        vehicle { type }
    }
}

# Request Ride
mutation RequestRide($pickup: LocationInput!, $dropoff: LocationInput!) {
    requestRide(pickup: $pickup, dropoff: $dropoff) {
        id
        estimatedFare
        estimatedArrival
        status
    }
}

# Track Active Ride (with subscription for real-time)
subscription TrackRide($rideId: ID!) {
    rideUpdated(rideId: $rideId) {
        id
        status
        driver {
            name
            currentLocation
            vehicle { model color }
        }
        estimatedArrival
    }
}

# Ride History
query RideHistory($limit: Int = 20, $after: String) {
    rides(first: $limit, after: $after) {
        edges {
            node {
                id
                date
                pickup { address }
                dropoff { address }
                fare
                driver { name rating }
            }
        }
        pageInfo {
            hasNextPage
            endCursor
        }
    }
}

/*
KEY DESIGN DECISIONS:

1. REAL-TIME TRACKING:
   - Use subscription for driver location
   - Reduces battery vs polling
   - Instant status updates

2. PAGINATION:
   - Cursor-based for ride history
   - Handles new rides being added
   - Efficient for large histories

3. NESTED DATA:
   - Driver info included in ride
   - One query instead of ride + driver
   - Reduces latency

4. ESTIMATED VALUES:
   - Include in initial response
   - Update via subscription
   - Better UX (show estimates immediately)
*/
```

### Q2: How would you handle cache invalidation in a GraphQL mobile app?

**Expected Answer:**

```swift
/*
CACHE INVALIDATION STRATEGIES:

1. TIME-BASED (TTL):
*/
class GraphQLCache {
    func get(query: String, ttl: TimeInterval = 300) -> Data? {
        guard let entry = cache[query] else { return nil }
        
        if Date().timeIntervalSince(entry.timestamp) > ttl {
            cache.removeValue(forKey: query)
            return nil
        }
        
        return entry.data
    }
}

// TTL by query type:
// - User profile: 5 minutes
// - Feed posts: 1 minute
// - Static content: 1 hour


/*
2. MUTATION-BASED:
*/
func mutate(_ mutation: String) async throws {
    // Execute mutation
    let result = try await execute(mutation)
    
    // Invalidate related queries
    if mutation.contains("createPost") {
        cache.invalidate(pattern: "feed")
        cache.invalidate(pattern: "userPosts")
    }
}

/*
3. FIELD-LEVEL INVALIDATION (Normalized cache):
*/
class NormalizedCache {
    func updateField(typename: String, id: String, field: String, value: Any) {
        let key = "\(typename):\(id)"
        objects[key]?[field] = value
        
        // All queries using this object are automatically updated!
    }
}

// Example:
cache.updateField(typename: "User", id: "123", field: "name", value: "New Name")
// All queries (profile, feed, comments) now have updated name!


/*
4. SUBSCRIPTION-BASED:
*/
subscription {
    userUpdated(id: "123") {
        name
        avatar
    }
}

// When received:
func handleSubscription(data: User) {
    cache.updateObject(id: data.id, typename: "User", fields: data)
}


/*
RECOMMENDATION:
Combine all strategies:
- TTL as baseline
- Invalidate on mutations
- Subscriptions for real-time
- Normalized cache for efficiency
*/
```

### Q3: GraphQL vs REST for mobile - when would you choose each?

**Expected Answer:**

```swift
/*
CHOOSE GRAPHQL WHEN:

✅ Complex, nested UI screens
   Example: Instagram post = image + author + comments + likes
   GraphQL: 1 query
   REST: 4-5 requests

✅ Different clients need different data
   iOS app needs: thumbnail + title
   Web needs: full image + description + metadata
   GraphQL: Each client queries exactly what it needs

✅ Frequent UI changes
   Designer adds new field to screen
   GraphQL: Just add to query (no backend change)
   REST: Backend must add to endpoint

✅ Bandwidth-constrained
   Mobile users on slow networks
   GraphQL: Only essential data
   REST: All fields always sent

✅ Multiple data sources
   User from DB + Posts from cache + Analytics from service
   GraphQL: Single unified API
   REST: Multiple endpoints


CHOOSE REST WHEN:

✅ Simple CRUD operations
   Blog: Get post → Edit → Save
   REST endpoints are simpler

✅ File uploads/downloads
   Profile picture, documents
   REST multipart/form-data works better

✅ HTTP caching is critical
   Public data (blog posts, news)
   REST uses standard HTTP cache headers

✅ Third-party integration
   Public API for developers
   REST is more familiar

✅ Simple team/app
   Small app, small team
   GraphQL adds complexity


REAL EXAMPLE - INSTAGRAM:

Screen: Post Detail

REST approach:
GET /posts/123           → 500KB (post + unused fields)
GET /users/456           → 200KB (author + unused fields)
GET /posts/123/likes     → 100KB
GET /posts/123/comments  → 300KB

Total: 4 requests, 1.1MB, ~2 seconds

GraphQL approach:
query {
    post(id: "123") {
        image
        caption
        author { username avatar }
        likeCount
        comments(first: 20) {
            text
            author { username }
        }
    }
}

Total: 1 request, 50KB, 0.3 seconds

Decision: GraphQL wins for Instagram! ✅


REAL EXAMPLE - SIMPLE BLOG:

REST approach:
GET /posts → List of posts
GET /posts/123 → Single post
POST /posts → Create post

Simple, standard, works great!

GraphQL approach:
query { posts { ... } }
query { post(id: "123") { ... } }
mutation { createPost(...) { ... } }

Decision: REST wins - simpler! ✅


MY RECOMMENDATION:
Start with REST for MVP
Migrate to GraphQL when:
- App has 10+ screens
- Lots of nested data
- Multiple mobile platforms
- Bandwidth is critical
*/
```

### Q4: Design an offline-first GraphQL mobile app

**Expected Answer:**

```swift
/*
ARCHITECTURE:

┌─────────────────────────────────────────┐
│              UI LAYER                    │
│  (Always shows data from cache)         │
├─────────────────────────────────────────┤
│           GRAPHQL CLIENT                 │
│  ┌──────────────┐  ┌──────────────┐    │
│  │ Query Manager│  │ Mutation     │    │
│  │              │  │ Queue        │    │
│  └──────────────┘  └──────────────┘    │
├─────────────────────────────────────────┤
│        NORMALIZED CACHE                  │
│    (Single source of truth)             │
├─────────────────────────────────────────┤
│          PERSISTENCE                     │
│  (SQLite / Core Data / Realm)           │
└─────────────────────────────────────────┘

IMPLEMENTATION:
*/

actor OfflineGraphQLClient {
    private let cache: NormalizedCache
    private let persistence: PersistenceLayer
    private let mutationQueue: MutationQueue
    
    // READS: Always from cache
    func query<T: Decodable>(_ query: String) async -> T {
        // 1. Get from cache (instant)
        if let cached = await cache.get(query: query) {
            print("📦 Cache hit (offline-safe)")
            
            // 2. Refresh in background if online
            if NetworkMonitor.shared.isOnline {
                Task {
                    try? await refreshFromNetwork(query)
                }
            }
            
            return cached
        }
        
        // 3. Load from persistence
        if let persisted = await persistence.load(query: query) {
            await cache.store(data: persisted, forQuery: query)
            return persisted
        }
        
        // 4. Must fetch (need network)
        if NetworkMonitor.shared.isOnline {
            return try await fetchAndStore(query)
        }
        
        // 5. Offline and no cache - return empty
        return emptyResponse()
    }
    
    // WRITES: Queue for sync
    func mutate(_ mutation: String) async {
        let mutationId = UUID()
        
        // 1. Apply optimistic update
        let optimisticData = generateOptimisticResponse(mutation)
        await cache.applyOptimistic(id: mutationId, data: optimisticData)
        
        // 2. Add to queue
        await mutationQueue.enqueue(
            id: mutationId,
            mutation: mutation,
            timestamp: Date()
        )
        
        // 3. Try to sync immediately
        if NetworkMonitor.shared.isOnline {
            await syncMutations()
        } else {
            print("📴 Offline - queued for later")
        }
    }
    
    private func syncMutations() async {
        let pending = await mutationQueue.getPending()
        
        for mutation in pending {
            do {
                // Execute on server
                let response = try await execute(mutation.query)
                
                // Update cache with real data
                await cache.replaceOptimistic(
                    id: mutation.id,
                    with: response
                )
                
                // Remove from queue
                await mutationQueue.markCompleted(mutation.id)
                
            } catch {
                print("❌ Sync failed: \(error)")
                // Keep in queue for retry
            }
        }
    }
}

// EXAMPLE: Offline Post Creation

class CreatePostViewModel: ObservableObject {
    @Published var posts: [Post] = []
    private let client = OfflineGraphQLClient()
    
    func createPost(text: String, image: UIImage) async {
        let mutation = """
        mutation {
            createPost(text: "\(text)", image: "\(imageUrl)") {
                id
                text
                createdAt
                author { name }
            }
        }
        """
        
        // This works offline!
        await client.mutate(mutation)
        
        // Post immediately appears in UI with optimistic data
        // Shows "Pending" badge
        // Syncs when online
    }
}

// UI shows pending state
struct PostRow: View {
    let post: Post
    
    var body: some View {
        VStack {
            Text(post.text)
            
            if post.isOptimistic {
                Label("Pending sync", systemImage: "clock")
                    .foregroundColor(.orange)
            }
        }
    }
}

/*
KEY FEATURES:

1. ALWAYS WORKS:
   ✅ No loading spinners
   ✅ No "no connection" errors
   ✅ Immediate feedback

2. CONFLICT RESOLUTION:
   - Last write wins
   - Or custom merge logic
   - Server can reject optimistic updates

3. SYNC STRATEGIES:
   - Immediate (when online)
   - Periodic (every 30 sec)
   - On app foreground
   - Manual refresh

4. STORAGE:
   - Normalized cache in memory
   - Persist to SQLite
   - Encrypt sensitive data
*/
```

### Q5: How do you handle authentication in GraphQL mobile apps?

**Expected Answer:**

```swift
/*
APPROACH 1: Header-based (Recommended)
*/

class AuthenticatedGraphQLClient {
    private var accessToken: String?
    private var refreshToken: String?
    
    func query<T: Decodable>(_ query: String) async throws -> T {
        var request = URLRequest(url: endpoint)
        request.httpMethod = "POST"
        
        // Add auth header
        if let token = accessToken {
            request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        }
        
        let body = ["query": query]
        request.httpBody = try JSONEncoder().encode(body)
        
        do {
            let (data, response) = try await URLSession.shared.data(for: request)
            
            // Check for 401 Unauthorized
            if let httpResponse = response as? HTTPURLResponse,
               httpResponse.statusCode == 401 {
                // Token expired - refresh
                try await refreshAccessToken()
                
                // Retry original query
                return try await self.query(query)
            }
            
            return try JSONDecoder().decode(T.self, from: data)
            
        } catch {
            throw error
        }
    }
    
    private func refreshAccessToken() async throws {
        guard let refreshToken = refreshToken else {
            throw AuthError.noRefreshToken
        }
        
        let mutation = """
        mutation {
            refreshToken(token: "\(refreshToken)") {
                accessToken
                refreshToken
                expiresIn
            }
        }
        """
        
        let response: TokenResponse = try await execute(mutation)
        
        // Update tokens
        self.accessToken = response.accessToken
        self.refreshToken = response.refreshToken
        
        // Store in keychain
        KeychainManager.save(accessToken: response.accessToken)
        KeychainManager.save(refreshToken: response.refreshToken)
    }
}

/*
APPROACH 2: Context-based (For some GraphQL servers)
*/

mutation Login($email: String!, $password: String!) {
    login(email: $email, password: $password) {
        token
        user {
            id
            name
        }
    }
}

// Then use token in subsequent requests

/*
SECURE STORAGE:
*/

class KeychainManager {
    static func save(accessToken: String) {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "accessToken",
            kSecValueData as String: accessToken.data(using: .utf8)!
        ]
        
        SecItemDelete(query as CFDictionary)
        SecItemAdd(query as CFDictionary, nil)
    }
    
    static func getAccessToken() -> String? {
        let query: [String: Any] = [
            kSecClass as String: kSecClassGenericPassword,
            kSecAttrAccount as String: "accessToken",
            kSecReturnData as String: true
        ]
        
        var result: AnyObject?
        SecItemCopyMatching(query as CFDictionary, &result)
        
        if let data = result as? Data {
            return String(data: data, encoding: .utf8)
        }
        
        return nil
    }
}

/*
HANDLING LOGOUT:
*/

func logout() async {
    // Clear tokens
    accessToken = nil
    refreshToken = nil
    
    // Clear keychain
    KeychainManager.deleteTokens()
    
    // Clear cache
    await cache.clearAll()
    
    // Navigate to login
    await MainActor.run {
        appState.isLoggedIn = false
    }
}
```

---

## 🎯 Key Takeaways

**For Interviews:**

1. **Explain trade-offs clearly**
   - "GraphQL is great for X because..."
   - "But REST is better for Y because..."

2. **Focus on mobile impacts**
   - Bandwidth savings
   - Battery life
   - Offline support
   - UI performance

3. **Know when NOT to use GraphQL**
   - Simple apps
   - File uploads
   - Standard CRUD

4. **Understand caching**
   - Memory + disk + normalized
   - Invalidation strategies
   - Offline-first patterns

5. **Real-time considerations**
   - Subscriptions vs polling
   - Battery impact
   - WebSocket management

This comprehensive guide covers GraphQL from beginner to advanced, with production-ready iOS code and interview preparation! 🚀
