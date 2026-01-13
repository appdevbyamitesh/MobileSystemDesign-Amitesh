P# Offline-First Architecture - Complete iOS System Design

> [!NOTE]
> This is a 60-minute Uber L4 interview simulation focused on designing an offline-first mobile app architecture.

**Problem Statement:** *Design an iOS app architecture that works seamlessly offline and syncs data when online. Example: A note-taking app like Evernote or a task manager like Todoist.*

---

## 🎯 SCALED Framework Coverage

> [!IMPORTANT]
> This design follows the **[SCALED framework](./00_SCALED_Framework_Guide.md)** - demonstrating advanced mobile engineering for offline-first apps.

| SCALED | Section in This File | What You'll Learn |
|--------|----------------------|-------------------|
| **S** - System Requirements | [0-10 min: Requirements](#0-10-min-requirements-clarification) | Offline CRUD, sync requirements, conflict scenarios |
| **C** - Design Considerations | [10-25 min: HLD](#10-25-min-high-level-design-hld) | Offline-first vs online-first, sync engine design |
| **A** - Architecture | [10-25 min: HLD](#10-25-min-high-level-design-hld) | Repository + Sync Engine architecture, CoreData layer |
| **L** - Low-Level Design | [25-45 min: LLD](#25-45-min-low-level-design-lld) | CoreData models, Sync queue, Conflict resolver |
| **E** - Evaluating NFRs | [45-55 min: Deep Dives](#45-55-min-deep-dives) | Reliability (network failures), consistency (conflicts) |
| **D** - API Design | [Part 4: Sync API Design](#part-4-sync-api-design-offline-first) | Delta sync, batch operations, idempotency, versioning |
| **T** - Trade-offs | Throughout + [Deep Dives](#45-55-min-deep-dives) | Last-write-wins vs CRDTs, optimistic vs pessimistic locking |

> [!CAUTION]
> Offline-first is considered an advanced L4/L5 topic. Master this to stand out!

---

## 0-10 min: Requirements Clarification

### Questions to Ask Interviewer

**You:** "Let me understand the requirements. First, what type of data are we working with?
- Is it user-generated content (notes, tasks)?
- Read-only content (articles, product catalog)?
- Collaborative data (shared documents)?"

**Interviewer:** "Focus on user-generated task management. Users create, update, delete tasks. No collaboration for now."

**You:** "Great. For offline behavior:
- Should all CRUD operations work offline?
- When we come online, how do we handle conflicts?
- What's the acceptable sync latency?
- Do we need to support multiple devices?"

**Interviewer:** "All CRUD offline. Last-write-wins for conflicts. Sync within 5 seconds of connectivity. Yes, multiple devices."

**You:** "For scale and constraints:
- How many tasks per user typically?
- What's the data retention policy?
- Should we optimize for battery or data?
- Any platform constraints (iOS version)?"

**Interviewer:** "Typical user has 100-500 tasks. Keep all data unless user deletes. Balance battery and data. iOS 15+."

### Documented Scope

```
✅ IN SCOPE:
- Full CRUD on tasks (create, read, update, delete) offline
- Automatic background sync when online
- Multi-device support with last-write-wins
- Local persistence with SQLite/CoreData
- Conflict resolution strategy
- Sync queue with retry mechanism

❌ OUT OF SCOPE:
- Real-time collaboration (operational transforms)
- Attachment handling (images, files)
- Complex conflict UIs (manual resolution)
- User authentication flow
```

**Non-Functional Requirements:**
- Sync latency: < 5s when online
- Offline data: All user tasks
- Conflict strategy: Last-write-wins (timestamp-based)
- Storage: Local database + cloud backend

---

## 10-25 min: High-Level Design (HLD)

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────┐
│                    UI Layer                             │
│   - TaskListView, TaskDetailView                       │
│   - Reactive updates                                    │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│              Presentation Layer                         │
│   - TaskListViewModel                                   │
│   - @Published properties (Combine)                     │
└──────────────────────┬──────────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────────┐
│                 Domain Layer                            │
│   ┌─────────────────────────────────────┐               │
│   │  TaskRepository (Single Source)     │               │
│   │  - Always returns from local DB     │               │
│   │  - Triggers sync in background      │               │
│   └────┬────────────────────────┬───────┘               │
└────────┼────────────────────────┼───────────────────────┘
         │                        │
    ┌────▼──────┐          ┌──────▼──────┐
    │ Local DB  │          │  SyncEngine │
    │ (CoreData)│          │             │
    │           │          │ - SyncQueue │
    │ - Tasks   │          │ - Conflict  │
    │ - Metadata│◄─────────┤   Resolver  │
    └───────────┘          │ - Retry     │
                           └──────┬──────┘
                                  │
                           ┌──────▼──────┐
                           │   Network   │
                           │ - REST API  │
                           │ - Reachable │
                           └─────────────┘
```

### Key Architectural Decisions

**You:** "I'm choosing an **offline-first with sync engine pattern**. Here's why:

**1. Repository as Single Source of Truth:**
- UI always reads from local database
- Even fresh data from server goes through local DB first
- Guarantees UI works offline

**2. Separate Sync Engine:**
- Decoupled from UI and repository
- Runs in background using `TaskScheduler` / Background Tasks
- Manages sync queue independently

**3. Why CoreData over SQLite:**
- ✅ Built-in change tracking (NSPersistentHistoryTracking)
- ✅ Merge policies for conflicts
- ✅ Background contexts for threading
- ❌ More complex than raw SQLite
- **Alternative:** Could use Realm for simpler syntax

**4. Sync Queue Pattern:**
- Local changes go to persistent queue
- Queue processes when online
- Retry with exponential backoff on failure"

### Data Flow - Create Task (Offline)

```
User taps "Add Task"
    │
    ▼
ViewModel.createTask(title: "...")
    │
    ▼
Repository.createTask(task)
    │
    ├─→ Save to Local DB immediately
    │   └─ Task(id: UUID, syncStatus: .pending, updatedAt: Date())
    │
    ├─→ Update UI immediately (optimistic update)
    │
    └─→ Enqueue sync operation
        └─ SyncQueue.enqueue(operation: .create(task))
        
(Later, when online)
        
SyncEngine detects connectivity
    │
    ▼
Process SyncQueue
    │
    ▼
POST /tasks { task JSON }
    │
    ├─→ Success:
    │   └─ Update local task: syncStatus = .synced
    │       serverId = response.id
    │
    └─→ Failure:
        └─ Retry with backoff (3 attempts)
```

### Data Flow - Update Task (Multi-Device Conflict)

```
Device A: User updates task title (offline)
    │
    ▼
LocalDB: Task(id: "123", title: "New", updatedAt: T1, syncStatus: .pending)
    │
    ▼
SyncQueue: enqueue(.update(id: "123", ...))

---

Device B: User updates same task title (online)
    │
    ▼
LocalDB: Task(id: "123", title: "Different", updatedAt: T2)
    │
    ▼
Syncs immediately to server
    │
    ▼
Server: Task(id: "123", title: "Different", updatedAt: T2)

---

Device A comes online later
    │
    ▼
SyncEngine processes queue
    │
    ▼
1. Fetch all tasks from server (detect changes)
    │
    ▼
2. Server returns: Task(id: "123", title: "Different", updatedAt: T2)
    │
    ▼
3. Conflict Detected!
   Local:  updatedAt = T1, title = "New"
   Server: updatedAt = T2, title = "Different"
    │
    ▼
4. ConflictResolver.resolve(local, remote)
    │
    ├─→ If T2 > T1: Use server version (last-write-wins)
    │   └─ Overwrite local, discard pending change
    │
    └─→ If T1 > T2: Push local version
        └─ POST update to server
```

---

## 25-45 min: Low-Level Design (LLD)

### Data Models

```swift
// Domain Model
struct Task: Identifiable, Codable {
    let id: UUID
    var title: String
    var isCompleted: Bool
    var createdAt: Date
    var updatedAt: Date
    
    // Sync metadata
    var syncStatus: SyncStatus
    var serverId: String? // Backend ID after sync
}

enum SyncStatus: String, Codable {
    case synced       // No pending changes
    case pending      // Local change not synced
    case syncing      // Currently uploading
    case failed       // Sync failed, will retry
}

// Sync operation for queue
struct SyncOperation: Codable {
    let id: UUID
    let type: OperationType
    let taskId: UUID
    let taskData: Task?
    let createdAt: Date
    var retryCount: Int
    
    enum OperationType: String, Codable {
        case create, update, delete
    }
}
```

### CoreData Entities

```swift
// TaskEntity (CoreData)
@objc(TaskEntity)
public class TaskEntity: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var title: String
    @NSManaged public var isCompleted: Bool
    @NSManaged public var createdAt: Date
    @NSManaged public var updatedAt: Date
    @NSManaged public var syncStatus: String
    @NSManaged public var serverId: String?
}

// SyncOperationEntity (Persistent queue)
@objc(SyncOperationEntity)
public class SyncOperationEntity: NSManagedObject {
    @NSManaged public var id: UUID
    @NSManaged public var type: String
    @NSManaged public var taskId: UUID
    @NSManaged public var taskJSON: Data?
    @NSManaged public var createdAt: Date
    @NSManaged public var retryCount: Int16
}
```

### Repository Implementation

```swift
protocol TaskRepositoryProtocol {
    func getAllTasks() -> AnyPublisher<[Task], Never>
    func createTask(_ task: Task) async throws
    func updateTask(_ task: Task) async throws
    func deleteTask(id: UUID) async throws
}

class TaskRepository: TaskRepositoryProtocol {
    private let localDataSource: LocalDataSource
    private let syncEngine: SyncEngine
    
    init(localDataSource: LocalDataSource, syncEngine: SyncEngine) {
        self.localDataSource = localDataSource
        self.syncEngine = syncEngine
    }
    
    // ALWAYS return local data for offline-first
    func getAllTasks() -> AnyPublisher<[Task], Never> {
        localDataSource.observeTasks()
            .map { entities in entities.map { $0.toDomain() } }
            .eraseToAnyPublisher()
    }
    
    func createTask(_ task: Task) async throws {
        var newTask = task
        newTask.syncStatus = .pending
        newTask.updatedAt = Date()
        
        // 1. Save locally first (optimistic update)
        try await localDataSource.save(newTask)
        
        // 2. Enqueue sync operation
        let operation = SyncOperation(
            id: UUID(),
            type: .create,
            taskId: newTask.id,
            taskData: newTask,
            createdAt: Date(),
            retryCount: 0
        )
        try await syncEngine.enqueue(operation)
        
        // 3. Trigger sync if online
        syncEngine.triggerSync()
    }
    
    func updateTask(_ task: Task) async throws {
        var updated = task
        updated.syncStatus = .pending
        updated.updatedAt = Date()
        
        try await localDataSource.update(updated)
        
        let operation = SyncOperation(
            id: UUID(),
            type: .update,
            taskId: task.id,
            taskData: updated,
            createdAt: Date(),
            retryCount: 0
        )
        try await syncEngine.enqueue(operation)
        syncEngine.triggerSync()
    }
    
    func deleteTask(id: UUID) async throws {
        // Soft delete: mark for deletion
        var task = try await localDataSource.getTask(id: id)
        task.syncStatus = .pending
        
        try await localDataSource.delete(id: id)
        
        let operation = SyncOperation(
            id: UUID(),
            type: .delete,
            taskId: id,
            taskData: nil,
            createdAt: Date(),
            retryCount: 0
        )
        try await syncEngine.enqueue(operation)
        syncEngine.triggerSync()
    }
}
```

### Local Data Source (CoreData)

```swift
actor LocalDataSource {
    private let container: NSPersistentContainer
    private let context: NSManagedObjectContext
    
    init() {
        container = NSPersistentContainer(name: "TaskModel")
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("CoreData failed: \(error)")
            }
        }
        
        context = container.newBackgroundContext()
        context.automaticallyMergesChangesFromParent = true
        context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
    }
    
    func save(_ task: Task) throws {
        let entity = TaskEntity(context: context)
        entity.id = task.id
        entity.title = task.title
        entity.isCompleted = task.isCompleted
        entity.createdAt = task.createdAt
        entity.updatedAt = task.updatedAt
        entity.syncStatus = task.syncStatus.rawValue
        entity.serverId = task.serverId
        
        try context.save()
    }
    
    func update(_ task: Task) throws {
        let fetchRequest = TaskEntity.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "id == %@", task.id as CVarArg)
        
        guard let entity = try context.fetch(fetchRequest).first else {
            throw DataError.notFound
        }
        
        entity.title = task.title
        entity.isCompleted = task.isCompleted
        entity.updatedAt = task.updatedAt
        entity.syncStatus = task.syncStatus.rawValue
        
        try context.save()
    }
    
    func observeTasks() -> AnyPublisher<[TaskEntity], Never> {
        let fetchRequest = TaskEntity.fetchRequest()
        fetchRequest.sortDescriptors = [NSSortDescriptor(key: "createdAt", ascending: false)]
        
        // Use NSFetchedResultsController for reactive updates
        let controller = NSFetchedResultsController(
            fetchRequest: fetchRequest,
            managedObjectContext: context,
            sectionNameKeyPath: nil,
            cacheName: nil
        )
        
        return controller.publisher
            .map { $0.fetchedObjects ?? [] }
            .eraseToAnyPublisher()
    }
}
```

### Sync Engine

```swift
actor SyncEngine {
    private let networkService: NetworkServiceProtocol
    private let localDataSource: LocalDataSource
    private let reachability: NetworkReachability
    
    private var isSyncing = false
    private let maxRetries = 3
    
    init(networkService: NetworkServiceProtocol,
         localDataSource: LocalDataSource,
         reachability: NetworkReachability) {
        self.networkService = networkService
        self.localDataSource = localDataSource
        self.reachability = reachability
        
        // Observe connectivity changes
        Task {
            for await isOnline in reachability.statusStream {
                if isOnline {
                    await self.sync()
                }
            }
        }
    }
    
    func enqueue(_ operation: SyncOperation) async throws {
        // Save to persistent queue (CoreData SyncOperationEntity)
        try await localDataSource.saveSyncOperation(operation)
    }
    
    func triggerSync() {
        Task {
            await sync()
        }
    }
    
    private func sync() async {
        guard !isSyncing, reachability.isConnected else { return }
        isSyncing = true
        defer { isSyncing = false }
        
        do {
            // Step 1: Fetch server changes first (for conflict detection)
            let serverTasks = try await networkService.fetchTasks()
            try await resolveConflicts(serverTasks: serverTasks)
            
            // Step 2: Push local changes
            let pendingOps = try await localDataSource.getPendingSyncOperations()
            
            for operation in pendingOps {
                try await processSyncOperation(operation)
            }
            
        } catch {
            print("Sync failed: \(error)")
            // Will retry on next connectivity change
        }
    }
    
    private func processSyncOperation(_ operation: SyncOperation) async throws {
        do {
            switch operation.type {
            case .create:
                guard let task = operation.taskData else { return }
                let serverTask = try await networkService.createTask(task)
                
                // Update local with server ID
                var localTask = task
                localTask.serverId = serverTask.id
                localTask.syncStatus = .synced
                try await localDataSource.update(localTask)
                
            case .update:
                guard let task = operation.taskData else { return }
                try await networkService.updateTask(task)
                
                var localTask = task
                localTask.syncStatus = .synced
                try await localDataSource.update(localTask)
                
            case .delete:
                try await networkService.deleteTask(id: operation.taskId)
            }
            
            // Remove from sync queue
            try await localDataSource.deleteSyncOperation(id: operation.id)
            
        } catch {
            // Retry logic
            if operation.retryCount < maxRetries {
                var retryOp = operation
                retryOp.retryCount += 1
                try await localDataSource.updateSyncOperation(retryOp)
            } else {
                // Mark as failed, notify user
                var failedTask = operation.taskData
                failedTask?.syncStatus = .failed
                if let failedTask = failedTask {
                    try await localDataSource.update(failedTask)
                }
            }
            throw error
        }
    }
    
    private func resolveConflicts(serverTasks: [Task]) async throws {
        let localTasks = try await localDataSource.getAllTasks()
        
        for serverTask in serverTasks {
            if let localTask = localTasks.first(where: { $0.serverId == serverTask.serverId }) {
                // Conflict detection
                if localTask.updatedAt != serverTask.updatedAt {
                    // Last-write-wins
                    if serverTask.updatedAt > localTask.updatedAt {
                        // Server wins, overwrite local
                        var updated = serverTask
                        updated.id = localTask.id // Keep local UUID
                        updated.syncStatus = .synced
                        try await localDataSource.update(updated)
                        
                        // Remove conflicting local change from queue
                        try await localDataSource.removePendingSyncForTask(id: localTask.id)
                    }
                    // If local.updatedAt > server.updatedAt, our pending change will sync next
                }
            } else {
                // New task from another device, insert locally
                var newTask = serverTask
                newTask.id = UUID() // Generate new local ID
                newTask.syncStatus = .synced
                try await localDataSource.save(newTask)
            }
        }
    }
}
```

### Network Service

```swift
protocol NetworkServiceProtocol {
    func fetchTasks() async throws -> [Task]
    func createTask(_ task: Task) async throws -> Task
    func updateTask(_ task: Task) async throws
    func deleteTask(id: UUID) async throws
}

class NetworkService: NetworkServiceProtocol {
    private let session: URLSession
    private let baseURL = "https://api.example.com/v1"
    
    func fetchTasks() async throws -> [Task] {
        let url = URL(string: "\(baseURL)/tasks")!
        let (data, _) = try await session.data(from: url)
        return try JSONDecoder().decode([Task].self, from: data)
    }
    
    func createTask(_ task: Task) async throws -> Task {
        var request = URLRequest(url: URL(string: "\(baseURL)/tasks")!)
        request.httpMethod = "POST"
        request.httpBody = try JSONEncoder().encode(task)
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        let (data, _) = try await session.data(for: request)
        return try JSONDecoder().decode(Task.self, from: data)
    }
    
    func updateTask(_ task: Task) async throws {
        guard let serverId = task.serverId else {
            throw NetworkError.missingServerId
        }
        
        var request = URLRequest(url: URL(string: "\(baseURL)/tasks/\(serverId)")!)
        request.httpMethod = "PUT"
        request.httpBody = try JSONEncoder().encode(task)
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")
        
        _ = try await session.data(for: request)
    }
    
    func deleteTask(id: UUID) async throws {
        // Requires server ID lookup first
        // Simplified for brevity
    }
}
```

### Network Reachability

```swift
class NetworkReachability {
    private let monitor = NWPathMonitor()
    private let queue = DispatchQueue(label: "com.app.reachability")
    
    var isConnected: Bool {
        monitor.currentPath.status == .satisfied
    }
    
    var statusStream: AsyncStream<Bool> {
        AsyncStream { continuation in
            monitor.pathUpdateHandler = { path in
                continuation.yield(path.status == .satisfied)
            }
            monitor.start(queue: queue)
        }
    }
}
```

---

## Part 4: Sync API Design (Offline-First)

### Task Management API Endpoints

**You:** "For offline-first architecture, I need to design sync-friendly APIs. Here's my approach:"

#### Core CRUD Endpoints

```
GET /v1/tasks
  Query Params:
    - since: ISO8601 timestamp (optional, for delta sync)
    - limit: Int (default: 100)
  
  Response: 200 OK
  {
    "data": {
      "tasks": [
        {
          "id": "task_abc123",
          "title": "Buy groceries",
          "isCompleted": false,
          "createdAt": "2026-01-13T10:00:00Z",
          "updatedAt": "2026-01-13T15:30:00Z",
          "version": 3
        }
      ],
      "syncMetadata": {
        "serverTimestamp": "2026-01-13T18:00:00Z",
        "hasMore": false
      }
    }
  }

POST /v1/tasks
  Request:
  {
    "title": "Buy groceries",
    "isCompleted": false,
    "clientId": "uuid-client-generated",  // For idempotency
    "clientTimestamp": "2026-01-13T15:30:00Z"
  }
  
  Response: 201 Created
  {
    "data": {
      "id": "task_abc123",  // Server-generated ID
      "title": "Buy groceries",
      "isCompleted": false,
      "createdAt": "2026-01-13T15:30:00Z",
      "updatedAt": "2026-01-13T15:30:00Z",
      "version": 1
    }
  }

PUT /v1/tasks/{id}
  Request:
  {
    "title": "Buy groceries and cook dinner",
    "isCompleted": false,
    "version": 1,  // For optimistic locking
    "clientTimestamp": "2026-01-13T16:00:00Z"
  }
  
  Response: 200 OK (updated)
  Response: 409 Conflict (version mismatch)
  {
    "error": {
      "code": "VERSION_CONFLICT",
      "message": "Task was modified by another client",
      "currentVersion": 2,
      "serverData": { /* latest task data */ }
    }
  }

DELETE /v1/tasks/{id}
  Response: 204 No Content
```

### Delta Sync API (Bandwidth Optimization)

**Interviewer:** "How do you minimize data transfer for sync?"

**You:** "I use delta sync with timestamps:

```
GET /v1/tasks?since=2026-01-13T10:00:00Z

// Only returns tasks modified after timestamp
Response:
{
  "data": {
    "tasks": [
      {"id": "task_abc", "updatedAt": "2026-01-13T11:00:00Z", ...}
    ],
    "deletedTaskIds": ["task_xyz"],  // Deleted since last sync
    "syncMetadata": {
      "serverTimestamp": "2026-01-13T18:00:00Z"
    }
  }
}
```

**Benefits:**
- ✅ Only transfers changed data (not all 500 tasks)
- ✅ Includes deletions (server knows what client should remove)
- ✅ Client stores server timestamp for next sync
- **Trade-off:** Server must track deletions (soft delete with tombstones)"

### Batch Sync API (Battery Optimization)

```
POST /v1/sync/batch
Request:
{
  "operations": [
    {"type": "create", "clientId": "uuid1", "data": {...}},
    {"type": "update", "id": "task_abc", "version": 1, "data": {...}},
    {"type": "delete", "id": "task_xyz"}
  ],
  "lastSyncTimestamp": "2026-01-13T10:00:00Z"
}

Response:
{
  "results": [
    {"clientId": "uuid1", "success": true, "serverId": "task_new"},
    {"id": "task_abc", "success": false, "error": {"code": "VERSION_CONFLICT"}},
    {"id": "task_xyz", "success": true}
  ],
  "deltaUpdates": {
    "tasks": [ /* tasks changed on server since lastSyncTimestamp */ ],
    "deletedTaskIds": ["task_old"]
  },
  "serverTimestamp": "2026-01-13T18:00:00Z"
}
```

**Why Batch API:**
- ✅ Single network call instead of N operations
- ✅ Saves battery (fewer radio wake-ups)
- ✅ Atomic - all-or-nothing transaction
- ✅ Returns delta updates in same response

### Conflict Resolution API Pattern

**Interviewer:** "How does the API handle conflicts?"

**You:** "Two approaches:

**1. Optimistic Locking (Version-based):**
```swift
// Client sends version number
PUT /v1/tasks/123
{
  "title": "Updated",
  "version": 1  // Client's current version
}

// Server response
409 Conflict {
  "error": {
    "code": "VERSION_CONFLICT",
    "message": "Version mismatch",
    "currentVersion": 2,
    "serverData": { /* latest task */ }
  }
}

// iOS client handling
if response.statusCode == 409 {
    let serverVersion = error.currentVersion
    let serverData = error.serverData
    
    // Resolve conflict
    let resolved = conflictResolver.resolve(local: localTask, remote: serverData)
    
    // Retry with new version
    try await updateTask(resolved, version: serverVersion)
}
```

**2. Timestamp-based (Last-Write-Wins):**
```
PUT /v1/tasks/123
{
  "title": "Updated",
  "updatedAt": "2026-01-13T15:00:00Z"
}

// Server compares timestamps
if request.updatedAt > server.updatedAt:
    accept update
else:
    return 409 Conflict with server data
```

**Trade-offs:**
- **Version-based:** More reliable, prevents lost updates
- **Timestamp-based:** Simpler, but clock sync issues
- **My choice:** Version-based for data integrity"

### Idempotency Strategy

```
POST /v1/tasks
Headers:
  Idempotency-Key: uuid-client-generated

// Server stores idempotency key
// If client retries with same key:
//   - Return 200 OK with original created resource
//   - Don't create duplicate

Response:
{
  "data": { /* task that was created */ },
  "meta": {
    "idempotent": true  // Indicates this was a retry
  }
}
```

**iOS Implementation:**
```swift
func createTask(_ task: Task) async throws -> Task {
    let idempotencyKey = UUID().uuidString
    var request = URLRequest(url: apiURL)
    request.setValue(idempotencyKey, forHTTPHeaderField: "Idempotency-Key")
    
    // If network fails and client retries, server won't create duplicate
    return try await performRequest(request)
}
```

### Sync State API

```
GET /v1/sync/status
Response:
{
  "lastSyncTimestamp": "2026-01-13T10:00:00Z",
  "pendingChanges": 5,
  "conflictsNeedingResolution": 2,
  "serverVersion": "2.1.0"
}
```

**Use case:** Show sync status in UI

### API Versioning for Sync

**Interviewer:** "How do you handle API changes for offline-first apps?"

**You:** "Very carefully! Offline apps can't force updates:

**Versioning Strategy:**
```swift
// Client sends supported API version
Headers:
  X-API-Version: 2
  X-Client-Version: 1.5.0

// Server responds with compatibility
{
  "data": {...},
  "meta": {
    "apiVersion": "2",
    "minSupportedClientVersion": "1.4.0",
    "deprecationNotices": [
      {
        "feature": "syncV1",
        "sunsetDate": "2026-06-01",
        "migrationGuide": "https://..."
      }
    ]
  }
}
```

**Backwards Compatibility Rules:**
1. Support N-1 API versions (current + previous)
2. Never remove fields, only deprecate
3. New required fields → make optional initially
4. Give 6-month notice for breaking changes

**iOS client handling:**
```swift
if meta.minSupportedClientVersion > currentAppVersion {
    showForceUpdateDialog()
} else if meta.deprecationNotices.isNotEmpty {
    logDeprecationWarning()
}
```"

### Error Responses for Sync

```json
// Network timeout during sync
{
  "error": {
    "code": "SYNC_INTERRUPTED",
    "message": "Sync was interrupted, safe to retry",
    "retryable": true,
    "lastSuccessfulOperation": 3  // Client can resume from operation 4
  }
}

// Server in maintenance mode
{
  "error": {
    "code": "SERVICE_UNAVAILABLE",
    "message": "Server maintenance in progress",
    "retryAfter": 300,  // 5 minutes
    "estimatedDuration": "15 minutes"
  }
}

// Account quota exceeded
{
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Free tier allows 100 tasks, you have 105",
    "currentUsage": 105,
    "limit": 100,
    "upgradeUrl": "https://..."
  }
}
```

### Authentication for Background Sync

```swift
// Long-lived refresh token for background sync
class BackgroundSyncAuth {
    func getBackgroundSyncToken() async throws -> String {
        // Use refresh token that doesn't expire
        let refreshToken = keychain.get("refresh_token")
        
        // Exchange for short-lived access token
        let accessToken = try await authAPI.refresh(refreshToken)
        
        return accessToken
    }
}

// API request with background token
var request = URLRequest(url: syncURL)
request.setValue("Bearer \(bgToken)", forHTTPHeaderField: "Authorization")
request.setValue("background", forHTTPHeaderField: "X-Sync-Context")  // Hint to server
```

### Mobile-Specific Optimizations

**1. Compression for Sync Payloads**

```
POST /v1/sync/batch
Headers:
  Content-Encoding: gzip
  Accept-Encoding: gzip

// Both request and response are compressed
// Typical compression: 70-80% size reduction for JSON
```

**2. Lightweight Endpoint for Status Check**

```
HEAD /v1/sync/status
Response Headers:
  X-Has-Updates: true
  X-Pending-Count: 5
  X-Last-Sync: 2026-01-13T10:00:00Z

// No body, just headers (saves bandwidth)
// Client can decide whether to full sync
```

### Interview Q&A: Sync API Design

**Q:** "Why separate endpoints for batch vs individual operations?"

**A:**
> "Flexibility for different scenarios:
> 
> **Individual endpoints** (`POST /tasks`, `PUT /tasks/{id}`):
> - Real-time updates when online
> - Simple, RESTful
> - Better for debugging
> 
> **Batch endpoint** (`POST /sync/batch`):
> - Background sync (queued operations)
> - Single network call (battery efficient)
> - Atomic transactions
> 
> Client chooses based on context:
> ```swift
> if isOnline && !isBackgroundSync {
>     // Individual call
>     try await api.createTask(task)
> } else {
>     // Queue for batch sync
>     syncQueue.enqueue(operation)
> }
> ```"

**Q:** "How do you prevent sync storms (all clients syncing at once)?"

**A:**
> "Several strategies:
> 
> **1. Jittered sync intervals:**
> ```swift
> let baseInterval = 300  // 5 minutes
> let jitter = Int.random(in: 0...60)  // 0-60 seconds
> let actualInterval = baseInterval + jitter
> ```
> 
> **2. Server-side rate limiting:**
> ```
> Response: 429 Too Many Requests
> Retry-After: 45  // Stagger retries
> ```
> 
> **3. Exponential backoff on conflict:**
> ```swift
> if conflict { retryAfter = 2^attemptCount }
> ```
> 
> **4. Server pushes sync schedule:**
> ```json
> {
>   \"suggestedSyncInterval\": 600,  // 10 min instead of 5
>   \"reason\": \"high_server_load\"
> }
> ```"

**Q:** "What if user makes changes on 3 devices while offline?"

**A:**
> "Multi-device conflict resolution:
> 
> **Scenario:** User edits task title on Phone, iPad, Mac (all offline)
> 
> **All come online simultaneously:**
> 1. Server receives 3 PUT requests
> 2. First one wins (version 1 → 2)
> 3. Others get 409 Conflict
> 4. Clients fetch server version
> 5. Last-write-wins based on `updatedAt` timestamp
> 6. Losing clients resubmit if their timestamp is newer
> 
> **Better approach:** Version vectors (complex but no data loss)
> ```json
> {
>   \"version\": {
>     \"phone\": 3,
>     \"ipad\": 2,
>     \"mac\": 1
>   }
> }
> ```
> 
> But for tasks, last-write-wins is acceptable."

---

## 45-55 min: Deep Dives

### 1. Conflict Resolution Strategies

**Interviewer:** "Why last-write-wins? What are the alternatives?"

**You:** "Last-write-wins is simplest for this scope. Here's the comparison:

| Strategy | How It Works | Pros | Cons |
|----------|-------------|------|------|
| **Last-Write-Wins** | Latest timestamp wins | Simple, automatic | Data loss possible |
| **Operational Transform** | Merge both changes | No data loss | Very complex, real-time needed |
| **CRDTs** | Commutative operations | Eventual consistency | Limited data types |
| **Manual Resolution** | Show UI to user | User decides | UX friction |

**For task app:** LWW is acceptable because:
- Tasks are simple entities (title, completed)
- Low collision probability (user rarely edits same task on 2 devices simultaneously)
- If loss occurs, it's visible (title change), user can re-enter

**If requirements changed to collaborative docs:** Would need OT or CRDTs (like Figma/Google Docs)"

### 2. Sync Queue Persistence

**Interviewer:** "What if app crashes during sync?"

**You:** "That's why sync queue is in CoreData, not memory:

```swift
// Persistent queue survives crashes
@NSManaged public var queuedOperations: NSOrderedSet // In CoreData

// On app restart
func application(_ application: UIApplication, didFinishLaunchingWithOptions...) {
    Task {
        await syncEngine.resumePendingSync()
    }
}
```

**Recovery scenarios:**

| Scenario | Handling |
|----------|----------|
| Crash mid-network call | URLSession retries automatically, or operation stays in queue |
| Database write fails | Transaction rollback, operation remains queued |
| App killed by iOS | Next launch processes queue |

**Trade-off:** CoreData overhead vs reliability (worth it for data integrity)"

### 3. Battery & Performance

**Interviewer:** "How do you optimize for battery with background sync?"

**You:** "Several strategies:

**1. Adaptive Sync Frequency:**
```swift
actor SyncEngine {
    private var syncInterval: TimeInterval {
        switch reachability.connectionType {
        case .wifi: return 5.0  // Frequent on WiFi
        case .cellular: return 30.0 // Conservative on cellular
        case .none: return .infinity
        }
    }
}
```

**2. Background Task Scheduling:**
```swift
// Use BGTaskScheduler for efficient background work
func scheduleSync() {
    let request = BGAppRefreshTaskRequest(identifier: "com.app.sync")
    request.earliestBeginDate = Date(timeIntervalSinceNow: 15 * 60) // 15 min
    
    try? BGTaskScheduler.shared.submit(request)
}
```

**3. Coalescing:**
- Batch multiple operations into single network call
- Debounce rapid changes (e.g., typing in title)

**4. Opportunistic Sync:**
- Piggyback on user-initiated actions (app foreground)
- Don't wake device just for sync

**Trade-offs:**
- ✅ 10x better battery life
- ❌ Sync delay up to 15 min (acceptable for tasks)
- **Critical apps (messaging):** Would need push notifications to wake app"

### 4. Testing Strategy

**You:** "Testability is key for offline-first:

**Unit Tests:**
```swift
func testConflictResolution() async throws {
    let localTask = Task(id: UUID(), title: "Local", updatedAt: Date())
    let serverTask = Task(id: UUID(), title: "Server", updatedAt: Date().addingTimeInterval(100))
    
    let resolver = ConflictResolver()
    let result = resolver.resolve(local: localTask, remote: serverTask)
    
    XCTAssertEqual(result.title, "Server") // Server wins (newer)
}

func testSyncQueueRetry() async throws {
    let mockNetwork = MockNetworkService()
    mockNetwork.shouldFail = true
    
    let syncEngine = SyncEngine(network: mockNetwork, ...)
    let operation = SyncOperation(type: .create, task: Task(...), retryCount: 0)
    
    try await syncEngine.processSyncOperation(operation)
    
    // Should increment retry count
    let updated = try await localDataSource.getSyncOperation(id: operation.id)
    XCTAssertEqual(updated.retryCount, 1)
}
```

**Integration Tests:**
```swift
// Simulate offline → online transition
func testOfflineToOnlineFlow() async throws {
    // 1. Go offline
    reachability.simulateOffline()
    
    // 2. Create task
    try await repository.createTask(Task(title: "Test"))
    
    // 3. Verify local save
    let localTasks = try await localDataSource.getAllTasks()
    XCTAssertEqual(localTasks.count, 1)
    XCTAssertEqual(localTasks[0].syncStatus, .pending)
    
    // 4. Go online
    reachability.simulateOnline()
    
    // 5. Wait for sync
    try await Task.sleep(nanoseconds: 2_000_000_000)
    
    // 6. Verify synced
    let synced = try await localDataSource.getAllTasks()
    XCTAssertEqual(synced[0].syncStatus, .synced)
    XCTAssertNotNil(synced[0].serverId)
}
```

**Manual Testing Scenarios:**
1. Airplane mode → create task → enable WiFi → verify sync
2. Two devices → edit same task → verify LWW
3. Force quit app mid-sync → relaunch → verify queue processed
4. Poor network (Network Link Conditioner) → verify retries"

### 5. Scalability

**Interviewer:** "What if user has 10,000 tasks?"

**You:** "Several optimizations:

**1. Pagination for Sync:**
```swift
func fetchTasks(since: Date) async throws -> [Task] {
    // Only fetch tasks modified since last sync
    let url = "\(baseURL)/tasks?updatedSince=\(since.iso8601)"
    // ...
}

// Track last sync timestamp
var lastSyncDate: Date = Date.distantPast

func sync() async {
    let tasks = try await networkService.fetchTasks(since: lastSyncDate)
    // Only process changed tasks
    lastSyncDate = Date()
}
```

**2. Database Indexing:**
```swift
// Add index on updatedAt for fast queries
entity.updatedAt.isIndexed = true
```

**3. Lazy Loading UI:**
```swift
// Use NSFetchedResultsController with batching
fetchRequest.fetchBatchSize = 20
```

**4. Differential Sync:**
- Server sends delta (changed tasks only)
- Use If-Modified-Since headers

**Memory Impact:**
- 10,000 tasks × ~200 bytes = ~2 MB (acceptable)
- If had attachments → remote references only, lazy load

**Trade-off:** Complexity vs scalability (only add if needed)"

---

## Common Uber L4 Follow-Up Questions

### Q1: "How would you handle schema migrations?"

**Answer:**
"CoreData has built-in lightweight migration:

```swift
let container = NSPersistentContainer(name: "TaskModel")
let description = container.persistentStoreDescriptions.first
description?.shouldMigrateStoreAutomatically = true
description?.shouldInferMappingModelAutomatically = true

// For complex migrations
if !isLightweightMigrationPossible {
    let migrationManager = NSMigrationManager(...)
    // Custom mapping model
}
```

**Migration strategy:**
1. Version models (TaskModel_v1, TaskModel_v2)
2. Test migration path before release
3. Keep migration code for 2-3 versions back

**Backwards compatibility:**
- API versioning (v1, v2)
- Client sends version header
- Server supports N-1 versions"

### Q2: "What about data privacy/encryption?"

**Answer:**
"Multiple layers:

**1. At-Rest Encryption:**
```swift
description?.setOption(FileProtectionType.complete as NSObject,
                       forKey: NSPersistentStoreFileProtectionKey)
```

**2. In-Transit:**
- TLS 1.3 for all API calls
- Certificate pinning

**3. Sensitive Fields:**
```swift
// Encrypt task title if contains sensitive data
extension Task {
    var encryptedTitle: String {
        CryptoKit.encrypt(title, with: deviceKey)
    }
}
```

**Trade-off:** Performance overhead vs security (justifiable for user data)"

### Q3: "How would you debug sync issues in production?"

**Answer:**
"Observability is key:

**1. Local Logging:**
```swift
os_log(.debug, log: .sync, "Syncing operation %{public}@", operation.id)

// In Settings app → show sync log
func exportSyncLog() -> String {
    OSLogStore.local().getEntries()
}
```

**2. Sync Status UI:**
```swift
struct SyncIndicatorView: View {
    @StateObject var syncMonitor: SyncMonitor
    
    var body: some View {
        HStack {
            if syncMonitor.isSyncing {
                ProgressView()
                Text("Syncing...")
            } else if syncMonitor.failedCount > 0 {
                Image(systemName: "exclamationmark.triangle")
                Text("\(syncMonitor.failedCount) items failed")
            }
        }
    }
}
```

**3. Remote Monitoring:**
- Send anonymous sync metrics (success/fail rate)
- Crash reports with sync queue state

**4. Debug Menu:**
```swift
#if DEBUG
Button("Force Sync") { syncEngine.triggerSync() }
Button("Simulate Conflict") { injectConflict() }
Button("Clear Queue") { clearSyncQueue() }
#endif
```"

---

## L4 Success Checklist

✅ Designed offline-first architecture (local DB as source of truth)
✅ Explained sync engine with persistent queue
✅ Showed conflict resolution with last-write-wins
✅ Used actors for thread safety
✅ Handled network failures with retry + exponential backoff
✅ Discussed battery optimization (adaptive sync, BGTaskScheduler)
✅ Showed testability (unit + integration tests)
✅ Covered scalability (delta sync, indexed queries)
✅ Addressed data persistence across crashes
✅ Compared trade-offs for each decision

---

## Interview Mistakes to Avoid

| Mistake | Why It Fails | Fix |
|---------|-------------|-----|
| Sync on every change | Battery drain, network spam | Debounce, batch operations |
| Ignoring conflicts | Data loss in production | Implement resolution strategy |
| No retry mechanism | Temp failures become permanent | Exponential backoff with max retries |
| Sync blocking UI | Poor UX | Background queue, optimistic updates |
| No offline indication | User confused why changes don't appear | Sync status UI |

> [!TIP]
> Practice explaining conflict resolution out loud. It's the #1 deep dive for offline-first designs.
