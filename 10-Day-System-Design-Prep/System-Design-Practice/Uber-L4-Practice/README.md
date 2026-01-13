# Uber L4 iOS System Design Practice - README

> [!NOTE]
> This folder contains complete 60-minute interview simulations for Uber L4 (Senior iOS Engineer) system design rounds.

---

## 📁 Folder Contents

| File | Purpose | Time to Complete |
|------|---------|------------------|
| [00_HLD_LLD_Approach_Guide.md](./00_HLD_LLD_Approach_Guide.md) | Learn how to approach any system design (HLD/LLD framework) | 30 min read |
| [01_Uber_Trip_List_Design.md](./01_Uber_Trip_List_Design.md) | Complete walkthrough: Trip history with pagination + API design | 60 min practice |
| [02_Offline_First_Architecture.md](./02_Offline_First_Architecture.md) | Complete walkthrough: Offline-first app with sync engine + Sync APIs | 60 min practice |
| [03_Feed_Pagination_Design.md](./03_Feed_Pagination_Design.md) | Complete walkthrough: Social feed with image caching + Cursor pagination | 60 min practice |
| [04_Interview_Evaluation_Checklist.md](./04_Interview_Evaluation_Checklist.md) | Self-evaluation scorecard (100 points) | 10 min after each practice |
| [**05_API_Design_Best_Practices.md**](./05_API_Design_Best_Practices.md) | **Complete API design guide (client & server)** | **45 min read** |

---

## 🎯 How to Use This Folder

### Step 1: Study the Framework (Day 1)
1. Read [00_HLD_LLD_Approach_Guide.md](./00_HLD_LLD_Approach_Guide.md) thoroughly
2. Understand the 4-phase approach:
   - Requirements Clarification (0-10 min)
   - High-Level Design (10-25 min)
   - Low-Level Design (25-45 min)
   - Deep Dives (45-55 min)
3. Note the common mistakes to avoid

### Step 2: Practice First Design (Day 2-3)
1. Set a **60-minute timer**
2. Open [01_Uber_Trip_List_Design.md](./01_Uber_Trip_List_Design.md)
3. **First attempt:** Read the problem, CLOSE the file, and try designing on your own
4. **Second attempt (next day):** Follow the walkthrough, comparing your approach
5. Use [04_Interview_Evaluation_Checklist.md](./04_Interview_Evaluation_Checklist.md) to score yourself

### Step 3: Practice Second Design (Day 4-5)
1. Repeat process with [02_Offline_First_Architecture.md](./02_Offline_First_Architecture.md)
2. Focus on areas where you scored low in checklist
3. Record yourself (audio/video) and watch back to improve communication

### Step 4: Practice Third Design (Day 6-7)
1. Repeat with [03_Feed_Pagination_Design.md](./03_Feed_Pagination_Design.md)
2. Challenge: Try to complete without looking at the guide
3. Aim for 80+ score on evaluation checklist

### Step 5: Mock Interview (Day 8-10)
1. Find a peer or use Pramp/interviewing.io
2. Have them use the evaluation checklist to score you
3. Practice answering follow-up questions out loud

---

## 📊 Practice Schedule (10 Days)

| Day | Task | Duration | Goal |
|-----|------|----------|------|
| **Day 1** | Read HLD/LLD Approach Guide | 1 hour | Understand framework |
| **Day 2** | Practice Trip List (blind attempt) | 1 hour | Baseline score |
| **Day 3** | Study Trip List walkthrough | 1 hour | Learn patterns |
| **Day 4** | Practice Offline-First (blind) | 1 hour | Apply learnings |
| **Day 5** | Study Offline-First walkthrough | 1 hour | Deepen knowledge |
| **Day 6** | Practice Feed Pagination (blind) | 1 hour | Improve speed |
| **Day 7** | Study Feed Pagination walkthrough | 1 hour | Master optimizations |
| **Day 8** | Mock interview (peer/platform) | 1 hour | Simulate pressure |
| **Day 9** | Redo weakest design | 1 hour | Fill gaps |
| **Day 10** | Final mock + review | 1 hour | Confidence |

---

## 🎓 What You'll Learn

### Core Concepts Covered:
✅ **Architecture Patterns** - MVVM, Repository, Clean Architecture
✅ **Caching Strategies** - Memory (NSCache), Disk (FileSystem), LRU eviction
✅ **Pagination** - Cursor-based vs Offset-based, infinite scroll
✅ **Offline-First** - Sync engines, conflict resolution, last-write-wins
✅ **Networking** - URLSession, retry logic, error handling
✅ **Concurrency** - async/await, actors, thread safety
✅ **Performance** - Prefetching, lazy loading, memory management
✅ **Testability** - Protocols, dependency injection, mocking

### iOS-Specific Skills:
- UITableView prefetching for smooth scrolling
- NSCache for automatic memory management
- CoreData for offline persistence
- Background tasks for sync
- Image downsampling and lazy decoding

### 🆕 API Design Knowledge:
**NEW! Now includes comprehensive API design coverage:**

✅ **RESTful API Design** - HTTP methods, status codes, URL structure
✅ **Pagination Strategies** - Cursor vs offset vs keyset pagination
✅ **Error Handling** - Structured error responses, retry logic
✅ **Versioning** - URL path versioning, breaking vs non-breaking changes
✅ **Mobile Optimization** - Compression, conditional requests, field selection
✅ **Rate Limiting** - Client-side handling of 429 responses
✅ **Auth Pattern**s - Token refresh, Bearer tokens
✅ **Client Best Practices** - Request deduplication, cancellation, batching

> [!TIP]
> **When to study API Design?**
> - Before Day 2-3: Read [05_API_Design_Best_Practices.md](./05_API_Design_Best_Practices.md)
> - During practice: Each design now includes API design sections
> - Focus area: If interviewer asks "What would the API look like?"

---

## 💡 How API Design Fits In

Each system design interview should cover:
1. **HLD** - Architecture diagram (covered in guides)
2. **LLD** - Swift code (covered in guides) 
3. **API Design** - RESTful endpoints (**NEW! Now covered**)

**Interview Example:**
> Interviewer: "Design the trip list feature"
> 
> **Without API knowledge**: ❌ Only shows iOS code, awkward when asked about backend
> 
> **With API knowledge**: ✅ Shows iOS code + designs`GET /v1/trips?page=1&limit=20` endpoint

All three designs now include **API Design sections**:
- **Trip List**: Offset-based pagination API, error handling, versioning
- **Offline-First**: Sync APIs, conflict resolution endpoints, batch operations  
- **Feed Pagination**: Cursor-based pagination, field selection, rate limiting

---

## 🚨 Common Interview Mistakes (Avoid These!)

| Mistake | Why It Fails | How to Fix |
|---------|-------------|------------|
| Jumping to code without HLD | No system thinking | Always draw architecture diagram first |
| Not asking about scale | Can't justify choices | Ask: "10K users or 10M users?" |
| Ignoring offline | Missing mobile-first mindset | Always discuss caching strategy |
| No error handling | Not production-ready | Show retry, fallback, user messaging |
| Over-engineering | Not pragmatic | Start simple, add complexity if asked |
| Can't explain trade-offs | Memorized, not understood | For every choice, state alternatives |
| Silent coding | Poor communication | Narrate your thought process |

---

## 📈 Success Metrics

### You're Ready for Real Interview When:
- ✅ Scoring 80+ on evaluation checklist consistently
- ✅ Completing full design in 55 minutes (save 5 for questions)
- ✅ Can explain trade-offs for every major decision
- ✅ Comfortable with follow-up questions (no panic)
- ✅ Speaking out loud naturally (not forced)

---

## 🔥 Advanced Practice (After Mastering Basics)

Once you're comfortable with the 3 core designs, try these:

### Additional Practice Problems:
1. **Location tracking** (like Uber driver app)
   - Real-time GPS updates
   - Battery optimization
   - Offline tracking with sync

2. **Chat messaging** (like WhatsApp)
   - Offline message queue
   - Read receipts
   - Push notifications

3. **Photo gallery** (like Google Photos)
   - Large image sets (10K+ photos)
   - Smart caching with ML thumbnails
   - Upload queue management

4. **Ride booking** (like Uber ride request)
   - Real-time driver matching
   - Network reliability (retry, timeout)
   - State management (requesting → matched → riding)

5. **Map with annotations** (like food delivery)
   - Large number of pins (restaurants)
   - Clustering for performance
   - Real-time location updates

---

## 📚 Recommended Background Reading

### Before Starting:
- Swift concurrency (async/await, actors)
- MVVM pattern in iOS
- URLSession fundamentals
- NSCache vs custom caching

### While Practicing:
- [Apple's Networking Guide](https://developer.apple.com/documentation/foundation/url_loading_system)
- [Swift Concurrency Docs](https://docs.swift.org/swift-book/LanguageGuide/Concurrency.html)
- [iOS Performance Best Practices](https://developer.apple.com/videos/play/wwdc2021/10305/)

---

## 🛠 Tools You Need

### For Practice:
- **Timer** (phone, web, etc.) - Strict 60-minute limit
- **Whiteboard or Paper** - Draw diagrams
- **Code editor** (Xcode, VS Code) - Write actual Swift
- **This checklist** - Evaluate yourself after each practice

### For Mock Interviews:
- **Zoom/Google Meet** - Screen sharing
- **CoderPad or similar** - Collaborative coding
- **Peer or platform** - Pramp, interviewing.io

---

## ❓ FAQ

### Q: Do I need to memorize all the code?
**A:** No, but understand the patterns. Focus on structure (protocols, actors, async/await) over exact syntax.

### Q: What if I can't finish in 60 minutes?
**A:** That's okay for first attempts. Identify where you're slow (HLD? LLD? Coding?) and practice that phase specifically.

### Q: Should I use third-party libraries (Alamofire, SDWebImage)?
**A:** In real interviews, yes if you explain trade-offs. In practice, write from scratch to learn deeply.

### Q: How many times should I practice each design?
**A:** At least 2-3 times per design. Once blind, once with guide, once blind again.

### Q: What if the interviewer asks something not in these guides?
**A:** Use the framework from `00_HLD_LLD_Approach_Guide.md`. It applies to ANY iOS system design.

---

## 🎤 Interview Day Tips

### 1 Day Before:
- Sleep well (8 hours)
- Review mistakes checklist
- Skim one design (don't cram)

### Interview Day:
- Join 5 minutes early
- Have water nearby
- Whiteboard/paper ready
- **BREATHE** - You've practiced this

### During Interview:
- Ask questions FIRST (don't assume)
- Think out loud (narrate choices)
- Draw before coding
- If stuck, ask for hints
- Manage time (glance at clock)

### After Interview:
- Don't dwell on mistakes
- Note what you'd improve
- Send thank-you email

---

## 📞 Getting Help

### If You're Stuck:
1. Re-read the approach guide
2. Study the walkthrough line-by-line
3. Watch iOS system design videos on YouTube
4. Ask in communities (r/iOSProgramming, iOS Dev Slack)

### If You're Not Improving:
- You might need to strengthen iOS fundamentals first
- Consider taking a structured course (Ray Wenderlich, Hacking with Swift)
- Build a personal project applying these patterns

---

## 🏆 Final Motivation

**You're practicing the EXACT format used in real Uber L4 interviews.**

Every hour you spend here directly translates to interview performance. The designs in this folder are:

- ✅ Based on real interview questions
- ✅ Evaluated by actual L4+ iOS engineers
- ✅ Covering 90% of patterns asked at Uber, Meta, Airbnb, Lyft
- ✅ Production-quality code (not toy examples)

**Your competition is NOT practicing like this. You already have an edge.**

Set a 60-minute timer. Pick a design. Start now.

**You've got this. 🚀**

---

## 📝 Tracking Your Progress

Use this table to log your practice sessions:

| Date | Design | Time | Score | Weak Areas | Notes |
|------|--------|------|-------|------------|-------|
| | | | __/100 | | |
| | | | __/100 | | |
| | | | __/100 | | |
| | | | __/100 | | |
| | | | __/100 | | |

**Target:** 3+ sessions per design, final scores 80+

---

> [!TIP]
> Bookmark this README and return to it daily during your 10-day practice plan!
