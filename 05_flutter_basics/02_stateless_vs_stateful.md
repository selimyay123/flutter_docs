# 02 — StatelessWidget vs StatefulWidget

## Goal

Make UI that **changes**. Learn what state is and how `setState` works.

---

## 1. What is state?

**State = data that can change over time and affects the UI.**

Examples: a counter value, whether a checkbox is checked, text in a field, a list of todos, "is loading".

Remember: `UI = f(state)`. Change state → Flutter rebuilds the UI.

---

## 2. StatelessWidget

No internal changing data. Its output depends only on constructor arguments.

```dart
class Greeting extends StatelessWidget {
  const Greeting({super.key, required this.name});
  final String name;

  @override
  Widget build(BuildContext context) => Text('Hello, $name');
}
```

It rebuilds only when its **parent** gives it new arguments.

---

## 3. StatefulWidget

Two classes: the **widget** (immutable config) and the **State** (mutable, long-lived).

```dart
class CounterPage extends StatefulWidget {
  const CounterPage({super.key});

  @override
  State<CounterPage> createState() => _CounterPageState();
}

class _CounterPageState extends State<CounterPage> {
  int _count = 0;

  void _increment() {
    setState(() {
      _count++;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text('Counter')),
      body: Center(
        child: Text('Count: $_count', style: Theme.of(context).textTheme.headlineMedium),
      ),
      floatingActionButton: FloatingActionButton(
        onPressed: _increment,
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

### Why two classes?

Widgets are **recreated constantly** (every rebuild). The `State` object **survives** rebuilds, so your data isn't lost.

---

## 4. `setState`

```dart
setState(() {
  _count++;
});
```

What it does:

1. Runs your callback (changes data).
2. Marks the widget **dirty**.
3. Flutter calls `build()` again **in the next frame**.

Important facts:

- Changing a variable **without** `setState` does not update the UI.
- The callback must be **synchronous** (no `async` inside).
- Keep the callback small; do slow work **before** it.

```dart
// Good: compute first, set after
final result = await fetch();
if (!mounted) return;
setState(() => _data = result);
```

---

## 5. State lifecycle

```
createState()
   │
initState()          <- once. Init controllers, start listeners.
   │
didChangeDependencies()   <- after initState + when an InheritedWidget you use changes
   │
build()              <- many times
   │      ▲
didUpdateWidget()    <- parent rebuilt with new config
   │
setState() ──► build()
   │
deactivate()
   │
dispose()            <- once. Cancel streams, dispose controllers.
```

Rules:

- `initState`: call `super.initState()` **first**. Don't use `context` for `InheritedWidget` lookups here (use `didChangeDependencies`).
- `dispose`: release everything, then `super.dispose()` **last**.
- `mounted`: is the State still in the tree? Check it after every `await` before `setState` or navigation.

```dart
class _TimerPageState extends State<TimerPage> {
  late final Timer _timer;
  int _seconds = 0;

  @override
  void initState() {
    super.initState();
    _timer = Timer.periodic(const Duration(seconds: 1), (_) {
      setState(() => _seconds++);
    });
  }

  @override
  void dispose() {
    _timer.cancel();       // otherwise: leak + setState on a dead widget
    super.dispose();
  }

  @override
  Widget build(BuildContext context) => Text('$_seconds s');
}
```

---

## 6. `widget` property

Inside `State`, `widget` gives you the current configuration.

```dart
class Greeting extends StatefulWidget {
  const Greeting({super.key, required this.name});
  final String name;
  @override
  State<Greeting> createState() => _GreetingState();
}

class _GreetingState extends State<Greeting> {
  @override
  Widget build(BuildContext context) => Text('Hi ${widget.name}');
}
```

---

## 7. Where should state live?

**Rule: keep state as low as possible, but as high as necessary.**

- Only one widget needs it → store it inside that widget.
- Two sibling widgets need it → move it up to their common parent ("lifting state up") and pass data down + callbacks up.
- Many distant widgets need it → state management (folder `06`).

```dart
// Parent owns the state
class Parent extends StatefulWidget { ... }
class _ParentState extends State<Parent> {
  bool _liked = false;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        LikeButton(liked: _liked, onChanged: (v) => setState(() => _liked = v)),
        Text(_liked ? 'You like this' : 'Not liked'),
      ],
    );
  }
}

// Child is stateless: data down, events up
class LikeButton extends StatelessWidget {
  const LikeButton({super.key, required this.liked, required this.onChanged});
  final bool liked;
  final ValueChanged<bool> onChanged;

  @override
  Widget build(BuildContext context) => IconButton(
        icon: Icon(liked ? Icons.favorite : Icons.favorite_border),
        onPressed: () => onChanged(!liked),
      );
}
```

**"Data flows down, events flow up."** This pattern is the base of all Flutter architecture.

---

## 8. Reduce rebuild cost

`setState` rebuilds the **whole** `build` of that widget. To keep it cheap:

1. Make the stateful widget **small** (only the part that changes).
2. Use `const` for children that don't change.
3. Extract static parts into their own widgets.

```dart
// Better: only the counter text is stateful
Column(
  children: const [
    ExpensiveStaticHeader(),   // const -> skipped on rebuilds
    CounterText(),             // StatefulWidget that owns _count
  ],
)
```

---

## Common mistakes

- Changing state without `setState`.
- Calling `setState` after `dispose` ("setState called after dispose").
- Forgetting `dispose` for controllers/timers/subscriptions.
- Putting all state at the top of the app.
- Creating things (controllers, futures) inside `build` — they get recreated every rebuild.
- Doing async work inside `setState`.

---

## Code challenge

1. Build a **counter** with `+`, `-`, and `reset`. Disable `-` at 0.
2. Build a **stopwatch** (start, stop, reset) with a `Timer.periodic`. Dispose it correctly. Show time as `mm:ss.s`.
3. **Lifting state:** two widgets — a slider that sets a font size and a text that uses it — living in different widget classes sharing one parent state.

<details>
<summary>Hint — disabled button</summary>

```dart
IconButton(
  onPressed: _count > 0 ? () => setState(() => _count--) : null,
  icon: const Icon(Icons.remove),
)
```
</details>

---

## Interview questions

1. StatelessWidget vs StatefulWidget: when do you need each?
2. Why are there two classes (widget + State)?
3. What does `setState` do internally?
4. List the lifecycle methods of `State` in order. What belongs in `initState` and `dispose`?
5. What does `mounted` mean and when do you check it?
6. What is "lifting state up"?
7. Why is it bad to create a `Future` or controller inside `build`?

<details>
<summary>Hints</summary>

1. Stateless for output depending only on inputs; Stateful when it owns changing data or resources.
2. Widgets are cheap and recreated; State persists across rebuilds.
3. Runs the callback, marks the element dirty, schedules a rebuild for the next frame.
4. createState, initState, didChangeDependencies, build, didUpdateWidget, deactivate, dispose. Init: setup; dispose: cleanup.
5. Whether the State is still attached to the tree; check after `await` before `setState`/navigation/context use.
6. Move state to the nearest common ancestor and pass data down and callbacks up.
7. `build` can run every frame; you would recreate it (and leak/restart network calls) each rebuild.
</details>
