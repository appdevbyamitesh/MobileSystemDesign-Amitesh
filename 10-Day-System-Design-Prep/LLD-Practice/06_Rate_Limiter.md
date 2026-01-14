# LLD Problem 6: Client-Side Rate Limiter

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Algorithm Knowledge**: Token bucket, sliding window
- **Concurrency**: Thread-safe request management
- **API Design**: Clean interfaces
- **Mobile Awareness**: Battery and network efficiency

---

# R — Requirements

## Functional Requirements

```markdown
1. Rate Limiting
   - Limit requests per time window
   - Per-endpoint configuration
   - Multiple strategies (token bucket, sliding window)

2. Request Management
   - Queue or reject exceeded requests
   - Retry-After header support
   - Priority request handling

3. Monitoring
   - Remaining quota visibility
   - Rate limit exceeded callbacks
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Overhead | < 1ms | Don't slow requests |
| Memory | < 1MB | Minimal footprint |
| Thread safety | Required | Concurrent calls |
| Testability | High | Unit test strategies |

---

# E — Entities

```swift
// MARK: - Core Types

struct RateLimitConfig {
    let endpoint: String
    let maxRequests: Int
    let windowSeconds: TimeInterval
    let strategy: RateLimitStrategyType
}

enum RateLimitStrategyType {
    case tokenBucket
    case slidingWindow
    case fixedWindow
}

enum RateLimitResult {
    case allowed(remainingRequests: Int)
    case rejected(retryAfter: TimeInterval)
}

// MARK: - Request

struct APIRequest {
    let id: UUID
    let endpoint: String
    let priority: RequestPriority
    let timestamp: Date
}

enum RequestPriority: Int, Comparable {
    case low = 0
    case normal = 1
    case high = 2
    case critical = 3  // Always allowed
    
    static func < (lhs: RequestPriority, rhs: RequestPriority) -> Bool {
        lhs.rawValue < rhs.rawValue
    }
}
```

---

# S — States

```swift
enum RateLimiterState {
    case accepting
    case throttled(until: Date)
    case disabled
}
```

## State Diagram

```
[Accepting] ──quota exhausted──▶ [Throttled]
     ▲                               │
     │         window reset          │
     └───────────────────────────────┘
```

---

# H — Handling Concurrency

```swift
actor RateLimiterManager {
    private var limiters: [String: RateLimiter] = [:]
    
    func shouldAllow(request: APIRequest) async -> RateLimitResult {
        let limiter = getOrCreate(for: request.endpoint)
        return await limiter.checkAndConsume()
    }
    
    private func getOrCreate(for endpoint: String) -> RateLimiter {
        if let existing = limiters[endpoint] {
            return existing
        }
        let config = RateLimitConfig.default(for: endpoint)
        let limiter = config.strategy.createLimiter(config: config)
        limiters[endpoint] = limiter
        return limiter
    }
}
```

---

# A — Architecture & Patterns

## Pattern 1: Strategy (Rate Limiting Algorithms)

### Token Bucket Strategy

```swift
protocol RateLimiter: Actor {
    func checkAndConsume() async -> RateLimitResult
    func reset()
}

actor TokenBucketLimiter: RateLimiter {
    private let maxTokens: Int
    private let refillRate: Double  // tokens per second
    private var tokens: Double
    private var lastRefillTime: Date
    
    init(maxTokens: Int, refillRate: Double) {
        self.maxTokens = maxTokens
        self.refillRate = refillRate
        self.tokens = Double(maxTokens)
        self.lastRefillTime = Date()
    }
    
    func checkAndConsume() async -> RateLimitResult {
        refillTokens()
        
        if tokens >= 1 {
            tokens -= 1
            return .allowed(remainingRequests: Int(tokens))
        } else {
            let waitTime = (1 - tokens) / refillRate
            return .rejected(retryAfter: waitTime)
        }
    }
    
    private func refillTokens() {
        let now = Date()
        let elapsed = now.timeIntervalSince(lastRefillTime)
        let tokensToAdd = elapsed * refillRate
        
        tokens = min(Double(maxTokens), tokens + tokensToAdd)
        lastRefillTime = now
    }
    
    func reset() {
        tokens = Double(maxTokens)
        lastRefillTime = Date()
    }
}
```

### Sliding Window Strategy

```swift
actor SlidingWindowLimiter: RateLimiter {
    private let maxRequests: Int
    private let windowSeconds: TimeInterval
    private var requestTimestamps: [Date] = []
    
    init(maxRequests: Int, windowSeconds: TimeInterval) {
        self.maxRequests = maxRequests
        self.windowSeconds = windowSeconds
    }
    
    func checkAndConsume() async -> RateLimitResult {
        let now = Date()
        let windowStart = now.addingTimeInterval(-windowSeconds)
        
        // Remove timestamps outside window
        requestTimestamps.removeAll { $0 < windowStart }
        
        if requestTimestamps.count < maxRequests {
            requestTimestamps.append(now)
            return .allowed(remainingRequests: maxRequests - requestTimestamps.count)
        } else {
            // Calculate when oldest will expire
            let oldestRequest = requestTimestamps.first!
            let retryAfter = windowSeconds - now.timeIntervalSince(oldestRequest)
            return .rejected(retryAfter: retryAfter)
        }
    }
    
    func reset() {
        requestTimestamps.removeAll()
    }
}
```

### Why Strategy Pattern

| Alternative | Why Rejected |
|-------------|--------------|
| if-else in single class | Not extensible |
| Inheritance | Algorithms are interchangeable |
| Hardcoded | Can't swap at runtime |

---

## Pattern 2: Singleton (Shared Limiter)

```swift
actor RateLimiterManager {
    static let shared = RateLimiterManager()
    
    private var limiters: [String: any RateLimiter] = [:]
    private let configProvider: RateLimitConfigProvider
    
    private init(configProvider: RateLimitConfigProvider = .default) {
        self.configProvider = configProvider
    }
    
    func shouldAllow(endpoint: String, priority: RequestPriority = .normal) async -> RateLimitResult {
        // Critical priority bypasses rate limiting
        if priority == .critical {
            return .allowed(remainingRequests: -1)
        }
        
        let limiter = await getOrCreateLimiter(for: endpoint)
        return await limiter.checkAndConsume()
    }
}
```

### ⚠️ Singleton Caution

```swift
// Make testable with protocol
protocol RateLimiterManagerProtocol {
    func shouldAllow(endpoint: String, priority: RequestPriority) async -> RateLimitResult
}

// For testing
actor MockRateLimiterManager: RateLimiterManagerProtocol {
    var allowAll = true
    
    func shouldAllow(endpoint: String, priority: RequestPriority) async -> RateLimitResult {
        allowAll ? .allowed(remainingRequests: 100) : .rejected(retryAfter: 60)
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Request Throttling

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  View  │ │ APIClient  │ │RateLimiter │ │TokenBucket │ │   Server   │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │
    │ fetchData()│              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ shouldAllow? │              │              │
    │            │─────────────▶│              │              │
    │            │              │              │              │
    │            │              │ checkAndConsume()           │
    │            │              │─────────────▶│              │
    │            │              │              │              │
    │            │              │              │──┐           │
    │            │              │              │  │ refill    │
    │            │              │              │  │ tokens    │
    │            │              │              │◀─┘           │
    │            │              │              │              │
    │            │              │◀─────────────│ .allowed(5)  │
    │            │              │              │              │
    │            │◀─────────────│              │              │
    │            │              │              │              │
    │            │ make request │              │              │
    │            │────────────────────────────────────────────▶
    │            │              │              │              │
    │            │◀────────────────────────────────────────────│
    │            │              │              │              │
    │◀───────────│ response     │              │              │
    │            │              │              │              │
```

---

# E — Edge Cases

## Edge Case 1: Burst of Requests

```swift
// Handle burst gracefully
class ThrottledAPIClient {
    private let rateLimiter: RateLimiterManager
    private let requestQueue = DispatchQueue(label: "request.queue")
    private var pendingRequests: [PendingRequest] = []
    
    func request<T>(_ endpoint: String) async throws -> T {
        let result = await rateLimiter.shouldAllow(endpoint: endpoint)
        
        switch result {
        case .allowed:
            return try await executeRequest(endpoint)
        case .rejected(let retryAfter):
            // Queue the request
            return try await queueAndRetry(endpoint, after: retryAfter)
        }
    }
    
    private func queueAndRetry<T>(_ endpoint: String, after delay: TimeInterval) async throws -> T {
        try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
        return try await request(endpoint)  // Recursive retry
    }
}
```

## Edge Case 2: Clock Drift

```swift
actor SlidingWindowLimiter: RateLimiter {
    // Use monotonic time to avoid clock drift issues
    private var requestDescriptors: [(timestamp: CFAbsoluteTime, id: UUID)] = []
    
    func checkAndConsume() async -> RateLimitResult {
        let now = CFAbsoluteTimeGetCurrent()
        let windowStart = now - windowSeconds
        
        requestDescriptors.removeAll { $0.timestamp < windowStart }
        // ... rest of logic
    }
}
```

---

# D — Design Trade-offs

| Token Bucket | Sliding Window |
|--------------|----------------|
| Allows bursts | Strict limit |
| Fixed memory | Grows with requests |
| Constant time | O(n) cleanup |

**Decision:** Token bucket for most APIs, sliding window for payment APIs

---

# iOS Implementation

```swift
class APIClient {
    private let rateLimiter: RateLimiterManager
    private let session: URLSession
    
    func request<T: Decodable>(_ endpoint: Endpoint) async throws -> T {
        // Check rate limit
        let result = await rateLimiter.shouldAllow(
            endpoint: endpoint.path,
            priority: endpoint.priority
        )
        
        switch result {
        case .allowed(let remaining):
            // Update UI with remaining quota
            await MainActor.run {
                QuotaIndicator.shared.update(remaining: remaining)
            }
            
            return try await executeRequest(endpoint)
            
        case .rejected(let retryAfter):
            // Check if server also sent 429
            throw APIError.rateLimited(retryAfter: retryAfter)
        }
    }
}

// Usage
let client = APIClient()

do {
    let data = try await client.request(.fetchPosts)
} catch APIError.rateLimited(let retryAfter) {
    showRetryAfterAlert(seconds: retryAfter)
}
```

---

# Interview Tips

## What to Say

```markdown
1. "I'll use Strategy pattern for different rate limiting algorithms..."
2. "Token bucket allows bursts, sliding window is stricter..."
3. "Singleton with protocol extraction for testability..."
4. "Actor ensures thread safety without manual locking..."
```

## Red Flags to Avoid

```markdown
❌ "I'll use a simple counter that resets every minute"
   → Doesn't handle edge of window

❌ "I'll just catch 429 from server"
   → Server rate limits are last resort

❌ Using NSLock instead of Actor
   → Modern Swift uses actors
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Rate Limiter problem!*
