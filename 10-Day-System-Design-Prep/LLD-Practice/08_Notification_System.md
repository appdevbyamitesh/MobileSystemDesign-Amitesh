# LLD Problem 8: Notification System

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **iOS-Specific**: UNUserNotificationCenter usage
- **Multiple Channels**: Push, local, in-app
- **Design Patterns**: Observer, Strategy, Factory
- **Background Handling**: Notification delivery

---

# R — Requirements

## Functional Requirements

```markdown
1. Notification Channels
   - Push notifications (APNs)
   - Local notifications
   - In-app notifications

2. Notification Management
   - Schedule notifications
   - Cancel notifications
   - Update pending notifications
   - Group notifications

3. User Preferences
   - Per-channel opt-in/out
   - Quiet hours
   - Category-based settings
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Delivery | Best effort | APNs not guaranteed |
| Background | Required | Deliver when app closed |
| Permission | Graceful | Handle denied gracefully |

---

# E — Entities

```swift
// MARK: - Notification Types

struct AppNotification: Identifiable, Codable {
    let id: String
    let type: NotificationType
    let title: String
    let body: String
    let data: [String: String]
    let category: NotificationCategory
    var channel: NotificationChannel
    let createdAt: Date
    var scheduledFor: Date?
    var isRead: Bool = false
}

enum NotificationType: String, Codable {
    case orderUpdate
    case promotion
    case reminder
    case social
    case system
}

enum NotificationCategory: String, Codable {
    case transactional  // Order updates, critical
    case marketing      // Promotions, can be suppressed
    case social         // Friend activity
    case system         // App updates
}

enum NotificationChannel: String, Codable {
    case push
    case local
    case inApp
    
    var requiresPermission: Bool {
        switch self {
        case .push, .local: return true
        case .inApp: return false
        }
    }
}

// MARK: - User Preferences

struct NotificationPreferences: Codable {
    var pushEnabled: Bool
    var localEnabled: Bool
    var inAppEnabled: Bool
    var quietHoursStart: Date?
    var quietHoursEnd: Date?
    var enabledCategories: Set<NotificationCategory>
}
```

---

# S — States

```swift
enum NotificationState {
    case pending
    case delivered
    case read
    case dismissed
    case failed
}
```

## State Diagram

```
[Pending] ──deliver──▶ [Delivered] ──tap──▶ [Read]
                              │
                              │ swipe
                              ▼
                        [Dismissed]
```

---

# A — Architecture & Patterns

## Pattern 1: Observer (Notification Received)

```swift
import Combine
import UserNotifications

class NotificationObserver: NSObject, UNUserNotificationCenterDelegate, ObservableObject {
    static let shared = NotificationObserver()
    
    @Published private(set) var receivedNotifications: [AppNotification] = []
    @Published private(set) var unreadCount: Int = 0
    
    private let notificationSubject = PassthroughSubject<AppNotification, Never>()
    
    var notificationPublisher: AnyPublisher<AppNotification, Never> {
        notificationSubject.eraseToAnyPublisher()
    }
    
    override init() {
        super.init()
        UNUserNotificationCenter.current().delegate = self
    }
    
    // Foreground notification
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        willPresent notification: UNNotification
    ) async -> UNNotificationPresentationOptions {
        let appNotification = parse(notification)
        
        await MainActor.run {
            receivedNotifications.append(appNotification)
            notificationSubject.send(appNotification)
            unreadCount += 1
        }
        
        return [.banner, .sound, .badge]
    }
    
    // Notification tapped
    func userNotificationCenter(
        _ center: UNUserNotificationCenter,
        didReceive response: UNNotificationResponse
    ) async {
        let appNotification = parse(response.notification)
        
        // Handle action
        switch response.actionIdentifier {
        case UNNotificationDefaultActionIdentifier:
            handleNotificationTap(appNotification)
        case "markAsRead":
            markAsRead(appNotification.id)
        default:
            break
        }
    }
}
```

### Why Observer Pattern

| Alternative | Why Rejected |
|-------------|--------------|
| Polling | Wastes resources |
| Delegate only | Only 1:1 |
| No pattern | Tight coupling |

---

## Pattern 2: Strategy (Channel Selection)

```swift
protocol NotificationChannelStrategy {
    var channel: NotificationChannel { get }
    func send(_ notification: AppNotification) async throws
    func cancel(_ notificationId: String) async
}

// Push Notification Strategy
class PushNotificationStrategy: NotificationChannelStrategy {
    let channel: NotificationChannel = .push
    
    func send(_ notification: AppNotification) async throws {
        // Push is server-initiated, we just register token
        throw NotificationError.pushRequiresServer
    }
    
    func cancel(_ notificationId: String) async {
        // Can't cancel remote push
    }
}

// Local Notification Strategy
class LocalNotificationStrategy: NotificationChannelStrategy {
    let channel: NotificationChannel = .local
    
    func send(_ notification: AppNotification) async throws {
        let content = UNMutableNotificationContent()
        content.title = notification.title
        content.body = notification.body
        content.sound = .default
        content.userInfo = notification.data
        
        let trigger: UNNotificationTrigger?
        if let scheduledFor = notification.scheduledFor {
            let components = Calendar.current.dateComponents(
                [.year, .month, .day, .hour, .minute],
                from: scheduledFor
            )
            trigger = UNCalendarNotificationTrigger(dateMatching: components, repeats: false)
        } else {
            trigger = UNTimeIntervalNotificationTrigger(timeInterval: 1, repeats: false)
        }
        
        let request = UNNotificationRequest(
            identifier: notification.id,
            content: content,
            trigger: trigger
        )
        
        try await UNUserNotificationCenter.current().add(request)
    }
    
    func cancel(_ notificationId: String) async {
        UNUserNotificationCenter.current().removePendingNotificationRequests(
            withIdentifiers: [notificationId]
        )
    }
}

// In-App Notification Strategy
class InAppNotificationStrategy: NotificationChannelStrategy {
    let channel: NotificationChannel = .inApp
    
    private let inAppManager: InAppNotificationManager
    
    init(inAppManager: InAppNotificationManager) {
        self.inAppManager = inAppManager
    }
    
    func send(_ notification: AppNotification) async throws {
        await inAppManager.show(notification)
    }
    
    func cancel(_ notificationId: String) async {
        await inAppManager.dismiss(notificationId)
    }
}
```

---

## Pattern 3: Factory (Channel Creation)

```swift
class NotificationChannelFactory {
    private let inAppManager: InAppNotificationManager
    
    init(inAppManager: InAppNotificationManager) {
        self.inAppManager = inAppManager
    }
    
    func createChannel(_ type: NotificationChannel) -> NotificationChannelStrategy {
        switch type {
        case .push:
            return PushNotificationStrategy()
        case .local:
            return LocalNotificationStrategy()
        case .inApp:
            return InAppNotificationStrategy(inAppManager: inAppManager)
        }
    }
    
    func createChannels(for notification: AppNotification) -> [NotificationChannelStrategy] {
        // Based on notification category and user preferences
        var channels: [NotificationChannel] = []
        
        switch notification.category {
        case .transactional:
            channels = [.push, .inApp]  // Always both
        case .marketing:
            channels = [.push]  // Push only
        case .social:
            channels = [.inApp]  // In-app only
        case .system:
            channels = [.local, .inApp]
        }
        
        return channels.map { createChannel($0) }
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Notification Dispatch

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│ Server │ │  AppDel    │ │ NotifMgr   │ │  Factory   │ │  Strategy  │
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │
    │ APNs push  │              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ didReceive   │              │              │
    │            │─────────────▶│              │              │
    │            │              │              │              │
    │            │              │ parse notification          │
    │            │              │──┐           │              │
    │            │              │◀─┘           │              │
    │            │              │              │              │
    │            │              │ createChannels(notification)│
    │            │              │─────────────▶│              │
    │            │              │              │              │
    │            │              │◀─────────────│ [strategies] │
    │            │              │              │              │
    │            │              │ send(notification)          │
    │            │              │────────────────────────────▶│
    │            │              │              │              │
    │            │              │              │              │ show banner
    │            │              │              │              │──┐
    │            │              │              │              │◀─┘
    │            │              │              │              │
    │            │              │ publish update              │
    │            │              │──────────────▶ Observer     │
    │            │              │              │              │
```

---

# E — Edge Cases

## Edge Case 1: Permission Denied

```swift
class NotificationPermissionManager {
    func requestPermission() async -> Bool {
        let center = UNUserNotificationCenter.current()
        
        do {
            let granted = try await center.requestAuthorization(options: [.alert, .sound, .badge])
            
            if !granted {
                // Show in-app setting to re-enable
                await showPermissionDeniedUI()
            }
            
            return granted
        } catch {
            return false
        }
    }
    
    func checkPermissionStatus() async -> UNAuthorizationStatus {
        let settings = await UNUserNotificationCenter.current().notificationSettings()
        return settings.authorizationStatus
    }
    
    func showPermissionDeniedUI() async {
        // Deep link to Settings
        await MainActor.run {
            NotificationCenter.default.post(
                name: .showNotificationSettings,
                object: nil
            )
        }
    }
}
```

## Edge Case 2: Quiet Hours

```swift
class NotificationScheduler {
    func shouldDeliverNow(_ notification: AppNotification, preferences: NotificationPreferences) -> Bool {
        guard let start = preferences.quietHoursStart,
              let end = preferences.quietHoursEnd else {
            return true
        }
        
        let now = Date()
        let calendar = Calendar.current
        
        let currentHour = calendar.component(.hour, from: now)
        let startHour = calendar.component(.hour, from: start)
        let endHour = calendar.component(.hour, from: end)
        
        // In quiet hours
        if startHour < endHour {
            // Same day: 22:00 - 07:00 (next day)
            if currentHour >= startHour || currentHour < endHour {
                return notification.category == .transactional  // Only urgent
            }
        } else {
            // Overnight: 22:00 - 07:00
            if currentHour >= startHour || currentHour < endHour {
                return notification.category == .transactional
            }
        }
        
        return true
    }
}
```

## Edge Case 3: Background Delivery

```swift
class AppDelegate: UIApplicationDelegate {
    func application(
        _ application: UIApplication,
        didReceiveRemoteNotification userInfo: [AnyHashable: Any]
    ) async -> UIBackgroundFetchResult {
        
        guard let notification = parseNotification(userInfo) else {
            return .noData
        }
        
        // Process in background
        do {
            // Update local data based on notification
            try await processBackgroundNotification(notification)
            return .newData
        } catch {
            return .failed
        }
    }
}
```

---

# D — Design Trade-offs

| Push | Local | In-App |
|------|-------|--------|
| Works when app closed | Scheduled, offline | Requires app open |
| Server-initiated | Client-initiated | Instant |
| Requires permission | Requires permission | No permission |

**Decision:** Use all three with priority (push > local > in-app for critical)

---

# iOS Implementation

```swift
@MainActor
class NotificationManager: ObservableObject {
    static let shared = NotificationManager()
    
    @Published private(set) var notifications: [AppNotification] = []
    @Published private(set) var unreadCount: Int = 0
    
    private let channelFactory: NotificationChannelFactory
    private let preferences: NotificationPreferences
    private var cancellables = Set<AnyCancellable>()
    
    init() {
        self.channelFactory = NotificationChannelFactory(
            inAppManager: InAppNotificationManager.shared
        )
        self.preferences = UserDefaults.standard.notificationPreferences
        
        // Subscribe to received notifications
        NotificationObserver.shared.notificationPublisher
            .sink { [weak self] notification in
                self?.handleReceived(notification)
            }
            .store(in: &cancellables)
    }
    
    func send(_ notification: AppNotification) async throws {
        // Check preferences
        guard preferences.enabledCategories.contains(notification.category) else {
            throw NotificationError.categoryDisabled
        }
        
        // Get appropriate channels
        let channels = channelFactory.createChannels(for: notification)
        
        // Send via each channel
        for channel in channels {
            try await channel.send(notification)
        }
        
        notifications.append(notification)
    }
    
    func markAsRead(_ id: String) {
        if let index = notifications.firstIndex(where: { $0.id == id }) {
            notifications[index].isRead = true
            unreadCount = notifications.filter { !$0.isRead }.count
        }
    }
    
    func clearAll() {
        notifications.removeAll()
        unreadCount = 0
        
        // Clear system notifications
        UNUserNotificationCenter.current().removeAllDeliveredNotifications()
    }
}
```

---

# Interview Tips

## What to Say

```markdown
1. "I'll use Observer pattern for receiving notifications..."
2. "Strategy pattern for different delivery channels..."
3. "Factory to create the right channel based on notification type..."
4. "UNUserNotificationCenter is the iOS notification API..."
```

## Red Flags to Avoid

```markdown
❌ "I'll just use UILocalNotification"
   → Deprecated since iOS 10

❌ Forgetting permission handling
   → Critical for iOS

❌ No background delivery handling
   → Incomplete implementation
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Notification System problem!*
