# Data Synchronization for Mobile System Design (iOS)

Data synchronization on mobile means keeping data consistent between the device's local storage and a remote server, so changes made on one device or the server eventually appear everywhere. Unlike a simple backup (one-way copy at a point in time), sync is bidirectional and continuous, handling changes from both client and server.[1][2]

## Beginner Level

### What data sync means (simple English)
Data sync is the process of making sure the information on your phone matches the information on the server (and other devices). When you add a note offline, sync uploads it when internet returns; when someone else updates shared data, sync downloads it to your phone.[2][1]

### Why sync is hard on mobile
Mobile networks drop frequently (elevator, subway, airplane mode), iOS background execution is limited by the OS to save battery, and continuous syncing drains battery and cellular data. Apps must handle partial failures gracefully—what happens if the upload starts but the app is terminated mid-sync?[3][4][1]

### Real-life analogy
Think of sync like a postal service between two notebooks (phone and server):
- You write entries in your local notebook (offline writes).
- When the mailman arrives (internet returns), letters go both ways: your new entries are sent to the server, and the server's new entries arrive at your device.
- If both notebooks have changes to the same entry, you need a rule to decide which one wins (conflict resolution).

### Sync vs backup vs replication
| Term | Direction | Frequency | Handles conflicts? | Use case |
|---|---|---|---|---|
| Sync | Bidirectional [2] | Continuous/periodic | Yes [5] | WhatsApp messages, Notes app |
| Backup | One-way (device → cloud) | Manual/scheduled | No | iCloud backup, device restore |
| Replication | One-way or bidirectional | Continuous | Sometimes | Database clustering, content delivery |

### Simple iOS sync example
**Food delivery "Saved addresses":**
- User adds a new address offline → saved to Core Data locally, marked `syncStatus = .pending`.
- When app returns online (detected via `NWPathMonitor`), a background worker uploads the address.[6]
- Server confirms receipt → local record marked `syncStatus = .synced`.
- If another device updated the same address, conflict resolution applies (typically server-authoritative for critical data like addresses).

---

## Intermediate Level (SDE-2 Ready)

### Common mobile sync models
**Pull sync (client-initiated):** Client periodically asks server "what's new since my last sync?" using a timestamp or token, then downloads changes. Simple and battery-friendly (fewer wake-ups), but updates are delayed until the next pull.[7][2]

**Push sync (server-initiated):** Server sends real-time notifications (APNs on iOS) when data changes, triggering the client to fetch updates immediately. Lower latency but requires maintaining a notification infrastructure and handling token management.[2]

**Two-way sync:** Client pushes local changes to the server and pulls server changes, merging both directions. Most offline-first apps use this model—WhatsApp, Google Drive, Uber trip history all sync bidirectionally.[8][9]

### Sync triggers (when sync happens on iOS)
- **App launch:** Common for simple apps; sync on `application(_:didFinishLaunchingWithOptions:)`.[4]
- **Foreground transition:** Sync when app becomes active via `applicationDidBecomeActive(_:)`.
- **Manual refresh:** User pulls-to-refresh; immediate sync with loading indicator.
- **Background fetch:** iOS wakes the app periodically (controlled by OS) to sync via `application(_:performFetchWithCompletionHandler:)`. Must call `completionHandler` promptly with result (`.newData`, `.noData`, or `.failed`) so iOS learns your app's behavior and schedules future fetches accordingly.[4]
- **Background URLSession:** For large uploads/downloads that survive app termination using `URLSessionConfiguration.background(withIdentifier:)`. The system manages the transfer out-of-process and relaunches your app when done.[10][3]

### Handling conflicts (simple cases)
A conflict occurs when the same data is modified in two places before syncing (e.g., editing a cart item offline while the server updates the price). Common strategies:[11]
- **Last-write-wins (LWW):** Compare timestamps; most recent change wins. Simple but risky for critical data (financial records, addresses).[5][11]
- **Server-authoritative:** Server's version always wins (common for pricing, inventory, payments).
- **Client-authoritative:** Rare; used for user-generated content like drafts where local edits take precedence.

### Incremental sync (delta sync)
Instead of downloading the entire dataset on every sync, request only changes since the last sync using timestamps, version numbers, or change tokens. For example, Microsoft Graph API uses delta query with `@odata.deltaLink` tokens to fetch only new/updated/deleted items since the previous request.[12][13]

**iOS example (ride-hailing trip history):**
```swift
struct SyncMetadata: Codable {
    var lastSyncTimestamp: Date
    var changeToken: String?
}

func syncTrips() async throws {
    let metadata = loadSyncMetadata() // from UserDefaults or DB
    
    // Incremental fetch: only trips changed since lastSyncTimestamp
    let url = URL(string: "https://api.example.com/trips?since=\(metadata.lastSyncTimestamp.iso8601)")!
    let (data, _) = try await URLSession.shared.data(from: url)
    let response = try JSONDecoder().decode(TripsResponse.self, from: data)
    
    // Merge changes into local DB
    for trip in response.trips {
        try mergeTrip(trip)
    }
    
    // Save new sync metadata
    saveSyncMetadata(SyncMetadata(lastSyncTimestamp: Date(), changeToken: response.nextToken))
}
```

### Retry strategies and idempotency
Network requests can fail or timeout; retries with exponential backoff prevent hammering the server.
- Start with a base delay (e.g., 1s), double after each failure, and cap at a max delay (e.g., 60s).
- Add jitter (randomness) to prevent retry storms when many clients fail simultaneously.

**Idempotency** means repeating the same request multiple times produces the same result (no duplicate side effects).
- Use unique request IDs or server-side deduplication to make sync operations idempotent—crucial for retries.

### Common iOS sync mistakes
- **Calling network APIs directly from UI code:** Bypasses the local database; different screens show inconsistent state.
- **Not persisting pending operations:** User actions lost on app crash; use a durable "sync queue" or "outbox" table.
- **Tight retry loops without backoff:** Drains battery and triggers rate limits.
- **Ignoring background execution limits:** iOS suspends background tasks after ~30 seconds; use `BGTaskScheduler` for longer work.
- **Blocking main thread during sync:** Use background queues (`DispatchQueue.global()` or `Task { }`) to avoid UI freezes.

---

## Advanced Level (Big Tech Interviews)

### Designing a robust sync engine
At scale, treat sync as a dedicated subsystem with its own state machine, not scattered API calls. Architecture components:

- **Local database (source of truth):** Core Data, Realm, or SQLite stores all user-visible data.
- **Sync queue (outbox pattern):** Persistent queue of pending operations (`SyncOperation` entities with `id`, `type`, `payload`, `status`, `retryCount`, `createdAt`). Operations survive app restarts and crashes.
- **Sync worker:** Background service that processes the outbox, executes network requests with retries/backoff, and updates operation status (`pending → inProgress → completed/failed`).
- **Conflict resolver:** Applies resolution strategies per entity type when local and server versions diverge.
- **Connectivity monitor:** Observes network state with `NWPathMonitor` and triggers sync on offline→online transitions.

### Conflict resolution strategies (advanced)
**Last-write-wins (LWW) with hybrid logical clocks:** Use vector clocks or hybrid logical timestamps (combining wall-clock time and logical counters) to deterministically resolve conflicts across distributed clients. Couchbase Sync Gateway 4.0+ uses this approach with version vectors.

**Operational transformation (OT) / CRDTs:** For collaborative editing (Google Docs-style), use conflict-free replicated data types (CRDTs) that mathematically guarantee convergence without explicit resolution. Complex to implement but enables true real-time collaboration.

**Custom merge rules:** Per-field merging based on business logic. Example for ecommerce cart:
- Cart items: merge by item ID (union of both sets).
- Quantity: sum if same item added on multiple devices.
- Promo codes: server-authoritative (validate on server).

**User intervention:** For critical conflicts (e.g., editing the same invoice field), surface a UI for the user to choose which version to keep or manually merge.

### Syncing large datasets efficiently
**Pagination:** Fetch data in chunks (e.g., 50 records per request) to avoid memory spikes and long network hangs.

**Batching:** Group multiple small operations into a single request to reduce overhead (HTTP headers, TLS handshakes).

**Delta updates (change data capture):** Send only modified fields, not entire records. For example, if a user updates just the address line 1, send `{"addressLine1": "new value"}` instead of the full address object.

**Compression:** gzip or Brotli compress request/response bodies to save bandwidth, especially for text-heavy data (chat messages, documents).

**Selective sync:** Allow users to choose which data to sync (e.g., Dropbox selective sync, WhatsApp "media auto-download" settings) to control storage and data usage.

### Data sync with offline-first
Offline-first and sync are complementary: offline-first ensures the app remains functional without network, while sync reconciles changes when connectivity returns. Key patterns:
- **Eventual consistency:** Accept that data won't be immediately consistent across devices; define acceptable staleness windows (e.g., chat messages appear within 5 seconds).
- **Optimistic UI:** Show the user's action immediately (e.g., message sent), then reconcile with server later; rollback if server rejects.
- **Persistent outbox:** Queue all writes locally and sync asynchronously; never lose user intent even if network fails.

### Real-time sync vs eventual consistency
**Real-time sync** (low latency, strong consistency) requires WebSockets or server-sent events (SSE) for live updates, consuming battery and data. Uber uses this for driver location tracking and real-time ETA updates.

**Eventual consistency** (higher latency, looser guarantees) batches syncs periodically or on triggers (app launch, background fetch). Food delivery order history doesn't need real-time updates—syncing every few minutes is acceptable.

**Trade-offs:**
- Real-time: lower latency, higher battery drain, complex infrastructure (WebSocket lifecycle, reconnection logic).
- Eventual: better battery life, simpler client code, but stale data until next sync.

### Battery, performance, storage
**Battery:** Minimize radio wake-ups by batching sync operations and leveraging iOS scheduling (`BGTaskScheduler` instead of continuous polling). Avoid syncing on cellular if possible (offer Wi-Fi-only mode).

**Performance:** Background sync must not block the main thread; use concurrent queues and async/await. Lazy-load large datasets (virtualized lists, paginated queries) instead of syncing everything upfront.

**Storage:** Implement cache eviction policies (LRU, TTL) to prevent unbounded growth. For media-heavy apps (photo sync), store thumbnails locally and full-res on-demand.

### Security and privacy
**Encryption in transit:** Always use HTTPS (TLS 1.3+) for sync traffic.

**Encryption at rest:** Encrypt sensitive local data (PII, tokens) using iOS Data Protection APIs or third-party libraries (SQLCipher for SQLite).

**Authentication:** Use OAuth 2.0 tokens with refresh; store access tokens in Keychain (never UserDefaults). Handle token expiration gracefully during background sync.

**Data integrity:** Use checksums or hashes to verify downloaded data hasn't been tampered with during transit.

### How big tech apps sync data (mobile client view)
**Uber trip history:** Pulls trip list incrementally (timestamp-based) on app launch and foreground; caches locally in database; server-authoritative for pricing/driver info; real-time location updates use WebSocket while trip is active.

**WhatsApp messages:** Uses custom binary protocol over HTTPS; messages stored locally in SQLite; push notifications (APNs) trigger sync; end-to-end encrypted (sync layer sees only encrypted payloads); conflict-free by design (each message has unique ID).

**Google Drive iOS:** iCloud-style file sync using delta sync and change tokens; conflict resolution shows both versions if edits happen offline; thumbnails cached locally, full files on-demand; background sync via `URLSessionConfiguration.background`.

**Amazon mobile app:** Cart and order history use two-way sync; cart is eventually consistent (merge items from multiple devices); pricing is server-authoritative; images cached with TTL; offline browsing relies on cached catalog data.

---

## Interview Preparation

### Real mobile system design questions (sync-focused)
1. "Design offline chat for iOS with reliable message delivery and conflict-free ordering."
2. "Design a collaborative notes app (Google Keep-style) with multi-device sync and conflict resolution."
3. "Design trip history sync for a ride-hailing app (Uber-like) that works offline and syncs incrementally."
4. "Design a shopping cart that syncs across devices with conflict handling for quantity and pricing."
5. "Design offline map data sync (Google Maps-style) with selective download and storage limits."

### Interview-ready answer structure (say it out loud)

**1. Clarify requirements (ask these questions):**
- What must work offline? (read-only, write, or both?)
- Conflict resolution policy? (LWW, server-authority, or user intervention?)
- Data volume? (messages: tens of thousands; orders: hundreds)
- Real-time or eventual consistency? (chat: real-time preferred; order history: eventual OK)

**2. High-level architecture:**
- Local DB (Core Data / Realm): single source of truth for UI.
- Sync queue (outbox): persistent table of pending writes.
- Sync service: processes outbox, handles retries, applies conflict resolution.
- Network monitor: `NWPathMonitor` triggers sync on connectivity changes.

**3. Data model (example: chat):**
```
Message: id (UUID), chatId, senderId, text, timestamp, syncStatus (pending/synced/failed), serverMessageId
SyncOperation: id, operationType (send/delete/edit), payload (JSON), retryCount, lastAttempt
```

**4. Sync flow (write path):**
- User sends message → insert into local DB with `syncStatus=.pending` and generate local UUID.
- Enqueue `SyncOperation(type: .sendMessage, payload: message)`.
- Sync worker picks up operation, POSTs to server, receives `serverMessageId`.
- Update local message with `serverMessageId` and `syncStatus=.synced`.
- On failure: exponential backoff, retry up to N times, then mark `.failed` and notify user.

**5. Sync flow (read path):**
- On app launch or foreground: fetch `GET /messages?since={lastSyncTimestamp}`.
- Server returns delta (new/updated/deleted messages).
- Merge into local DB: upsert by `serverMessageId`, resolve conflicts if local pending message matches.
- Update `lastSyncTimestamp`.

**6. Conflict resolution (state your choice and justify):**
- For messages: conflict-free (each has unique ID; server assigns canonical order by timestamp).
- For edits: LWW based on `timestamp` field; server discards stale edits.

**7. Trade-offs:**
- **Sync frequency:** more frequent = fresher data but higher battery cost; use background fetch (iOS-scheduled) for balance.
- **Storage:** cache all messages locally vs. only recent N days; depends on user needs and device storage.
- **Conflict strategy:** LWW is simple but loses data; user intervention is safest but adds friction.
- **Real-time:** WebSocket for live updates vs. pull-based for simplicity and battery savings.

### Common follow-up questions and answers

**Q: "What if the app crashes mid-sync?"**
**A:** Persistent outbox ensures operations survive crashes. On restart, the sync worker resumes processing from where it left off; mark operations `inProgress` before executing and reset to `pending` if app terminates without completion (using timestamps or app lifecycle hooks).

**Q: "How do you avoid duplicate uploads (idempotency)?"**
**A:** Include a unique `requestId` (UUID) in each sync operation; server deduplicates by `requestId` (e.g., store in Redis cache for 24 hours and reject duplicates). For messages, client-generated UUID serves as idempotency key.

**Q: "How do you handle slow networks or timeouts?"**
**A:** Set reasonable timeouts (e.g., 30s for sync requests), use exponential backoff on retries (doubling delay from 1s → 2s → 4s → ... up to 60s max), and add jitter to prevent retry storms. Background URLSession handles long-running uploads/downloads even if app is suspended.

**Q: "Two users edit the same field offline. How do you merge?"**
**A:** Define per-field or per-entity resolution rules:
- Cart quantity: sum (or max).
- Address: server-authoritative (user must re-enter if mismatch).
- Collaborative doc: use CRDT (complex) or prompt user to choose version (simpler).

**Q: "How do you test sync logic?"**
**A:** Mock network layer and local DB; simulate scenarios (offline writes, slow network, server conflicts, app crash mid-sync); use Xcode's network link conditioner for latency/packet loss testing; integration tests with a staging server.

**Q: "How do you sync large files (photos, videos)?"**
**A:** Use background `URLSession` with `uploadTask` or `downloadTask`, which survives app suspension. Resume interrupted transfers using HTTP Range headers or multipart uploads (S3-style). Store metadata (thumbnail, fileURL) in DB; download full file on-demand.

---

### Resources & References
[1] [Pattern Paper](https://www.cs.wm.edu/~dcschmidt/PDF/PatternPaperv11.pdf)
[2] [Offline Data Sync](https://intellisoft.io/implementing-offline-data-synchronization-in-logistics-applications/)
[3] [URLSession Background Config](https://stackoverflow.com/questions/76136268/what-happens-behind-when-i-create-urlsession-with-background-configuration)
[4] [Background Fetch iOS](https://mobiforge.com/design-development/using-background-fetch-ios)
[5] [Couchbase Conflict Resolution](https://docs.couchbase.com/sync-gateway/current/conflict-resolution.html)
[6] [NWPathMonitor](https://www.appypievibe.ai/blog/swift-code/nwpathmonitor-internet-connectivity/)
[7] [Sync Strategies](https://endmr11.github.io/system_design/en/mobile/storage/sync-strategies.html)
[8] [Couchbase Mobile Sync](https://www.couchbase.com/blog/data-sync-on-ios-couchbase-mobile/)
[9] [Google Drive Sync](https://community.latenode.com/t/whats-the-best-way-to-implement-cloud-synchronization-with-google-drive-in-ios-applications/35175)
[10] [URLSession Background Transfers](https://williamboles.com/keeping-things-going-when-the-user-leaves-with-urlsession-and-background-transfers/)
[11] [Offline Conflict Resolution](https://peerdh.com/blogs/programming-insights/implementing-data-conflict-resolution-strategies-for-offline-first-applications)
[12] [Real-time Data Sync](https://www.dots-mobile.com/blog-posts/best-practices-real-time-data-sync)
[13] [Microsoft Graph Delta Query](https://learn.microsoft.com/en-us/graph/delta-query-messages)
[14] [ACM Digital Library](https://dl.acm.org/doi/10.5555/2821679.2831282)
[15] [Large Volume Data Sync](https://www.slideshare.net/slideshow/mobile-synchronization-patterns-for-large-volumes-of-data/183807453)
[16] [URLSession Documentation](https://developer.apple.com/documentation/foundation/urlsession)
[17] [URLSessionConfiguration](https://developer.apple.com/documentation/foundation/urlsessionconfiguration)
[18] [URLSession Background StackOverflow](https://stackoverflow.com/questions/57784097/how-to-make-urlsession-to-work-when-app-is-moved-to-background)
[19] [CoreData CloudKit Sync](https://cdf1982.com/2020/06/21/CoreData_CloudKit_app_with_NSPersistentCloudKitContainer_only_syncing_upon_launch_not_when_is_active.html)
[20] [PowerSync Background Tasks](https://www.powersync.com/blog/keep-background-apps-fresh-with-expo-background-tasks-and-powersync)
[21] [iOS Background Networking](https://blog.stackademic.com/ios-18-background-survival-guide-part-3-unstoppable-networking-with-background-urlsession-f9c8f01f665b)
[22] [Apple Developer Forums](https://developer.apple.com/forums/thread/767395)
[23] [Uber HiveSync](https://www.uber.com/en-IN/blog/building-ubers-data-lake-batch-data-replication-using-hivesync/)
[24] [Uber System Architecture](https://www.geeksforgeeks.org/system-design/system-design-of-uber-app-uber-system-architecture/)
[25] [Uber SOA Design](https://www.techaheadcorp.com/blog/uber-service-oriented-architecture-design-part/)
[26] [Uber Data Infrastructure](https://www.uber.com/en-IN/blog/modernizing-ubers-data-infrastructure-with-gcp/)
[27] [Uber Real-time Architecture](https://siliconangle.com/2023/06/17/ubers-real-time-architecture-represents-future-data-apps-meet-architects-built/)
[28] [WhatsApp Architecture](https://www.app-whatsappws.com/?p=220)
[29] [Uber Petabytes Management](https://blog.bytebytego.com/p/how-uber-manages-petabytes-of-real)
[30] [Offline Messaging](https://sendontime.io/features/offline-messaging-scheduling)
[31] [Logseq Sync](https://discuss.logseq.com/t/sync-on-ios-with-onedrive-dropbox-googledrive-icloud/4685?page=2)
