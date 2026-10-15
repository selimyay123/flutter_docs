# 06 — Immutability and Equality

## Goal

Understand why Flutter loves immutable objects and how Dart compares objects.

---

## 1. Identity vs equality

| Concept | Question | Dart |
|---------|----------|------|
| Identity | Same object in memory? | `identical(a, b)` |
| Equality | Same value? | `a == b` |

By default `==` is identity. You must override it for value equality.

```dart
class Point {
  const Point(this.x, this.y);
  final int x, y;

  @override
  bool operator ==(Object other) =>
      identical(this, other) ||
      other is Point && other.runtimeType == runtimeType && other.x == x && other.y == y;

  @override
  int get hashCode => Object.hash(x, y);
}

void main() {
  print(Point(1, 2) == Point(1, 2)); // true
  final set = {Point(1, 2), Point(1, 2)};
  print(set.length); // 1
}
```

### The contract

1. If `a == b` then `a.hashCode == b.hashCode`. (**Always!**)
2. `==` must be reflexive, symmetric, transitive.
3. Equal fields used in `==` must also be used in `hashCode`.

If you break rule 1, `Set` and `Map` silently misbehave: they use `hashCode` first to find a bucket.

Use `Object.hash(a, b, c)` and `Object.hashAll(list)`.

---

## 2. Why immutability?

An immutable object **never changes after creation**.

Benefits:

- **Predictable:** no one can change it behind your back.
- **Thread/isolate safe:** nothing to lock.
- **Easy equality & caching.**
- **Flutter loves it:** to know whether to rebuild, Flutter compares old and new state. If you mutate a list in place, the "old" and "new" are the same object → nothing seems to change.

```dart
// BUG pattern in state management:
state.items.add(x);        // same list instance, listeners not notified
// FIX:
state = state.copyWith(items: [...state.items, x]);
```

---

## 3. How to make a class immutable

1. All fields `final`.
2. `const` constructor if possible.
3. Collections: copy in, expose unmodifiable views.
4. Provide `copyWith` to "change" something by making a new object.

```dart
class User {
  const User({required this.name, required this.age});
  final String name;
  final int age;

  User copyWith({String? name, int? age}) =>
      User(name: name ?? this.name, age: age ?? this.age);

  @override
  bool operator ==(Object other) =>
      other is User && other.name == name && other.age == age;

  @override
  int get hashCode => Object.hash(name, age);

  @override
  String toString() => 'User($name, $age)';
}

void main() {
  const a = User(name: 'Ada', age: 36);
  final b = a.copyWith(age: 37);
  print(a); // User(Ada, 36)  - unchanged
  print(b); // User(Ada, 37)
}
```

### Known weakness of `copyWith`

You can't set a nullable field **back to null** (`null` means "keep"). Solutions: a wrapper, a sentinel, or use code generation (`freezed`).

---

## 4. Reduce the boilerplate

Writing `==`, `hashCode`, `copyWith`, `toString`, `toJson` by hand is error-prone. Teams use:

- **`freezed`** + `json_serializable` (code generation).
- **`equatable`** (just implement `props`).
- **Dart records** for small value types (equality is free).
- (Dart macros were planned but the feature was cancelled — ignore old tutorials.)

```dart
// equatable example
class User extends Equatable {
  const User(this.name, this.age);
  final String name;
  final int age;
  @override
  List<Object?> get props => [name, age];
}
```

You should **understand** the manual version for interviews.

---

## 5. Deep vs shallow immutability

```dart
class Team {
  const Team(this.members);
  final List<String> members;     // field is final, but the LIST is still mutable!
}

final t = Team(['a']);
t.members.add('b');               // allowed. Not truly immutable.
```

Fix: `List.unmodifiable(members)` in a constructor, or use `const` lists.

---

## Common mistakes

- Overriding `==` but not `hashCode`.
- Including mutable fields in `hashCode` of objects stored in a Set/Map (hash changes → object "lost").
- Believing `final` means deeply immutable.
- Comparing with `==` when you need `identical`.

---

## Code challenge

1. Make a `Money` value class (`amount` in cents as `int`, `currency` as `String`) with `==`, `hashCode`, `+` operator (same currency only, else throw), and `toString`.
2. Make `Team` truly immutable (members can't be changed from outside).

<details>
<summary>Solution</summary>

```dart
class Money {
  const Money(this.cents, this.currency);
  final int cents;
  final String currency;

  Money operator +(Money other) {
    if (other.currency != currency) {
      throw ArgumentError('Currency mismatch');
    }
    return Money(cents + other.cents, currency);
  }

  @override
  bool operator ==(Object other) =>
      other is Money && other.cents == cents && other.currency == currency;

  @override
  int get hashCode => Object.hash(cents, currency);

  @override
  String toString() => '${(cents / 100).toStringAsFixed(2)} $currency';
}

class Team {
  Team(List<String> members) : members = List.unmodifiable(members);
  final List<String> members;
}
```
</details>

---

## Interview questions

1. `identical` vs `==`: difference?
2. State the `==`/`hashCode` contract. What breaks if you violate it?
3. Why does Flutter state management prefer immutable state?
4. Does `final List` make the list immutable?
5. What is `copyWith` and what is its limitation?
6. What are options to avoid writing equality boilerplate?

<details>
<summary>Hints</summary>

1. Identity = same memory object; `==` = overridable value comparison.
2. Equal objects must have equal hash codes. Violation breaks `Set`/`Map` lookups.
3. Changes are detected by reference comparison; immutable data avoids hidden mutations.
4. No, only the reference is final.
5. Create a modified copy; can't easily reset nullable fields to null.
6. `freezed`, `equatable`, records.
</details>
