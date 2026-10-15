# Project 3 — Capstone

## Goal

Build **one** portfolio-quality app that proves you are interview-ready. You choose the idea. You make real decisions, write them down, and defend them.
Estimated time: 3–4 weeks.

---

## Pick an idea (or invent one)

| Idea | Why it's good |
|------|---------------|
| **Habit tracker** with streaks, charts, reminders | Dates, algorithms (streaks), local DB, notifications, custom painting |
| **Expense manager** with categories and monthly reports | Money handling, DB, aggregation, charts, export |
| **Flashcards (spaced repetition)** | Scheduling algorithm (SM-2), offline, stats |
| **Recipe app** with search and shopping list | Lists, search, images, offline cache |
| **Chat app** (Firebase/Supabase) | Streams, auth, real time, optimistic UI |
| **Markdown notes** with folders and full-text search | Trees, search (trie/inverted index), storage |
| **Pomodoro + stats** | Timers, background behavior, charts |

Pick something you would use yourself. Motivation matters.

---

## Non-negotiable requirements

1. **Architecture:** feature-first, layered (UI → view model → repository → data), dependency injection.
2. **State management:** one consistent choice, with explicit sealed states.
3. **Persistence:** local database with at least **one migration** (v1 → v2).
4. **Networking or sync:** at least one remote/backed feature or a clean fake that could be swapped.
5. **Navigation:** `go_router` with a redirect (onboarding/auth) and deep link support.
6. **Error handling:** typed errors, retry, global error hook, crash reporting (Crashlytics/Sentry or a clean interface).
7. **Theming:** light/dark, tokens, Material 3, accessible (large text, contrast, semantics).
8. **Responsive:** phone + tablet/desktop layout.
9. **Testing:** ≥ 70% coverage on domain/view-model code, ≥ 5 widget tests, ≥ 1 integration test, ≥ 1 golden.
10. **CI:** GitHub Actions running format, analyze, tests.
11. **Performance:** profiled in profile mode; document one real optimization you made (with before/after numbers).
12. **Algorithm component:** at least **one** non-trivial algorithm you designed and tested (see below).
13. **Docs:** README with screenshots, architecture diagram, setup steps, decisions.

---

## The algorithm component (must-have)

Choose one that fits your app, implement it as **pure Dart**, with tests including a **stress test** against a brute force:

| App | Algorithm |
|-----|-----------|
| Habit tracker | Longest/current streak with timezone-aware days; best week |
| Expenses | Group-by + prefix sums for range totals; split a bill among people minimizing transactions (greedy with heaps) |
| Flashcards | SM-2 scheduling; due-queue with a priority queue |
| Recipes | Ingredient search with inverted index; scaling quantities with fractions |
| Notes | Trie/inverted index search; diff between versions (LCS) |
| Chat | Message ordering/merging with timestamps; unread count |
| Pomodoro | Interval scheduling & statistics |

Write a short `ALGORITHM.md`: problem, constraints, brute force, optimized approach, invariants, Big-O, tests.

---

## Milestones

### Week 1 — Foundations
- [ ] Idea brief (1 page): users, problems, MVP scope, **what you will NOT build**.
- [ ] Wireframes (paper or Figma).
- [ ] Project skeleton, lints, CI running, theme.
- [ ] Domain models + repository interfaces + fakes + tests.

### Week 2 — Core features
- [ ] Local DB and migrations.
- [ ] View models/state and the main screens.
- [ ] Navigation + redirects.
- [ ] Algorithm component implemented and tested.

### Week 3 — Quality
- [ ] Error handling, loading/empty states, retry.
- [ ] Responsive layouts, accessibility pass.
- [ ] Widget/golden/integration tests.
- [ ] Performance profiling and fixes.

### Week 4 — Ship
- [ ] Flavors, icons, splash, release build.
- [ ] README, screenshots, `ARCHITECTURE.md`, `DECISIONS.md`.
- [ ] Beta (TestFlight/Play internal track) with 3+ real users; collect feedback; fix.
- [ ] Record a 2-minute demo video.

---

## Architecture Decision Records (ADRs)

For each important decision write a short note in `DECISIONS.md`:

```
## ADR-003: Use Riverpod for state management
Context: Need testable async state and DI without BuildContext.
Options: Provider, Bloc, Riverpod.
Decision: Riverpod.
Consequences: + compile-safe, easy overrides in tests; − learning curve, codegen optional.
```

This is the single best preparation for senior interviews: you can explain **why**, not only **what**.

---

## Evaluation rubric (be honest, score 1–5)

| Category | What a 5 looks like |
|----------|--------------------|
| Architecture | Clear layers, no leaks (widgets don't know DB/HTTP), DI, small files |
| Code quality | Meaningful names, small functions/widgets, no duplication, zero analyzer issues |
| State | Immutable, explicit states, no impossible combinations |
| Correctness | Edge cases covered, no crashes in 30 minutes of random use |
| Testing | Fast, deterministic, meaningful asserts, CI green |
| UX & accessibility | Loading/empty/error, large text OK, contrast OK, semantic labels |
| Performance | Measured, 60 FPS on a real mid-range device, justified optimizations |
| Algorithms | Correct, documented, tested, complexity understood |
| Delivery | CI, flavors, versioning, signed build, changelog |
| Communication | README, ADRs, demo video |

Target: average ≥ 4. Re-iterate on the lowest category.

---

## Code review yourself

Print your diff (or open a PR on your own repo) and review as a senior would:

- Would I understand this in 6 months?
- What breaks if the API returns `null` here?
- What happens on a slow network? On rotation? On a 200% font size?
- What's the simplest change that removes this complexity?
- Is this tested?

---

## Mock "defend your project" interview (do it out loud!)

Record yourself answering:

1. Give me a 2-minute overview of the app and who it's for.
2. Draw the architecture. Why these layers?
3. Show me the trickiest bug you fixed. How did you find it?
4. How does data flow from a button tap to the screen update?
5. How would the app behave with 1,000,000 records?
6. How would you add real-time sync with a server?
7. What are the weaknesses of your app and how would you fix them?
8. What would you change if you had two more weeks?
9. Walk me through your algorithm and its complexity.
10. How would you onboard a new developer to this codebase?

---

## Interview questions (about your capstone)

1. What was the hardest technical decision, and what alternatives did you consider?
2. How do you know your app is correct? What is your test strategy?
3. Which part of your code are you least proud of? Why?
4. How did you measure and improve performance?
5. How would the architecture change for a team of 10?
6. What are the security and privacy considerations of your app?
7. How would you migrate users' data safely to a new schema?
8. Show me one place where you applied a data structure/algorithm to solve a real product need.

<details>
<summary>Hints</summary>

- Use the **STAR** format (Situation, Task, Action, Result) and mention **trade-offs** and **numbers** (before/after).
- Be honest about weaknesses; seniors are valued for self-awareness and judgment.
- For a team of 10: modularization, ownership, code review rules, CI gates, design system package.
- Security: secure storage, no secrets in the client, least-privilege permissions, data deletion, encrypted backups.
- Migration: versioned schema, tested upgrade paths, backup before migrating, feature flags.
</details>
