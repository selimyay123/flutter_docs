# 02 — Streams

## Goal

Handle **many values over time**: user taps, sensor data, chat messages, live prices.

---

## 1. Future vs Stream

| | Future | Stream |
|--|--------|--------|
| Values | **One** (later) | **Zero or many** (over time) |
| Analogy | A package delivered once | A water pipe / YouTube live |
| Await with | `await` | `await for` / `listen` |

---

## 2. Consuming a stream

```dart
Stream<int> countDown(int from) async* {   // async* = generator
  for (var i = from; i >= 0; i--) {
    await Future.delayed(const Duration(seconds: 1));
    yield i;                                // emit a value
  }
}

Future<void> main() async {
  await for (final n in countDown(3)) {
    print(n);                               // 3 2 1 0
  }
  print('liftoff');
}
```

- `async*` + `yield` = **produce** a stream.
- `await for` = **consume** it, one event at a time.

Using `listen`:

```dart
final sub = countDown(3).listen(
  (n) => print(n),
  onError: (e) => print('error $e'),
  onDone: () => print('done'),
);

// later:
await sub.cancel();   // always cancel to avoid leaks!
```

---

## 3. Single-subscription vs broadcast

| Type | Listeners | Replays missed events? |
|------|-----------|------------------------|
| Single-subscription (default) | Exactly one | Buffers until listened |
| Broadcast | Many | No, you miss events before you listen |

```dart
final controller = StreamController<int>.broadcast();
controller.stream.listen((x) => print('A $x'));
controller.stream.listen((x) => print('B $x'));
controller.add(1);   // both print
```

---

## 4. StreamController

You **create** a stream and push events manually.

```dart
class Counter {
  final _controller = StreamController<int>();
  var _value = 0;

  Stream<int> get stream => _controller.stream;

  void increment() => _controller.add(++_value);

  void dispose() => _controller.close();   // close to free resources
}
```

Methods: `add`, `addError`, `close`.

---

## 5. Transforming streams

Streams have operators like lists:

```dart
Stream.periodic(const Duration(seconds: 1), (i) => i)   // 0,1,2,...
    .where((n) => n.isEven)
    .map((n) => n * 10)
    .take(3)
    .listen(print);                                      // 0, 20, 40
```

More: `expand`, `skip`, `distinct`, `asyncMap`, `handleError`, `timeout`, `first`, `toList`.

Advanced combinations (debounce, switchMap, combineLatest) live in **`rxdart`**.

### Search-box debounce idea

User types "flutter" fast. You don't want 7 network calls. You wait until they stop for 300 ms, then search. Streams (or a simple `Timer`) solve this.

```dart
Timer? _debounce;
void onChanged(String text) {
  _debounce?.cancel();
  _debounce = Timer(const Duration(milliseconds: 300), () => search(text));
}
```

---

## 6. Streams in Flutter

`StreamBuilder` rebuilds the UI for each event:

```dart
StreamBuilder<int>(
  stream: counter.stream,
  initialData: 0,
  builder: (context, snapshot) {
    if (snapshot.hasError) return Text('Error: ${snapshot.error}');
    if (!snapshot.hasData) return const CircularProgressIndicator();
    return Text('Count: ${snapshot.data}');
  },
)
```

Real uses: Firebase snapshots, WebSockets, location updates, Bloc's `Stream<State>`.

---

## 7. Cancel and close everything

Memory leaks are the #1 stream bug.

- Cancel every `StreamSubscription` in `dispose()`.
- Close every `StreamController` you own.

```dart
class _MyState extends State<MyWidget> {
  late final StreamSubscription<int> _sub;

  @override
  void initState() {
    super.initState();
    _sub = stream.listen((_) => setState(() {}));
  }

  @override
  void dispose() {
    _sub.cancel();
    super.dispose();
  }
  // build omitted
}
```

---

## 8. Back-pressure (concept)

If a producer is faster than the consumer, events pile up in memory. `pause()`/`resume()` on a subscription, or throttling, controls this. Know the word for interviews.

---

## Common mistakes

- Listening twice on a single-subscription stream (throws `Bad state`).
- Forgetting to cancel/close → leaks.
- Creating a new stream inside `build()` → resubscribes every rebuild.
- Using a Stream for something that is just one value (use Future).

---

## Code challenge

1. Write `Stream<int> fibonacci()` (infinite). Print the first 10 using `.take(10)`.
2. Write a function that listens to `Stream<int>` and returns a `Future<int>` with the sum of all events (use `await for`).
3. Create a `StreamController<String>` and make a "chat room" where two listeners each print messages with their name.

<details>
<summary>Solution</summary>

```dart
Stream<int> fibonacci() async* {
  var a = 0, b = 1;
  while (true) {
    yield a;
    final next = a + b;
    a = b;
    b = next;
  }
}

Future<int> sumStream(Stream<int> s) async {
  var total = 0;
  await for (final n in s) {
    total += n;
  }
  return total;
}

Future<void> main() async {
  await fibonacci().take(10).forEach(print);
  print(await sumStream(Stream.fromIterable([1, 2, 3]))); // 6

  final room = StreamController<String>.broadcast();
  room.stream.listen((m) => print('Ann got: $m'));
  room.stream.listen((m) => print('Bob got: $m'));
  room.add('Hello!');
  await room.close();
}
```
</details>

---

## Interview questions

1. Future vs Stream?
2. Single-subscription vs broadcast stream?
3. What does `async*` and `yield` do?
4. How do you avoid leaks with streams in a `State` object?
5. What is back-pressure?
6. How would you implement a search-as-you-type feature?

<details>
<summary>Hints</summary>

1. One async value vs a sequence of async values.
2. One listener with buffering vs many listeners without replay.
3. Declares an async generator; `yield` emits an event.
4. Cancel subscriptions and close controllers in `dispose()`.
5. Producer faster than consumer, causing unbounded buffering.
6. Debounce input (~300 ms), cancel previous requests / ignore stale results, handle loading & errors.
</details>
