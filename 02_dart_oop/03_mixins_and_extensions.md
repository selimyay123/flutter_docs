# 03 — Mixins and Extensions

## Goal

Share code **without** inheritance and add methods to types you do not own.

---

## 1. Mixins

A mixin is a bag of behavior you can "mix into" any class. A class can use **many** mixins.

```dart
mixin Swimmer {
  void swim() => print('Swimming');
}

mixin Flyer {
  void fly() => print('Flying');
}

class Animal {}

class Duck extends Animal with Swimmer, Flyer {}
class Fish extends Animal with Swimmer {}

void main() {
  Duck().fly();
  Fish().swim();
}
```

### Restrict a mixin with `on`

```dart
abstract class Widgetish {
  void rebuild();
}

mixin Refreshable on Widgetish {
  void refresh() {
    print('refreshing');
    rebuild();               // allowed: we know it's a Widgetish
  }
}
```

Only classes that extend/implement `Widgetish` can use `Refreshable`.

### Order matters

When two mixins define the same method, the **last** one wins.

```dart
class C extends Base with A, B {}  // B overrides A overrides Base
```

### Real Flutter mixins

- `SingleTickerProviderStateMixin` — gives a `State` a ticker for animations.
- `AutomaticKeepAliveClientMixin` — keep tab state alive.
- `WidgetsBindingObserver` — app lifecycle callbacks.

### Mixin vs abstract class

| | Mixin | Abstract class |
|--|-------|----------------|
| Multiple per class | Yes | No (single `extends`) |
| Has constructors | No | Yes |
| Use for | Reusable capability | Base type with identity |

---

## 2. Extension methods

Add methods to **existing types** (even built-in ones) without editing them.

```dart
extension StringX on String {
  bool get isBlank => trim().isEmpty;

  String capitalize() =>
      isEmpty ? this : '${this[0].toUpperCase()}${substring(1)}';
}

extension IntX on int {
  bool get isPrime {
    if (this < 2) return false;
    for (var i = 2; i * i <= this; i++) {
      if (this % i == 0) return false;
    }
    return true;
  }
}

void main() {
  print('  '.isBlank);          // true
  print('dart'.capitalize());   // Dart
  print(7.isPrime);             // true
}
```

### Extensions on generics and nullable types

```dart
extension ListX<T> on List<T> {
  T? get second => length >= 2 ? this[1] : null;
}

extension NullableStringX on String? {
  bool get isNullOrEmpty => this == null || this!.isEmpty;
}
```

### Flutter favorite: `BuildContext` extensions

```dart
extension ContextX on BuildContext {
  ThemeData get theme => Theme.of(this);
  Size get screenSize => MediaQuery.sizeOf(this);
}
// usage: context.theme.colorScheme.primary
```

### Extension types (Dart 3.3+)

Zero-cost wrappers that give a type a new interface at compile time.

```dart
extension type UserId(int value) {
  bool get isValid => value > 0;
}

void load(UserId id) {}

void main() {
  load(UserId(5));
  // load(5);    // ERROR: int is not UserId — prevents mixing up ids
}
```

Great for avoiding the bug "I passed `orderId` where `userId` was expected."

### Limits of extensions

- They are **static**: resolved at compile time based on the variable's static type.
- They can't add fields (state).
- A real member of the class always wins over an extension member.
- If two extensions clash, you must resolve manually: `StringX(s).capitalize()`.

---

## 3. Choosing the tool

| Need | Use |
|------|-----|
| Is-a relationship + shared state | Class inheritance |
| Reusable behavior across unrelated classes | Mixin |
| Add helpers to a type you don't own | Extension |
| Strongly typed wrapper with no overhead | Extension type |
| Just a utility function | Top-level function |

---

## Common mistakes

- Using mixins to hold state shared across too many classes (hidden coupling).
- Large extension "junk drawers" with unrelated helpers.
- Expecting extensions to be dynamically dispatched (they are not).

---

## Code challenge

1. Write an extension on `List<int>` named `sum`, `average` (returns `double?`, null when empty).
2. Write a mixin `Loggable` with `void log(String message)` that prints `[ClassName] message` (hint: `runtimeType`). Use it in two classes.

<details>
<summary>Solution</summary>

```dart
extension IntListX on List<int> {
  int get sum => fold(0, (a, b) => a + b);
  double? get average => isEmpty ? null : sum / length;
}

mixin Loggable {
  void log(String message) => print('[$runtimeType] $message');
}

class Cart with Loggable {
  void add() => log('item added');
}

class Auth with Loggable {
  void login() => log('logged in');
}
```
</details>

---

## Interview questions

1. What is a mixin, and how does it differ from inheritance?
2. What does `on` do in `mixin M on X`?
3. Are extension methods dynamically dispatched? Why does it matter?
4. Can an extension add a field? Why not?
5. What problem do extension types solve?
6. If two mixins define the same method, which wins?

<details>
<summary>Hints</summary>

1. Reuse behavior across class hierarchies; many per class; no constructor.
2. Restricts which classes can use the mixin and lets it call that superclass's members.
3. No, resolved statically by the declared type; calling on a `dynamic` variable won't find it.
4. Extensions don't change the object's layout, so no storage.
5. Type-safe zero-cost wrappers (e.g. different ID types).
6. The last mixin listed in `with`.
</details>
