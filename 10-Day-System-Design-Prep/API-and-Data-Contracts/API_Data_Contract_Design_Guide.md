# API & Data Contract Design for iOS Engineers

> A comprehensive guide from beginner to Big Tech interview level — focused on how iOS apps consume APIs

---

## Table of Contents

1. [Beginner Level — Simple English](#beginner-level--simple-english)
2. [Intermediate Level — SDE-2 / L4 Ready](#intermediate-level--sde-2--l4-ready)
3. [Advanced Level — Big Tech Interviews](#advanced-level--big-tech-interviews)
4. [Mobile System Design Application — Real-World Example](#mobile-system-design-application--real-world-example)
5. [Interview Preparation Section](#interview-preparation-section)

---

# Beginner Level — Simple English

## What is an API?

Think of an API (Application Programming Interface) like a **waiter in a restaurant**.

- You (the iOS app) are the customer
- The kitchen (backend server) prepares the food (data)
- The waiter (API) takes your order to the kitchen and brings back the food

**In mobile terms:**
```
Your iPhone App → Makes a Request → API → Backend Server
                ← Gets a Response ←
```

### Real Example: Uber App

When you open Uber and see your trip history:

1. The app sends a request: "Give me the last 20 trips for this user"
2. The API receives this request
3. The backend fetches the data
4. The API sends back a response with trip details
5. Your app displays the trips

---

## Why Mobile Apps Need Clear Data Contracts

A **data contract** is like a promise between the app and the server about what data looks like.

### Imagine This Problem:

**Server sends:**
```json
{
  "user_name": "John",
  "tripCost": "15.50"
}
```

**But iOS app expects:**
```json
{
  "userName": "John",
  "tripCost": 15.50
}
```

**Result:** App crashes or shows wrong data!

### Why Contracts Matter:

| Without Clear Contract | With Clear Contract |
|------------------------|---------------------|
| App crashes on unexpected data | App knows exactly what to expect |
| Developers guess field names | Everyone agrees on field names |
| Numbers might come as strings | Data types are fixed |
| New fields break old apps | Backward compatibility is maintained |

---

## What "RESTful Design" Means in Practice

REST (Representational State Transfer) is a way of designing APIs that is easy to understand and use.

### Key REST Concepts (Simple Explanation):

1. **Resources = Things**
   - A trip, a user, an order, a payment — these are resources
   - Each resource has a unique address (URL)

2. **URLs = Addresses**
   - `/users/123` → User with ID 123
   - `/trips/456` → Trip with ID 456
   - `/orders/789` → Order with ID 789

3. **HTTP Methods = Actions**
   - GET = Read (get something)
   - POST = Create (make something new)
   - PUT/PATCH = Update (change something)
   - DELETE = Remove (delete something)

4. **Stateless = Server Forgets You**
   - Each request must carry all needed information
   - Server doesn't remember your previous request

---

## HTTP Methods with Real Mobile Examples

### GET — Fetch Data (Read Only)

Used when the app wants to **read** data without changing anything.

```
GET /users/me/trips
```

**Real Examples:**

| App | Action | API Call |
|-----|--------|----------|
| Uber | View trip history | `GET /trips` |
| DoorDash | See past orders | `GET /orders` |
| Instagram | Load your feed | `GET /feed` |
| WhatsApp | Fetch messages | `GET /chats/{chatId}/messages` |

**iOS Code Example:**
```swift
func fetchTrips() async throws -> [Trip] {
    let url = URL(string: "https://api.uber.com/v1/trips")!
    var request = URLRequest(url: url)
    request.httpMethod = "GET"
    request.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
    
    let (data, _) = try await URLSession.shared.data(for: request)
    return try JSONDecoder().decode([Trip].self, from: data)
}
```

---

### POST — Create New Data

Used when the app wants to **create** something new.

```
POST /trips
Body: { destination: "Airport", pickupLocation: "123 Main St" }
```

**Real Examples:**

| App | Action | API Call |
|-----|--------|----------|
| Uber | Request a ride | `POST /trips` |
| DoorDash | Place new order | `POST /orders` |
| Venmo | Send payment | `POST /payments` |
| Twitter | Create a tweet | `POST /tweets` |

**iOS Code Example:**
```swift
struct CreateTripRequest: Encodable {
    let pickupLocation: Location
    let destination: Location
    let rideType: String
}

func createTrip(request: CreateTripRequest) async throws -> Trip {
    let url = URL(string: "https://api.uber.com/v1/trips")!
    var urlRequest = URLRequest(url: url)
    urlRequest.httpMethod = "POST"
    urlRequest.setValue("application/json", forHTTPHeaderField: "Content-Type")
    urlRequest.httpBody = try JSONEncoder().encode(request)
    
    let (data, _) = try await URLSession.shared.data(for: urlRequest)
    return try JSONDecoder().decode(Trip.self, from: data)
}
```

---

### PUT — Replace Entire Resource

Used to **completely replace** an existing resource.

```
PUT /users/me/profile
Body: { name: "John", phone: "123-456-7890", email: "john@email.com" }
```

**When to use PUT:**
- You're sending ALL fields
- Missing fields should be cleared/removed

---

### PATCH — Update Part of Resource

Used to **partially update** an existing resource.

```
PATCH /users/me/profile
Body: { phone: "123-456-7890" }
```

**Real Examples:**

| App | Action | API Call |
|-----|--------|----------|
| Uber | Update destination | `PATCH /trips/{id}` |
| DoorDash | Change delivery instructions | `PATCH /orders/{id}` |
| WhatsApp | Update profile photo | `PATCH /profile` |

**PUT vs PATCH — Mobile Developer Rule:**

| Use PUT When | Use PATCH When |
|--------------|----------------|
| Replacing entire profile | Updating just phone number |
| Overwriting a document | Adding a single preference |
| Form has all fields | Form has one field |

---

### DELETE — Remove Data

Used to **remove** a resource.

```
DELETE /payments/card/123
```

**Real Examples:**

| App | Action | API Call |
|-----|--------|----------|
| Uber | Remove saved card | `DELETE /payments/{cardId}` |
| DoorDash | Delete saved address | `DELETE /addresses/{id}` |
| Spotify | Remove from playlist | `DELETE /playlists/{id}/tracks/{trackId}` |

---

## Simple Data Models and Entity Relationships

### What is a Data Model?

A **data model** defines the structure of data your app works with.

**Example: Trip Model**
```swift
struct Trip: Codable {
    let id: String
    let userId: String
    let driverId: String?
    let pickupLocation: Location
    let destination: Location
    let status: TripStatus
    let fare: Money
    let createdAt: Date
}

struct Location: Codable {
    let latitude: Double
    let longitude: Double
    let address: String
}

struct Money: Codable {
    let amount: Decimal
    let currency: String  // "USD", "INR"
}

enum TripStatus: String, Codable {
    case requested
    case driverAssigned
    case driverArrived
    case inProgress
    case completed
    case cancelled
}
```

---

### Entity Relationships (How Data Connects)

**Real-World Example: Uber Data Relationships**

```
┌──────────┐       ┌──────────┐       ┌──────────┐
│   User   │──────▶│   Trip   │◀──────│  Driver  │
└──────────┘       └──────────┘       └──────────┘
     │                   │
     │                   │
     ▼                   ▼
┌──────────┐       ┌──────────┐
│ Payment  │       │  Rating  │
│ Methods  │       └──────────┘
└──────────┘
```

**In Simple Terms:**
- A **User** can have many **Trips**
- A **Trip** has one **User** and one **Driver**
- A **User** can have many **Payment Methods**
- Each **Trip** can have **Ratings** (passenger rates driver, driver rates passenger)

---

### Common Entity Relationships in Mobile Apps

| App | Entities | Relationship |
|-----|----------|--------------|
| Uber | User, Trip, Driver | User has many Trips; Trip has one Driver |
| DoorDash | User, Order, Restaurant | User has many Orders; Order has one Restaurant |
| Instagram | User, Post, Comment | User has many Posts; Post has many Comments |
| WhatsApp | User, Chat, Message | Chat has many Users; Chat has many Messages |

---

## Why Predictable Request/Response Shapes Matter

### What is a "Shape"?

The **shape** is the structure of JSON data — what fields exist and what types they have.

**Predictable Shape:**
```json
{
  "id": "trip_123",
  "status": "completed",
  "fare": {
    "amount": 15.50,
    "currency": "USD"
  },
  "driver": {
    "id": "driver_456",
    "name": "John",
    "rating": 4.8
  }
}
```

**Unpredictable Shape (BAD!):**
```json
{
  "id": 123,                    // Sometimes number, sometimes string
  "status": "COMPLETED",        // Sometimes uppercase, sometimes lowercase
  "fare": "15.50 USD",          // Sometimes string, sometimes object
  "driver": "John (4.8 stars)"  // Sometimes string, sometimes object
}
```

---

### Why This Matters for iOS

**Problem with Unpredictable Data:**

```swift
// This crashes if "id" is sometimes a number
struct Trip: Codable {
    let id: String  // 💥 Crash if server sends: "id": 123
}
```

**How iOS Handles This (Codable):**

```swift
struct Trip: Codable {
    let id: String
    let status: TripStatus
    let fare: Money
    let driver: Driver?  // Optional if driver not assigned yet
}

// Decoding
do {
    let trip = try JSONDecoder().decode(Trip.self, from: jsonData)
} catch {
    // Handle decoding error - shape didn't match!
    print("Failed to decode trip: \(error)")
}
```

---

### Rules for Predictable Shapes

| Rule | Good Example | Bad Example |
|------|--------------|-------------|
| Consistent field names | `userName` everywhere | `userName`, `user_name`, `username` mixed |
| Consistent types | `"id": "123"` always string | Sometimes string, sometimes number |
| Consistent null handling | `"driver": null` | Sometimes missing, sometimes null |
| Consistent date format | ISO 8601: `"2024-01-15T10:30:00Z"` | `"Jan 15, 2024"`, `"1705314600"` mixed |

---

# Intermediate Level — SDE-2 / L4 Ready

## Designing RESTful Endpoints for Mobile Usage

### Mobile-First API Design Principles

| Principle | Why It Matters for Mobile |
|-----------|---------------------------|
| Minimize round trips | Slow network = bad UX |
| Return only needed data | Limited bandwidth & battery |
| Support partial loading | Cellular data caps |
| Handle offline gracefully | Users enter tunnels, elevators |

---

### URL Structure Best Practices

**Good URL Design:**
```
Base: https://api.uber.com

/v1/users/{userId}                    → User profile
/v1/users/{userId}/trips              → User's trips
/v1/users/{userId}/trips/{tripId}     → Specific trip
/v1/users/{userId}/payment-methods    → User's cards
/v1/trips/{tripId}/ratings            → Ratings for a trip
```

**Bad URL Design:**
```
/getTripsForUser?user=123             → Verbs in URL
/v1/user_trips/getUserTrips           → Inconsistent naming
/trips/list                           → Redundant "list"
```

---

### Designing Endpoints for Common Mobile Screens

**Uber Trip List Screen:**

| Screen Element | API Endpoint | Method |
|---------------|--------------|--------|
| Trip history | `/v1/trips?status=completed` | GET |
| Upcoming trips | `/v1/trips?status=scheduled` | GET |
| Trip details | `/v1/trips/{tripId}` | GET |
| Cancel trip | `/v1/trips/{tripId}/cancel` | POST |

**Food Delivery Order Screen:**

| Screen Element | API Endpoint | Method |
|---------------|--------------|--------|
| Current order status | `/v1/orders/{orderId}` | GET |
| Order history | `/v1/orders?status=delivered` | GET |
| Track delivery | `/v1/orders/{orderId}/tracking` | GET |
| Rate order | `/v1/orders/{orderId}/rating` | POST |

---

## Request / Response DTOs (Data Transfer Objects)

### What is a DTO?

A **DTO** is a data structure specifically designed for API communication.

**Why DTOs?**

```
┌─────────────┐                          ┌─────────────┐
│  Database   │                          │   iOS App   │
│   Model     │ ──X── Not Same ──X── ─▶  │   Model     │
└─────────────┘                          └─────────────┘
       │                                        ▲
       ▼                                        │
┌─────────────┐                          ┌─────────────┐
│  Backend    │ ◀────── API ────────────▶│   iOS DTO   │
│    DTO      │         Contract          └─────────────┘
└─────────────┘
```

### Real Example: Why DTOs Differ from Internal Models

**Database Model (Server Side):**
```
User Table:
- id (UUID)
- email
- password_hash        ← Never send to client!
- created_at
- last_login_ip        ← Never send to client!
- account_status
- internal_notes       ← Never send to client!
```

**API Response DTO (What iOS Receives):**
```swift
struct UserResponse: Codable {
    let id: String
    let email: String
    let displayName: String
    let profileImageUrl: String?
    let memberSince: Date
}
// No password, no IP, no internal notes!
```

---

### Request DTOs vs Response DTOs

**Request DTO (iOS → Server):**
```swift
// What iOS sends to create a trip
struct CreateTripRequest: Encodable {
    let pickupLocation: LocationDTO
    let dropoffLocation: LocationDTO
    let paymentMethodId: String
    let rideType: String  // "uberX", "uberXL", "comfort"
}

struct LocationDTO: Codable {
    let latitude: Double
    let longitude: Double
    let formattedAddress: String
}
```

**Response DTO (Server → iOS):**
```swift
// What iOS receives after creating trip
struct CreateTripResponse: Decodable {
    let tripId: String
    let estimatedPickupTime: Date
    let estimatedFare: MoneyDTO
    let driver: DriverDTO?  // null until driver accepts
    let vehicle: VehicleDTO?
    let status: String
}

struct DriverDTO: Decodable {
    let id: String
    let name: String
    let photoUrl: String
    let rating: Double
    let vehicleMake: String
    let vehicleModel: String
    let licensePlate: String
}
```

---

### Why DTOs Should Differ from App's Domain Models

**API Response DTO:**
```swift
struct TripResponse: Decodable {
    let tripId: String
    let pickupLat: Double
    let pickupLng: Double
    let pickupAddress: String
    let dropoffLat: Double
    let dropoffLng: Double
    let dropoffAddress: String
    let fareAmount: Double
    let fareCurrency: String
    let status: String
    let createdAt: String  // ISO8601 string
}
```

**App's Domain Model (Cleaner for UI):**
```swift
struct Trip {
    let id: String
    let pickup: Location
    let dropoff: Location
    let fare: Money
    let status: TripStatus
    let createdAt: Date
    
    // Computed properties for UI
    var formattedFare: String {
        return fare.formatted()
    }
    
    var isActive: Bool {
        return status == .inProgress || status == .driverAssigned
    }
}

// Mapper: DTO → Domain Model
extension Trip {
    init(from response: TripResponse) {
        self.id = response.tripId
        self.pickup = Location(
            latitude: response.pickupLat,
            longitude: response.pickupLng,
            address: response.pickupAddress
        )
        self.dropoff = Location(
            latitude: response.dropoffLat,
            longitude: response.dropoffLng,
            address: response.dropoffAddress
        )
        self.fare = Money(
            amount: Decimal(response.fareAmount),
            currency: response.fareCurrency
        )
        self.status = TripStatus(rawValue: response.status) ?? .unknown
        
        let formatter = ISO8601DateFormatter()
        self.createdAt = formatter.date(from: response.createdAt) ?? Date()
    }
}
```

---

## Handling Optional Fields & Backward Compatibility

### Optional Fields in Swift

```swift
struct TripResponse: Decodable {
    let tripId: String
    let status: String
    
    // Optional fields - may or may not be present
    let driver: DriverDTO?           // null before driver accepts
    let estimatedArrival: String?    // null if unknown
    let promotionApplied: String?    // null if no promo
    
    // New field added in API v2 - old responses won't have it
    let carbonOffset: Double?        // Added later, make optional!
}
```

### Backward Compatibility Rules for iOS

| Rule | Implementation |
|------|----------------|
| New fields should be optional | `let newField: Type?` |
| Use default values | `let newField: Type = defaultValue` |
| Never remove fields | Just stop sending them (keep optional) |
| Never change field types | Add new field with new type instead |

### Handling Unknown Enum Values

**Problem:** What if server sends a status your app doesn't know?

```swift
// BAD: App crashes on unknown status
enum TripStatus: String, Codable {
    case requested
    case completed
    case cancelled
    // Server sends "scheduled" - CRASH!
}

// GOOD: Handle unknown values gracefully
enum TripStatus: String, Codable {
    case requested
    case completed
    case cancelled
    case unknown  // Fallback for future values
    
    init(from decoder: Decoder) throws {
        let container = try decoder.singleValueContainer()
        let rawValue = try container.decode(String.self)
        self = TripStatus(rawValue: rawValue) ?? .unknown
    }
}
```

---

## Pagination Strategies for Mobile Apps

### Why Pagination Matters for Mobile

| Without Pagination | With Pagination |
|-------------------|-----------------|
| Download 10,000 trips at once | Download 20 trips at a time |
| App freezes for 30 seconds | App loads in 500ms |
| Uses 50MB of data | Uses 50KB per page |
| Battery drains fast | Battery efficient |
| Memory issues, crashes | Smooth scrolling |

---

### Offset-Based Pagination

**How it works:**
```
Page 1: GET /trips?offset=0&limit=20    → Items 1-20
Page 2: GET /trips?offset=20&limit=20   → Items 21-40
Page 3: GET /trips?offset=40&limit=20   → Items 41-60
```

**Response Example:**
```json
{
  "trips": [...],
  "pagination": {
    "offset": 0,
    "limit": 20,
    "total": 150,
    "hasMore": true
  }
}
```

**iOS Implementation:**
```swift
class TripRepository {
    private var currentOffset = 0
    private let pageSize = 20
    private var hasMore = true
    
    func loadNextPage() async throws -> [Trip] {
        guard hasMore else { return [] }
        
        let url = URL(string: "https://api.uber.com/v1/trips?offset=\(currentOffset)&limit=\(pageSize)")!
        let response = try await apiClient.get(TripListResponse.self, from: url)
        
        currentOffset += pageSize
        hasMore = response.pagination.hasMore
        
        return response.trips.map { Trip(from: $0) }
    }
    
    func refresh() {
        currentOffset = 0
        hasMore = true
    }
}
```

**Pros:**
- Simple to implement
- Can jump to any page
- Easy to show "Page 3 of 10"

**Cons:**
- Data shifts cause duplicates or missed items
- Slow on large datasets (DB has to skip N rows)
- Not suitable for real-time changing data

**Offset Problem Illustrated:**
```
Initial data: [A, B, C, D, E, F, G, H, I, J]

User loads page 1 (offset=0, limit=5): [A, B, C, D, E]

New item Z is added at the beginning: [Z, A, B, C, D, E, F, G, H, I, J]

User loads page 2 (offset=5, limit=5): [E, F, G, H, I]

Result: User sees E twice! 😱
```

---

### Cursor-Based Pagination

**How it works:**
```
Page 1: GET /trips?limit=20              → Returns cursor: "abc123"
Page 2: GET /trips?cursor=abc123&limit=20 → Returns cursor: "def456"
Page 3: GET /trips?cursor=def456&limit=20 → Returns cursor: null (no more)
```

**Response Example:**
```json
{
  "trips": [...],
  "pagination": {
    "nextCursor": "eyJ0cmlwSWQiOiJ0cmlwXzEyMzQ1IiwiY3JlYXRlZEF0IjoiMjAyNC0wMS0xNVQxMDozMDowMFoifQ==",
    "hasMore": true
  }
}
```

**iOS Implementation:**
```swift
class TripRepository {
    private var nextCursor: String? = nil
    private var hasMore = true
    
    func loadNextPage() async throws -> [Trip] {
        guard hasMore else { return [] }
        
        var urlString = "https://api.uber.com/v1/trips?limit=20"
        if let cursor = nextCursor {
            urlString += "&cursor=\(cursor)"
        }
        
        let url = URL(string: urlString)!
        let response = try await apiClient.get(TripListResponse.self, from: url)
        
        nextCursor = response.pagination.nextCursor
        hasMore = response.pagination.hasMore
        
        return response.trips.map { Trip(from: $0) }
    }
    
    func refresh() {
        nextCursor = nil
        hasMore = true
    }
}
```

**Pros:**
- No duplicates when data changes
- Fast on large datasets
- Works well with real-time data

**Cons:**
- Can't jump to arbitrary page
- Can't show "Page X of Y"
- Slightly more complex

---

### Why Large Apps Prefer Cursor-Based

**Uber, Instagram, Twitter all use cursor-based because:**

1. **Data changes constantly**
   - New trips added
   - Posts deleted
   - Feed reordered

2. **Datasets are huge**
   - Millions of records
   - Offset has to skip N rows (slow)
   - Cursor jumps directly to position (fast)

3. **Better user experience**
   - No duplicate items in feed
   - Infinite scroll works smoothly

---

## Error Handling and Status Codes

### HTTP Status Codes for Mobile

| Code | Meaning | iOS Action |
|------|---------|------------|
| 200 | Success | Parse and display data |
| 201 | Created | Resource created successfully |
| 204 | No Content | Delete successful, no body |
| 400 | Bad Request | Show validation errors |
| 401 | Unauthorized | Redirect to login |
| 403 | Forbidden | Show permission denied |
| 404 | Not Found | Show "not found" message |
| 409 | Conflict | Handle duplicate/conflict |
| 422 | Validation Error | Show field-specific errors |
| 429 | Rate Limited | Back off, retry later |
| 500 | Server Error | Show generic error, retry |
| 503 | Service Unavailable | Retry with backoff |

---

### Standardized Error Response Format

```json
{
  "error": {
    "code": "PAYMENT_FAILED",
    "message": "Your card was declined",
    "details": {
      "reason": "insufficient_funds",
      "cardLast4": "1234"
    },
    "retryable": false
  }
}
```

**iOS Error Handling:**
```swift
struct APIError: Decodable {
    let code: String
    let message: String
    let details: [String: String]?
    let retryable: Bool
}

enum NetworkError: Error {
    case unauthorized
    case notFound
    case serverError(APIError)
    case rateLimited(retryAfter: TimeInterval)
    case noConnection
    case unknown
}

func handleResponse(_ response: HTTPURLResponse, data: Data) throws {
    switch response.statusCode {
    case 200...299:
        return  // Success
        
    case 401:
        throw NetworkError.unauthorized
        
    case 404:
        throw NetworkError.notFound
        
    case 429:
        let retryAfter = Double(response.value(forHTTPHeaderField: "Retry-After") ?? "60") ?? 60
        throw NetworkError.rateLimited(retryAfter: retryAfter)
        
    case 400...499:
        let error = try JSONDecoder().decode(APIError.self, from: data)
        throw NetworkError.serverError(error)
        
    case 500...599:
        throw NetworkError.unknown
        
    default:
        throw NetworkError.unknown
    }
}
```

---

### Displaying Errors to Users

```swift
extension NetworkError {
    var userFacingMessage: String {
        switch self {
        case .unauthorized:
            return "Please log in again"
        case .notFound:
            return "This item no longer exists"
        case .serverError(let error):
            return error.message  // Server provides user-friendly message
        case .rateLimited:
            return "Too many requests. Please wait a moment."
        case .noConnection:
            return "No internet connection"
        case .unknown:
            return "Something went wrong. Please try again."
        }
    }
    
    var shouldRetry: Bool {
        switch self {
        case .rateLimited, .unknown:
            return true
        case .serverError(let error):
            return error.retryable
        default:
            return false
        }
    }
}
```

---

# Advanced Level — Big Tech Interviews

## Designing Stable API Contracts for Long-Lived Mobile Apps

### The Challenge

```
App Store has:
├── App v1.0 (released 2 years ago)
├── App v1.5 (released 1 year ago)
├── App v2.0 (released 6 months ago)
└── App v2.5 (current)

All versions call the SAME API! 😰
```

### Stability Rules

| Rule | Why |
|------|-----|
| Never remove fields | Old apps depend on them |
| Never rename fields | Old apps won't find them |
| Never change field types | Old apps will crash |
| Add new fields as optional | Old apps ignore them |
| Use nullable types generously | Gives you flexibility |

### Additive-Only Changes (Safe)

```swift
// Version 1 Response
struct TripResponseV1: Decodable {
    let tripId: String
    let status: String
    let fare: Double
}

// Version 2 Response (Additive = Safe)
struct TripResponseV2: Decodable {
    let tripId: String
    let status: String
    let fare: Double
    let carbonOffset: Double?      // NEW - optional
    let sustainabilityBadge: String?  // NEW - optional
}

// Old v1 apps still work - they just ignore new fields!
```

---

## API Versioning Strategies

### 1. URL-Based Versioning

```
https://api.uber.com/v1/trips
https://api.uber.com/v2/trips
```

**Pros:**
- Very explicit and clear
- Easy to route on backend
- Easy to test and debug

**Cons:**
- Multiple endpoints to maintain
- Harder to deprecate old versions
- URL changes can break caching

**iOS Implementation:**
```swift
enum APIVersion {
    case v1
    case v2
    
    var baseURL: String {
        switch self {
        case .v1: return "https://api.uber.com/v1"
        case .v2: return "https://api.uber.com/v2"
        }
    }
}

class APIClient {
    let version: APIVersion
    
    init(version: APIVersion = .v2) {
        self.version = version
    }
    
    func url(for path: String) -> URL {
        URL(string: "\(version.baseURL)\(path)")!
    }
}
```

---

### 2. Header-Based Versioning

```
GET https://api.uber.com/trips
Headers:
  Accept: application/vnd.uber.v2+json
  X-API-Version: 2
```

**Pros:**
- Clean URLs
- Easy to switch versions
- Works well with content negotiation

**Cons:**
- Version not visible in URL
- Harder to test in browser
- Easy to forget the header

**iOS Implementation:**
```swift
class APIClient {
    private let apiVersion = 2
    
    func request(for url: URL) -> URLRequest {
        var request = URLRequest(url: url)
        request.setValue("application/vnd.uber.v\(apiVersion)+json", 
                        forHTTPHeaderField: "Accept")
        request.setValue("\(apiVersion)", 
                        forHTTPHeaderField: "X-API-Version")
        return request
    }
}
```

---

### 3. Field-Based (Feature Flags)

```json
// Request includes capabilities
POST /trips
Headers:
  X-Client-Capabilities: ["surge_pricing_v2", "carbon_offset"]

// Server includes/excludes fields based on capabilities
{
  "tripId": "123",
  "fare": 25.00,
  "carbonOffset": 0.5  // Only if client requested this capability
}
```

**Pros:**
- Fine-grained control
- Gradual rollouts
- A/B testing friendly

**Cons:**
- Complex implementation
- Hard to document
- Testing all combinations is difficult

**iOS Implementation:**
```swift
struct ClientCapabilities {
    static let current: [String] = [
        "surge_pricing_v2",
        "carbon_offset",
        "multi_stop_trips",
        "real_time_eta"
    ]
}

class APIClient {
    func request(for url: URL) -> URLRequest {
        var request = URLRequest(url: url)
        request.setValue(ClientCapabilities.current.joined(separator: ","), 
                        forHTTPHeaderField: "X-Client-Capabilities")
        return request
    }
}
```

---

### Mobile Trade-offs Comparison

| Strategy | App Store | Testing | Flexibility | Complexity |
|----------|-----------|---------|-------------|------------|
| URL-based | ✅ Easy to pin version | ✅ Easy | ⚠️ All or nothing | Low |
| Header-based | ✅ Easy to pin | ⚠️ Need tools | ⚠️ All or nothing | Medium |
| Field-based | ⚠️ Need capability management | ❌ Complex | ✅ Very flexible | High |

**Interview Answer:**
> "For most mobile apps, I'd recommend **URL-based versioning** for major changes and **additive field changes** for minor updates. URL versioning is explicit and easy to manage in mobile apps where we can't force users to update. The app's build can simply point to `/v2/` and we know exactly what contract to expect."

---

## Evolving APIs Without Breaking Old Apps

### The Golden Rules

1. **Add, never remove**
   ```json
   // Original
   { "price": 25.00 }
   
   // Evolution (safe)
   { "price": 25.00, "priceBreakdown": { "base": 20, "surge": 5 } }
   ```

2. **Use optional types from day one**
   ```swift
   struct Trip: Decodable {
       let driver: Driver?  // Optional from start = easy to evolve
   }
   ```

3. **Deprecate gracefully**
   ```json
   {
     "price": 25.00,           // Old field - keep for backward compat
     "fare": {                  // New field - modern apps use this
       "amount": 25.00,
       "currency": "USD"
     },
     "_deprecated": ["price"]   // Hint to developers
   }
   ```

4. **Use feature flags for big changes**
   ```swift
   // Check if new feature is available
   if response.capabilities.contains("multi_stop_trips") {
       showMultiStopUI()
   }
   ```

---

## Performance Considerations

### Payload Size Optimization

**Problem: Over-fetching**
```json
// iOS only needs name and photo for a list
GET /users/123

// Server returns EVERYTHING
{
  "id": "123",
  "name": "John",
  "photo": "...",
  "email": "john@email.com",
  "phone": "+1234567890",
  "address": { ... },
  "preferences": { ... },
  "paymentMethods": [...],
  "tripHistory": [...]  // 1000 trips!
}
```

**Solutions:**

1. **Sparse fieldsets**
   ```
   GET /users/123?fields=name,photo
   ```

2. **Different endpoints for different use cases**
   ```
   GET /users/123/profile    → Full profile
   GET /users/123/summary    → Just name + photo
   ```

3. **GraphQL**
   ```graphql
   query {
     user(id: "123") {
       name
       photo
     }
   }
   ```

---

### Under-fetching Problem

**Problem:**
```
Screen needs: User + Trips + Payments

Request 1: GET /users/123
Request 2: GET /users/123/trips
Request 3: GET /users/123/payments

= 3 round trips = slow!
```

**Solutions:**

1. **Compound endpoints**
   ```
   GET /users/123/dashboard
   
   {
     "user": {...},
     "recentTrips": [...],
     "defaultPayment": {...}
   }
   ```

2. **Include parameter**
   ```
   GET /users/123?include=trips,payments
   ```

---

### Mobile-Specific Performance Tips

| Technique | Benefit |
|-----------|---------|
| GZIP compression | 60-80% smaller payloads |
| Response caching | Faster repeat loads |
| ETags | Skip download if unchanged |
| Image URL CDN | Faster image loads |
| Delta sync | Only changed data |

**iOS Implementation: ETag Caching**
```swift
class APIClient {
    private var etags: [URL: String] = [:]
    
    func get<T: Decodable>(_ type: T.Type, from url: URL) async throws -> T? {
        var request = URLRequest(url: url)
        
        // Add ETag if we have one
        if let etag = etags[url] {
            request.setValue(etag, forHTTPHeaderField: "If-None-Match")
        }
        
        let (data, response) = try await URLSession.shared.data(for: request)
        let httpResponse = response as! HTTPURLResponse
        
        // Store new ETag
        if let etag = httpResponse.value(forHTTPHeaderField: "ETag") {
            etags[url] = etag
        }
        
        // 304 = Not Modified, use cached version
        if httpResponse.statusCode == 304 {
            return nil  // Caller uses cached version
        }
        
        return try JSONDecoder().decode(T.self, from: data)
    }
}
```

---

## Consistency Guarantees and Idempotent APIs

### What is Idempotency?

**Idempotent** = Same request, same result, no matter how many times you call it.

**Why it matters for mobile:**
- Network can fail mid-request
- User might tap button twice
- App might retry automatically

**Idempotent Example:**
```
PUT /users/123/profile
{ "name": "John" }

Call once: User name is John
Call twice: User name is still John ✅
Call 100 times: User name is still John ✅
```

**Non-Idempotent Example:**
```
POST /payments
{ "amount": 100 }

Call once: $100 charged
Call twice: $200 charged! 😱
```

---

### Idempotency Keys

**Solution for non-idempotent operations:**

```swift
struct PaymentRequest: Encodable {
    let amount: Decimal
    let idempotencyKey: String  // Client-generated unique ID
}

func createPayment(amount: Decimal) async throws -> Payment {
    // Generate unique key that survives retries
    let idempotencyKey = UUID().uuidString
    
    let request = PaymentRequest(
        amount: amount,
        idempotencyKey: idempotencyKey
    )
    
    // Even if this is called twice with same key,
    // only one payment is created
    return try await apiClient.post(request, to: "/payments")
}
```

**Server behavior with idempotency key:**
```
Request 1: POST /payments (key: "abc123", amount: 100)
→ Creates payment, returns Payment(id: "pay_456")

Request 2: POST /payments (key: "abc123", amount: 100)  // Retry
→ Returns same Payment(id: "pay_456"), no duplicate!
```

---

### iOS Retry Strategy with Idempotency

```swift
class NetworkManager {
    func performWithRetry<T: Decodable>(
        request: URLRequest,
        idempotencyKey: String,
        maxRetries: Int = 3
    ) async throws -> T {
        var lastError: Error?
        var request = request
        request.setValue(idempotencyKey, forHTTPHeaderField: "Idempotency-Key")
        
        for attempt in 0..<maxRetries {
            do {
                let (data, response) = try await URLSession.shared.data(for: request)
                return try JSONDecoder().decode(T.self, from: data)
            } catch {
                lastError = error
                
                // Exponential backoff
                if attempt < maxRetries - 1 {
                    let delay = pow(2.0, Double(attempt))
                    try await Task.sleep(nanoseconds: UInt64(delay * 1_000_000_000))
                }
            }
        }
        
        throw lastError!
    }
}
```

---

## How Big Tech Designs Mobile-Friendly APIs

### Uber's Approach

```swift
// Uber uses a "Trip State Machine" pattern
// Single endpoint returns current state with all needed data

GET /v1/trips/current

// Response includes everything for current state
{
  "state": "DRIVER_EN_ROUTE",
  "trip": { ... },
  "driver": { ... },
  "vehicle": { ... },
  "eta": { ... },
  "route": { ... },
  
  // Available actions based on state
  "actions": [
    { "type": "CANCEL", "url": "/trips/123/cancel" },
    { "type": "CONTACT_DRIVER", "url": "/trips/123/contact" }
  ],
  
  // Polling hint
  "refreshAfterMs": 5000
}
```

### Instagram's Feed Design

```swift
// Cursor-based infinite scroll
GET /v1/feed?cursor=abc123

{
  "posts": [...],
  "cursor": {
    "next": "def456",
    "hasMore": true
  },
  
  // Injected content
  "suggestions": [...],  // "People you might know"
  "ads": [...]           // Ad placements
}
```

### Amazon's Order System

```swift
// Order state endpoint with actions
GET /v1/orders/123

{
  "order": { ... },
  "items": [...],
  "tracking": { ... },
  
  // Dynamic actions based on order state
  "actions": [
    { 
      "type": "CANCEL",
      "available": true,
      "deadline": "2024-01-15T10:00:00Z"
    },
    {
      "type": "RETURN",
      "available": false,
      "reason": "Order not delivered yet"
    }
  ]
}
```

---

# Mobile System Design Application — Real-World Example

## Case Study: Uber Trip History Screen

### Screen Requirements

```
┌─────────────────────────────────────┐
│ ← Trip History                       │
├─────────────────────────────────────┤
│ Filter: All ▼   Sort: Recent ▼      │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │ 📍 Home → Airport               │ │
│ │ Jan 15, 2024 • $45.50           │ │
│ │ ★★★★★ UberX                     │ │
│ └─────────────────────────────────┘ │
│ ┌─────────────────────────────────┐ │
│ │ 📍 Office → Downtown            │ │
│ │ Jan 14, 2024 • $18.25           │ │
│ │ ★★★★☆ UberX                     │ │
│ └─────────────────────────────────┘ │
│                                      │
│         [Load More...]               │
└─────────────────────────────────────┘
```

---

### 1. Endpoint Design

```yaml
Base URL: https://api.uber.com/v1

Endpoints:
  # List trips with filters
  GET /users/{userId}/trips
    Query Parameters:
      - status: completed | cancelled | all
      - cursor: string (for pagination)
      - limit: int (default: 20, max: 50)
      - sortBy: date | price
      - sortOrder: asc | desc
    
  # Single trip details
  GET /trips/{tripId}
  
  # Submit rating (after trip)
  POST /trips/{tripId}/rating
    Body: { rating: 1-5, comment: string? }
```

---

### 2. Data Models and Relationships

```swift
// Core Entities
struct User {
    let id: String
    let name: String
    let email: String
    let phone: String
}

struct Trip {
    let id: String
    let userId: String        // FK → User
    let driverId: String?     // FK → Driver
    let pickup: Location
    let dropoff: Location
    let rideType: RideType
    let status: TripStatus
    let fare: Money
    let rating: Int?
    let createdAt: Date
    let completedAt: Date?
}

struct Driver {
    let id: String
    let name: String
    let photoUrl: String
    let rating: Double
    let vehicle: Vehicle
}

struct Vehicle {
    let make: String
    let model: String
    let color: String
    let licensePlate: String
}

// Relationships:
// User (1) ←→ (N) Trip
// Driver (1) ←→ (N) Trip
// Trip (1) ←→ (1) Vehicle (via Driver)
```

---

### 3. Request / Response DTOs

**Request: List Trips**
```swift
// No body, just query parameters
// GET /users/me/trips?status=completed&cursor=abc123&limit=20
```

**Response: Trip List**
```swift
struct TripListResponse: Decodable {
    let trips: [TripDTO]
    let pagination: PaginationDTO
}

struct TripDTO: Decodable {
    let tripId: String
    let status: String
    let rideType: String
    let pickup: LocationDTO
    let dropoff: LocationDTO
    let fare: MoneyDTO
    let rating: Int?
    let driver: DriverSummaryDTO?
    let createdAt: String
    let completedAt: String?
}

struct LocationDTO: Decodable {
    let latitude: Double
    let longitude: Double
    let address: String
    let shortName: String  // "Home", "Airport"
}

struct MoneyDTO: Decodable {
    let amount: Double
    let currencyCode: String
    let formattedString: String  // "$45.50" - server-formatted
}

struct DriverSummaryDTO: Decodable {
    let driverId: String
    let name: String
    let photoUrl: String
    let rating: Double
}

struct PaginationDTO: Decodable {
    let nextCursor: String?
    let hasMore: Bool
    let totalCount: Int?  // Optional, expensive to compute
}
```

---

### 4. Pagination Approach

**Decision: Cursor-based pagination**

**Why:**
- Trips are continually added
- Users scroll through potentially 1000s of trips
- No duplicate issues when new trips added
- Fast performance at scale

**Implementation:**
```swift
class TripHistoryRepository {
    private var nextCursor: String? = nil
    private var hasMore = true
    private var trips: [Trip] = []
    
    func loadInitial() async throws -> [Trip] {
        nextCursor = nil
        hasMore = true
        trips = []
        return try await loadNextPage()
    }
    
    func loadNextPage() async throws -> [Trip] {
        guard hasMore else { return trips }
        
        var components = URLComponents(string: "https://api.uber.com/v1/users/me/trips")!
        var queryItems = [URLQueryItem(name: "status", value: "completed"),
                          URLQueryItem(name: "limit", value: "20")]
        
        if let cursor = nextCursor {
            queryItems.append(URLQueryItem(name: "cursor", value: cursor))
        }
        components.queryItems = queryItems
        
        let response = try await apiClient.get(TripListResponse.self, from: components.url!)
        
        // Update pagination state
        nextCursor = response.pagination.nextCursor
        hasMore = response.pagination.hasMore
        
        // Convert DTOs to domain models
        let newTrips = response.trips.map { Trip(from: $0) }
        trips.append(contentsOf: newTrips)
        
        return trips
    }
}
```

---

### 5. Versioning Strategy

**Approach: URL-based versioning with additive changes**

```
Current: https://api.uber.com/v1/users/me/trips
Future:  https://api.uber.com/v2/users/me/trips (for breaking changes)
```

**Handling Version Changes:**
```swift
struct AppConfig {
    static let apiVersion = "v1"
    static let baseURL = "https://api.uber.com/\(apiVersion)"
}

// When v2 is released:
// 1. Release new app version pointing to v2
// 2. Keep v1 running for old app versions
// 3. After 6-12 months, show "Update Required" for v1 users
```

**Additive Changes (No Version Bump):**
```swift
// v1 original
struct TripDTO: Decodable {
    let tripId: String
    let fare: MoneyDTO
}

// v1 with new optional field (backward compatible)
struct TripDTO: Decodable {
    let tripId: String
    let fare: MoneyDTO
    let carbonOffset: Double?        // NEW - old apps ignore
    let sustainabilityBadge: String? // NEW - old apps ignore
}
```

---

### 6. How iOS App Consumes and Evolves

**Initial Implementation (v1.0):**
```swift
class TripHistoryViewModel: ObservableObject {
    @Published var trips: [Trip] = []
    @Published var isLoading = false
    @Published var error: Error?
    
    private let repository: TripHistoryRepository
    
    func loadTrips() async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            trips = try await repository.loadInitial()
        } catch {
            self.error = error
        }
    }
    
    func loadMore() async {
        guard !isLoading else { return }
        isLoading = true
        defer { isLoading = false }
        
        do {
            trips = try await repository.loadNextPage()
        } catch {
            // Silently fail on "load more" errors
        }
    }
}
```

**Evolution (v1.5) - Adding Carbon Offset:**
```swift
// Only DTO and UI change - repository unchanged!
struct TripDTO: Decodable {
    // ... existing fields
    let carbonOffset: Double?  // NEW
}

struct Trip {
    // ... existing properties
    let carbonOffset: Double?  // NEW
    
    var carbonOffsetLabel: String? {
        guard let offset = carbonOffset else { return nil }
        return "\(offset) kg CO₂ offset"
    }
}

// UI now shows carbonOffsetLabel if available
```

**Evolution (v2.0) - Multi-Stop Trips:**
```swift
// v2 introduces multi-stop trips - breaking change
// New app version required

struct TripDTO: Decodable {
    let tripId: String
    let stops: [StopDTO]  // NEW - replaces pickup/dropoff
    let fare: MoneyDTO
}

struct StopDTO: Decodable {
    let location: LocationDTO
    let type: String  // "pickup", "dropoff", "stop"
    let arrivedAt: String?
}
```

---

## Case Study 2: Instagram-Style Feed with Infinite Scroll

### Screen Requirements

```
┌─────────────────────────────────────┐
│ 🏠 Home                       🔔 💬 │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │ 👤 john_doe • 2h ago            │ │
│ │ ┌─────────────────────────────┐ │ │
│ │ │                             │ │ │
│ │ │      [Photo/Video]          │ │ │
│ │ │                             │ │ │
│ │ └─────────────────────────────┘ │ │
│ │ ❤️ 1,234  💬 56  📤           │ │
│ │ Caption text goes here...     │ │
│ └─────────────────────────────────┘ │
│ ┌─────────────────────────────────┐ │
│ │ 👤 jane_smith • 5h ago          │ │
│ │ [Next Post...]                  │ │
│ └─────────────────────────────────┘ │
│                                      │
│       ↓ Loading more...              │
└─────────────────────────────────────┘
```

---

### 1. Endpoint Design

```yaml
Base URL: https://api.instagram.com/v1

Endpoints:
  # Main feed (algorithm-sorted)
  GET /feed
    Query Parameters:
      - cursor: string (opaque cursor for pagination)
      - limit: int (default: 10, max: 25)
    
  # Single post details
  GET /posts/{postId}
  
  # Like/Unlike a post
  POST /posts/{postId}/like
  DELETE /posts/{postId}/like
  
  # Comments on a post
  GET /posts/{postId}/comments
    Query Parameters:
      - cursor: string
      - limit: int (default: 20)
  
  # Add comment
  POST /posts/{postId}/comments
    Body: { text: string, replyToCommentId?: string }
```

---

### 2. Data Models and Relationships

```swift
// Core Entities
struct User {
    let id: String
    let username: String
    let displayName: String
    let profileImageUrl: String
    let isVerified: Bool
}

struct Post {
    let id: String
    let authorId: String           // FK → User
    let media: [MediaItem]
    let caption: String?
    let location: Location?
    let createdAt: Date
    let stats: PostStats
    let viewerContext: ViewerContext  // Current user's relationship
}

struct MediaItem {
    let id: String
    let type: MediaType  // image, video
    let url: String
    let thumbnailUrl: String?
    let dimensions: Dimensions
    let duration: TimeInterval?  // For videos
}

struct PostStats {
    let likeCount: Int
    let commentCount: Int
    let shareCount: Int
}

struct ViewerContext {
    let hasLiked: Bool
    let hasSaved: Bool
    let hasShared: Bool
}

// Relationships:
// User (1) ←→ (N) Post
// Post (1) ←→ (N) MediaItem  
// Post (1) ←→ (N) Comment
// User (N) ←→ (N) Post (via Likes)
```

---

### 3. Request / Response DTOs

**Response: Feed**
```swift
struct FeedResponse: Decodable {
    let posts: [FeedPostDTO]
    let pagination: CursorPaginationDTO
    let injectedContent: InjectedContentDTO?  // Ads, suggestions
}

struct FeedPostDTO: Decodable {
    let postId: String
    let author: AuthorDTO
    let media: [MediaDTO]
    let caption: String?
    let locationName: String?
    let createdAt: String
    let stats: StatsDTO
    let viewerContext: ViewerContextDTO
    
    // For carousel posts
    let currentMediaIndex: Int?
}

struct AuthorDTO: Decodable {
    let userId: String
    let username: String
    let displayName: String
    let profileImageUrl: String
    let isVerified: Bool
    let isFollowing: Bool  // Current user follows this author?
}

struct MediaDTO: Decodable {
    let mediaId: String
    let type: String  // "image", "video"
    let url: String
    let thumbnailUrl: String?
    let width: Int
    let height: Int
    let durationMs: Int?  // Video duration
    let hasAudio: Bool?   // Video has sound?
}

struct StatsDTO: Decodable {
    let likeCount: Int
    let commentCount: Int
    
    // Formatted strings for display
    let likeCountFormatted: String   // "1,234" or "1.2M"
    let commentCountFormatted: String
}

struct ViewerContextDTO: Decodable {
    let hasLiked: Bool
    let hasSaved: Bool
    let canComment: Bool  // Comments may be disabled
    let canShare: Bool
}

struct CursorPaginationDTO: Decodable {
    let nextCursor: String?
    let hasMore: Bool
}

struct InjectedContentDTO: Decodable {
    let ads: [AdDTO]?
    let suggestions: [SuggestionDTO]?
    let insertPositions: [String: Int]  // "ad_1": 3 means insert ad after position 3
}
```

---

### 4. Feed-Specific Design Considerations

**Why Cursor-Based is Essential for Feeds:**

```
User opens app:
  Feed = [Post A, Post B, Post C, Post D...]
  
User scrolls down to Post D, pauses...

Meanwhile, new posts are published:
  [New Post X, New Post Y, Post A, Post B, Post C, Post D...]

User continues scrolling:
  With OFFSET: Would re-fetch Post D! (offset shifted by 2)
  With CURSOR: Continues from exactly after Post D ✅
```

**Handling Feed Refresh:**

```swift
class FeedRepository {
    private var cursor: String? = nil
    private var posts: [Post] = []
    
    // Pull-to-refresh: Start fresh
    func refresh() async throws -> [Post] {
        cursor = nil
        posts = []
        return try await loadMore()
    }
    
    // Infinite scroll: Continue from cursor
    func loadMore() async throws -> [Post] {
        var urlString = "https://api.instagram.com/v1/feed?limit=10"
        if let cursor = cursor {
            urlString += "&cursor=\(cursor)"
        }
        
        let response = try await apiClient.get(FeedResponse.self, from: URL(string: urlString)!)
        
        // Update cursor
        cursor = response.pagination.nextCursor
        
        // Convert and append
        let newPosts = response.posts.map { Post(from: $0) }
        posts.append(contentsOf: newPosts)
        
        return posts
    }
    
    // Check for new posts since last refresh
    func checkForNewPosts() async throws -> Int {
        // Fetch just the count without loading full posts
        let response = try await apiClient.get(
            NewPostsCheckResponse.self,
            from: URL(string: "https://api.instagram.com/v1/feed/new-count")!
        )
        return response.newPostCount
    }
}
```

---

### 5. Like/Unlike with Optimistic Updates

**API Design for Likes:**

```
POST /posts/{postId}/like    → Like the post
DELETE /posts/{postId}/like  → Unlike the post

Response:
{
  "success": true,
  "likeCount": 1235,
  "hasLiked": true
}
```

**iOS Optimistic Update Pattern:**

```swift
class PostViewModel: ObservableObject {
    @Published var post: Post
    private let apiClient: APIClient
    
    func toggleLike() {
        // 1. Optimistic update (instant UI feedback)
        let previousState = post.hasLiked
        let previousCount = post.likeCount
        
        post.hasLiked.toggle()
        post.likeCount += post.hasLiked ? 1 : -1
        
        // 2. Make API call
        Task {
            do {
                if post.hasLiked {
                    try await apiClient.post(to: "/posts/\(post.id)/like")
                } else {
                    try await apiClient.delete(from: "/posts/\(post.id)/like")
                }
            } catch {
                // 3. Rollback on failure
                await MainActor.run {
                    post.hasLiked = previousState
                    post.likeCount = previousCount
                }
            }
        }
    }
}
```

---

### 6. Versioning for Feed Evolution

**Feed v1 → v2 Evolution (Reels Integration):**

```swift
// v1 Response
struct FeedPostDTO_v1: Decodable {
    let postId: String
    let imageUrl: String
    let caption: String?
}

// v2 Response (Backward Compatible)
struct FeedPostDTO_v2: Decodable {
    let postId: String
    let imageUrl: String          // Kept for v1 clients
    let caption: String?
    
    // New fields (optional for v1 clients)
    let media: [MediaDTO]?        // Multiple media support
    let contentType: String?      // "post", "reel", "story_highlight"
    let reelMetadata: ReelDTO?    // Only for reels
}

// v1 clients: Read imageUrl, ignore new fields
// v2 clients: Use media array, contentType for rich experience
```

---

## Case Study 3: Food Delivery Orders History (DoorDash Style)

### Screen Requirements

```
┌─────────────────────────────────────┐
│ ← Orders                            │
├─────────────────────────────────────┤
│ [Active] [Past]                     │
├─────────────────────────────────────┤
│ ┌─────────────────────────────────┐ │
│ │ 🍔 McDonald's                   │ │
│ │ Jan 14 • $24.50 • 3 items       │ │
│ │ ✅ Delivered                     │ │
│ │ [Reorder] [View Receipt]        │ │
│ └─────────────────────────────────┘ │
│ ┌─────────────────────────────────┐ │
│ │ 🍕 Domino's Pizza               │ │
│ │ Jan 12 • $32.00 • 2 items       │ │
│ │ ✅ Delivered                     │ │
│ │ [Reorder] [View Receipt]        │ │
│ └─────────────────────────────────┘ │
│                                      │
│         [Load More...]               │
└─────────────────────────────────────┘
```

---

### 1. Endpoint Design

```yaml
Base URL: https://api.doordash.com/v1

Endpoints:
  # List orders
  GET /users/{userId}/orders
    Query Parameters:
      - status: active | past | all
      - cursor: string
      - limit: int (default: 15)
    
  # Single order details
  GET /orders/{orderId}
  
  # Reorder (create new order from past order)
  POST /orders/{orderId}/reorder
    Body: { deliveryAddress?: AddressDTO, paymentMethodId?: string }
  
  # Active order tracking
  GET /orders/{orderId}/tracking
    Returns: real-time delivery status, driver location, ETA
  
  # Rate order
  POST /orders/{orderId}/rating
    Body: { 
      overallRating: 1-5,
      foodRating: 1-5?,
      deliveryRating: 1-5?,
      comment: string?,
      tip: MoneyDTO?
    }
  
  # Report issue
  POST /orders/{orderId}/issues
    Body: { type: string, description: string, itemIds?: [string] }
```

---

### 2. Data Models and Relationships

```swift
// Core Entities
struct Order {
    let id: String
    let userId: String              // FK → User
    let restaurantId: String        // FK → Restaurant
    let items: [OrderItem]
    let deliveryAddress: Address
    let status: OrderStatus
    let subtotal: Money
    let fees: OrderFees
    let total: Money
    let tip: Money?
    let estimatedDeliveryTime: Date?
    let actualDeliveryTime: Date?
    let createdAt: Date
    let driver: Driver?
}

struct OrderItem {
    let id: String
    let menuItemId: String
    let name: String
    let quantity: Int
    let price: Money
    let customizations: [Customization]
    let specialInstructions: String?
}

struct Restaurant {
    let id: String
    let name: String
    let logoUrl: String
    let cuisineType: String
    let rating: Double
    let deliveryTime: TimeRange
}

struct OrderFees {
    let deliveryFee: Money
    let serviceFee: Money
    let smallOrderFee: Money?
    let taxes: Money
}

enum OrderStatus: String, Codable {
    case placed
    case confirmed
    case preparing
    case readyForPickup
    case driverAssigned
    case pickedUp
    case arriving
    case delivered
    case cancelled
    case unknown
}

// Relationships:
// User (1) ←→ (N) Order
// Restaurant (1) ←→ (N) Order
// Order (1) ←→ (N) OrderItem
// Order (1) ←→ (0..1) Driver (assigned during delivery)
```

---

### 3. Request / Response DTOs

**Response: Orders List**
```swift
struct OrdersListResponse: Decodable {
    let orders: [OrderSummaryDTO]
    let pagination: CursorPaginationDTO
}

struct OrderSummaryDTO: Decodable {
    let orderId: String
    let restaurant: RestaurantSummaryDTO
    let itemCount: Int
    let itemsSummary: String  // "Big Mac, Fries, and 1 more"
    let total: MoneyDTO
    let status: String
    let statusDisplayText: String  // "Delivered" - localized
    let createdAt: String
    let deliveredAt: String?
    
    // Actions available for this order
    let actions: [OrderActionDTO]
}

struct RestaurantSummaryDTO: Decodable {
    let restaurantId: String
    let name: String
    let logoUrl: String
    let cuisineType: String
}

struct OrderActionDTO: Decodable {
    let type: String        // "reorder", "view_receipt", "rate", "report_issue"
    let label: String       // "Reorder", "View Receipt"
    let available: Bool
    let unavailableReason: String?  // "Restaurant is closed"
}

struct MoneyDTO: Decodable {
    let amount: Double
    let currencyCode: String
    let formatted: String  // "$24.50"
}

struct CursorPaginationDTO: Decodable {
    let nextCursor: String?
    let hasMore: Bool
}
```

**Response: Order Details**
```swift
struct OrderDetailsResponse: Decodable {
    let orderId: String
    let restaurant: RestaurantDetailDTO
    let items: [OrderItemDTO]
    let deliveryAddress: AddressDTO
    let status: String
    let statusHistory: [StatusEventDTO]  // Timeline of status changes
    
    // Pricing breakdown
    let pricing: PricingBreakdownDTO
    
    // Delivery info
    let driver: DriverDTO?
    let estimatedDeliveryAt: String?
    let actualDeliveryAt: String?
    
    // Actions
    let actions: [OrderActionDTO]
    
    // Rating (if already rated)
    let rating: OrderRatingDTO?
}

struct OrderItemDTO: Decodable {
    let itemId: String
    let name: String
    let quantity: Int
    let unitPrice: MoneyDTO
    let totalPrice: MoneyDTO
    let customizations: [String]  // ["No onions", "Extra cheese"]
    let specialInstructions: String?
    let imageUrl: String?
}

struct PricingBreakdownDTO: Decodable {
    let subtotal: MoneyDTO
    let deliveryFee: MoneyDTO
    let serviceFee: MoneyDTO
    let taxes: MoneyDTO
    let tip: MoneyDTO?
    let discount: MoneyDTO?
    let discountDescription: String?  // "20% off first order"
    let total: MoneyDTO
}

struct StatusEventDTO: Decodable {
    let status: String
    let timestamp: String
    let description: String  // "Your order was picked up by John"
}
```

---

### 4. Pagination Approach

**Cursor-Based with Status Filtering:**

```swift
class OrdersRepository {
    private var cursors: [OrderFilter: String?] = [:]
    private var hasMore: [OrderFilter: Bool] = [:]
    
    enum OrderFilter: String {
        case active = "active"
        case past = "past"
    }
    
    func loadOrders(filter: OrderFilter) async throws -> [Order] {
        var urlComponents = URLComponents(string: "https://api.doordash.com/v1/users/me/orders")!
        var queryItems = [
            URLQueryItem(name: "status", value: filter.rawValue),
            URLQueryItem(name: "limit", value: "15")
        ]
        
        if let cursor = cursors[filter], let cursorValue = cursor {
            queryItems.append(URLQueryItem(name: "cursor", value: cursorValue))
        }
        urlComponents.queryItems = queryItems
        
        let response = try await apiClient.get(OrdersListResponse.self, from: urlComponents.url!)
        
        // Update pagination state for this filter
        cursors[filter] = response.pagination.nextCursor
        hasMore[filter] = response.pagination.hasMore
        
        return response.orders.map { Order(from: $0) }
    }
    
    func hasMoreOrders(for filter: OrderFilter) -> Bool {
        return hasMore[filter] ?? true
    }
    
    func refresh(filter: OrderFilter) {
        cursors[filter] = nil
        hasMore[filter] = true
    }
}
```

---

### 5. Reorder Flow API Design

**Challenge:** Items or prices may have changed since the original order.

**API Design:**

```yaml
POST /orders/{orderId}/reorder

Response (Success):
{
  "cart": {
    "cartId": "cart_789",
    "items": [...],
    "total": { ... },
    "warnings": []  // Empty = no issues
  },
  "requiresReview": false
}

Response (Items Changed):
{
  "cart": {
    "cartId": "cart_789",
    "items": [...],
    "total": { ... },
    "warnings": [
      {
        "type": "ITEM_UNAVAILABLE",
        "itemName": "McFlurry",
        "message": "McFlurry is no longer available"
      },
      {
        "type": "PRICE_CHANGED",
        "itemName": "Big Mac",
        "oldPrice": "$5.99",
        "newPrice": "$6.49"
      }
    ]
  },
  "requiresReview": true
}
```

**iOS Implementation:**

```swift
func reorder(orderId: String) async throws -> ReorderResult {
    let response = try await apiClient.post(
        ReorderResponse.self,
        to: "/orders/\(orderId)/reorder"
    )
    
    if response.requiresReview {
        // Show review screen with warnings
        return .needsReview(cart: response.cart, warnings: response.cart.warnings)
    } else {
        // Go directly to checkout
        return .readyToCheckout(cart: response.cart)
    }
}

enum ReorderResult {
    case readyToCheckout(cart: Cart)
    case needsReview(cart: Cart, warnings: [ReorderWarning])
    case restaurantClosed
    case error(Error)
}
```

---

### 6. Active Order Tracking (Real-Time Updates)

**Polling vs WebSocket Decision:**

| Approach | Pros | Cons |
|----------|------|------|
| Polling | Simple, works everywhere | Battery drain, stale data |
| WebSocket | Real-time, efficient | Complex, connection management |
| Server-Sent Events | Real-time, simpler than WS | Limited browser support |

**Hybrid Approach (What DoorDash Uses):**

```swift
class OrderTrackingManager {
    private var webSocket: URLSessionWebSocketTask?
    private var pollingTimer: Timer?
    
    func startTracking(orderId: String) {
        // 1. Try WebSocket first
        connectWebSocket(orderId: orderId)
        
        // 2. Fallback to polling if WebSocket fails
        startPollingFallback(orderId: orderId)
    }
    
    private func connectWebSocket(orderId: String) {
        let url = URL(string: "wss://tracking.doordash.com/orders/\(orderId)")!
        webSocket = URLSession.shared.webSocketTask(with: url)
        webSocket?.resume()
        
        receiveWebSocketMessage()
    }
    
    private func receiveWebSocketMessage() {
        webSocket?.receive { [weak self] result in
            switch result {
            case .success(let message):
                self?.handleTrackingUpdate(message)
                self?.receiveWebSocketMessage()  // Continue listening
            case .failure:
                self?.fallbackToPolling()
            }
        }
    }
    
    private func startPollingFallback(orderId: String) {
        // Poll every 30 seconds as backup
        pollingTimer = Timer.scheduledTimer(withTimeInterval: 30, repeats: true) { [weak self] _ in
            Task {
                try? await self?.fetchTrackingUpdate(orderId: orderId)
            }
        }
    }
}
```

---

### 7. Versioning Strategy

**Food Delivery API Evolution:**

```swift
// v1: Basic order info
struct OrderDTO_v1: Decodable {
    let orderId: String
    let restaurantName: String
    let total: Double
    let status: String
}

// v2: Rich order data (additive changes)
struct OrderDTO_v2: Decodable {
    let orderId: String
    let restaurantName: String
    let total: Double
    let status: String
    
    // New optional fields (v1 clients ignore these)
    let restaurant: RestaurantDTO?       // Full restaurant info
    let pricing: PricingBreakdownDTO?    // Detailed pricing
    let sustainability: SustainabilityDTO?  // Carbon footprint
    let groupOrder: GroupOrderDTO?       // Group order info
}

// v3: Breaking change - different status structure
// Requires new endpoint version: /v3/orders
struct OrderDTO_v3: Decodable {
    let orderId: String
    let restaurant: RestaurantDTO
    let statusV2: StatusV2DTO  // New status format with ETA
}
```

---

# Interview Preparation Section

## Common API & Data Contract Interview Questions

### Question 1: Design an API for Uber's Trip History

**Expected Answer Structure:**

```
1. Clarify Requirements (1-2 min)
   - What data should be shown?
   - How far back should trips go?
   - What actions can users take?

2. High-Level Endpoint Design (2-3 min)
   - List trips: GET /users/{userId}/trips
   - Trip details: GET /trips/{tripId}
   - Rate trip: POST /trips/{tripId}/rating

3. Data Models (3-4 min)
   - Trip, User, Driver, Location, Money
   - Relationships between entities

4. Pagination Strategy (2-3 min)
   - Cursor-based for scalability
   - Why not offset-based

5. Error Handling (1-2 min)
   - Status codes
   - Retry strategies

6. Versioning (1-2 min)
   - URL-based for major changes
   - Additive fields for minor changes
```

---

### Question 2: How would you handle API backward compatibility?

**Model Answer:**

> "For mobile apps, backward compatibility is critical because we can't force users to update. I follow these principles:
>
> **First**, I design APIs with optional fields from the start. Any new field is nullable, so old apps simply ignore it.
>
> **Second**, I never remove or rename fields. If a field needs to change, I add a new one and deprecate the old one gradually.
>
> **Third**, for major breaking changes, I use URL versioning like `/v1/` to `/v2/`. Old app versions keep calling v1, while new app versions use v2. After 6-12 months, I show 'update required' for users on very old versions.
>
> **Fourth**, I use feature flags to progressively roll out new capabilities. The server can return different data based on what the client supports."

---

### Question 3: Offset vs Cursor pagination - when to use each?

**Model Answer:**

> "Both have their place, but for most mobile apps, especially ones with real-time data like social feeds or ride history, I prefer cursor-based pagination.
>
> **Offset-based** is simple and works well when:
> - Data is relatively static
> - Dataset is small
> - You need to show 'Page X of Y'
> - Users need to jump to specific pages
>
> **Cursor-based** is better when:
> - Data changes frequently (new items added)
> - Dataset is large (millions of records)
> - Infinite scroll UX is used
> - You can't afford duplicates
>
> **The key issue with offset** is data shifting. If I load page 1, then a new item is added, my page 2 request will include the last item from page 1 again - a duplicate. Cursor-based avoids this by saying 'give me items after this specific item' rather than 'skip N items.'"

---

### Question 4: How do you handle API errors in a mobile app?

**Model Answer:**

> "I categorize errors into three buckets with different handling strategies:
>
> **Client errors (400-499):**
> - 400/422: Show validation errors from the response body
> - 401: Clear tokens, redirect to login
> - 404: Show 'not found' message
> - 429: Implement exponential backoff
>
> **Server errors (500-599):**
> - Show generic error message
> - Implement automatic retry with backoff
> - For critical operations, queue offline and retry later
>
> **Network errors:**
> - No connection: Queue action for when connectivity returns
> - Timeout: Retry with longer timeout
> - SSL errors: Fail securely, don't downgrade
>
> **Implementation details:**
> - Standard error response format with code, message, retryable flag
> - User-facing messages separate from technical error codes
> - Idempotency keys for critical operations like payments to prevent double-charging on retry"

---

### Question 5: How would you design an API for offline-first mobile apps?

**Model Answer:**

> "For offline-first, the API needs to support synchronization patterns:
>
> **Delta sync endpoint:**
> ```
> GET /sync?since=2024-01-15T10:00:00Z
> ```
> Returns only records changed since the last sync.
>
> **Conflict resolution:**
> Each record has a `version` or `updatedAt` field. When the client syncs changes:
> - If client version < server version, server wins
> - If client version = server version, client update applies
> - For complex cases, return both versions and let user choose
>
> **Idempotency:**
> All write operations include client-generated IDs so the same change can be retried safely.
>
> **Batch operations:**
> Instead of individual API calls for each change, sync supports batch:
> ```
> POST /sync
> {
>   \"create\": [...],
>   \"update\": [...],
>   \"delete\": [...]
> }
> ```
>
> **Last sync tracking:**
> Client stores the timestamp of last successful sync and sends it on each sync request."

---

## How to Explain Trade-offs and Decisions

### The STAR-T Framework for API Design Answers

| Letter | Meaning | Example |
|--------|---------|---------|
| **S** | Situation | "For a feed with constantly changing data..." |
| **T** | Task | "...we need pagination that handles new items without duplicates" |
| **A** | Approach | "I'd use cursor-based pagination because..." |
| **R** | Result | "This ensures no duplicates and O(1) performance at scale" |
| **T** | Trade-offs | "The trade-off is we can't jump to specific pages" |

### Example Answer Using STAR-T:

**Question:** How would you paginate Uber's trip history?

**Answer:**
> "**Situation:** Uber's trip history can have thousands of trips, and new trips are constantly added.
>
> **Task:** We need efficient pagination that doesn't show duplicates when new trips appear during scrolling.
>
> **Approach:** I'd use cursor-based pagination. The cursor would be the last trip's ID and timestamp. Each request asks for 'trips older than this cursor.'
>
> **Result:** This gives O(1) performance regardless of page depth, and no duplicates even when new trips are added.
>
> **Trade-off:** We can't show 'Page 3 of 50' or let users jump to page 50. But for a mobile app with infinite scroll, that's acceptable. If we needed page jumping, we could use offset-based with a 'refresh for latest' feature to handle duplicates."

---

## Common Mistakes That Cause Candidates to Fail

### Mistake 1: Not Thinking Mobile-First

❌ **Bad:** "The API returns all 10,000 trips at once with full user objects embedded"

✅ **Good:** "The API paginates with 20 trips per page, includes only essential fields, and uses IDs for related entities with an option to expand"

---

### Mistake 2: Ignoring Backward Compatibility

❌ **Bad:** "When we add the carbon offset feature, we'll change the fare from a number to an object"

✅ **Good:** "We'll add `carbonOffset` as a new optional field. Old apps will ignore it. The `fare` field stays the same for backward compatibility."

---

### Mistake 3: Wrong Pagination Choice

❌ **Bad:** "For the Twitter feed, I'd use offset pagination because it's simpler"

✅ **Good:** "For a feed with constantly changing data, cursor-based pagination prevents duplicates when new items appear"

---

### Mistake 4: Forgetting Error Handling

❌ **Bad:** "The app calls the API and displays the result"

✅ **Good:** "The app handles network errors with retry logic, auth errors by redirecting to login, and shows user-friendly messages for validation errors"

---

### Mistake 5: Over-Engineering

❌ **Bad:** "We'll use GraphQL with subscriptions, event sourcing, and CQRS for the trip history API"

✅ **Good:** "For a read-heavy, relatively simple list, REST with cursor pagination is simpler and sufficient. We'd consider GraphQL if we had many different screens needing different combinations of data."

---

### Mistake 6: Not Discussing Versioning

❌ **Bad:** "We'll just update the API when needed"

✅ **Good:** "We'll use URL versioning for major changes and additive optional fields for minor changes, ensuring old app versions continue working"

---

## Quick Reference: API Design Checklist

```markdown
□ Endpoint Design
  □ RESTful resource naming
  □ Appropriate HTTP methods
  □ Query parameters for filtering

□ Data Contracts
  □ Clear request/response DTOs
  □ Optional fields for flexibility
  □ Unknown enum handling

□ Pagination
  □ Chose appropriate strategy
  □ Included hasMore flag
  □ Reasonable page size

□ Error Handling
  □ Standard error format
  □ Appropriate status codes
  □ Retry guidance

□ Versioning
  □ Versioning strategy chosen
  □ Backward compatibility plan
  □ Deprecation process

□ Performance
  □ Payload size optimized
  □ N+1 queries avoided
  □ Caching headers

□ Reliability
  □ Idempotent operations
  □ Retry safety
  □ Offline handling
```

---

## Final Interview Tips

1. **Always start with clarifying questions** - Shows you think about requirements before coding

2. **Think out loud** - Explain your reasoning as you design

3. **Consider the full lifecycle** - Design, implementation, versioning, deprecation

4. **Mention trade-offs** - Every design decision has pros and cons

5. **Use concrete examples** - "Like how Uber's feed..." makes answers memorable

6. **Draw diagrams** - Visual representations help communication

7. **Discuss what you'd monitor** - Error rates, latency, payload sizes

8. **Acknowledge alternatives** - "Another approach would be... but I chose this because..."

---

*Good luck with your iOS Mobile System Design interviews! 🍀*
