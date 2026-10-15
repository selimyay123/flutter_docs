# 03 — Event Loop and Isolates

## Goal

Know **how** Dart runs code. This is a favorite senior interview topic.

---

## 1. Dart is single-threaded per isolate

An **isolate** is a worker with:

- its own memory (heap), **not shared** with others
- its own event loop
- no locks needed (no shared memory = no data races)

Your Flutter app starts with a **main isolate**. UI, your code, and event handling all run there.

---

## 2. The event loop

Two queues:

| Queue | Holds | Priority |
|-------|-------|----------|
| **Microtask queue** | tiny tasks (`scheduleMicrotask`, Future `.then` of completed futures) | Higher |
| **Event queue** | I/O, timers, taps, drawing, `Future.delayed` | Lower |

Loop:

```
while (true) {
  run ALL microtasks until the microtask queue is empty
  take ONE event from the event queue and run it
}
```

### Test yourself — what prints?

```dart
import 'dart:async';

void main() {
  print('1 main start');

  Future(() => print('4 event'));                        // event queue
  scheduleMicrotask(() => print('3 microtask'));         // microtask queue
  Future.microtask(() => print('3b microtask'));

  print('2 main end');
}
```

Output:

```
1 main start
2 main end
3 microtask
3b microtask
4 event
```

`main` finishes first (synchronous code always runs to completion). Then microtasks. Then events.

### Why it matters

- A long synchronous loop **blocks everything**: the UI cannot draw until it ends.
- Never starve the event queue with endless microtasks.

---

## 3. When async is not enough

```dart
Future<int> heavy() async {
  var sum = 0;
  for (var i = 0; i < 2000000000; i++) {
    sum += i;
  }
  return sum;           // async keyword doesn't help: UI frozen while looping!
}
```

`async` only helps when waiting on **I/O**. For heavy **CPU** work (parsing a huge JSON, image processing, encryption), use a separate isolate.

---

## 4. `Isolate.run` (the simple way)

```dart
import 'dart:isolate';

int heavySum(int n) {
  var s = 0;
  for (var i = 0; i < n; i++) s += i;
  return s;
}

Future<void> main() async {
  final result = await Isolate.run(() => heavySum(1000000000));
  print(result);
}
```

It spawns an isolate, runs the function, returns the result, and shuts the isolate down.

### In Flutter: `compute`

```dart
final list = await compute(parseBigJson, jsonString);   // top-level function
```

(`compute` is basically a wrapper over `Isolate.run`; newer code can use `Isolate.run` directly.)

### What can cross the isolate boundary?

Messages are **copied** (with some optimizations). Allowed: numbers, strings, lists, maps, records, simple objects composed of those, `SendPort`. Not allowed: objects holding native resources (sockets, open files), closures capturing non-sendable state.

Copying has a cost. Sending a 100 MB object back and forth can cost more than the work!

---

## 5. Long-lived isolates with ports

```dart
Future<void> main() async {
  final receive = ReceivePort();
  await Isolate.spawn(worker, receive.sendPort);

  await for (final msg in receive) {
    print('main got: $msg');
    if (msg == 'done') break;
  }
  receive.close();
}

void worker(SendPort toMain) {
  for (var i = 0; i < 3; i++) {
    toMain.send('progress $i');
  }
  toMain.send('done');
}
```

Use when you need continuous two-way communication (e.g. a background audio engine).

---

## 6. Decision guide

| Situation | Tool |
|-----------|------|
| Waiting for network/disk/timer | `async/await` |
| Many events over time | `Stream` |
| Heavy CPU, one-off | `Isolate.run` / `compute` |
| Heavy CPU, long-running with messaging | `Isolate.spawn` + ports |
| Native work (camera, sensors) | Platform plugin |

Rule: **measure first.** Use DevTools. Do not guess.

Rule: a frame at 60 FPS = **16 ms** budget (8 ms at 120 FPS). Any single task taking more than that on the UI isolate causes jank.

---

## 7. Web note

On Flutter Web, isolates are mapped to Web Workers with limitations (`Isolate.run` works differently, `compute` runs on the same thread in some cases). Always check platform support.

---

## Common mistakes

- Thinking `async` = parallel.
- Parsing large JSON on the UI isolate.
- Sending huge objects between isolates.
- Trying to share a variable between isolates (impossible; each has its own copy).

---

## Code challenge

1. Predict the output (write it before running):

```dart
void main() {
  print('A');
  Future(() => print('B'));
  Future.microtask(() => print('C'));
  Future.delayed(Duration.zero, () => print('D'));
  scheduleMicrotask(() => print('E'));
  print('F');
}
```

2. Write a function that computes the number of primes below 5,000,000 inside `Isolate.run`. Compare time with running it directly.

<details>
<summary>Solution</summary>

1. `A F C E B D`
   - `A`, `F` synchronous. Microtasks `C`, `E` next (in order added). Then events `B`, `D` (in order added).

2.
```dart
import 'dart:isolate';

int countPrimes(int limit) {
  final sieve = List<bool>.filled(limit, true);
  var count = 0;
  for (var i = 2; i < limit; i++) {
    if (sieve[i]) {
      count++;
      for (var j = i * i; j < limit; j += i) {
        sieve[j] = false;
      }
    }
  }
  return count;
}

Future<void> main() async {
  final result = await Isolate.run(() => countPrimes(5000000));
  print(result); // 348513
}
```
</details>

---

## Interview questions

1. Explain Dart's event loop. Which runs first: microtask or event?
2. Is Dart multi-threaded? What is an isolate?
3. When do you need an isolate if you already have `async/await`?
4. How do isolates communicate and why is there no shared memory?
5. What is `compute` in Flutter?
6. What is the frame budget at 60 FPS and how does it relate to isolates?

<details>
<summary>Hints</summary>

1. Sync code → all microtasks → one event → repeat. Microtasks first.
2. Each isolate is single-threaded; many isolates can run in parallel on different threads with isolated memory.
3. CPU-bound heavy work; async only helps while waiting on I/O.
4. Message passing via ports with copying; no shared state means no locks or races.
5. A helper to run a function in a background isolate and return the result.
6. ~16 ms; heavy work on the UI isolate eats the budget and drops frames.
</details>
