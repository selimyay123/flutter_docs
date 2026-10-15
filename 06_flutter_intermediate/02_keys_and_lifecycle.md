# 02 — Keys and Lifecycle (Deep Dive)

## Goal

Know when and which key to use, and manage resources correctly through a widget's life.

---

## 1. What is a key?

A key is an **identity tag** on a widget. It helps Flutter match old and new widgets during rebuild (see the Three Trees lesson).

```dart
ListView(
  children: [
    for (final todo in todos)
      TodoTile(key: ValueKey(todo.id), todo: todo),
  ],
)
```

**Rule:** if the order, insertion, or removal of a list of **stateful** widgets can change, give each a stable, unique key.

---

## 2. Key types

| Key | Identity based on | Use |
|-----|-------------------|-----|
| `ValueKey<T>(value)` | Equality of the value | Items with a natural ID (`ValueKey(user.id)`) |
| `ObjectKey(object)` | Identity of an object | When a whole object is the identity |
| `UniqueKey()` | Always unique (new every time) | Force a fresh State on each rebuild — careful! |
| `PageStorageKey(value)` | Stores scroll position etc. | Preserve scroll position of lists in tabs |
| `GlobalKey<T>()` | App-wide unique | Access a widget's State from elsewhere; move a widget between parents |

### Local vs global keys

- **Local keys** (`Value/Object/Unique`) only need to be unique **among siblings**.
- **`GlobalKey`** must be unique in the **whole app** and is **expensive**. Use sparingly.

### Common `GlobalKey` uses

```dart
final formKey = GlobalKey<FormState>();       // validate a Form
formKey.currentState!.validate();

final scaffoldKey = GlobalKey<ScaffoldState>();
```

Anti-pattern: calling methods on child states through `GlobalKey` to control them. Prefer controllers or callbacks.

---

## 3. A key trick: reset state on purpose

Changing a key tells Flutter: "this is a **different** widget". The old element/State is discarded.

```dart
// Reset a form/animation when the user switches
ProfileEditor(key: ValueKey(selectedUser.id), user: selectedUser)
```

When `selectedUser.id` changes, the editor gets a **fresh State** (new text controllers, etc.). Much simpler than manually resetting.

### `AnimatedSwitcher` relies on keys

```dart
AnimatedSwitcher(
  duration: const Duration(milliseconds: 200),
  child: Text('$count', key: ValueKey(count)),
)
```

---

## 4. Don't misuse keys

- `UniqueKey()` inside `build` → new key every rebuild → State destroyed every rebuild → lost text input, animations restart, poor performance.
- Using the **index** as a key in a reorderable list defeats the purpose.
- Keys for stateless widgets that never move are unnecessary.

---

## 5. State lifecycle — full reference

```dart
class _MyState extends State<MyWidget> {
  @override
  void initState() {
    super.initState();
    // 1x: init controllers, subscribe, start timers, kick off first load
  }

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    // After initState, and whenever an InheritedWidget you depend on changes
    // (Theme.of(context), MediaQuery.of(context), Localizations...)
  }

  @override
  void didUpdateWidget(covariant MyWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    // Parent rebuilt us with a new config: compare oldWidget vs widget
    if (oldWidget.userId != widget.userId) _reload();
  }

  @override
  Widget build(BuildContext context) { ... }   // pure! no side effects

  @override
  void deactivate() {
    super.deactivate();
    // Removed from tree (may be re-inserted, e.g. via GlobalKey move)
  }

  @override
  void dispose() {
    // 1x: cancel timers/subscriptions, dispose controllers/focus nodes
    super.dispose();
  }
}
```

### Which method for what

| Task | Where |
|------|-------|
| Create `TextEditingController`, `AnimationController` | `initState` (or `late final` field) |
| Start listening to a stream | `initState`; cancel in `dispose` |
| Use `Theme.of(context)`, `MediaQuery.of(context)` to set up something | `didChangeDependencies` |
| React to changed constructor params | `didUpdateWidget` |
| Free resources | `dispose` |
| Anything async that touches UI afterward | check `mounted` |

### `didUpdateWidget` example — a widget that reloads when its ID changes

```dart
class UserView extends StatefulWidget {
  const UserView({super.key, required this.userId});
  final int userId;
  @override
  State<UserView> createState() => _UserViewState();
}

class _UserViewState extends State<UserView> {
  late Future<User> _future;

  @override
  void initState() {
    super.initState();
    _future = repo.fetch(widget.userId);
  }

  @override
  void didUpdateWidget(UserView old) {
    super.didUpdateWidget(old);
    if (old.userId != widget.userId) {
      _future = repo.fetch(widget.userId);   // new request for the new ID
    }
  }
  ...
}
```

---

## 6. App lifecycle

```dart
class _State extends State<MyApp> with WidgetsBindingObserver {
  @override
  void initState() {
    super.initState();
    WidgetsBinding.instance.addObserver(this);
  }

  @override
  void dispose() {
    WidgetsBinding.instance.removeObserver(this);
    super.dispose();
  }

  @override
  void didChangeAppLifecycleState(AppLifecycleState state) {
    switch (state) {
      case AppLifecycleState.resumed:   // back in the foreground
      case AppLifecycleState.inactive:  // transitioning (e.g. incoming call)
      case AppLifecycleState.paused:    // in background
      case AppLifecycleState.detached:
      case AppLifecycleState.hidden:
    }
  }
}
```

Use it to refresh data on `resumed`, pause video on `paused`, save drafts, stop sensors.

---

## 7. Keeping state alive

Tabs and `PageView`s dispose off-screen pages. To keep them:

```dart
class _TabState extends State<Tab> with AutomaticKeepAliveClientMixin {
  @override
  bool get wantKeepAlive => true;

  @override
  Widget build(BuildContext context) {
    super.build(context);                  // required!
    return ...;
  }
}
```

Or use `IndexedStack` (keeps all children alive; costs memory) or `go_router`'s `StatefulShellRoute`.

Better design: keep important state **outside** widgets (in a state-management layer), then the UI can be rebuilt freely.

---

## 8. Initialization patterns

### Async init (a very common bug!)

```dart
// WRONG: initState can't be async
@override
void initState() async { ... }

// RIGHT: kick off a future, handle with FutureBuilder or state variable
late final Future<Data> _future = repo.load();      // created ONCE (not in build!)

@override
Widget build(BuildContext context) {
  return FutureBuilder<Data>(
    future: _future,
    builder: (context, snap) { ... },
  );
}
```

Common bug: `future: repo.load()` **inside** `build` → a **new request every rebuild**.

---

## Common mistakes

- `UniqueKey()` in `build`.
- Using list index as key for dynamic lists.
- Heavy `GlobalKey` use as a shortcut for communication.
- Forgetting `super.initState()` / `super.dispose()`.
- Using `context` inherited lookups in `initState`.
- Not cancelling subscriptions → "setState() called after dispose()".
- Creating futures inside `build`.

---

## Code challenge

1. Reproduce the "swapped stateful boxes" bug and fix it with `ValueKey`.
2. Build a `UserView(userId)` that re-fetches when `userId` changes, using `didUpdateWidget`. Then achieve the same with `key: ValueKey(userId)` and compare approaches.
3. Make a page that logs every lifecycle method with a timestamp. Open it, rebuild it, navigate away, come back, and watch the order.
4. Build a tab layout where each tab has a scrolling list that **preserves scroll position** when switching tabs (try `PageStorageKey` and `AutomaticKeepAliveClientMixin`).

<details>
<summary>Hint — logging</summary>

```dart
void _log(String msg) => debugPrint('${DateTime.now().toIso8601String()} $msg');
// call _log('initState'), _log('build'), ... in each override
```
</details>

---

## Interview questions

1. What problem do keys solve?
2. `ValueKey` vs `ObjectKey` vs `UniqueKey` vs `GlobalKey`: when to use each?
3. Why is `UniqueKey()` inside `build` harmful?
4. How can you reset a widget's State from outside?
5. Order of lifecycle methods? Where do you start a stream, and where do you cancel it?
6. What is `didChangeDependencies` for?
7. When is `didUpdateWidget` called?
8. How do you keep a tab's state alive? What are the trade-offs?
9. Why shouldn't you create a `Future` in `build`?

<details>
<summary>Hints</summary>

1. Correctly associating State/elements with items when position or order changes.
2. Value: natural IDs. Object: object identity. Unique: force new. Global: Form/Scaffold access or moving widgets across the tree.
3. A new identity each rebuild destroys State and animations every time.
4. Change its key.
5. initState → didChangeDependencies → build → (didUpdateWidget/setState) → deactivate → dispose. Start in `initState`, cancel in `dispose`.
6. Reacting to changes of inherited dependencies (and first-time setup needing context).
7. When the parent rebuilds this widget with a new configuration.
8. Keep-alive mixin, `IndexedStack`, or external state. Costs memory and CPU for off-screen pages.
9. Every rebuild would start a new request.
</details>
