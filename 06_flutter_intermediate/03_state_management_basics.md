# 03 — State Management Basics

## Goal

Share and update state across many widgets **cleanly**. Understand the ideas first; packages come next lesson.

---

## 1. Two kinds of state

| Kind | Definition | Example | Where |
|------|------------|---------|-------|
| **Ephemeral (local/UI) state** | Belongs to one widget | Current tab, animation progress, text in a field, "is expanded" | `StatefulWidget` + `setState` |
| **App (shared) state** | Needed by many parts / survives screens | User session, cart, settings, fetched data | State management solution |

Not everything needs a package. **Start simple.** Move up only when you feel pain.

---

## 2. The problem

```
App
 ├─ HomePage ── ProductList ── ProductCard ── [AddToCart button]
 └─ CartPage ── CartBadge (shows count)
```

`ProductCard` and `CartBadge` are far apart, but both need the cart. Passing the cart through every constructor ("prop drilling") is painful and brittle.

---

## 3. Evolution of solutions

### Level 0 — lift state up + pass down

Fine for 2–3 levels.

### Level 1 — `ValueNotifier` + `ValueListenableBuilder` (built in!)

```dart
final counter = ValueNotifier<int>(0);

// anywhere
counter.value++;

// UI rebuilds ONLY this part when the value changes
ValueListenableBuilder<int>(
  valueListenable: counter,
  builder: (context, value, child) => Text('$value'),
)
```

Dispose it when you own it: `counter.dispose()`.

### Level 2 — `ChangeNotifier` + `ListenableBuilder`

```dart
class CartModel extends ChangeNotifier {
  final List<Product> _items = [];
  List<Product> get items => List.unmodifiable(_items);
  int get count => _items.length;
  double get total => _items.fold(0, (s, p) => s + p.price);

  void add(Product p) {
    _items.add(p);
    notifyListeners();               // tell the UI to rebuild
  }

  void remove(Product p) {
    _items.remove(p);
    notifyListeners();
  }
}

ListenableBuilder(
  listenable: cart,
  builder: (context, _) => Text('Items: ${cart.count}'),
)
```

How does the UI get the same `cart` instance everywhere? → Level 3.

### Level 3 — InheritedWidget / dependency injection

Put the model **above** the widgets that need it, and look it up through the tree. Packages (`provider`, `riverpod`, `flutter_bloc`) do this for you with less boilerplate. See next lesson.

---

## 4. Architecture of state: three concepts to separate

```
UI (widgets)  ──events──►  Logic holder (ChangeNotifier / Cubit / Notifier)  ──►  Repository  ──►  API / DB
      ▲                              │
      └─────────── state ◄───────────┘
```

1. **State** — a plain data class describing what the screen needs.
2. **Logic** — methods that change state (`addToCart`, `load`, `search`).
3. **UI** — renders state and sends events.

Separation lets you **test logic without widgets**.

### Model state explicitly (avoid boolean soup)

```dart
// BAD: impossible combinations (isLoading && hasError && data != null ...)
bool isLoading = false;
String? error;
List<Item>? items;

// GOOD: exactly one situation at a time
sealed class ItemsState {}
class ItemsLoading extends ItemsState {}
class ItemsError extends ItemsState { ItemsError(this.message); final String message; }
class ItemsData extends ItemsState { ItemsData(this.items); final List<Item> items; }

Widget build(BuildContext context) => switch (state) {
      ItemsLoading() => const CircularProgressIndicator(),
      ItemsError(:final message) => Text(message),
      ItemsData(:final items) => ItemList(items),
    };
```

**Make impossible states unrepresentable.** A senior principle.

---

## 5. Immutable state & rebuilds

Frameworks compare old and new state to decide to rebuild. Don't mutate in place:

```dart
// BAD: same list instance, nothing notices (in value-compare frameworks)
state.items.add(item);

// GOOD
state = state.copyWith(items: [...state.items, item]);
```

---

## 6. Rebuild only what changes

- Listen **as low in the tree as possible**.
- Select only the data you need (`context.select`, `ref.watch(provider.select(...))`, `BlocSelector`).
- Pass static children via the `child` parameter of builders.
- Split big widgets so a change affects a small subtree.

---

## 7. A tiny, full example (ChangeNotifier without packages)

```dart
class CounterModel extends ChangeNotifier {
  int _value = 0;
  int get value => _value;
  void increment() {
    _value++;
    notifyListeners();
  }
}

class CounterScope extends InheritedNotifier<CounterModel> {
  const CounterScope({super.key, required CounterModel model, required super.child})
      : super(notifier: model);

  static CounterModel of(BuildContext context) =>
      context.dependOnInheritedWidgetOfExactType<CounterScope>()!.notifier!;
}

void main() {
  final model = CounterModel();
  runApp(CounterScope(model: model, child: const MaterialApp(home: CounterPage())));
}

class CounterPage extends StatelessWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context) {
    final model = CounterScope.of(context);       // subscribes to changes
    return Scaffold(
      body: Center(child: Text('${model.value}')),
      floatingActionButton: FloatingActionButton(
        onPressed: model.increment,
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

That is, in essence, how `provider` works. Packages add lifecycle management, laziness, selection and tooling.

---

## 8. How to choose (guide)

| Situation | Choice |
|-----------|--------|
| Local UI state | `setState` |
| One value, small scope | `ValueNotifier` |
| Small/medium app, simple | `provider` + `ChangeNotifier` |
| Compile-safe, testable, async-heavy, scalable | **Riverpod** |
| Strict event→state, large team, audit trail | **Bloc / Cubit** |
| Existing team standard | **Whatever the team uses** |

There is no "best". There is *consistent* and *understood by the team*.

---

## Common mistakes

- Putting all state in one huge global object.
- Mutating state in place and forgetting `notifyListeners()`.
- Calling `notifyListeners` in `build`.
- Using a package for one boolean.
- Boolean soup instead of a sealed state.
- Business logic inside widgets.

---

## Code challenge

1. Build a **shopping cart** with a `CartModel extends ChangeNotifier` provided with `InheritedNotifier`: product list page with "Add" buttons, an app bar **badge** showing count, and a cart page with total and remove.
2. Refactor a `bool isLoading / String? error / List? data` screen into a **sealed state** with exhaustive `switch`.
3. Add `copyWith` to an immutable `TodoState` and implement `toggle(id)` without mutating.

<details>
<summary>Hint — #3</summary>

```dart
TodoState toggle(String id) => copyWith(
  todos: [for (final t in todos) t.id == id ? t.copyWith(done: !t.done) : t],
);
```
</details>

---

## Interview questions

1. Ephemeral vs app state: how do you decide where state lives?
2. What is "prop drilling" and how do you avoid it?
3. `ValueNotifier` vs `ChangeNotifier`?
4. What does `notifyListeners()` do?
5. Why separate UI from logic? What do you gain for testing?
6. What does "make impossible states unrepresentable" mean? Example?
7. Why should state be immutable in most solutions?
8. How do you limit rebuilds when state changes?
9. How does an `InheritedWidget`-based solution work under the hood?

<details>
<summary>Hints</summary>

1. If only one widget cares → local; if many/persist → shared state.
2. Passing data via many constructors; use DI/inherited lookup/state management.
3. ValueNotifier holds one value; ChangeNotifier can hold arbitrary state and notify manually.
4. Informs listeners/builders to rebuild.
5. Logic can be unit-tested without a widget tree.
6. Model states with sealed classes so contradictory flags can't exist.
7. Identity/equality-based change detection; predictability.
8. Local listening, selectors, splitting widgets, const children.
9. Model stored in an inherited element; dependents registered in `of(context)`; notified on change.
</details>
