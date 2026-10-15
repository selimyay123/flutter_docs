# 01 — Classes and Objects

## Goal

Model real things with code. Understand constructors, fields, methods, getters, and `this`.

---

## 1. A class is a blueprint, an object is a thing built from it

```dart
class Dog {
  String name;
  int age;

  Dog(this.name, this.age);       // constructor with shorthand

  void bark() => print('$name says Woof!');
}

void main() {
  final rex = Dog('Rex', 3);      // no `new` needed
  rex.bark();
}
```

- **Fields** = data (`name`, `age`).
- **Methods** = behavior (`bark`).
- **Constructor** = builds an object.

`this.name` in the constructor parameter list is **syntactic sugar** for assigning the field.

---

## 2. Constructors in depth

### Named parameters (most common in Flutter)

```dart
class User {
  User({required this.name, this.age = 0, this.email});

  final String name;
  final int age;
  final String? email;
}

final u = User(name: 'Ada', email: 'a@x.com');
```

### Named constructors

```dart
class Point {
  const Point(this.x, this.y);
  const Point.origin() : x = 0, y = 0;                 // initializer list
  Point.fromJson(Map<String, dynamic> json)
      : x = json['x'] as double,
        y = json['y'] as double;

  final double x;
  final double y;
}
```

The part after `:` is the **initializer list**. It runs **before** the constructor body.
Use it to set `final` fields and add `assert`s.

### Const constructors

```dart
const p = Point(1, 2);
const q = Point(1, 2);
print(identical(p, q)); // true  - same object in memory!
```

All fields must be `final`. Widgets use this a lot.

### Factory constructors

A factory can **return an existing object** or a subtype (a normal constructor always builds a new one).

```dart
class Logger {
  static final Map<String, Logger> _cache = {};
  final String name;

  Logger._(this.name);                        // private constructor

  factory Logger(String name) =>
      _cache.putIfAbsent(name, () => Logger._(name));
}

void main() {
  print(identical(Logger('a'), Logger('a'))); // true
}
```

### Redirecting constructors

```dart
class Rect {
  Rect(this.w, this.h);
  Rect.square(double side) : this(side, side);
  final double w, h;
}
```

---

## 3. Getters and setters

Look like fields but run code.

```dart
class Circle {
  Circle(this.radius);
  double radius;

  double get area => 3.14159 * radius * radius;   // computed property

  set diameter(double d) => radius = d / 2;
}

final c = Circle(2);
print(c.area);
c.diameter = 10;
```

Prefer a getter over a method when it is cheap, has no side effects, and feels like data.

---

## 4. Static members

Belong to the **class**, not an instance.

```dart
class MathUtils {
  static const pi = 3.14159;
  static int square(int x) => x * x;
}

print(MathUtils.square(3));
```

In Dart, top-level functions and constants are often better than static-only classes.

---

## 5. Encapsulation

Hide what callers should not touch. Prefix with `_`.

```dart
class BankAccount {
  double _balance = 0;                // private to the file

  double get balance => _balance;     // read-only from outside

  void deposit(double amount) {
    if (amount <= 0) throw ArgumentError('Amount must be positive');
    _balance += amount;
  }
}
```

Why? You can protect rules ("balance never negative") in **one place**.

---

## 6. `toString`, `==`, `hashCode`

By default, two objects are equal only if they are the **same instance**.

```dart
class P {
  P(this.x);
  final int x;
}
print(P(1) == P(1)); // false
```

To compare by **value** you override `==` and `hashCode` (see lesson 06).
Override `toString` for helpful debugging output:

```dart
@override
String toString() => 'Dog($name, $age)';
```

---

## 7. Null-safe fields and `late`

Every non-nullable field must be set before the constructor body ends:

```dart
class Foo {
  Foo(this.a);
  final int a;
  // int b; // ERROR: non-nullable field not initialized
  late final int b = a * 2; // lazy: computed on first read
}
```

---

## Common mistakes

- Making all fields public and mutable. Prefer `final`.
- Using constructors that leave objects half-built (invalid state).
- Forgetting `const` on constructors for immutable classes.
- Heavy work inside constructors.

---

## Code challenge

Create a `Rectangle` class with `width`, `height` (final, required named params, must be > 0 — use `assert`),
a getter `area`, a getter `isSquare`, a named constructor `Rectangle.square(double side)`, and a nice `toString`.

<details>
<summary>Solution</summary>

```dart
class Rectangle {
  Rectangle({required this.width, required this.height})
      : assert(width > 0 && height > 0, 'Sides must be positive');

  Rectangle.square(double side) : this(width: side, height: side);

  final double width;
  final double height;

  double get area => width * height;
  bool get isSquare => width == height;

  @override
  String toString() => 'Rectangle(${width}x$height)';
}
```
</details>

---

## Interview questions

1. What is the difference between a normal constructor, a named constructor, and a factory constructor?
2. What is the initializer list and when does it run?
3. What does a `const` constructor give you?
4. Getter vs method: how do you choose?
5. Is `_name` private to the class or to the file in Dart?
6. Why prefer `final` fields?

<details>
<summary>Hints</summary>

1. Normal builds a new instance; named gives intent/alternate ways; factory may return cached/subtype instances and can't use `this`.
2. After `:` before the body; used for final fields and asserts.
3. Compile-time canonical immutable instances → memory sharing, and in Flutter skipped rebuilds.
4. Getter for cheap, side-effect-free, data-like values; method for work or side effects.
5. File (library).
6. Immutability → predictable, thread-safe, easier to reason about.
</details>
