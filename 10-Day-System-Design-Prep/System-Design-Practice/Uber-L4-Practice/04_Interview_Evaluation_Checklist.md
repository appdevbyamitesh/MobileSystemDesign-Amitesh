# Uber L4 iOS Interview - Evaluation Checklist

> [!IMPORTANT]
> Use this checklist to evaluate your mock interview performance. Uber L4 = Senior iOS Engineer expectations.

---

## Interview Performance Scorecard

### Category 1: Requirements Clarification (10 points)

| Criteria | Points | Self-Score |
|----------|--------|------------|
| Asked about functional requirements (features, user flows) | 2 | __/2 |
| Asked about non-functional requirements (scale, performance, offline) | 2 | __/2 |
| Clarified constraints and assumptions (iOS version, battery, data) | 2 | __/2 |
| Documented agreed scope (in/out of scope) | 2 | __/2 |
| Managed time well (8-10 minutes, not rushed) | 2 | __/2 |
| **Total** | **10** | **__/10** |

---

### Category 2: High-Level Design (20 points)

| Criteria | Points | Self-Score |
|----------|--------|------------|
| Drew clear architecture diagram (layers, components) | 4 | __/4 |
| Explained architecture pattern choice (MVVM, Clean, etc.) with justification | 3 | __/3 |
| Showed data flow with arrows (user action → response) | 4 | __/4 |
| Separated concerns (UI, ViewModel, Repository, Network, Cache) | 3 | __/3 |
| Discussed offline-first or caching strategy | 3 | __/3 |
| Mentioned alternatives and trade-offs | 3 | __/3 |
| **Total** | **20** | **__/20** |

---

### Category 3: Low-Level Design (25 points)

| Criteria | Points | Self-Score |
|----------|--------|------------|
| Defined clear data models (structs/classes with proper types) | 4 | __/4 |
| Wrote production-quality code (not pseudocode) | 5 | __/5 |
| Used protocols for dependency injection and testability | 4 | __/4 |
| Showed proper async/await or concurrency handling | 4 | __/4 |
| Implemented thread safety (actors, DispatchQueue, locks) | 4 | __/4 |
| Handled errors and edge cases (retry, fallback, cancellation) | 4 | __/4 |
| **Total** | **25** | **__/25** |

---

### Category 4: Deep Technical Knowledge (20 points)

| Criteria | Points | Self-Score |
|----------|--------|------------|
| Discussed caching strategy (memory vs disk, TTL, eviction) | 5 | __/5 |
| Explained pagination approach (cursor vs offset) with justification | 4 | __/4 |
| Covered performance optimizations (prefetching, lazy loading, batching) | 4 | __/4 |
| Addressed scalability (large datasets, memory limits) | 4 | __/4 |
| Showed understanding of iOS-specific APIs (NSCache, CoreData, URLSession) | 3 | __/3 |
| **Total** | **20** | **__/20** |

---

### Category 5: Communication & Problem Solving (15 points)

| Criteria | Points | Self-Score |
|----------|--------|------------|
| Explained thought process out loud (not silent coding) | 3 | __/3 |
| Asked clarifying questions when uncertain | 3 | __/3 |
| Listened to interviewer feedback and adapted | 3 | __/3 |
| Organized approach (didn't jump around randomly) | 3 | __/3 |
| Managed time effectively (finished in 55 minutes) | 3 | __/3 |
| **Total** | **15** | **__/15** |

---

### Category 6: Trade-offs & Alternatives (10 points)

| Criteria | Points | Self-Score |
|----------|--------|------------|
| For each major decision, mentioned alternatives | 3 | __/3 |
| Clearly stated pros AND cons of chosen approach | 3 | __/3 |
| Justified choices with context (scale, team, constraints) | 2 | __/2 |
| Showed awareness of production considerations (monitoring, debugging) | 2 | __/2 |
| **Total** | **10** | **__/10** |

---

## Overall Score

| Category | Max | Your Score |
|----------|-----|------------|
| Requirements Clarification | 10 | __ |
| High-Level Design | 20 | __ |
| Low-Level Design | 25 | __ |
| Deep Technical Knowledge | 20 | __ |
| Communication & Problem Solving | 15 | __ |
| Trade-offs & Alternatives | 10 | __ |
| **TOTAL** | **100** | **__/100** |

---

## Scoring Guide

| Score | Level | Interpretation |
|-------|-------|----------------|
| **85-100** | 🟢 Strong Hire | Exceeds Uber L4 expectations, likely L5 performance |
| **70-84** | 🟢 Hire | Solid L4, would pass interview |
| **60-69** | 🟡 Borderline | Needs improvement in 1-2 areas, might pass |
| **50-59** | 🔴 No Hire | Significant gaps, needs more practice |
| **< 50** | 🔴 Strong No Hire | Not ready for L4 interviews |

---

## Red Flags (Instant Fail Signals)

Check if you did any of these. **Even ONE is a major concern:**

- [ ] Jumped straight to code without HLD diagram
- [ ] Could not explain trade-offs when asked "why this approach?"
- [ ] Ignored offline/caching (mobile-first thinking missing)
- [ ] No error handling or retry logic
- [ ] Thread safety issues (race conditions, crashes)
- [ ] Over-engineered solution (VIPER for simple feature)
- [ ] Couldn't answer basic iOS questions (URLSession, NSCache)
- [ ] Ran out of time before completing LLD
- [ ] Didn't ask ANY requirements questions
- [ ] Silent for long periods (not explaining thought process)

> [!CAUTION]
> If you checked 2+ red flags, focus on fundamentals before practicing more designs.

---

## Common L4 Gap Analysis

### If You Scored Low in Requirements (< 7/10):

**Practice:**
- Memorize the template questions from [00_HLD_LLD_Approach_Guide.md](./00_HLD_LLD_Approach_Guide.md)
- Set a timer for 10 minutes, don't go over
- Write down scope explicitly before starting HLD

### If You Scored Low in HLD (< 15/20):

**Practice:**
- Draw architecture diagrams for 5 different apps (Uber, Instagram, Notes, Weather, Maps)
- For each, explain out loud WHY you chose that architecture
- Practice data flow diagrams (request → response paths)

### If You Scored Low in LLD (< 18/25):

**Practice:**
- Write more Swift code by hand (not just reading)
- Study production code: [Swift official examples](https://github.com/apple/swift)
- Focus on protocols, actors, async/await
- Do coding challenges on LeetCode (Easy/Medium)

### If You Scored Low in Deep Knowledge (< 14/20):

**Study Topics:**
- NSCache vs custom cache: When to use which
- Cursor vs offset pagination: Database implications
- Memory management: Strong vs weak, retain cycles
- Threading: GCD, actors, OperationQueue
- iOS performance: Instruments, time profiler

### If You Scored Low in Communication (< 11/15):

**Practice:**
- Record yourself doing a mock interview (video)
- Watch it back: Are you explaining or just coding silently?
- Practice with a friend/peer who can interrupt with questions
- Join mock interview platforms (Pramp, interviewing.io)

---

## Follow-Up Questions Performance

Rate your performance on these common follow-ups:

| Question Category | Answered Well? | Notes |
|-------------------|----------------|-------|
| "Why did you choose X over Y?" | ☐ | |
| "What are the trade-offs?" | ☐ | |
| "How would you test this?" | ☐ | |
| "What if we had 10x more users?" | ☐ | |
| "How would you debug this in production?" | ☐ | |
| "What about battery/memory?" | ☐ | |
| "How does this handle offline?" | ☐ | |
| "What if the network is slow (2G)?" | ☐ | |

**Target:** Should answer 7/8 confidently for L4.

---

## Interviewer Perspective: What L4 Expects

### Must-Haves (Dealbreakers if missing):

✅ **Separation of concerns** - Clear layers (UI, ViewModel, Data)
✅ **Production code quality** - Not pseudocode, actual Swift
✅ **Error handling** - Network failures, retries, edge cases
✅ **Testability** - Protocols, dependency injection
✅ **Mobile-first thinking** - Offline, caching, battery
✅ **Communication** - Explaining rationale, asking questions

### Nice-to-Haves (Differentiators):

⭐ **Advanced iOS knowledge** - Background tasks, NSPersistentHistory, Instruments
⭐ **Performance optimization** - Prefetching, downsampling, lazy loading
⭐ **Monitoring/observability** - Logging, metrics, crash reporting
⭐ **Team/scale considerations** - "At Uber's scale...", code review, onboarding

---

## Action Items Based on Your Score

### Score 85-100 (Strong Hire):
- ✅ Ready to interview! Focus on company-specific prep (Uber's tech stack)
- Practice 1-2 more designs to stay sharp
- Prepare behavioral answers (STAR format)

### Score 70-84 (Hire):
- Practice 2-3 more full designs (60 min each)
- Deep dive on your weakest category
- Do 1-2 mock interviews with peers

### Score 60-69 (Borderline):
- Practice 5+ designs covering all patterns
- Study the 3 guides in this folder thoroughly
- Get feedback from a senior iOS engineer
- Timeline: 2-3 weeks before ready

### Score 50-59 (No Hire):
- Take 1 month to strengthen fundamentals
- Complete online course (e.g., Ray Wenderlich iOS Architecture)
- Build a personal project using MVVM + Repository
- Revisit after implementing a real offline-first feature

### Score < 50 (Strong No Hire):
- Focus on iOS fundamentals first (not system design yet)
- Study: Swift, UIKit/SwiftUI, networking, concurrency
- Build 2-3 small apps end-to-end
- Revisit system design in 2-3 months

---

## Next Steps

1. **Calculate your score** across all 6 categories
2. **Identify your weakest area** (lowest score)
3. **Review the corresponding guide:**
   - [HLD/LLD Approach](./00_HLD_LLD_Approach_Guide.md)
   - [Trip List Design](./01_Uber_Trip_List_Design.md)
   - [Offline-First](./02_Offline_First_Architecture.md)
   - [Feed Pagination](./03_Feed_Pagination_Design.md)
4. **Practice 1 design per day** for next 5 days
5. **Record yourself** and re-evaluate with this checklist

---

## Interview Simulation Timer

Use this timeline for self-practice:

```
00:00 - 10:00  │ Requirements Clarification
10:00 - 25:00  │ High-Level Design (HLD)
25:00 - 45:00  │ Low-Level Design (LLD)
45:00 - 55:00  │ Deep Dives & Follow-ups
55:00 - 60:00  │ Your Questions to Interviewer
```

**Set a timer and STOP when time's up.** Real interviews are strict on pacing.

---

## Additional Resources

### Recommended Study Materials:
- [iOS Interview Guide](https://github.com/raywenderlich/ios-interview)
- [System Design Primer](https://github.com/donnemartin/system-design-primer)
- [Swift by Sundell](https://www.swiftbysundell.com/)

### Practice Platforms:
- **Pramp** - Free peer mock interviews
- **interviewing.io** - Anonymous interviews with engineers
- **LeetCode** - System design problems

### Uber-Specific:
- Read Uber Engineering Blog (focused on mobile posts)
- Study RIBs architecture (Uber's framework)
- Understand their scale (millions of trips/day)

---

> [!TIP]
> Print this checklist and use it after every practice session. Track your progress over time!

**Good luck! You've got this. 🚀**
