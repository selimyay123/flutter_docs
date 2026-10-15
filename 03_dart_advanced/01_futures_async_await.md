# 01 — Futures, async, await

## Goal

Understand asynchronous code. Apps wait for network, disk, timers. Waiting must **never freeze the UI**.

---

## 1. Why async?

Imagine ordering coffee:

- **Sync:** you stand at the counter doing nothing until it is ready.
- **Async:** you get a ticket, sit down, and the barista calls you when it is ready.

A phone app's UI thread must keep drawing at 60+ FPS. If you block it → frozen screen ("jank").

---

## 2. Future

A `Future<T>` is a **promise of a value (or an error) that will arrive later**.

States: **uncompleted** → **completed with value** or **completed with error**.

```dart
Future<String> fetchName() {
  return Future.delayed(const Duration(seconds: 1), () => 'Ada');
}

void main() {
  print('start');
  fetchName().then((name) => print('got $name'));
  print('end');
}
// start
// end
// got Ada      <- arrives later
```

Notice: `end` prints **before** `got Ada`. The program did not wait.

---

## 3. async / await

`await` makes async code **read like** sync code.

```dart
Future<void> main() async {
  print('start');
  final name = await fetchName();   // pause THIS function, not the app
  print('got $name');
  print('end');
}
// start
// got Ada
// end
```

Rules:

- `await` can only be used inside an `async` function.
- An `async` function **always returns a Future** (wrapped automatically).
- `await` pauses only the current function. Other code keeps running.

### Error handling

```dart
Future<void> load() async {
  try {
    final data = await fetchData();
    print(data);
  } on TimeoutException {
    print('Too slow');
  } catch (e) {
    print('Failed: $e');
  } finally {
    print('done');
  }
}
```

With `.then`:

```dart
fetchData().then(print).catchError((e) => print('err $e')).whenComplete(() {});
```

Prefer `async/await` for readability.

---

## 4. Running Futures in parallel

**Sequential (slow):**

```dart
final a = await fetchA();   // 2 s
final b = await fetchB();   // 2 s  -> total 4 s
```

**Parallel (fast):**

```dart
final results = await Future.wait([fetchA(), fetchB()]);  // total ~2 s
```

Records version, keeps types:

```dart
final (a, b) = await (fetchA(), fetchB()).wait;
```

Other helpers:

| API | Meaning |
|-----|---------|
| `Future.value(x)` | Already completed |
| `Future.error(e)` | Already failed |
| `Future.delayed(d, f)` | After a delay |
| `Future.any([...])` | First one to finish wins |
| `future.timeout(d)` | Fail with `TimeoutException` |

---

## 5. Common pitfalls

### Forgetting `await`

```dart
Future<void> save() async { ... }

void onTap() {
  save();       // fire and forget: errors are lost, order not guaranteed
}
```

The linter rule `unawaited_futures` helps. If it's intended, write `unawaited(save());` (from `dart:async`).

### `await` in a loop (accidentally sequential)

```dart
for (final id in ids) {
  await fetch(id);       // one by one
}
// Parallel:
await Future.wait(ids.map(fetch));
```

(Sometimes sequential is correct, e.g. to respect rate limits.)

### Using `BuildContext` after `await` (Flutter!)

```dart
Future<void> onPressed() async {
  await save();
  if (!context.mounted) return;     // widget could be gone!
  Navigator.of(context).pop();
}
```

### `async` doesn't mean "another thread"

An `async` function runs on the **same** thread. It just yields while waiting for I/O.
Heavy **CPU** work still blocks. For that, use isolates (lesson 03).

---

## 6. Completer — bridge from callbacks

```dart
Future<int> wrapCallbackApi() {
  final completer = Completer<int>();
  legacyApi(onDone: (v) => completer.complete(v), onFail: completer.completeError);
  return completer.future;
}
```

---

## 7. Mental model

```
await foo();
   │
   ├─ foo starts (e.g. network request, handed to the OS)
   ├─ this function is suspended, control returns to the event loop
   ├─ other code & UI frames run
   └─ when foo completes, the function resumes right after the await
```

---

## Common mistakes

- Not awaiting.
- Not catching errors → unhandled exceptions.
- Awaiting in a loop when parallel is fine.
- Using context after await without `mounted` check.
- Believing async = faster. It only avoids blocking.

---

## Code challenge

1. Write `Future<int> fetchNumber(int n)` that waits 1 second and returns `n * 2`.
2. Fetch 1, 2, 3 **in parallel** and print their sum. Measure time with `Stopwatch` (should be ~1 s, not 3 s).
3. Write `Future<T> retry<T>(Future<T> Function() task, {int times = 3})` that retries on failure, rethrowing after the last attempt.

<details>
<summary>Solution</summary>

```dart
Future<int> fetchNumber(int n) =>
    Future.delayed(const Duration(seconds: 1), () => n * 2);

Future<void> main() async {
  final sw = Stopwatch()..start();
  final results = await Future.wait([1, 2, 3].map(fetchNumber));
  print(results.reduce((a, b) => a + b)); // 12
  print('${sw.elapsedMilliseconds} ms');   // ~1000
}

Future<T> retry<T>(Future<T> Function() task, {int times = 3}) async {
  for (var attempt = 1; ; attempt++) {
    try {
      return await task();
    } catch (_) {
      if (attempt >= times) rethrow;
      await Future.delayed(Duration(milliseconds: 200 * attempt)); // back-off
    }
  }
}
```
</details>

---

## Interview questions

1. What is a `Future`? What states can it be in?
2. What does `await` actually do? Does it block the thread?
3. Does `async` run code on another thread?
4. How do you run three independent requests in parallel?
5. Why check `context.mounted` after an `await`?
6. What happens to an error thrown inside a Future you never await or catch?
7. What is a `Completer` used for?

<details>
<summary>Hints</summary>

1. Value/error available later. Uncompleted, completed (value), completed (error).
2. Suspends the current function and yields to the event loop. Thread not blocked.
3. No. Same isolate/thread; only CPU work in an Isolate runs in parallel.
4. `Future.wait([...])` or record `.wait`.
5. The widget may have been removed; using its context crashes or misbehaves.
6. It becomes an unhandled async error (reported to the zone/error handler); can crash in some setups.
7. Convert callback-based APIs to Futures; complete manually.
</details>
