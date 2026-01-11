# iOS Local Data Storage & Persistence Guide

Local data storage on iOS means saving information directly on the user's device so the app can access it later—even without an internet connection. This includes user preferences, cached API responses, offline content, and structured app data.

---

## Beginner Level

### What local data storage means on iOS (simple English)
When you use an app, it often needs to remember things: your login status, your favorite settings, the last search you made, or items in your cart. Instead of fetching everything from the internet each time, apps store this data locally on your iPhone. This makes the app faster, works offline, and provides a smoother experience.

### When and why mobile apps need local databases
- **Offline access:** A food delivery app caches your recent orders so you can view them without internet.
- **Performance:** Loading cached data is instant; network requests take time.
- **User preferences:** Saving theme settings, notification preferences, or language choices.
- **Reducing server load:** Caching API responses reduces repeated calls.
- **Data persistence:** Chat apps store messages locally so they're available immediately on app launch.

### Overview of iOS storage options

| Storage Option | Best For | Data Type | Complexity |
|----------------|----------|-----------|------------|
| **UserDefaults** | Small settings, flags | Key-value (primitives, small objects) | Very simple |
| **File System** | Documents, images, JSON files | Files, binary data | Simple |
| **SQLite** | Complex queries, large datasets | Relational tables | Moderate |
| **Core Data** | Complex object graphs, relationships | Managed objects | Moderate-High |
| **Realm** | Fast reads/writes, reactive UI | Objects with live updates | Moderate |
| **Keychain** | Secrets, passwords, tokens | Encrypted key-value | Simple (via wrappers) |

### Real-life analogies for each storage option

**UserDefaults – Sticky notes on your fridge:**
Quick reminders like "dark mode = ON" or "last opened tab = 3". Fast to write and read, but not for storing your entire grocery list.

**File System – Filing cabinet:**
Store documents, images, or JSON files in folders. Good for larger files but you manage the organization yourself.

**SQLite – Spreadsheet with formulas:**
Rows and columns of data with powerful queries. Great when you need to search, filter, or join data (e.g., finding all orders from last week).

**Core Data – Smart filing system with relationships:**
Not just rows—objects with connections. A `Customer` has many `Orders`, each `Order` has many `Items`. Core Data manages these relationships and memory efficiently.

**Realm – Live dashboard:**
Data updates automatically reflect in your UI. When a new message arrives, the chat list updates without manual refresh.

**Keychain – Bank vault:**
Encrypted storage for sensitive data like passwords and API tokens. The system protects it even if the device is compromised.

### Basic examples

**Saving user settings with UserDefaults:**
```swift
// Save
UserDefaults.standard.set(true, forKey: "isDarkModeEnabled")
UserDefaults.standard.set("en", forKey: "preferredLanguage")

// Read
let isDarkMode = UserDefaults.standard.bool(forKey: "isDarkModeEnabled")
let language = UserDefaults.standard.string(forKey: "preferredLanguage") ?? "en"
```

**Caching API response to file:**
```swift
func cacheResponse(_ data: Data, filename: String) throws {
    let cacheDir = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask).first!
    let fileURL = cacheDir.appendingPathComponent(filename)
    try data.write(to: fileURL)
}

func loadCachedResponse(filename: String) -> Data? {
    let cacheDir = FileManager.default.urls(for: .cachesDirectory, in: .userDomainMask).first!
    let fileURL = cacheDir.appendingPathComponent(filename)
    return try? Data(contentsOf: fileURL)
}
```

---

## Intermediate Level (SDE-2 Ready)

### Deep dive into Core Data

**Core Data stack components:**
```
┌─────────────────────────────────────────┐
│     NSManagedObjectContext              │  ← In-memory scratchpad
│     (viewContext / backgroundContext)   │
├─────────────────────────────────────────┤
│     NSPersistentStoreCoordinator        │  ← Mediates between context & store
├─────────────────────────────────────────┤
│     NSPersistentStore (SQLite file)     │  ← Actual storage on disk
└─────────────────────────────────────────┘
```

**Entities and relationships:**
```swift
// Entity: Order
// Attributes: id (UUID), totalAmount (Double), createdAt (Date)
// Relationships: customer (to-one), items (to-many)

// Entity: OrderItem
// Attributes: id (UUID), name (String), price (Double), quantity (Int)
// Relationships: order (to-one, inverse of Order.items)
```

**Creating the Core Data stack (modern approach):**
```swift
class PersistenceController {
    static let shared = PersistenceController()
    
    let container: NSPersistentContainer
    
    var viewContext: NSManagedObjectContext {
        container.viewContext
    }
    
    init() {
        container = NSPersistentContainer(name: "AppModel")
        container.loadPersistentStores { description, error in
            if let error = error {
                fatalError("Core Data failed to load: \(error)")
            }
        }
        container.viewContext.automaticallyMergesChangesFromParent = true
    }
    
    func newBackgroundContext() -> NSManagedObjectContext {
        container.newBackgroundContext()
    }
    
    func save() {
        let context = viewContext
        if context.hasChanges {
            try? context.save()
        }
    }
}
```

**Threading model:**
- `viewContext`: Main thread only—use for UI reads.
- `backgroundContext`: Background thread—use for writes, imports, heavy fetches.
- Never pass `NSManagedObject` across threads; pass `objectID` instead.

```swift
// Safe background write
func importOrders(_ ordersData: [OrderDTO]) {
    let context = PersistenceController.shared.newBackgroundContext()
    context.perform {
        for dto in ordersData {
            let order = Order(context: context)
            order.id = dto.id
            order.totalAmount = dto.total
            order.createdAt = dto.createdAt
        }
        try? context.save()
    }
}
```

### Comparing storage options

| Criteria | UserDefaults | SQLite | Core Data | Realm |
|----------|--------------|--------|-----------|-------|
| **Data size** | Small (< 1 MB) | Large | Large | Large |
| **Queries** | Key lookup only | Full SQL | NSPredicate/Fetch | Type-safe queries |
| **Relationships** | None | Manual joins | Automatic | Automatic |
| **Threading** | Thread-safe | Manual | Context-based | Thread-confined |
| **Learning curve** | Very low | Medium | Medium-High | Medium |
| **Performance** | Fast for small data | Very fast | Good | Very fast |
| **Migrations** | N/A | Manual | Automatic (lightweight) | Automatic |
| **Reactive/Live** | No | No | With Combine | Yes (built-in) |

**When to use each:**

- **UserDefaults:** App settings, feature flags, simple preferences, onboarding completion.
- **SQLite (raw or via GRDB/FMDB):** Maximum control, complex joins, full-text search, existing SQL expertise.
- **Core Data:** Apple ecosystem integration, iCloud sync, complex object graphs, SwiftUI `@FetchRequest`.
- **Realm:** Cross-platform (Android/iOS sharing), real-time UI updates, simpler API than Core Data.

### Choosing storage based on requirements

**Food delivery app example:**

| Data | Storage Choice | Reason |
|------|----------------|--------|
| User preferences (theme, notifications) | UserDefaults | Small, simple key-value |
| Cached menu items | Core Data / Realm | Structured data, offline browsing |
| Order history | Core Data / Realm | Relationships (Order → Items → Restaurant) |
| Authentication token | Keychain | Security-sensitive |
| Cached restaurant images | File System (Caches) | Binary data, can be re-downloaded |

### Designing a clean local database schema

**Principles:**
1. **Normalize where sensible:** Avoid duplicating data (e.g., restaurant info repeated in every order).
2. **Denormalize for performance:** Sometimes duplicating data (e.g., restaurant name in Order) avoids expensive joins.
3. **Use UUIDs as primary keys:** Consistency with server, conflict-free sync.
4. **Add timestamps:** `createdAt`, `updatedAt` for sync and debugging.
5. **Add sync status:** `syncStatus` enum (`pending`, `synced`, `failed`) for offline-first apps.

**Example schema (ride-hailing app):**
```
User
├── id: UUID
├── name: String
├── email: String
└── createdAt: Date

Trip
├── id: UUID
├── userId: UUID (relationship to User)
├── pickupAddress: String
├── dropoffAddress: String
├── fare: Double
├── status: TripStatus (enum)
├── startedAt: Date?
├── completedAt: Date?
├── syncStatus: SyncStatus
└── createdAt: Date

Driver
├── id: UUID
├── name: String
├── vehicleNumber: String
├── rating: Double
└── photoURL: String?

TripDriver (join or relationship)
├── tripId: UUID
└── driverId: UUID
```

### Data access layer (Repository pattern)

Separate database access from UI and networking:

```swift
// Protocol for abstraction
protocol TripRepository {
    func fetchRecentTrips(limit: Int) async throws -> [Trip]
    func saveTrip(_ trip: Trip) async throws
    func updateTripStatus(_ tripId: UUID, status: TripStatus) async throws
    func deleteTripsPendingSync() async throws
}

// Core Data implementation
class CoreDataTripRepository: TripRepository {
    private let context: NSManagedObjectContext
    
    init(context: NSManagedObjectContext) {
        self.context = context
    }
    
    func fetchRecentTrips(limit: Int) async throws -> [Trip] {
        try await context.perform {
            let request = TripEntity.fetchRequest()
            request.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
            request.fetchLimit = limit
            let entities = try self.context.fetch(request)
            return entities.map { Trip(entity: $0) }
        }
    }
    
    func saveTrip(_ trip: Trip) async throws {
        try await context.perform {
            let entity = TripEntity(context: self.context)
            entity.populate(from: trip)
            try self.context.save()
        }
    }
    
    // ... other methods
}
```

**Benefits:**
- Swap Core Data for Realm without changing UI code.
- Easy to mock for unit tests.
- Clear separation of concerns.

---

## Advanced Level (Big Tech Interviews)

### Data migration strategies

**Types of migrations:**

1. **Lightweight migration (Core Data):**
   - Handled automatically for simple changes (add attribute, rename, add relationship).
   - Enable with `shouldMigrateStoreAutomatically` and `shouldInferMappingModelAutomatically`.

2. **Heavyweight migration:**
   - Required for complex changes (splitting entities, transforming data).
   - Define custom `NSMappingModel` with migration policies.

3. **Manual migration (SQLite/Realm):**
   - Write SQL `ALTER TABLE` statements or Realm migration blocks.

**Best practices:**
- **Version your schema:** Increment model version with each release.
- **Backward compatibility:** Old app versions should gracefully handle (or reject) newer data.
- **Test migrations:** Create unit tests that migrate from version N to N+1.
- **Staged rollout:** Test migrations in beta before wide release.

**Core Data lightweight migration setup:**
```swift
let description = NSPersistentStoreDescription()
description.shouldMigrateStoreAutomatically = true
description.shouldInferMappingModelAutomatically = true
container.persistentStoreDescriptions = [description]
```

**Realm migration example:**
```swift
let config = Realm.Configuration(
    schemaVersion: 3,
    migrationBlock: { migration, oldSchemaVersion in
        if oldSchemaVersion < 2 {
            migration.enumerateObjects(ofType: Order.className()) { oldObject, newObject in
                // Transform data
                newObject!["totalAmount"] = (oldObject!["amount"] as! Double) + (oldObject!["tax"] as! Double)
            }
        }
        if oldSchemaVersion < 3 {
            // Add new property with default value
            migration.enumerateObjects(ofType: Order.className()) { _, newObject in
                newObject!["currency"] = "USD"
            }
        }
    }
)
Realm.Configuration.defaultConfiguration = config
```

### Handling large datasets and performance optimization

**Pagination / Batch fetching:**
```swift
// Core Data: Fetch in batches
let request = OrderEntity.fetchRequest()
request.fetchBatchSize = 50  // Load 50 objects at a time
request.fetchLimit = 200     // Maximum 200 results
```

**Faulting:**
Core Data loads objects as "faults" (placeholders) until accessed—memory efficient for large datasets.

**Indexing:**
Add indexes to frequently queried attributes in the Core Data model editor or via `isIndexed`.

**Background fetching:**
```swift
// Perform heavy fetch on background context
let backgroundContext = persistenceController.newBackgroundContext()
backgroundContext.perform {
    let request = OrderEntity.fetchRequest()
    request.predicate = NSPredicate(format: "createdAt > %@", lastSyncDate as NSDate)
    let results = try? backgroundContext.fetch(request)
    // Process results
}
```

**Lazy loading for relationships:**
Don't prefetch all related objects; let Core Data fault them on access.

**Denormalization for read performance:**
Store computed values (e.g., `orderTotal`) instead of calculating every time.

### Thread safety and concurrency

**Core Data threading rules:**
1. Never share `NSManagedObject` across threads—use `objectID`.
2. Use `perform { }` or `performAndWait { }` for context operations.
3. Merge changes from background to main context.

```swift
// Safe cross-thread access
func updateOrderOnBackground(orderId: NSManagedObjectID, newStatus: OrderStatus) {
    let backgroundContext = persistenceController.newBackgroundContext()
    backgroundContext.perform {
        guard let order = try? backgroundContext.existingObject(with: orderId) as? OrderEntity else { return }
        order.status = newStatus.rawValue
        try? backgroundContext.save()
    }
}
```

**Realm threading:**
Realm objects are thread-confined. Pass primary keys or use `ThreadSafeReference` for cross-thread access.

```swift
// Realm cross-thread
let orderRef = ThreadSafeReference(to: order)

DispatchQueue.global().async {
    let realm = try! Realm()
    guard let order = realm.resolve(orderRef) else { return }
    try! realm.write {
        order.status = "completed"
    }
}
```

**SQLite threading:**
Use separate connections per thread or serialize access with a dedicated queue.

### Encryption and security for local data at rest

**iOS Data Protection:**
Files inherit protection levels based on app entitlements:
- `NSFileProtectionComplete`: Data inaccessible when device locked.
- `NSFileProtectionCompleteUntilFirstUserAuthentication`: Data accessible after first unlock.

```swift
// Set file protection
let attributes: [FileAttributeKey: Any] = [
    .protectionKey: FileProtectionType.complete
]
try FileManager.default.setAttributes(attributes, ofItemAtPath: filePath)
```

**Core Data encryption:**
- Core Data uses SQLite, which is unencrypted by default.
- Use **SQLCipher** (via encrypted SQLite) for full database encryption.
- Alternatively, encrypt sensitive fields before storage.

**Realm encryption:**
```swift
var key = Data(count: 64)
_ = key.withUnsafeMutableBytes { SecRandomCopyBytes(kSecRandomDefault, 64, $0.baseAddress!) }

let config = Realm.Configuration(encryptionKey: key)
let realm = try! Realm(configuration: config)
```

**Store encryption key in Keychain, not UserDefaults!**

### Secure storage of secrets

**Keychain for tokens and credentials:**
```swift
import Security

func saveToKeychain(key: String, data: Data) -> Bool {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecValueData as String: data,
        kSecAttrAccessible as String: kSecAttrAccessibleAfterFirstUnlock
    ]
    SecItemDelete(query as CFDictionary) // Remove existing
    return SecItemAdd(query as CFDictionary, nil) == errSecSuccess
}

func loadFromKeychain(key: String) -> Data? {
    let query: [String: Any] = [
        kSecClass as String: kSecClassGenericPassword,
        kSecAttrAccount as String: key,
        kSecReturnData as String: true,
        kSecMatchLimit as String: kSecMatchLimitOne
    ]
    var result: AnyObject?
    SecItemCopyMatching(query as CFDictionary, &result)
    return result as? Data
}
```

**Never store in UserDefaults:**
- API keys, tokens, passwords → Keychain
- User preferences, non-sensitive settings → UserDefaults

### How big tech iOS apps manage local databases

**Uber:**
- Uses Core Data or a custom SQLite layer for trip history, driver info, payment methods.
- Encrypted storage for payment tokens (Keychain).
- Aggressive cache eviction to limit app size.
- Background sync with conflict resolution.

**WhatsApp:**
- SQLite (custom, possibly SQLCipher-encrypted) for messages.
- Millions of messages per user with efficient indexing.
- Attachment thumbnails in file system, full media on-demand.
- End-to-end encryption keys in Keychain.

**Amazon:**
- Product catalog cached in local database for offline browsing.
- Order history synced incrementally.
- Cart persisted locally, synced with server on conflict.
- Images cached with TTL-based eviction.

**Google Maps:**
- Offline map tiles stored in file system (Documents or Application Support).
- User-managed storage (download/remove regions).
- Location history and preferences in local database.
- Encryption for sensitive location data.

---

## Interview Preparation

### Real mobile system design questions (local storage focused)

1. "Design local caching for a food delivery app that works offline."
2. "How would you store and sync a user's order history across devices?"
3. "Design a chat app's local message storage with efficient search."
4. "How would you handle database migrations when your app schema changes?"
5. "Design secure storage for user credentials and payment tokens on iOS."

### Interview-ready answer structure

**1. Clarify requirements (ask these questions):**
- What data needs to be stored locally? (structure, size, sensitivity)
- Does it need to work offline? (read-only or read-write?)
- How often does data change? (real-time updates vs. periodic sync)
- Any security requirements? (encryption, PII handling)
- What's the expected dataset size? (100 items vs. millions)

**2. Choose storage solution and justify:**
```
"For order history in a food delivery app, I'd use Core Data because:
- Structured data with relationships (Order → Items → Restaurant)
- Moderate dataset size (hundreds to thousands of orders)
- Apple ecosystem integration (CloudKit sync potential)
- SwiftUI @FetchRequest for reactive UI

If we needed cross-platform code sharing with Android, I'd consider Realm instead."
```

**3. Design the schema:**
```
Order
├── id: UUID (primary key, synced from server)
├── userId: UUID
├── restaurantId: UUID → Restaurant
├── items: [OrderItem] (to-many relationship)
├── totalAmount: Double
├── status: OrderStatus (enum)
├── createdAt: Date
├── syncStatus: SyncStatus
└── updatedAt: Date
```

**4. Data access layer:**
```
"I'd use the Repository pattern to abstract database access:
- TripRepository protocol with methods like fetch, save, delete
- CoreDataTripRepository implementation
- Easy to mock for tests, swap implementations later"
```

**5. Handle threading:**
```
"Core Data contexts are not thread-safe. I'd use:
- viewContext on main thread for UI reads
- backgroundContext for imports and heavy writes
- Merge changes using automaticallyMergesChangesFromParent"
```

**6. Security considerations:**
```
"Sensitive data like auth tokens go in Keychain, not local database.
For the database itself:
- Enable iOS Data Protection (NSFileProtectionComplete)
- Consider SQLCipher for full encryption if handling PII
- Never log or cache sensitive fields in plain text"
```

**7. Trade-offs to discuss:**
- **Core Data vs. Realm:** Apple-native vs. cross-platform; complexity vs. simplicity.
- **Normalization vs. denormalization:** Storage efficiency vs. query speed.
- **Encryption:** Security vs. performance overhead.
- **Cache size:** More local data = better offline experience but larger app footprint.

### Common follow-up questions and answers

**Q: "How do you handle schema changes in production?"**
**A:** Use versioned migrations. Core Data supports lightweight migrations for simple changes (add/remove attributes). For complex changes, define custom mapping models. Always test migrations with real production data before release. Consider staged rollouts to catch issues early.

**Q: "What if the database gets corrupted?"**
**A:** Wrap database access in error handling. If Core Data fails to load the store, the fallback strategy depends on data criticality:
- For cache: Delete and recreate (data can be re-fetched).
- For user-generated content: Attempt recovery, prompt user, or sync from server.
Log corruption events for monitoring.

**Q: "How do you test database code?"**
**A:** Use in-memory stores for unit tests (faster, isolated). Create test fixtures with known data. Test migrations by loading old database versions and verifying successful upgrade. Use XCTest expectations for async operations.

**Q: "Core Data vs. SwiftData?"**
**A:** SwiftData (iOS 17+) is the modern, Swift-native wrapper around Core Data. Use SwiftData for new projects targeting iOS 17+; use Core Data for backward compatibility or existing projects. Both use the same underlying SQLite store.

**Q: "How do you optimize for low memory devices?"**
**A:**
- Use Core Data faulting (lazy loading).
- Fetch in batches (`fetchBatchSize`).
- Release objects not in use (`refreshAllObjects()`).
- Avoid fetching large blobs; store on file system.
- Monitor memory with Instruments.

**Q: "How do you secure local data if the device is jailbroken?"**
**A:** Jailbroken devices bypass many protections, but you can still:
- Encrypt the database with SQLCipher.
- Store encryption keys in Keychain with `kSecAttrAccessibleWhenUnlockedThisDeviceOnly`.
- Implement jailbreak detection (though not foolproof).
- Avoid storing highly sensitive data locally if possible.

---

## Quick Reference Cheat Sheet

| Data Type | Recommended Storage | Example |
|-----------|---------------------|---------|
| User preferences | UserDefaults | Dark mode, language |
| Auth tokens | Keychain | OAuth access/refresh tokens |
| Cached API responses | File System (Caches) | JSON responses |
| Structured app data | Core Data / Realm | Orders, messages, products |
| Large files | File System (Documents) | Downloaded PDFs, videos |
| Temporary data | File System (tmp) | Upload staging files |
| Encrypted database | SQLCipher / Realm encrypted | Financial data, health records |

---

## Resources & References

- [Apple Core Data Documentation](https://developer.apple.com/documentation/coredata)
- [Apple Keychain Services](https://developer.apple.com/documentation/security/keychain_services)
- [Realm Swift Documentation](https://realm.io/docs/swift/latest/)
- [SQLite with Swift (GRDB)](https://github.com/groue/GRDB.swift)
- [iOS Data Protection](https://developer.apple.com/documentation/uikit/protecting_the_user_s_privacy/encrypting_your_app_s_files)
- [SwiftData (iOS 17+)](https://developer.apple.com/documentation/swiftdata)
