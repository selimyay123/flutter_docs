# 04 — Enums, Records, and Patterns

## Goal

Use Dart 3's modern features to write short, safe, expressive code.

---

## 1. Enums

A fixed set of named values.

```dart
enum Status { pending, active, done }

void main() {
  final s = Status.active;
  print(s.name);                 // active
  print(s.index);                // 1
  print(Status.values);          // [Status.pending, ...]
  print(Status.values.byName('done'));
}
```

### Enhanced enums (with fields and methods)

```dart
enum Planet {
  mercury(mass: 3.30e23, radius: 2.44e6),
  earth(mass: 5.97e24, radius: 6.37e6);

  const Planet({required this.mass, required this.radius});
  final double mass;
  final double radius;

  double get gravity => 6.674e-11 * mass / (radius * radius);
}
```

### Exhaustive switch on enums

```dart
String label(Status s) => switch (s) {
      Status.pending => 'Waiting',
      Status.active => 'Running',
      Status.done => 'Finished',
    };
```

Add a new enum value and forget a case → **compile error**. Never use `default` on enums if you can avoid it, because it hides this safety.

---

## 2. Records

A record is an **anonymous, immutable group of values**. Great for returning many things.

```dart
(String, int) person = ('Ada', 36);
print(person.$1);   // Ada
print(person.$2);   // 36

// Named fields
({String name, int age}) p = (name: 'Ada', age: 36);
print(p.name);

// Mixed
(int, {String label}) m = (1, label: 'one');
```

Records have **value equality** automatically:

```dart
print((1, 2) == (1, 2)); // true
```

When to use a record vs a class:

| Record | Class |
|--------|-------|
| Small, temporary grouping | Concept with identity/behavior |
| Return multiple values | Needs methods, validation |
| No boilerplate | Used widely in the app |

---

## 3. Patterns

Patterns let you **match** and **destructure** data.

### Destructuring

```dart
final (name, age) = ('Ada', 36);

final [first, second, ...rest] = [1, 2, 3, 4, 5];   // rest = [3, 4, 5]

final {'id': id, 'tag': tag} = {'id': 7, 'tag': 'x'};

// Swap without temp
var a = 1, b = 2;
(a, b) = (b, a);
```

### Object patterns

```dart
class Point {
  Point(this.x, this.y);
  final int x, y;
}

void main() {
  final Point(:x, :y) = Point(3, 4);
  print('$x,$y');
}
```

### Pattern matching in `switch`

```dart
String classify(Object o) => switch (o) {
      int n when n < 0 => 'negative int',
      int() => 'int',
      String s when s.isEmpty => 'empty string',
      String() => 'string',
      [] => 'empty list',
      [var one] => 'list with one: $one',
      [var a, var b] => 'pair: $a, $b',
      (int x, int y) => 'point $x,$y',
      _ => 'something else',
    };
```

### `if-case`

```dart
final json = {'name': 'Ada', 'age': 36};

if (json case {'name': String name, 'age': int age}) {
  print('$name is $age');   // both variables are safely typed here
}
```

This is **safe JSON parsing** without casts. Very senior.

### Relational and logical patterns

```dart
String bucket(int n) => switch (n) {
      < 0 => 'neg',
      0 => 'zero',
      >= 1 && < 10 => 'small',
      _ => 'big',
    };
```

---

## 4. Putting it together: a tiny expression evaluator

```dart
sealed class Expr {}
class Num extends Expr { Num(this.v); final double v; }
class Add extends Expr { Add(this.l, this.r); final Expr l, r; }
class Mul extends Expr { Mul(this.l, this.r); final Expr l, r; }

double eval(Expr e) => switch (e) {
      Num(:final v) => v,
      Add(:final l, :final r) => eval(l) + eval(r),
      Mul(:final l, :final r) => eval(l) * eval(r),
    };

void main() {
  print(eval(Add(Num(2), Mul(Num(3), Num(4))))); // 14.0
}
```

Sealed + patterns = the compiler proves you handled every node type.

---

## Common mistakes

- Using `default`/`_` and silencing exhaustiveness checks.
- Returning a huge record `(a, b, c, d, e)` — use a class when it grows.
- Forgetting that record fields positions start at `$1`.

---

## Code challenge

1. Write `(int, int) divMod(int a, int b)` returning quotient and remainder.
2. Add a `Sub` class to the evaluator above. What does the compiler tell you before you add the case?
3. Write `String describeJson(Map<String, Object?> json)` using `if-case`, returning `"User Ada (36)"` or `"Invalid"`.

<details>
<summary>Solution</summary>

```dart
(int, int) divMod(int a, int b) => (a ~/ b, a % b);

class Sub extends Expr { Sub(this.l, this.r); final Expr l, r; }
// Compiler error: "The type 'Expr' is not exhaustively matched by the switch cases"
// Fix: add  Sub(:final l, :final r) => eval(l) - eval(r),

String describeJson(Map<String, Object?> json) {
  if (json case {'name': String name, 'age': int age}) {
    return 'User $name ($age)';
  }
  return 'Invalid';
}
```
</details>

---

## Interview questions

1. What is an enhanced enum? Give a use case.
2. Record vs class vs `Map` for returning multiple values?
3. What is pattern destructuring? Show a swap.
4. Why is exhaustive switching important?
5. How does `if-case` improve JSON parsing?

<details>
<summary>Hints</summary>

1. Enum with fields/methods/interfaces, e.g. `HttpStatus` with code and message.
2. Record: typed, immutable, lightweight; Map: no type safety; class: when behavior/identity needed.
3. Unpack values into variables: `(a, b) = (b, a)`.
4. Compiler tells you when new cases are unhandled → no forgotten branch bugs.
5. Check shape and types together; variables are already typed, no unsafe casts.
</details>
