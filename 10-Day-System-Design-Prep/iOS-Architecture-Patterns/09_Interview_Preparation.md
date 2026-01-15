# 🎯 iOS Architecture Interview Preparation

> **Real Questions, Strong Answers, and Common Pitfalls**

---

## 📋 Table of Contents

1. [Architecture Decision Questions](#architecture-decision-questions)
2. [Pattern-Specific Deep Dives](#pattern-specific-deep-dives)
3. [System Design + Architecture Questions](#system-design--architecture-questions)
4. [Behavioral Architecture Questions](#behavioral-architecture-questions)
5. [Common Mistakes to Avoid](#common-mistakes-to-avoid)
6. [Defense Strategies](#defense-strategies)

---

## 🏗️ Architecture Decision Questions

### Q1: "How do you choose an architecture for a new iOS project?"

**Strong Answer:**

> "I evaluate four key factors:
>
> 1. **Team size** — Larger teams need stricter boundaries. Solo dev? MVC is fine. 10+ devs? Clean Architecture or VIPER.
>
> 2. **App complexity** — Simple CRUD? MVVM. Complex state management? Clean Architecture. Uber-scale with viewless business logic? RIBs.
>
> 3. **Testing requirements** — If we need 70%+ coverage, I rule out MVC. MVVM is minimum, MVVM+C or Clean Architecture is better.
>
> 4. **Team familiarity** — I won't force VIPER on a team that's never used it with a tight deadline. We'd ship faster with MVVM+C.
>
> For most cases, I start with MVVM+C because it balances testability, navigation decoupling, and pragmatism. We can evolve to Clean Architecture if complexity grows."

**Follow-up probe:** "What if the team wants to use RIBs?"

> "I'd ask: Do we have 30+ iOS developers? Is the app truly complex with viewless state machines? Is the team experienced with reactive programming? If no to any, I'd suggest Clean Architecture as a stepping stone—it's 80% of RIBs' benefits with 50% of the complexity."

---

### Q2: "You're building a new e-commerce app. What architecture would you choose and why?"

**Strong Answer:**

> "For e-commerce, I'd choose **Clean Architecture with Coordinators** because:
>
> 1. **Complex navigation** — Product list → Detail → Cart → Checkout → Payment → Confirmation. Coordinators handle this cleanly.
>
> 2. **Reusable business logic** — Cart, Payment, and Inventory logic need to be shared across features. Use Cases in the Domain layer enable this.
>
> 3. **Multiple data sources** — Products from API, cart from local persistence, user session from Keychain. Repository pattern handles this elegantly.
>
> 4. **Testability** — Checkout flow must be thoroughly tested. Clean Architecture lets me test Use Cases without UI.
>
> I'd structure it as:
> - **Presentation**: Feature-based Coordinators + MVVM
> - **Domain**: Shared Use Cases (GetProductsUseCase, AddToCartUseCase, CheckoutUseCase)
> - **Data**: Repositories for Products, Cart, Orders, User"

---

### Q3: "When would you NOT use MVVM?"

**Strong Answer:**

> "I'd skip MVVM in three scenarios:
>
> 1. **Prototypes** — If I'm building a hackathon demo, MVC is faster. No need for ViewModel abstraction.
>
> 2. **Very simple apps** — A settings screen with 5 toggles doesn't need a ViewModel. Putting toggle handling in ViewController is acceptable.
>
> 3. **When RIBs/VIPER is mandated** — Some companies have standardized on these. MVVM inside VIPER's Presenter layer is redundant.
>
> That said, for any production app with 10+ screens, I'd use MVVM as a minimum because:
> - It's SwiftUI-ready
> - It enables unit testing
> - It keeps ViewControllers small"

---

### Q4: "How do you handle navigation in iOS architectures?"

**Strong Answer:**

> "Navigation belongs in a dedicated layer, not in ViewControllers or ViewModels:
>
> **MVC/MVVM (default):** Navigation in ViewController → Hard to test, hard to reuse
>
> **MVVM+C:** Coordinator creates VCs, handles push/present → Testable, reusable
>
> **VIPER:** Router handles navigation per module → Strict isolation
>
> **RIBs:** Router attaches/detaches child RIBs → Tree-based state management
>
> I prefer Coordinators because:
> 1. ViewModels stay UIKit-free (fully testable)
> 2. Deep linking is straightforward (Coordinator can navigate directly)
> 3. Same ViewController works in different flows
>
> Example structure:
> ```swift
> class AppCoordinator {
>     func start() {
>         if isLoggedIn { showMain() }
>         else { showAuth() }
>     }
> }
> ```"

---

### Q5: "What's the difference between VIPER and Clean Architecture?"

**Strong Answer:**

> "They share principles but differ in focus:
>
> | Aspect | VIPER | Clean Architecture |
> |--------|-------|-------------------|
> | **Unit** | Per-screen module | Per-layer (Presentation/Domain/Data) |
> | **Business logic** | Interactor per module | Use Cases (shared) |
> | **Navigation** | Router per module | Coordinators (presentation layer) |
> | **Reuse** | Limited across modules | High via shared Domain |
>
> **VIPER** is better when:
> - Strict per-screen isolation is required
> - Using code generation templates
> - Enterprise with compliance audits
>
> **Clean Architecture** is better when:
> - Business logic is shared across features
> - Building with SwiftUI
> - Domain-driven design mindset
> - Want flexibility over prescription"

---

## 🔬 Pattern-Specific Deep Dives

### Q6: "Why does Massive View Controller happen in MVC?"

**Strong Answer:**

> "MVC has no clear place for three things:
>
> 1. **Business logic** — 'Where does validation go?' → Controller!
> 2. **Data transformation** — 'Where do I format dates?' → Controller!
> 3. **Navigation** — 'Where do I push the next screen?' → Controller!
>
> The result is a Controller with 500+ lines handling:
> - TableView delegate/datasource
> - Networking
> - Data mapping
> - Navigation
> - Error handling
>
> MVVM fixes this by extracting logic to ViewModel. MVVM+C goes further by extracting navigation to Coordinator."

---

### Q7: "How does data binding work in MVVM with Combine?"

**Strong Answer:**

> "The ViewModel exposes `@Published` properties. The View subscribes via Combine:
>
> ```swift
> // ViewModel
> class FeedViewModel: ObservableObject {
>     @Published var posts: [Post] = []
>     @Published var isLoading = false
> }
>
> // ViewController
> viewModel.$posts
>     .receive(on: DispatchQueue.main)
>     .sink { [weak self] posts in
>         self?.tableView.reloadData()
>     }
>     .store(in: &cancellables)
> ```
>
> Key benefits:
> - Automatic UI updates when state changes
> - No manual `updateUI()` calls
> - Works seamlessly with SwiftUI's `@StateObject`
>
> Common pitfall: Forgetting `[weak self]` in sink closure → retain cycle."

---

### Q8: "Explain the Coordinator pattern and when you'd use it."

**Strong Answer:**

> "Coordinators handle navigation flow, removing this responsibility from ViewControllers:
>
> **Responsibilities:**
> - Create ViewControllers and inject dependencies
> - Push, present, dismiss screens
> - Manage child flows (e.g., AuthCoordinator, CheckoutCoordinator)
>
> **Example:**
> ```swift
> class CheckoutCoordinator {
>     func showCart() { ... }
>     func showPayment() { ... }
>     func showConfirmation() { ... }
>     func finishFlow() {
>         parentCoordinator?.childDidFinish(self)
>     }
> }
> ```
>
> **When to use:**
> - Complex navigation (tabs, modals, nested stacks)
> - Deep linking requirements
> - Reusable ViewControllers in different contexts
>
> **When NOT to use:**
> - Simple 3-5 screen app with linear flow"

---

### Q9: "What's a viewless RIB and why would you use one?"

**Strong Answer:**

> "A viewless RIB has Router + Interactor but no View/Presenter. It manages business state without UI.
>
> **Examples:**
> - **AuthStateRIB** — Tracks logged-in state, manages token refresh
> - **LocationRIB** — GPS tracking during trip, no dedicated UI
> - **AnalyticsRIB** — Tracks user journey across the app
> - **FeatureFlagRIB** — Controls which features are active
>
> **Why viewless?**
> - State machines that outlive individual screens
> - Cross-cutting concerns (auth, analytics)
> - Background operations (sync, notifications)
>
> This is RIBs' unique strength: the RIB tree represents *app state*, not just *screen hierarchy*."

---

### Q10: "How do Use Cases work in Clean Architecture?"

**Strong Answer:**

> "Use Cases represent single business actions. They live in the Domain layer:
>
> ```swift
> class LikePostUseCase {
>     private let repository: PostRepositoryProtocol
>     
>     func execute(postId: String) async -> Result<Post, DomainError> {
>         // 1. Validate business rules
>         // 2. Call repository
>         // 3. Return domain result
>     }
> }
> ```
>
> **Key principles:**
> - One use case = one action (Single Responsibility)
> - Returns domain types, not DTOs
> - Knows nothing about UI or network specifics
>
> **ViewModel usage:**
> ```swift
> let result = await likePostUseCase.execute(postId)
> switch result {
>     case .success(let post): updateUI(post)
>     case .failure(let error): showError(error)
> }
> ```
>
> **Why Use Cases?**
> - Testable without UI
> - Reusable across ViewModels
> - Clear API for business operations"

---

## 🖥️ System Design + Architecture Questions

### Q11: "Design a social media feed. Walk me through the architecture."

**Strong Answer:**

> "I'll use Clean Architecture with MVVM+C:
>
> **Presentation Layer:**
> - `FeedCoordinator` — Handles navigation to post detail, user profile
> - `FeedViewModel` — Manages state (loading, posts, pagination)
> - `FeedViewController` — Binds to ViewModel, displays TableView
>
> **Domain Layer:**
> - `GetFeedUseCase` — Fetches paginated posts
> - `LikePostUseCase` — Handles like with optimistic update
> - `Post` (Entity) — Pure domain model
>
> **Data Layer:**
> - `PostRepository` — Coordinates API + local cache
> - `PostAPIService` — Network calls
> - `PostCache` — NSCache + disk persistence
>
> **Key decisions:**
> 1. **Pagination:** Cursor-based, triggered at 3 items from bottom
> 2. **Optimistic updates:** Like appears instantly, reverts on failure
> 3. **Caching:** First page cached, refreshed in background
>
> **Why this architecture?**
> - Testable at every layer
> - Use Cases are reusable (LikePostUseCase used in feed, detail, search)
> - Repository hides complexity of cache + API coordination"

---

### Q12: "How would you architect an offline-first app?"

**Strong Answer:**

> "Offline-first requires the Data layer to prioritize local storage:
>
> **Repository Strategy:**
> ```swift
> class TripRepository {
>     func getTrips() async throws -> [Trip] {
>         // 1. Return local data immediately
>         let local = try await database.getTrips()
>         
>         // 2. Refresh in background
>         Task {
>             let remote = try await api.getTrips()
>             await database.save(remote)
>             // Notify observers of update
>         }
>         
>         return local
>     }
> }
> ```
>
> **Sync strategy:**
> - Track pending changes in local queue
> - Retry on network restoration
> - Conflict resolution (last-write-wins or merge)
>
> **Architecture impact:**
> - Repository becomes more complex
> - Need observable pattern for data updates
> - Domain layer unchanged (doesn't know about offline)
>
> **This is Clean Architecture's strength:** Offline logic is isolated to Data layer. Use Cases and UI are oblivious."

---

## 🗣️ Behavioral Architecture Questions

### Q13: "Tell me about a time you had to migrate architectures."

**Strong Answer:**

> "At [Company], we migrated from MVC to MVVM+C over 6 months:
>
> **Problem:** 8 ViewControllers over 500 lines. Testing was impossible. New features took weeks.
>
> **Approach:**
> 1. New features in MVVM+C from day one
> 2. Extracted shared networking into Services first
> 3. High-churn screens migrated next (highest ROI)
> 4. Stable legacy screens left as MVC (not worth the risk)
>
> **Challenges:**
> - Team pushback ('Why change what works?')
> - Mixing patterns in same codebase
> - Coordinator lifecycle with legacy VCs
>
> **Results:**
> - Test coverage: 5% → 45%
> - Feature velocity: 2x faster
> - New dev onboarding: 2 weeks → 1 week
>
> **Lesson:** Gradual migration > big bang rewrite. Let the two patterns coexist."

---

### Q14: "How do you handle disagreements about architecture on your team?"

**Strong Answer:**

> "I focus on objective criteria, not opinions:
>
> 1. **Define success metrics** — What are we optimizing for? Testability? Development speed? Onboarding?
>
> 2. **Prototype both approaches** — Build the same feature both ways. Compare lines of code, test coverage, time spent.
>
> 3. **Consider constraints** — Team experience, deadline, existing codebase.
>
> **Example:** Team debated VIPER vs MVVM+C. I proposed:
> - Build one feature each way
> - Compare: files created, test coverage achieved, time to implement
>
> MVVM+C won because:
> - 40% less code
> - Same test coverage
> - Team was faster (familiar pattern)
>
> **Key:** Architecture is a tool, not a religion. Choose what fits the context."

---

## ❌ Common Mistakes to Avoid

### Mistake 1: Over-engineering

```
❌ "I always use VIPER because it's the most structured."

✅ "I choose VIPER when I need strict isolation for a large team. 
   For smaller teams, MVVM+C gives 80% of the benefit with less overhead."
```

### Mistake 2: Under-engineering

```
❌ "MVC is fine. Apple uses it."

✅ "MVC works for simple apps, but I've seen it fail at scale. 
   For apps over 10 screens, I move to MVVM minimum for testability."
```

### Mistake 3: Ignoring team context

```
❌ "RIBs is the best architecture, so I'd use it."

✅ "RIBs is powerful, but it requires reactive programming expertise and 
   significant infrastructure. I'd validate the team can support it first."
```

### Mistake 4: Not justifying trade-offs

```
❌ "I'd use Clean Architecture."

✅ "I'd use Clean Architecture because it separates business logic 
   (testable Use Cases) from infrastructure (swappable repositories). 
   The trade-off is more boilerplate, which is acceptable for 
   a long-lived product."
```

### Mistake 5: Forgetting navigation

```
❌ "MVVM handles everything."

✅ "MVVM handles state and business logic, but navigation is awkward. 
   I'd pair it with Coordinators to decouple navigation from ViewModels."
```

---

## 🛡️ Defense Strategies

### When Challenged on Complexity

**Interviewer:** "Isn't Clean Architecture overkill?"

**Strong Response:**
> "It can be. I wouldn't use it for a 5-screen app. But the question asked about a large e-commerce platform with 30 screens, multiple data sources, and a team of 15. In that context, the structure pays for itself in maintainability and testability."

### When Challenged on Simplicity

**Interviewer:** "Why not just use MVC? It's simpler."

**Strong Response:**
> "MVC is simpler initially, but it creates long-term pain. I've worked on MVC codebases with 1000+ line ViewControllers. Testing was impossible, onboarding was slow, and changes were risky. The upfront investment in MVVM or Clean Architecture prevents that technical debt."

### When Your Approach is Questioned

**Interviewer:** "Why not use [X] instead?"

**Strong Response:**
> "Good question. [X] is also valid here. I chose [Y] because [specific reason: testability, team familiarity, navigation complexity]. If those constraints were different—for example, if the team had deep [X] experience—I'd reconsider."

### When You Don't Know Something

**Interviewer:** "How does RIBs handle [obscure detail]?"

**Strong Response:**
> "I haven't worked with that specific aspect of RIBs. From what I know, RIBs uses [general principle]. If this were a real project, I'd dive into Uber's documentation and open-source examples to understand it fully before implementing."

---

## 🎓 Key Takeaways for Interviews

1. **Always ask clarifying questions** before choosing an architecture
2. **Justify trade-offs explicitly** — Every choice has costs and benefits
3. **Show flexibility** — "I would start with X, but if Y changes, I'd reconsider"
4. **Demonstrate experience** — Share real examples of migration, trade-offs, failures
5. **Know when NOT to use each pattern** — This shows maturity

### Quick Reference Card

| If Asked About | Mention |
|----------------|---------|
| **MVC** | Massive VC problem, when it's acceptable |
| **MVVM** | Testability, Combine binding, SwiftUI fit |
| **MVVM+C** | Navigation decoupling, deep linking |
| **VIPER** | Strict boundaries, boilerplate cost |
| **RIBs** | Viewless RIBs, tree-based state, scale |
| **Clean Arch** | Dependency inversion, layers, Use Cases |
