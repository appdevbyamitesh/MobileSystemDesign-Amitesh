# Error Handling Patterns and Result Types: Complete Deep Dive

## 🎈 Level 1: Beginner - Understanding Errors Like a Child

### What is an Error?

Imagine you're making a sandwich. Things can go wrong:
- 🍞 No bread in the kitchen
- 🥜 You're allergic to peanut butter
- 🔪 The knife is dirty

These are **errors** - problems that stop you from completing your task.

In programming, errors are the same - things that can go wrong when your code runs.

### Two Ways to Handle Errors in Swift

Swift gives you two main tools:

**1. Try-Catch (Traditional Way)**
```swift
do {
    try makeSandwich()
    print("Yay! Sandwich ready!")
} catch {
    print("Oops! Something went wrong: \(error)")
}
```

**2. Result Type (Modern Way)**
```swift
let result = makeSandwich()
switch result {
case .success(let sandwich):
    print("Yay! Got sandwich: \(sandwich)")
case .failure(let error):
    print("Oops! Something went wrong: \(error)")
}
```

### Simple Example: Division

**Problem:** What happens when you divide by zero? Error!

```swift
// Define possible errors
enum MathError: Error {
    case divisionByZero
}

// Function that can fail
func divide(_ a: Int, by b: Int) throws -> Int {
    if b == 0 {
        throw MathError.divisionByZero  // "Throw" an error
    }
    return a / b
}

// Using try-catch
do {
    let result = try divide(10, by: 2)
    print("Result: \(result)")  // 5
} catch {
    print("Error: \(error)")
}

// What if we divide by zero?
do {
    let result = try divide(10, by: 0)
    print("Result: \(result)")  // This line never runs
} catch MathError.divisionByZero {
    print("Cannot divide by zero!")  // This runs!
}
```

***

## 🚀 Level 2: Intermediate - Understanding Error Handling Patterns

### Pattern 1: Try-Catch-Throw

This is Swift's built-in error handling:

#### Step 1: Define Errors
```swift
enum NetworkError: Error {
    case noConnection
    case timeout
    case invalidURL
    case serverError(code: Int)
}
```

#### Step 2: Mark Function as Throwing
```swift
func fetchUser(id: String) throws -> User {
    // Check for problems
    guard hasConnection() else {
        throw NetworkError.noConnection
    }
    
    guard let url = URL(string: "https://api.com/user/\(id)") else {
        throw NetworkError.invalidURL
    }
    
    // Fetch and return user
    let user = try downloadUser(from: url)
    return user
}
```

#### Step 3: Handle Errors with Do-Catch
```swift
do {
    let user = try fetchUser(id: "123")
    print("Got user: \(user.name)")
} catch NetworkError.noConnection {
    print("No internet connection")
} catch NetworkError.timeout {
    print("Request timed out")
} catch NetworkError.serverError(let code) {
    print("Server error: \(code)")
} catch {
    print("Unknown error: \(error)")  // Catch-all
}
```

### Try Variants

Swift has three ways to use `try`:

**1. Regular `try` - Must use do-catch**
```swift
do {
    let result = try riskyFunction()
    print(result)
} catch {
    print("Error: \(error)")
}
```

**2. `try?` - Converts error to nil (Optional)**
```swift
let result = try? riskyFunction()  // Returns Optional
if let value = result {
    print("Success: \(value)")
} else {
    print("Failed, but I don't care why")
}
```

**3. `try!` - Force try (CRASHES if error occurs)**
```swift
let result = try! riskyFunction()  // Crashes on error - DANGEROUS!
// Only use when 100% certain no error can happen
```

**Comparison:**
| Try Variant | Returns | Error Handling | Use Case |
|-------------|---------|----------------|----------|
| `try` | Value or throws | Must catch | Standard - always preferred |
| `try?` | Optional | Converts to nil | When failure isn't critical |
| `try!` | Value (force) | Crashes app | Only when 100% sure it works |

### Pattern 2: Result Type

Introduced in Swift 5, `Result` is an enum with two cases:

```swift
enum Result<Success, Failure: Error> {
    case success(Success)
    case failure(Failure)
}
```

**Think of it as:** "The result is EITHER a success OR a failure, never both".

#### Basic Example

```swift
enum FileError: Error {
    case notFound
    case unreadable
    case empty
}

func readFile(named: String) -> Result<String, FileError> {
    // Check if file exists
    guard fileExists(named) else {
        return .failure(.notFound)
    }
    
    // Check if readable
    guard canRead(named) else {
        return .failure(.unreadable)
    }
    
    // Read content
    let content = getContent(named)
    guard !content.isEmpty else {
        return .failure(.empty)
    }
    
    return .success(content)
}

// Using the result
let result = readFile(named: "data.txt")

switch result {
case .success(let content):
    print("File content: \(content)")
case .failure(let error):
    print("Error reading file: \(error)")
}
```

### Result Type Methods

Result has helpful methods:

**1. map - Transform success value**
```swift
let result = readFile(named: "numbers.txt")
let countResult = result.map { $0.count }  // Count characters

// If success: .success(100)
// If failure: .failure(error) - unchanged
```

**2. mapError - Transform error**
```swift
enum UIError: Error {
    case displayError(String)
}

let uiResult = result.mapError { fileError in
    return UIError.displayError("File issue: \(fileError)")
}
```

**3. flatMap - Chain operations**
```swift
let result = readFile(named: "data.txt")
    .flatMap { content in
        parseJSON(content)  // Returns another Result
    }
```

**4. get() - Extract value or throw**
```swift
do {
    let content = try result.get()
    print(content)
} catch {
    print("Error: \(error)")
}
```

### When to Use What?

**Use Try-Catch when:**
- ✅ Function is doing immediate work
- ✅ Error should propagate up the call stack
- ✅ Writing synchronous code
- ✅ Want to handle errors right where they occur

**Use Result Type when:**
- ✅ Async operations (networking, file I/O)
- ✅ Want to store success/failure for later
- ✅ Need to transform or chain operations
- ✅ Building completion handlers
- ✅ Want explicit error types in function signature

***

## 💎 Level 3: Advanced - Senior Engineer Patterns

### Pattern 1: Async/Await with Error Handling

Modern Swift uses async/await:

```swift
enum APIError: Error {
    case invalidResponse
    case decodingFailed
    case serverError(Int)
}

// Async throwing function
func fetchUser(id: String) async throws -> User {
    let url = URL(string: "https://api.com/user/\(id)")!
    
    let (data, response) = try await URLSession.shared.data(from: url)
    
    guard let httpResponse = response as? HTTPURLResponse else {
        throw APIError.invalidResponse
    }
    
    guard httpResponse.statusCode == 200 else {
        throw APIError.serverError(httpResponse.statusCode)
    }
    
    do {
        let user = try JSONDecoder().decode(User.self, from: data)
        return user
    } catch {
        throw APIError.decodingFailed
    }
}

// Usage
Task {
    do {
        let user = try await fetchUser(id: "123")
        print("User: \(user.name)")
    } catch APIError.serverError(let code) {
        print("Server error: \(code)")
    } catch APIError.decodingFailed {
        print("Failed to decode JSON")
    } catch {
        print("Unknown error: \(error)")
    }
}
```

### Pattern 2: Result with Async/Await

Combining Result with async:

```swift
func fetchUser(id: String) async -> Result<User, APIError> {
    do {
        let url = URL(string: "https://api.com/user/\(id)")!
        let (data, response) = try await URLSession.shared.data(from: url)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            return .failure(.invalidResponse)
        }
        
        let user = try JSONDecoder().decode(User.self, from: data)
        return .success(user)
    } catch {
        return .failure(.decodingFailed)
    }
}

// Usage
Task {
    let result = await fetchUser(id: "123")
    switch result {
    case .success(let user):
        print("User: \(user.name)")
    case .failure(let error):
        print("Error: \(error)")
    }
}
```

### Pattern 3: Custom Result Builders

Building a type-safe error handling DSL:

```swift
struct NetworkResult<T> {
    let data: T?
    let error: Error?
    let statusCode: Int?
    
    var isSuccess: Bool {
        return error == nil && data != nil
    }
    
    static func success(_ data: T, statusCode: Int = 200) -> NetworkResult<T> {
        return NetworkResult(data: data, error: nil, statusCode: statusCode)
    }
    
    static func failure(_ error: Error, statusCode: Int? = nil) -> NetworkResult<T> {
        return NetworkResult(data: nil, error: error, statusCode: statusCode)
    }
}

// Usage with pattern matching
func handleResult<T>(_ result: NetworkResult<T>) {
    switch (result.data, result.error, result.statusCode) {
    case (let data?, nil, 200):
        print("Success: \(data)")
    case (nil, let error?, 404):
        print("Not found: \(error)")
    case (nil, let error?, 500...):
        print("Server error: \(error)")
    default:
        print("Unknown state")
    }
}
```

### Pattern 4: Error Recovery and Retry Logic

```swift
enum NetworkError: Error {
    case timeout
    case noConnection
    case serverError
}

func fetchWithRetry<T>(
    maxAttempts: Int = 3,
    delay: TimeInterval = 1.0,
    operation: () async throws -> T
) async throws -> T {
    var lastError: Error?
    
    for attempt in 1...maxAttempts {
        do {
            return try await operation()
        } catch {
            lastError = error
            print("Attempt \(attempt) failed: \(error)")
            
            if attempt < maxAttempts {
                try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
            }
        }
    }
    
    throw lastError ?? NetworkError.serverError
}

// Usage
Task {
    do {
        let user = try await fetchWithRetry {
            try await fetchUser(id: "123")
        }
        print("Success: \(user)")
    } catch {
        print("Failed after retries: \(error)")
    }
}
```

### Pattern 5: Type-Erased Error Wrapper

When you need to hide specific error types:

```swift
struct AnyError: Error, CustomStringConvertible {
    private let _description: String
    let underlyingError: Error
    
    init(_ error: Error) {
        self.underlyingError = error
        self._description = String(describing: error)
    }
    
    var description: String {
        return _description
    }
}

// Convert specific errors to generic
func fetchData() -> Result<Data, AnyError> {
    let result = performNetworkRequest()
    return result.mapError { AnyError($0) }
}
```

### Pattern 6: Error Context and Debugging

Adding context to errors:

```swift
struct ContextualError: Error {
    let context: String
    let underlyingError: Error
    let file: String
    let line: Int
    
    init(
        _ context: String,
        error: Error,
        file: String = #file,
        line: Int = #line
    ) {
        self.context = context
        self.underlyingError = error
        self.file = file
        self.line = line
    }
}

// Usage
func loadUserData(id: String) throws -> User {
    do {
        return try fetchUser(id: id)
    } catch {
        throw ContextualError("Failed to load user \(id)", error: error)
    }
}

// When caught:
// Error: Failed to load user 123
// File: UserService.swift, Line: 42
```

### Pattern 7: Result Builders for Complex Operations

Chaining multiple operations:

```swift
extension Result {
    func then<NewSuccess>(
        _ transform: (Success) -> Result<NewSuccess, Failure>
    ) -> Result<NewSuccess, Failure> {
        switch self {
        case .success(let value):
            return transform(value)
        case .failure(let error):
            return .failure(error)
        }
    }
}

// Chain multiple operations
let finalResult = fetchUser(id: "123")
    .then { user in
        fetchPosts(for: user)
    }
    .then { posts in
        processPosts(posts)
    }
    .then { processedData in
        saveToDatabase(processedData)
    }
```

### Pattern 8: Completion Handler with Result

Before async/await, this was the standard:

```swift
func fetchUser(
    id: String,
    completion: @escaping (Result<User, NetworkError>) -> Void
) {
    URLSession.shared.dataTask(with: userURL) { data, response, error in
        if let error = error {
            completion(.failure(.noConnection))
            return
        }
        
        guard let data = data else {
            completion(.failure(.invalidResponse))
            return
        }
        
        do {
            let user = try JSONDecoder().decode(User.self, from: data)
            completion(.success(user))
        } catch {
            completion(.failure(.decodingFailed))
        }
    }.resume()
}

// Usage
fetchUser(id: "123") { result in
    switch result {
    case .success(let user):
        print("User: \(user.name)")
    case .failure(let error):
        print("Error: \(error)")
    }
}
```

### Pattern 9: Throwing vs Result - Performance

**Throwing functions:**
```swift
func fetchUser() throws -> User {
    // Compiler optimizes this path
    // Zero-cost when no error occurs
    // Stack unwinding on error
}
```

**Result type:**
```swift
func fetchUser() -> Result<User, Error> {
    // Always allocates Result enum
    // Slightly slower than throws
    // But more explicit and composable
}
```

**Performance comparison:**
- **Throws**: Faster when no errors (~10-20% faster)
- **Result**: Consistent performance, better for async
- **Memory**: Throws uses less memory (no enum allocation)

### Pattern 10: Protocol-Oriented Error Handling

```swift
protocol ErrorHandling {
    associatedtype ErrorType: Error
    func handle(error: ErrorType)
}

class NetworkErrorHandler: ErrorHandling {
    func handle(error: NetworkError) {
        switch error {
        case .noConnection:
            showNoConnectionAlert()
        case .timeout:
            retryRequest()
        case .serverError(let code):
            logError(code: code)
        }
    }
    
    private func showNoConnectionAlert() { /* ... */ }
    private func retryRequest() { /* ... */ }
    private func logError(code: Int) { /* ... */ }
}

// Generic error handling function
func performOperation<Handler: ErrorHandling>(
    handler: Handler,
    operation: () throws -> Void
) where Handler.ErrorType == Error {
    do {
        try operation()
    } catch let error as Handler.ErrorType {
        handler.handle(error: error)
    } catch {
        print("Unhandled error: \(error)")
    }
}
```

### Real-World Architecture: Repository Pattern

Complete example with error handling:

```swift
// 1. Define domain errors
enum RepositoryError: Error, LocalizedError {
    case networkFailure(Error)
    case decodingFailure
    case notFound
    case unauthorized
    
    var errorDescription: String? {
        switch self {
        case .networkFailure(let error):
            return "Network error: \(error.localizedDescription)"
        case .decodingFailure:
            return "Failed to decode response"
        case .notFound:
            return "Resource not found"
        case .unauthorized:
            return "Unauthorized access"
        }
    }
}

// 2. Protocol with Result
protocol UserRepository {
    func fetchUser(id: String) async -> Result<User, RepositoryError>
    func updateUser(_ user: User) async -> Result<Void, RepositoryError>
}

// 3. Implementation
class NetworkUserRepository: UserRepository {
    private let session: URLSession
    
    init(session: URLSession = .shared) {
        self.session = session
    }
    
    func fetchUser(id: String) async -> Result<User, RepositoryError> {
        guard let url = URL(string: "https://api.com/users/\(id)") else {
            return .failure(.notFound)
        }
        
        do {
            let (data, response) = try await session.data(from: url)
            
            guard let httpResponse = response as? HTTPURLResponse else {
                return .failure(.networkFailure(NSError(domain: "", code: -1)))
            }
            
            switch httpResponse.statusCode {
            case 200:
                do {
                    let user = try JSONDecoder().decode(User.self, from: data)
                    return .success(user)
                } catch {
                    return .failure(.decodingFailure)
                }
            case 401:
                return .failure(.unauthorized)
            case 404:
                return .failure(.notFound)
            default:
                return .failure(.networkFailure(NSError(domain: "", code: httpResponse.statusCode)))
            }
        } catch {
            return .failure(.networkFailure(error))
        }
    }
    
    func updateUser(_ user: User) async -> Result<Void, RepositoryError> {
        // Implementation
        return .success(())
    }
}

// 4. ViewModel with error handling
@MainActor
class UserViewModel: ObservableObject {
    @Published var user: User?
    @Published var errorMessage: String?
    @Published var isLoading = false
    
    private let repository: UserRepository
    
    init(repository: UserRepository) {
        self.repository = repository
    }
    
    func loadUser(id: String) async {
        isLoading = true
        errorMessage = nil
        
        let result = await repository.fetchUser(id: id)
        
        isLoading = false
        
        switch result {
        case .success(let user):
            self.user = user
        case .failure(let error):
            self.errorMessage = error.errorDescription
        }
    }
}

// 5. SwiftUI View
struct UserView: View {
    @StateObject private var viewModel: UserViewModel
    
    var body: some View {
        VStack {
            if viewModel.isLoading {
                ProgressView()
            } else if let user = viewModel.user {
                Text("User: \(user.name)")
            } else if let error = viewModel.errorMessage {
                Text("Error: \(error)")
                    .foregroundColor(.red)
            }
        }
        .task {
            await viewModel.loadUser(id: "123")
        }
    }
}
```

***

## Interview Questions & Answers

### Q1: What's the difference between throws and Result type?

**Expert Answer:** `throws` is Swift's built-in error propagation mechanism where errors bubble up the call stack until caught by a do-catch block. `Result<Success, Failure>` is an enum that explicitly represents success or failure as a value.

**Key differences:**

**throws** is better for synchronous operations where errors should propagate immediately. It has zero performance overhead when no error occurs, uses less memory, and errors are handled via do-catch blocks.

**Result** is better for asynchronous operations, completion handlers, and when you want to store or pass around success/failure state. It makes error types explicit in function signatures, enables functional programming patterns like map/flatMap, but has slight overhead from enum allocation.

Use `throws` for immediate operations and `Result` for async callbacks or when you need to defer error handling.

### Q2: Explain try, try?, and try! with use cases.

**Expert Answer:** Swift provides three try variants for different error handling needs:

**`try`** requires a do-catch block and propagates errors explicitly. Use this as the default - it forces you to handle errors properly and makes error paths visible.

**`try?`** converts errors into optional nil values, discarding error information. Use when failure is acceptable and you don't need to know why it failed, such as loading cached data or parsing optional configuration.

**`try!`** force-unwraps and crashes on error. Only use when you're absolutely certain no error can occur, like loading bundled resources from your app bundle. Misuse causes production crashes.

Example: `try` for user actions, `try?` for non-critical features like loading avatars, `try!` only for impossible-to-fail scenarios like built-in resources.

### Q3: How do you design custom error types? Best practices?

**Expert Answer:** Custom error types should conform to the `Error` protocol and typically use enums for categorical errors. Best practices include:

**Use enums with associated values** to provide context: `enum NetworkError: Error { case serverError(statusCode: Int) }`.

**Conform to LocalizedError** for user-facing messages: implement `errorDescription` to return localized strings.

**Group related errors** hierarchically: have domain-specific errors like `DatabaseError`, `NetworkError`, `ValidationError` rather than one giant error enum.

**Include debugging information**: add file/line information or operation context using associated values.

**Make errors recoverable when possible**: include enough information for retry logic or alternative actions.

Avoid generic errors like returning bare `Error` - specific error types enable proper handling and recovery.

### Q4: What's the performance difference between throws and Result?

**Expert Answer:** `throws` has better performance characteristics than `Result` in the success path.

When no error occurs, throwing functions have essentially zero overhead - the compiler optimizes them to regular function calls. The error path uses stack unwinding which is more expensive but rarely taken.

`Result` always allocates an enum instance whether success or failure, adding overhead (~16 bytes for the enum). However, this overhead is consistent and predictable.

**Benchmarks show** throwing functions are typically 10-20% faster in the happy path, but the difference is negligible for most applications.

For async operations with completion handlers, `Result` is actually more efficient because it avoids separate success/failure callback parameters and provides a single, type-safe return value.

Choose based on API design needs, not micro-optimizations - the clarity and type safety are more important than the small performance difference.

### Q5: How do you handle errors in async/await code?

**Expert Answer:** async/await uses standard do-catch with `try await` for throwing async functions:

```swift
do {
    let user = try await fetchUser(id: "123")
    let posts = try await fetchPosts(for: user)
    await updateUI(with: posts)
} catch {
    await showError(error)
}
```

For multiple concurrent operations, use `async let` with try:

```swift
async let user = fetchUser()
async let settings = fetchSettings()

do {
    let (userData, userSettings) = try await (user, settings)
} catch {
    // Handle error from either operation
}
```

You can also return Result from async functions when you want explicit error types without throwing:

```swift
func fetchUser() async -> Result<User, NetworkError> {
    // Implementation
}
```

Use Task groups for error handling across multiple tasks, where one failure can cancel others or errors are collected for aggregate handling.

### Q6: When should you use rethrow?

**Expert Answer:** `rethrows` is used when a function takes a throwing closure and only throws if that closure throws. It's primarily for higher-order functions:

```swift
func map<T, U>(_ array: [T], transform: (T) throws -> U) rethrows -> [U] {
    var result: [U] = []
    for element in array {
        result.append(try transform(element))
    }
    return result
}
```

The `rethrows` keyword means: "I only throw if the closure you gave me throws." This allows calling the function without try if the closure doesn't throw:

```swift
let doubled = map([1,2,3]) { $0 * 2 }  // No try needed
let parsed = try map(["1","2"]) { try Int($0) }  // try required
```

This provides better ergonomics than always requiring try. Use `rethrows` for any function that takes a throwing closure and just propagates its errors without throwing its own.

### Q7: How do you test error handling code?

**Expert Answer:** Testing error paths is critical and often overlooked. Strategies include:

**1. Test specific error cases:**
```swift
func testNetworkErrorHandling() async {
    let result = await repository.fetchUser(id: "invalid")
    XCTAssertThrowsError(try result.get()) { error in
        XCTAssertEqual(error as? NetworkError, .notFound)
    }
}
```

**2. Use dependency injection with mock errors:**
```swift
class MockRepository: UserRepository {
    var errorToThrow: NetworkError?
    
    func fetchUser(id: String) async -> Result<User, NetworkError> {
        if let error = errorToThrow {
            return .failure(error)
        }
        return .success(mockUser)
    }
}
```

**3. Test error recovery logic:**
```swift
func testRetryOnFailure() async {
    var attempts = 0
    mock.onFetch = {
        attempts += 1
        if attempts < 3 { throw NetworkError.timeout }
        return user
    }
    let result = try await fetchWithRetry(operation: mock.fetch)
    XCTAssertEqual(attempts, 3)
}
```

**4. Test error messages and user-facing strings** to ensure they're appropriate and localized.

Always test both success and all error paths - error handling bugs are common in production.

### Q8: What are the anti-patterns in Swift error handling?

**Expert Answer:** Common anti-patterns include:

**1. Swallowing errors with try?** without logging: `let _ = try? dangerousOperation()` loses all error information.

**2. Overusing try!** - causes production crashes. Only acceptable for truly impossible failures.

**3. Catch-all blocks** that hide specific errors:
```swift
do { try operation() } catch { print("Error") }  // Too generic
```

**4. Using Error instead of specific types** loses type safety and makes exhaustive handling impossible.

**5. Not providing context** - throwing errors without information about what operation failed or why.

**6. Mixing error handling styles** - using both Result and throws inconsistently in the same codebase.

**7. Force unwrapping Result with try!** defeats the purpose: `try! result.get()`.

**8. Not testing error paths** - assuming happy path is sufficient.

Follow conventions: use throws for sync, Result for async; always catch specific errors; provide context; test all error paths.

***

## Summary

### Error Handling Patterns

**Try-Catch-Throw:**
- ✅ Built-in Swift error handling
- ✅ Zero overhead in success path
- ✅ Automatic error propagation
- ✅ Best for synchronous operations

**Result Type:**
- ✅ Explicit success/failure representation
- ✅ Composable with map/flatMap
- ✅ Perfect for async operations
- ✅ Type-safe error handling
- ⚠️ Slight enum allocation overhead

**Best Practices:**
- Use specific error types over generic Error
- Conform to LocalizedError for user messages
- Test both success and error paths
- Provide context in errors
- Choose throws for sync, Result for async
- Never swallow errors silently

Error handling is not optional - it's what separates robust, production-ready code from fragile prototypes. Master these patterns and your iOS apps will be significantly more reliable!
