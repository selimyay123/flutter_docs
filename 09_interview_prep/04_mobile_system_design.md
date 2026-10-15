# 04 — Mobile System Design

## Goal

Design scalable, robust, and performant mobile client architectures during senior system design interviews.

---

## 1. The Mobile System Design Framework (RADIO)

1. **R — Requirements & Scope**: Functional (must-haves) vs Non-functional (offline, latency, battery, memory, cache size).
2. **A — Architecture & High-Level Flow**: Layers (UI, State, Repository, Local Cache, Remote API).
3. **D — Data Models & Schemas**: Local DB entities, API payloads, synchronization state.
4. **I — Interface & APIs**: REST vs WebSocket vs GraphQL, endpoint design, pagination, idempotency keys.
5. **O — Optimizations & Deep Dives**: Offline sync, conflict resolution, battery/data efficiency, security, image pipeline.

---

## 2. Classic System Design: Real-Time Messaging App (e.g., WhatsApp / Slack)

```
┌────────────────────────────────────────────────────────┐
│ UI Layer (ChatView, MessageBubble, InputBar)           │
└───────────────────────────▲────────────────────────────┘
                            │ (State Stream)
┌───────────────────────────┴────────────────────────────┐
│ ViewModel / Cubit (ChatViewModel)                      │
└───────────────────────────▲────────────────────────────┘
                            │
┌───────────────────────────┴────────────────────────────┐
│ ChatRepository (Single Source of Truth)                │
│  - Optimistic updates                                  │
│  - Message status transitions (Sending, Sent, Read)    │
└─────────────┬────────────────────────────┬─────────────┘
              │                            │
   ┌──────────▼──────────┐      ┌──────────▼──────────┐
   │ SQLite / Drift DB   │      │ Sync Engine / WS    │
   │ (Local Message Store│      │ (WebSocket + REST)  │
   └─────────────────────┘      └─────────────────────┘
```

### Key Technical Challenges & Solutions

#### A. Optimistic UI Updates
When the user sends a message:
1. Immediately write to local DB with `status = MessageStatus.sending` and temporary UUID.
2. Emit local DB stream to update UI instantaneously.
3. Dispatch payload over WebSocket.
4. Upon server ACK with permanent ID: Update local record with `status = MessageStatus.sent`.
5. If timeout / error: Mark `status = MessageStatus.failed` and provide retry button.

#### B. Offline Synchronization & Idempotency
- Every sent message includes an **Idempotency Key** (client-generated UUID) to prevent duplicate messages when retrying over flaky networks.
- A background **Sync Worker** checks for un-sent messages when network connectivity is restored (`connectivity_plus`).

---

## 3. Classic System Design: Large-Scale Image Caching Pipeline

### Requirements:
- Display thousands of remote images in a fast-scrolling feed without jank or out-of-memory (OOM) crashes.

```
Request Image (URL, TargetWidth, TargetHeight)
         │
         ▼
[In-Memory LRU Cache] ──── Hit ────► Decode & Display
         │ Miss
         ▼
[Disk Cache (Encrypted/Temp)] ──── Hit ────► Resize/Decode ──► Save to Memory ──► Display
         │ Miss
         ▼
[Network Downloader (Pool)] ──► Stream bytes ──► Save to Disk ──► Decode ──► Display
```

### Senior Engineering Nuances:
1. **Target Sizing**: Never decode a 4000x3000 photo into memory to show in a 100x100 avatar. Use `instantiateImageCodecWithSize` or `cacheWidth`/`cacheHeight`.
2. **Download Prioritization**: Cancel image requests for items that have scrolled off-screen.
3. **Memory Budget**: Cap in-memory cache to a fraction (e.g., 20%) of available device RAM.

---

## System Design Exercise

Design an **Offline-First Expense Tracker** that syncs across multiple devices:
- How are concurrent edits to the same expense handled?
- How do you minimize battery drain when syncing?

<details>
<summary>Key Discussion Points</summary>

1. **Conflict Resolution**:
   - For simple fields: Last-Write-Wins (LWW) using NTP-synchronized timestamps or logical Lamport clocks.
   - For deletion vs edit: Deletion is marked via a "tombstone" boolean (`isDeleted = true`) rather than hard row removal until synced.
2. **Battery & Data Optimization**:
   - Batch sync updates rather than firing network requests on every keystroke.
   - Use exponential back-off and trigger sync on Wi-Fi or when charging for large attachments.
</details>

---

## Interview Questions

1. How do you design an API pagination strategy: offset-based vs cursor-based?
2. What happens if a mobile app receives a 401 Unauthorized while multiple network requests are in-flight?
3. How do you ensure push notifications wake up the app and update local data reliably?
4. How do you protect sensitive user tokens stored on-device?

<details>
<summary>Hints & Answers</summary>

1. **Cursor-based** is superior for real-time mobile feeds. Offset pagination suffers from duplicate/skipped items if new items are inserted at the head of the feed.
2. Use an **Auth Interceptor** that queues pending requests, executes a single token refresh request, updates storage, and replays queued requests with the new bearer token.
3. Use silent push notifications (data-only messages) with background handlers, respecting OS battery saver restrictions.
4. Use platform-native secure keychains (`flutter_secure_storage` backed by Keychain on iOS and Keystore + AES on Android).
</details>
