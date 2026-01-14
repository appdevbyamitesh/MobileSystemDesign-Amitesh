# LLD Problem 9: Meeting Room Booking System

> **Amazon iOS LLD Interview — Using RESHADED Framework**

---

## Why Amazon Asks This

- **Calendar Integration**: EventKit usage
- **Conflict Detection**: Overlapping bookings
- **Clean Architecture**: Facade pattern
- **Real-world Application**: Corporate apps

---

# R — Requirements

## Functional Requirements

```markdown
1. Room Management
   - View available rooms
   - Room capacity and amenities
   - Floor-wise organization

2. Booking
   - Check availability
   - Book room for a time slot
   - Recurring bookings
   - Cancel/modify booking

3. Calendar Integration
   - Sync with device calendar
   - Send invites to attendees
   - View bookings in calendar
```

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Conflict detection | < 100ms | Real-time feedback |
| Calendar sync | Bidirectional | Single source |
| Offline | View only | Booking requires network |

---

# E — Entities

```swift
// MARK: - Room & Building

struct Room: Identifiable, Codable {
    let id: String
    let name: String
    let floor: Int
    let capacity: Int
    let amenities: [Amenity]
    let imageUrl: String?
}

enum Amenity: String, Codable, CaseIterable {
    case projector
    case whiteboard
    case videoConference
    case phoneConference
    case wheelchair
}

struct Building: Identifiable, Codable {
    let id: String
    let name: String
    let address: String
    let floors: [Floor]
}

struct Floor: Identifiable, Codable {
    let id: String
    let level: Int
    let rooms: [Room]
}

// MARK: - Booking

struct Booking: Identifiable, Codable {
    let id: String
    let roomId: String
    let title: String
    let startTime: Date
    let endTime: Date
    let organizer: User
    let attendees: [User]
    let recurrence: Recurrence?
    var status: BookingStatus
    var calendarEventId: String?
}

struct Recurrence: Codable {
    let frequency: RecurrenceFrequency
    let interval: Int  // Every X weeks
    let endDate: Date?
    let count: Int?
}

enum RecurrenceFrequency: String, Codable {
    case daily, weekly, biweekly, monthly
}

enum BookingStatus: String, Codable {
    case pending
    case confirmed
    case cancelled
    case completed
}

// MARK: - Time Slot

struct TimeSlot: Hashable {
    let start: Date
    let end: Date
    
    func overlaps(with other: TimeSlot) -> Bool {
        start < other.end && end > other.start
    }
}
```

---

# S — States

```swift
enum BookingState {
    case draft
    case pending
    case confirmed
    case cancelled
}
```

## State Diagram

```
[Draft] ──submit──▶ [Pending] ──approve──▶ [Confirmed]
                        │                       │
                        │ reject                │ cancel
                        ▼                       ▼
                   [Cancelled]             [Cancelled]
```

---

# A — Architecture & Patterns

## Pattern 1: Facade (Booking API)

### What it does:
Simplifies complex booking operations behind a clean interface.

```swift
class BookingFacade {
    private let availabilityService: AvailabilityService
    private let bookingService: BookingService
    private let calendarService: CalendarService
    private let notificationService: NotificationService
    
    init(
        availabilityService: AvailabilityService,
        bookingService: BookingService,
        calendarService: CalendarService,
        notificationService: NotificationService
    ) {
        self.availabilityService = availabilityService
        self.bookingService = bookingService
        self.calendarService = calendarService
        self.notificationService = notificationService
    }
    
    // Simple interface for complex operation
    func bookRoom(
        room: Room,
        slot: TimeSlot,
        title: String,
        attendees: [User]
    ) async throws -> Booking {
        // 1. Check availability
        let isAvailable = try await availabilityService.check(room: room, slot: slot)
        guard isAvailable else {
            throw BookingError.roomNotAvailable
        }
        
        // 2. Create booking
        let booking = try await bookingService.create(
            room: room,
            slot: slot,
            title: title,
            attendees: attendees
        )
        
        // 3. Add to calendar
        let eventId = try await calendarService.createEvent(for: booking)
        var updatedBooking = booking
        updatedBooking.calendarEventId = eventId
        
        // 4. Send notifications
        await notificationService.sendBookingConfirmation(booking)
        
        return updatedBooking
    }
    
    func cancelBooking(_ booking: Booking) async throws {
        // 1. Cancel in system
        try await bookingService.cancel(booking)
        
        // 2. Remove from calendar
        if let eventId = booking.calendarEventId {
            try await calendarService.deleteEvent(eventId)
        }
        
        // 3. Notify attendees
        await notificationService.sendCancellation(booking)
    }
    
    func findAvailableRooms(
        slot: TimeSlot,
        capacity: Int,
        amenities: [Amenity]
    ) async throws -> [Room] {
        try await availabilityService.findAvailable(
            slot: slot,
            capacity: capacity,
            amenities: amenities
        )
    }
}
```

### Why Facade Pattern

| Without Facade | With Facade |
|----------------|-------------|
| ViewModel knows all services | Single entry point |
| Complex coordination | Simple method calls |
| Hard to test | Easily mockable |

---

## Pattern 2: Strategy (Conflict Resolution)

```swift
protocol ConflictResolutionStrategy {
    func resolve(booking: Booking, conflicts: [Booking]) -> ConflictResolution
}

enum ConflictResolution {
    case reject
    case waitlist(position: Int)
    case suggestAlternatives([TimeSlot])
    case override(reason: String)  // For managers
}

// Strategy 1: Strict - No conflicts allowed
class StrictConflictResolution: ConflictResolutionStrategy {
    func resolve(booking: Booking, conflicts: [Booking]) -> ConflictResolution {
        if conflicts.isEmpty {
            return .reject  // Just return, will be handled
        }
        return .reject
    }
}

// Strategy 2: Suggest alternatives
class SuggestAlternativeResolution: ConflictResolutionStrategy {
    private let availabilityService: AvailabilityService
    
    func resolve(booking: Booking, conflicts: [Booking]) -> ConflictResolution {
        // Find alternative slots
        let alternatives = findAlternativeSlots(
            near: TimeSlot(start: booking.startTime, end: booking.endTime),
            room: booking.roomId
        )
        
        return .suggestAlternatives(alternatives)
    }
    
    private func findAlternativeSlots(near slot: TimeSlot, room: String) -> [TimeSlot] {
        // Logic to find nearby available slots
        return []
    }
}

// Strategy 3: Waitlist
class WaitlistResolution: ConflictResolutionStrategy {
    private var waitlists: [String: [Booking]] = [:]
    
    func resolve(booking: Booking, conflicts: [Booking]) -> ConflictResolution {
        let key = "\(booking.roomId)_\(booking.startTime)"
        let position = (waitlists[key]?.count ?? 0) + 1
        
        waitlists[key, default: []].append(booking)
        
        return .waitlist(position: position)
    }
}

// Usage
class AvailabilityService {
    private var conflictStrategy: ConflictResolutionStrategy
    
    func setConflictStrategy(_ strategy: ConflictResolutionStrategy) {
        self.conflictStrategy = strategy
    }
    
    func handleConflict(booking: Booking, conflicts: [Booking]) -> ConflictResolution {
        conflictStrategy.resolve(booking: booking, conflicts: conflicts)
    }
}
```

---

# D — Data Flow

## Sequence Diagram: Booking + Conflict Detection

```
┌────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐
│  User  │ │   Facade   │ │AvailSvc   │ │ BookingSvc │ │ CalendarSvc│
└───┬────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘ └─────┬──────┘
    │            │              │              │              │
    │ bookRoom() │              │              │              │
    │───────────▶│              │              │              │
    │            │              │              │              │
    │            │ check(room, slot)           │              │
    │            │─────────────▶│              │              │
    │            │              │              │              │
    │            │              │──┐ query     │              │
    │            │              │  │ bookings  │              │
    │            │              │◀─┘           │              │
    │            │              │              │              │
    │            │              │──┐ detect    │              │
    │            │              │  │ conflicts │              │
    │            │              │◀─┘           │              │
    │            │              │              │              │
    │            │ isAvailable  │              │              │
    │            │◀─────────────│              │              │
    │            │              │              │              │
    │            │ (if conflict) suggestAlternatives         │
    │◀───────────│              │              │              │
    │            │              │              │              │
    │            │ (if available) create()    │              │
    │            │────────────────────────────▶│              │
    │            │              │              │              │
    │            │              │◀─────────────│ Booking      │
    │            │              │              │              │
    │            │ createEvent(booking)        │              │
    │            │───────────────────────────────────────────▶│
    │            │              │              │              │
    │            │◀───────────────────────────────────────────│ eventId
    │            │              │              │              │
    │ Booking    │              │              │              │
    │◀───────────│              │              │              │
```

---

# E — Edge Cases

## Edge Case 1: Overlapping Bookings

```swift
class AvailabilityService {
    func findConflicts(room: String, slot: TimeSlot, existingBookings: [Booking]) -> [Booking] {
        existingBookings.filter { booking in
            booking.roomId == room &&
            booking.status != .cancelled &&
            TimeSlot(start: booking.startTime, end: booking.endTime).overlaps(with: slot)
        }
    }
    
    func check(room: Room, slot: TimeSlot) async throws -> Bool {
        let existingBookings = try await fetchBookings(for: room.id, date: slot.start)
        let conflicts = findConflicts(room: room.id, slot: slot, existingBookings: existingBookings)
        return conflicts.isEmpty
    }
}
```

## Edge Case 2: Calendar Permission Denied

```swift
class CalendarService {
    func requestAccess() async throws -> Bool {
        let eventStore = EKEventStore()
        
        if #available(iOS 17.0, *) {
            return try await eventStore.requestFullAccessToEvents()
        } else {
            return try await eventStore.requestAccess(to: .event)
        }
    }
    
    func createEvent(for booking: Booking) async throws -> String {
        guard try await requestAccess() else {
            throw CalendarError.accessDenied
        }
        
        let eventStore = EKEventStore()
        let event = EKEvent(eventStore: eventStore)
        
        event.title = booking.title
        event.startDate = booking.startTime
        event.endDate = booking.endTime
        event.location = booking.roomId
        event.calendar = eventStore.defaultCalendarForNewEvents
        
        try eventStore.save(event, span: .thisEvent)
        
        return event.eventIdentifier
    }
}
```

## Edge Case 3: Recurring Booking Conflicts

```swift
func checkRecurringAvailability(
    room: Room,
    baseSlot: TimeSlot,
    recurrence: Recurrence
) async throws -> [Date: Bool] {
    var results: [Date: Bool] = [:]
    
    let occurrences = generateOccurrences(baseSlot: baseSlot, recurrence: recurrence)
    
    for occurrence in occurrences {
        let isAvailable = try await check(room: room, slot: occurrence)
        results[occurrence.start] = isAvailable
    }
    
    return results
}

private func generateOccurrences(baseSlot: TimeSlot, recurrence: Recurrence) -> [TimeSlot] {
    var occurrences: [TimeSlot] = [baseSlot]
    var currentDate = baseSlot.start
    
    let maxOccurrences = recurrence.count ?? 52  // Default 1 year
    
    while occurrences.count < maxOccurrences {
        currentDate = nextOccurrence(from: currentDate, frequency: recurrence.frequency, interval: recurrence.interval)
        
        if let endDate = recurrence.endDate, currentDate > endDate {
            break
        }
        
        let duration = baseSlot.end.timeIntervalSince(baseSlot.start)
        occurrences.append(TimeSlot(
            start: currentDate,
            end: currentDate.addingTimeInterval(duration)
        ))
    }
    
    return occurrences
}
```

---

# D — Design Trade-offs

| Approach | Pros | Cons |
|---------|------|------|
| Sync with device calendar | Single view | Permission needed |
| In-app calendar only | No permission | Separate view |
| EventKit | Native iOS | iOS only |

**Decision:** EventKit integration with fallback to in-app calendar

---

# iOS Implementation

```swift
@MainActor
class RoomBookingViewModel: ObservableObject {
    @Published private(set) var availableRooms: [Room] = []
    @Published private(set) var myBookings: [Booking] = []
    @Published private(set) var selectedDate = Date()
    @Published private(set) var isLoading = false
    @Published var error: BookingError?
    
    private let bookingFacade: BookingFacade
    
    init(bookingFacade: BookingFacade) {
        self.bookingFacade = bookingFacade
    }
    
    func searchAvailableRooms(
        slot: TimeSlot,
        capacity: Int,
        amenities: [Amenity]
    ) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            availableRooms = try await bookingFacade.findAvailableRooms(
                slot: slot,
                capacity: capacity,
                amenities: amenities
            )
        } catch {
            self.error = error as? BookingError
        }
    }
    
    func book(
        room: Room,
        slot: TimeSlot,
        title: String,
        attendees: [User]
    ) async {
        isLoading = true
        defer { isLoading = false }
        
        do {
            let booking = try await bookingFacade.bookRoom(
                room: room,
                slot: slot,
                title: title,
                attendees: attendees
            )
            myBookings.append(booking)
            
            // Haptic feedback
            UINotificationFeedbackGenerator().notificationOccurred(.success)
        } catch {
            self.error = error as? BookingError
        }
    }
    
    func cancel(_ booking: Booking) async {
        do {
            try await bookingFacade.cancelBooking(booking)
            myBookings.removeAll { $0.id == booking.id }
        } catch {
            self.error = error as? BookingError
        }
    }
}
```

---

# Interview Tips

## What to Say

```markdown
1. "Facade pattern hides complexity of booking flow..."
2. "Strategy pattern for different conflict resolution..."
3. "EventKit for native calendar integration..."
4. "Conflict detection checks overlapping time slots..."
```

## Red Flags to Avoid

```markdown
❌ "Simple if-else for conflict checking"
   → Doesn't scale with strategies

❌ Forgetting calendar permission handling
   → iOS-specific requirement

❌ No recurring booking support
   → Common real-world requirement
```

---

*This is how an Amazon iOS LLD interview expects you to approach the Meeting Room Booking problem!*
