# 03 — Operators and Control Flow

## Goal

Make decisions and repeat actions. These are the building blocks of every algorithm.

---

## 1. Operators

| Group | Operators |
|-------|-----------|
| Arithmetic | `+ - * / ~/ %` |
| Compare | `== != < > <= >=` |
| Logic | `&& \|\| !` |
| Assign | `= += -= *= /= ??=` |
| Increment | `++ --` |
| Null-aware | `?? ??= ?. ?[]` |
| Type | `is is! as` |
| Ternary | `condition ? a : b` |
| Cascade | `..` and `?..` |
| Spread | `...` and `...?` |

`&&` and `||` are **short-circuit**: the right side runs only if needed.

```dart
if (user != null && user.isAdmin) { ... } // safe: second part skipped when user is null
```

### Cascade `..`

Call several things on the same object.

```dart
final sb = StringBuffer()
  ..write('Hello')
  ..write(' ')
  ..write('World');
```

---

## 2. `if` / `else`

```dart
final score = 82;
if (score >= 90) {
  print('A');
} else if (score >= 80) {
  print('B');
} else {
  print('C or less');
}
```

### `if` as expression (ternary)

```dart
final label = score >= 50 ? 'pass' : 'fail';
```

Avoid nested ternaries. They are hard to read.

---

## 3. Loops

```dart
// classic for
for (var i = 0; i < 3; i++) {
  print(i);
}

// for-in: best for collections
for (final item in ['a', 'b']) {
  print(item);
}

// while
var n = 3;
while (n > 0) {
  n--;
}

// do-while: body runs at least once
do {
  n++;
} while (n < 3);
```

- `break` — exit the loop now.
- `continue` — skip to next iteration.

Labels for nested loops:

```dart
outer:
for (var i = 0; i < 3; i++) {
  for (var j = 0; j < 3; j++) {
    if (i * j == 2) break outer;
  }
}
```

---

## 4. `switch` (modern Dart 3)

Switch **statement** (no fall-through; `break` not needed):

```dart
switch (command) {
  case 'open':
    print('Opening');
  case 'close' || 'shut':   // multiple patterns
    print('Closing');
  default:
    print('Unknown');
}
```

Switch **expression** (returns a value):

```dart
final text = switch (score) {
  >= 90 => 'A',
  >= 80 => 'B',
  _ => 'C or less',     // _ is the wildcard
};
```

With **guards**:

```dart
final kind = switch (n) {
  int x when x < 0 => 'negative',
  0 => 'zero',
  _ => 'positive',
};
```

---

## 5. Collection `if` and `for` (used heavily in Flutter UI)

```dart
final items = [
  'Home',
  if (isLoggedIn) 'Profile',
  for (var i = 1; i <= 3; i++) 'Tab $i',
];
```

---

## 6. Thinking tools: loop patterns

These tiny patterns appear in almost every algorithm:

| Pattern | Idea |
|---------|------|
| Accumulator | `sum += x` — build a result |
| Counter | `count++` — count matches |
| Min/Max tracker | keep best so far |
| Flag | `found = true` — remember an event |
| Two indexes | `i` and `j` move through data |

### Example — FizzBuzz

```dart
void main() {
  for (var i = 1; i <= 15; i++) {
    if (i % 15 == 0) {
      print('FizzBuzz');
    } else if (i % 3 == 0) {
      print('Fizz');
    } else if (i % 5 == 0) {
      print('Buzz');
    } else {
      print(i);
    }
  }
}
```

Notice: we check the **most specific** condition (`% 15`) first. Order of conditions matters.

---

## Common mistakes

- Off-by-one errors: `<` vs `<=`.
- Modifying a list while looping over it (throws `ConcurrentModificationError`).
- Infinite loops (forgetting to change the loop variable).
- Deep nesting. Use **early return** (guard clauses) instead:

```dart
// Bad
String f(int? x) {
  if (x != null) {
    if (x > 0) { return 'ok'; }
  }
  return 'bad';
}
// Good
String g(int? x) {
  if (x == null) return 'bad';
  if (x <= 0) return 'bad';
  return 'ok';
}
```

---

## Code challenge

1. Print all prime numbers up to 50.
2. Hard mode: print the first 10 Fibonacci numbers using a loop (no recursion).

<details>
<summary>Solution</summary>

```dart
bool isPrime(int n) {
  if (n < 2) return false;
  for (var d = 2; d * d <= n; d++) {
    if (n % d == 0) return false;
  }
  return true;
}

void main() {
  for (var i = 1; i <= 50; i++) {
    if (isPrime(i)) print(i);
  }

  var a = 0, b = 1;
  for (var i = 0; i < 10; i++) {
    print(a);
    final next = a + b;
    a = b;
    b = next;
  }
}
```

Why `d * d <= n`? If `n` has a divisor bigger than its square root, it also has a smaller one. So we only check up to `√n`.
</details>

---

## Interview questions

1. What is short-circuit evaluation? Give a safety example.
2. What is the difference between a switch statement and switch expression in Dart 3?
3. What does the cascade operator `..` do?
4. What is a guard clause and why do seniors like it?
5. Why does Dart not let you modify a list while iterating with `for-in`?
6. What is an off-by-one error? How do you prevent it?

<details>
<summary>Hints</summary>

1. Right side skipped if left decides outcome: `a != null && a.b`.
2. Expression returns a value, is exhaustive-checked for sealed types/enums, uses `=>`.
3. Runs multiple operations on one object and returns that object.
4. Early return for invalid cases → flatter, readable code.
5. The iterator becomes invalid when the structure changes → `ConcurrentModificationError`.
6. Loop runs one time too many/few. Test boundaries (0, 1, n-1, n) and use `for-in` when possible.
</details>
