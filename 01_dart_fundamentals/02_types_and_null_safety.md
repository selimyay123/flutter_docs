# 02 — Types and Null Safety

## Goal

Know Dart's built-in types and master **null safety**, the feature that removes the "billion dollar mistake".

---

## 1. Built-in types

| Type | Example | Notes |
|------|---------|-------|
| `int` | `42` | 64-bit whole numbers (on web: JS number limits) |
| `double` | `3.14` | 64-bit decimals |
| `num` | `1` or `1.5` | Parent of `int` and `double` |
| `String` | `'hi'` | Immutable sequence of UTF-16 code units |
| `bool` | `true` | Only `true` / `false` (no "truthy" values!) |
| `List<T>` | `[1, 2]` | Ordered collection |
| `Set<T>` | `{1, 2}` | Unique values |
| `Map<K, V>` | `{'a': 1}` | Key → value |
| `Object` | any | Root of all non-null types |
| `Null` | `null` | The type of `null` |

Important: in Dart, **everything is an object**, even numbers. `5.isEven` works.

```dart
void main() {
  print(5.isEven);         // false
  print(3.7.round());      // 4
  print(int.parse('42'));  // 42
  print(double.tryParse('x')); // null (safe parse)
}
```

### Conditions need a real `bool`

```dart
var count = 1;
if (count) {}      // ERROR in Dart (no truthiness)
if (count != 0) {} // OK
```

---

## 2. Null safety

By default, a type **cannot** hold `null`.

```dart
String name = 'Ada';
name = null;        // ERROR

String? nickname = 'A';  // the ? means "may be null"
nickname = null;         // OK
```

A `String?` is a **different type** from `String`. The compiler forces you to handle `null`
before you can use the value. Null errors move from **runtime crashes** to **compile errors**.

### Tools to handle nullable values

```dart
String? name;

// 1. Check
if (name != null) {
  print(name.length); // Dart "promotes" name to String inside the if
}

// 2. ?. safe call -> null if name is null
print(name?.length);

// 3. ?? default value
print(name ?? 'Unknown');

// 4. ??= assign only if null
name ??= 'Guest';

// 5. ! "I promise it's not null" (dangerous!)
print(name!.length); // crashes if name is null
```

### Rules of thumb

- Prefer `??` and `?.` over `!`.
- Every `!` is a promise. If the promise breaks → crash. Use it only when you can prove non-null.
- Design APIs to avoid nullable types when possible (return an empty list, not `null`).

### Type promotion limits

Promotion works on **local variables**, not on fields that could change.

```dart
class User {
  String? name;
  void greet() {
    if (name != null) {
      // print(name.length); // ERROR: field might change between check and use
    }
  }
}
```

Fix: copy into a local variable.

```dart
void greet() {
  final n = name;
  if (n != null) print(n.length);
}
```

---

## 3. Type checks and casts

```dart
Object value = 'hello';

if (value is String) {
  print(value.length);      // promoted to String
}

final s = value as String;  // cast: throws if wrong type
```

- `is` — check (and promote)
- `is!` — negative check
- `as` — force cast (can throw)

---

## 4. Type inference and `Object?`

- `Object` = any non-null value.
- `Object?` = any value, including null. This is the real top type.
- `Never` = a function that never returns (e.g. always throws).

---

## 5. Numbers: common traps

```dart
print(7 / 2);   // 3.5 (always double)
print(7 ~/ 2);  // 3   (integer division)
print(7 % 2);   // 1   (remainder)
print(0.1 + 0.2 == 0.3); // false! floating point
```

Never compare doubles with `==` for money. Use integers (cents) or a decimal package.

---

## Common mistakes

- Using `!` to silence the compiler.
- Returning `null` instead of an empty collection.
- Comparing doubles with `==`.
- Forgetting that `/` returns a `double`.

---

## Code challenge

Write `String describe(String? name, int? age)`:

- Both present → `"Ada is 36"`.
- Name only → `"Ada (age unknown)"`.
- Age only → `"Someone is 36"`.
- Neither → `"Unknown person"`.

No `!` allowed.

<details>
<summary>Solution</summary>

```dart
String describe(String? name, int? age) {
  final who = name ?? 'Someone';
  if (name == null && age == null) return 'Unknown person';
  if (age == null) return '$who (age unknown)';
  return '$who is $age';
}
```
</details>

---

## Interview questions

1. What problem does null safety solve? How does it change error timing?
2. `String`, `String?`, `Object`, `Object?`, `dynamic`: order them from most to least restrictive.
3. What is type promotion, and why does it fail for class fields?
4. When is `!` acceptable?
5. What does `7 ~/ 2` return and why does `/` always return `double`?
6. Why can't you write `if (list)` in Dart?

<details>
<summary>Hints</summary>

1. Null dereference crashes. Null safety makes them compile-time errors.
2. `String` → `String?` → `Object` → `Object?` → `dynamic` (dynamic skips checks).
3. Compiler narrows a type after a check. Fields can be modified by other code between check and use, so it cannot guarantee.
4. When you have just proven non-null in a way the compiler cannot see, ideally with a comment explaining it.
5. `3` (truncating division). `/` is true division, so result may be fractional.
6. Dart requires `bool` conditions, avoiding accidental truthy/falsy bugs.
</details>
