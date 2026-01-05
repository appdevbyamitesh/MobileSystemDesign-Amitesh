# Memory Management with ARC: Deep Dive Notes for iOS Interviews

## Theory & Fundamentals

### What is Automatic Reference Counting (ARC)?

ARC is Swift's memory management technique that automatically tracks and manages memory usage of class instances. When you create a class instance, ARC allocates memory for it and keeps track of how many references exist to that instance through reference counting. When the reference count reaches zero, ARC automatically deallocates the memory, making it available for new allocations.

### How ARC Works

ARC maintains a counter in the object's header metadata stored in heap memory. Every time you create a reference to an instance, the count increases by 1. When a reference is removed or goes out of scope, the count decreases. When the count reaches zero, the `deinit` method is called and memory is freed.

### ARC vs Garbage Collection

Unlike garbage collection, ARC does not handle reference cycles automatically and eliminates performance issues like periodic GC pauses. ARC performs reference counting at compile time, making it more predictable and efficient for iOS devices with constrained resources.

## Reference Types in Swift

### Strong References (Default)

Strong references are the default in Swift and increase the reference count by 1. An object remains in memory as long as there's at least one strong reference to it.

```swift
class Person {
    let name: String
    
    init(name: String) {
        self.name = name
        print("\(name) is initialized")
    }
    
    deinit {
        print("\(name) is being deinitialized")
    }
}

var person1: Person? = Person(name: "Alice") // Reference count = 1
var person2 = person1 // Reference count = 2
person1 = nil // Reference count = 1
person2 = nil // Reference count = 0, deinit is called
```

### Weak References

Weak references do not increase the reference count and are always optional. They are used when the referenced object might be deallocated before the referencing object.

```swift
class Person {
    var name: String
    weak var friend: Person? // Must be optional
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) deallocated")
    }
}

var alice: Person? = Person(name: "Alice")
var bob: Person? = Person(name: "Bob")

alice?.friend = bob // Weak reference doesn't increase bob's count
bob?.friend = alice

alice = nil // Alice can be deallocated even though bob.friend references it
bob = nil // Bob can now be deallocated
```

**Key characteristics of weak references**:

- Always optional types
- Must be declared with `var` (not `let`)
- Automatically set to `nil` when the referenced object is deallocated
- Do not prevent the referenced object from being deallocated

### Unowned References

Unowned references also don't increase reference count but are non-optional and assume the referenced object always exists. Accessing an unowned reference after its object is deallocated causes a runtime crash.

```swift
class Customer {
    var name: String
    var card: CreditCard?
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) deallocated")
    }
}

class CreditCard {
    let number: String
    unowned let customer: Customer // Non-optional
    
    init(number: String, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    
    deinit {
        print("Card \(number) deallocated")
    }
}

var john: Customer? = Customer(name: "John")
john?.card = CreditCard(number: "1234-5678-9876", customer: john!)
john = nil // Both john and card are deallocated
```

### Weak vs Unowned: When to Use What

| Aspect | Weak | Unowned |
|--------|------|---------|
| Optional | Always optional | Non-optional |
| Can become nil | Yes | No (crashes if accessed after deallocation) |
| Use case | Referenced object may be deallocated first | Referenced object lives at least as long as referencing object |
| Safety | Safer, returns nil | Risky if lifetime assumptions are wrong |

## Retain Cycles (Reference Cycles)

### What is a Retain Cycle?

A retain cycle occurs when two or more objects hold strong references to each other, creating a circular dependency that prevents ARC from deallocating them. This leads to memory leaks where allocated memory is never released.

### Classic Retain Cycle Example

```swift
class Person {
    var name: String
    var dog: Dog?
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("\(name) deallocated")
    }
}

class Dog {
    var name: String
    var owner: Person? // Strong reference creates cycle
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("Dog \(name) deallocated")
    }
}

func createPersonAndDog() {
    var alice = Person(name: "Alice")
    var fido = Dog(name: "Fido")
    
    alice.dog = fido // Person -> Dog strong reference
    fido.owner = alice // Dog -> Person strong reference
} // Neither deinit is called - MEMORY LEAK!
```

### Breaking Retain Cycles

```swift
class Dog {
    var name: String
    weak var owner: Person? // Weak breaks the cycle
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("Dog \(name) deallocated")
    }
}

func createPersonAndDog() {
    var alice = Person(name: "Alice")
    var fido = Dog(name: "Fido")
    
    alice.dog = fido
    fido.owner = alice
} // Both deinit are called - NO LEAK!
```

## Closures and Capture Lists

### Retain Cycles with Closures

Closures capture strong references to `self` by default, creating potential retain cycles.

```swift
class ViewController {
    var name: String = "MainVC"
    var buttonAction: (() -> Void)?
    
    func setupButton() {
        // WRONG: Creates retain cycle
        buttonAction = {
            print("Button pressed in \(self.name)")
            self.doSomething()
        }
    }
    
    func doSomething() {
        print("Doing something...")
    }
    
    deinit {
        print("\(name) deallocated")
    }
}
```

### Breaking Closure Retain Cycles

**Using [weak self]** (safest approach):

```swift
class ViewController {
    var name: String = "MainVC"
    var buttonAction: (() -> Void)?
    
    func setupButton() {
        buttonAction = { [weak self] in
            guard let self = self else { return }
            print("Button pressed in \(self.name)")
            self.doSomething()
        }
    }
    
    deinit {
        print("\(name) deallocated")
    }
}
```

**Using [unowned self]** (use when self will always exist):

```swift
class ViewController {
    var buttonAction: (() -> Void)?
    
    func setupButton() {
        buttonAction = { [unowned self] in
            // No need for optional unwrapping
            print("Button pressed")
            self.doSomething()
        }
    }
}
```

### Capturing Multiple References

```swift
class NetworkManager {
    func fetchData(completion: @escaping () -> Void) {
        // ...
    }
}

class ViewModel {
    var manager = NetworkManager()
    var delegate: ViewDelegate?
    
    func loadData() {
        manager.fetchData { [weak self, weak delegate = self.delegate] in
            guard let self = self else { return }
            // Use self and delegate safely
            delegate?.dataDidLoad()
        }
    }
}
```

## Delegate Pattern and Memory Management

### Wrong Way (Creates Retain Cycle)

```swift
protocol TaskDelegate {
    func taskCompleted()
}

class TaskManager {
    var delegate: TaskDelegate? // Strong reference
}

class ViewController: UIViewController, TaskDelegate {
    var taskManager: TaskManager?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        taskManager = TaskManager()
        taskManager?.delegate = self // Cycle created
    }
    
    func taskCompleted() {
        print("Task completed")
    }
}
```

### Correct Way (Using Weak Delegate)

```swift
protocol TaskDelegate: AnyObject { // Must be class-only protocol
    func taskCompleted()
}

class TaskManager {
    weak var delegate: TaskDelegate? // Weak prevents cycle
}

class ViewController: UIViewController, TaskDelegate {
    var taskManager: TaskManager?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        taskManager = TaskManager()
        taskManager?.delegate = self // No cycle
    }
    
    func taskCompleted() {
        print("Task completed")
    }
    
    deinit {
        print("ViewController deallocated")
    }
}
```

## Advanced Scenarios

### Nested Dependencies

```swift
class Person {
    var name: String
    var dog: Dog?
    
    init(name: String, dog: Dog?) {
        self.name = name
        self.dog = dog
    }
    
    deinit {
        print("\(name) deallocated")
    }
}

class Dog {
    var name: String
    
    init(name: String) {
        self.name = name
    }
    
    deinit {
        print("Dog \(name) deallocated")
    }
}

func createPerson() {
    var alice = Person(name: "Alice", dog: nil) // alice count = 1
    var fido = Dog(name: "Fido") // fido count = 1
    
    alice.dog = fido // fido count = 2
} 
/* Deallocation sequence:
1. alice count drops to 0, alice is deallocated
2. fido count drops from 2 to 1 (alice.dog released)
3. fido count drops to 0, fido is deallocated
*/
```

### Using autoreleasepool for Memory-Intensive Operations

```swift
func processLargeDataSet() {
    for i in 0..<10000 {
        autoreleasepool {
            let largeObject = LargeDataObject(data: generateData())
            process(largeObject)
            // largeObject released at end of autoreleasepool
        }
    }
}
```

## Interview Questions & Answers

### Q1: What is ARC and how does it differ from garbage collection?

**Answer:** ARC (Automatic Reference Counting) is Swift's memory management technique that tracks the number of strong references to class instances at compile time. When the reference count reaches zero, memory is immediately deallocated. Unlike garbage collection, ARC doesn't handle reference cycles automatically and requires developers to use weak/unowned references to break cycles. ARC has no runtime overhead or periodic pause times like garbage collectors, making it more suitable for iOS devices with limited resources.

### Q2: Explain the difference between weak and unowned references. When would you use each?

**Answer:** Both weak and unowned references don't increase the reference count, but they differ in optionality and safety. Weak references are always optional and automatically become nil when the referenced object is deallocated. Unowned references are non-optional and assume the referenced object will always exist during the referencing object's lifetime. Use weak when the referenced object might be deallocated first (like delegates), and use unowned when you're certain the referenced object will outlive the referencing object (like a CreditCard always having a Customer). Accessing a deallocated unowned reference causes a crash.

### Q3: What is a retain cycle and how do you identify and fix it?

**Answer:** A retain cycle occurs when two or more objects hold strong references to each other, preventing ARC from deallocating them and causing memory leaks. Common scenarios include two-way relationships between objects, closures capturing self strongly, and delegate patterns without weak references. To identify retain cycles, use Xcode's Memory Graph Debugger (shows purple dots for cycles) or Instruments' Leaks tool. Fix them by making one reference weak or unowned, using capture lists `[weak self]` or `[unowned self]` in closures, and declaring delegates as weak with class-only protocols.

### Q4: Why must delegates be declared as weak? Show an example

**Answer:** Delegates must be declared weak to prevent retain cycles. When a parent object holds a strong reference to a child object that acts as its delegate, and the child holds a strong reference back to the parent through the delegate property, neither can be deallocated. Example:

```swift
protocol TaskDelegate: AnyObject {
    func taskCompleted()
}

class TaskManager {
    weak var delegate: TaskDelegate? // Prevents cycle
}

class ViewController: TaskDelegate {
    var manager = TaskManager() // VC -> Manager (strong)
    
    init() {
        manager.delegate = self // Manager -> VC (weak, no cycle)
    }
}
```

The protocol must conform to `AnyObject` to restrict it to class types, allowing weak references.

### Q5: What happens if you try to use a weak reference after the object it references has been deallocated?

**Answer:** When a weak reference's object is deallocated, the weak reference is automatically set to nil. This is safe and won't cause crashes. You must unwrap weak references since they're always optional. Example:

```swift
weak var person: Person? = Person(name: "Alice")
print(person?.name) // Optional("Alice")
person = nil // Object deallocated
print(person?.name) // nil (safe)
```

### Q6: Explain closure capture lists. Why do we need them?

**Answer:** Closure capture lists allow you to specify how closures capture references to variables, particularly self. Without capture lists, closures create strong references to captured objects by default, causing retain cycles when the object holds the closure. Use `[weak self]` when the closure might outlive the object, or `[unowned self]` when you're certain the object will outlive the closure. Example:

```swift
class DataManager {
    var completion: (() -> Void)?
    
    func loadData() {
        completion = { [weak self] in
            guard let self = self else { return }
            self.processData() // Safe, no retain cycle
        }
    }
}
```

### Q7: When would you use unowned instead of weak in a closure?

**Answer:** Use `[unowned self]` when you're absolutely certain that self will exist for the closure's entire lifetime and you want cleaner code without optional unwrapping. This is common in non-escaping closures or when the closure's lifetime is tied to the object's lifetime. However, weak is generally safer because accessing deallocated unowned references causes crashes. Example:

```swift
class ViewController {
    func setupAnimation() {
        UIView.animate(withDuration: 0.3) { [unowned self] in
            self.view.alpha = 0.5 // Safe: animation completes before VC deallocates
        }
    }
}
```

### Q8: How do you debug memory leaks in iOS? Name specific tools

**Answer:** Use multiple Xcode tools:

1. **Memory Graph Debugger**: Click the debug memory graph button during runtime to visualize all objects and their references; purple dots indicate retain cycles
2. **Instruments - Leaks**: Profile with Product > Profile, select Leaks template to detect memory leaks with stack traces showing allocation points
3. **Instruments - Allocations**: Monitor memory allocations, object lifetimes, and identify objects not being deallocated
4. **Zombies Instrument**: Detects over-released objects by replacing deallocated objects with "zombies"

Regular profiling during development helps catch issues early.

### Q9: What is the purpose of autoreleasepool in Swift?

**Answer:** `autoreleasepool` manages temporary objects' memory more efficiently in memory-intensive operations. It releases objects within its scope immediately when the block ends, reducing peak memory usage. This is especially useful in loops processing large datasets:

```swift
for i in 0..<10000 {
    autoreleasepool {
        let image = processImage(i)
        save(image)
        // image released here, not at end of entire loop
    }
}
```

Without autoreleasepool, all 10,000 images would accumulate in memory.

### Q10: Can value types (structs, enums) cause retain cycles?

**Answer:** No, value types cannot cause retain cycles because they don't use reference counting. Structs and enums are copied when assigned, not referenced, so ARC doesn't track them. However, if a struct contains a closure that captures self (when used in a class context), or if a struct property is a reference type (class instance), those can still create retain cycles. Only reference types (classes) participate in ARC and can form retain cycles.

### Q11: How does ARC handle multi-threaded environments?

**Answer:** ARC handles basic atomic operations on reference counts automatically, but multi-threaded access to shared objects requires explicit synchronization to avoid race conditions. Best practices include: using serial dispatch queues for sequential access, NSLock or other synchronization primitives for critical sections, and atomic property wrappers for thread-safe access. Example:

```swift
let serialQueue = DispatchQueue(label: "com.app.queue")
var sharedResource = SharedData()

serialQueue.async {
    sharedResource.data.append("Task 1") // Thread-safe
}
```

Without synchronization, concurrent reference count modifications can cause undefined behavior or crashes.

### Q12: What's the relationship between ARC and deinit?

**Answer:** The `deinit` method is called automatically by ARC immediately before an instance is deallocated, when its reference count reaches zero. You can't call deinit manually. Use deinit to perform cleanup tasks like invalidating timers, removing observers, or closing file handles. If deinit isn't called when expected, it indicates a retain cycle preventing deallocation. Example:

```swift
class DatabaseConnection {
    deinit {
        closeConnection()
        print("Connection closed")
    }
}
```

Deinit helps verify that ARC is properly managing memory.

These comprehensive notes cover the essential concepts, implementations, and interview questions for memory management with ARC in Swift.
