# 02 — Inheritance, Abstract Classes, Interfaces

## Goal

Reuse and abstract behavior. Know when to use `extends`, `implements`, and when **not** to.

---

## 1. Inheritance (`extends`)

A child class **is a** parent class with extras.

```dart
class Animal {
  Animal(this.name);
  final String name;

  void speak() => print('$name makes a sound');
}

class Dog extends Animal {
  Dog(super.name);                 // pass to parent constructor

  @override
  void speak() {
    super.speak();                 // call parent version
    print('$name barks');
  }
}
```

- `super` refers to the parent.
- `@override` — tell the compiler you override. It catches typos.
- Dart has **single inheritance**: one parent only.

---

## 2. Abstract classes

An abstract class can't be instantiated. It defines a **contract** (and optionally shared code).

```dart
abstract class Shape {
  double get area;                  // abstract: no body
  void describe() => print('Area: $area');   // concrete: shared code
}

class Square extends Shape {
  Square(this.side);
  final double side;
  @override
  double get area => side * side;
}

class Circle extends Shape {
  Circle(this.r);
  final double r;
  @override
  double get area => 3.14159 * r * r;
}

void main() {
  final shapes = <Shape>[Square(2), Circle(1)];
  for (final s in shapes) {
    s.describe();      // polymorphism!
  }
}
```

---

## 3. Polymorphism

One interface, many behaviors. The **actual** object decides which method runs at runtime.

```dart
Shape s = Circle(1);   // variable type is Shape, object is Circle
s.describe();          // uses Circle's area
```

This is the secret of flexible code: depend on the **abstraction** (`Shape`), not on concrete classes.

---

## 4. Interfaces (`implements`)

In Dart every class **is also an interface**. `implements` means: "I promise to provide all the members, but I get **no code** from it."

```dart
abstract interface class Storage {
  Future<void> save(String key, String value);
  Future<String?> load(String key);
}

class MemoryStorage implements Storage {
  final _data = <String, String>{};

  @override
  Future<void> save(String key, String value) async => _data[key] = value;

  @override
  Future<String?> load(String key) async => _data[key];
}
```

You can implement **many** interfaces: `class A implements B, C {}`.

### `extends` vs `implements`

| | `extends` | `implements` |
|--|-----------|--------------|
| Inherits code | Yes | No |
| How many | One | Many |
| Use for | "is-a" with shared behavior | Pure contract |

---

## 5. Dart 3 class modifiers

| Modifier | Meaning |
|----------|---------|
| `abstract` | Can't be instantiated |
| `interface` | Can be implemented but not extended (outside its library) |
| `base` | Can be extended but not implemented |
| `final` | Can't be extended or implemented (outside library) |
| `sealed` | Fixed set of subtypes → **exhaustive switch** |
| `mixin class` | Can be used as class and mixin |

### Sealed classes — the best tool for modeling states

```dart
sealed class LoadState {}
class Loading extends LoadState {}
class Loaded extends LoadState {
  Loaded(this.data);
  final String data;
}
class Failed extends LoadState {
  Failed(this.error);
  final String error;
}

String render(LoadState s) => switch (s) {
      Loading() => 'Spinner',
      Loaded(:final data) => 'Show $data',
      Failed(:final error) => 'Show error: $error',
    };
```

If you add a new subclass and forget a case, the compiler **errors**. This catches bugs at compile time.

---

## 6. Composition over inheritance

Deep inheritance trees become rigid. Prefer **has-a** over **is-a**.

```dart
// Bad idea: class FlyingSwimmingDuck extends FlyingDuck extends SwimmingDuck ...

// Better: compose small behaviors
class Duck {
  Duck({required this.flying, required this.swimming});
  final FlyBehavior flying;
  final SwimBehavior swimming;
}
```

Flutter itself follows this: you **compose** small widgets rather than inherit big ones.

---

## 7. SOLID in one minute

| Letter | Idea |
|--------|------|
| **S** | A class has **one** reason to change |
| **O** | Extend behavior without editing old code |
| **L** | A subtype must work wherever the parent works |
| **I** | Many small interfaces beat one giant one |
| **D** | Depend on abstractions, not concretions |

Dependency inversion example: `UserRepository` depends on the `Storage` interface above,
not on a specific database. You can swap in `MemoryStorage` for tests.

---

## Common mistakes

- Inheriting just to reuse a few lines (use composition or a helper function).
- Deep hierarchies (> 2–3 levels).
- Using `dynamic` instead of an abstract type.
- Forgetting `@override`.

---

## Code challenge

Model a payment system:

- `abstract interface class PaymentMethod { double fee(double amount); }`
- `CreditCard` (2% fee), `Paypal` (1% + 0.30), `Cash` (no fee).
- Write `double total(double price, PaymentMethod m)`.
- Add a new `Crypto` method **without changing** `total`.

<details>
<summary>Solution</summary>

```dart
abstract interface class PaymentMethod {
  double fee(double amount);
}

class CreditCard implements PaymentMethod {
  @override
  double fee(double amount) => amount * 0.02;
}

class Paypal implements PaymentMethod {
  @override
  double fee(double amount) => amount * 0.01 + 0.30;
}

class Cash implements PaymentMethod {
  @override
  double fee(double amount) => 0;
}

class Crypto implements PaymentMethod {
  @override
  double fee(double amount) => 0.5;
}

double total(double price, PaymentMethod m) => price + m.fee(price);
```

This follows the **Open/Closed** principle: add new methods with no edits to `total`.
</details>

---

## Interview questions

1. `extends` vs `implements` vs `with`: what does each give you?
2. What is polymorphism? Show an example.
3. Why are sealed classes useful for state modeling?
4. Composition vs inheritance: when do you choose which?
5. Explain the Liskov Substitution Principle with a violation example.
6. What is dependency inversion and how does it help testing?

<details>
<summary>Hints</summary>

1. extends: code + type from one parent; implements: contract only; with: mixin code.
2. A `List<Shape>` calling `area` on different subclasses.
3. Exhaustive switches → compiler enforces handling every state.
4. Inheritance for true is-a with stable hierarchy; otherwise compose.
5. `Square extends Rectangle` where setting width changes height breaks code expecting a Rectangle's independent sides.
6. Depend on interfaces so you can inject fakes/mocks.
</details>
