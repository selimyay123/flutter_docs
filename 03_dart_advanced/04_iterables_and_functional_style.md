# 04 — Iterables and Functional Style

## Goal

Transform data in short, readable chains. Understand **lazy** evaluation.

---

## 1. Iterable

`List`, `Set`, and `Map.keys` are all `Iterable`s: "something you can loop over."
Iterables offer powerful methods.

```dart
final nums = [1, 2, 3, 4, 5, 6];

nums.where((n) => n.isEven);                  // 2, 4, 6        (filter)
nums.map((n) => n * n);                       // 1,4,9,...      (transform)
nums.any((n) => n > 5);                       // true
nums.every((n) => n > 0);                     // true
nums.firstWhere((n) => n > 3);                // 4  (throws if none)
nums.firstWhere((n) => n > 10, orElse: () => -1); // -1
nums.reduce((a, b) => a + b);                 // 21 (throws if empty)
nums.fold(0, (sum, n) => sum + n);            // 21 (safe with empty)
nums.take(2);                                 // 1, 2
nums.skip(4);                                 // 5, 6
nums.takeWhile((n) => n < 4);                 // 1, 2, 3
nums.expand((n) => [n, n]);                   // 1,1,2,2,... (flatMap)
```

Also: `singleWhere`, `whereType<T>()`, `toList()`, `toSet()`, `join()`, `isEmpty`, `lastWhere`, `followedBy`.

---

## 2. Chaining

```dart
final result = users
    .where((u) => u.isActive)
    .map((u) => u.name.toUpperCase())
    .toList()
  ..sort();
```

Read it like a sentence: "active users → their upper-case names → as a list → sorted".

---

## 3. Laziness — important!

`where`, `map`, `take`, `skip` return a **lazy** `Iterable`. Nothing runs until you iterate it.

```dart
final lazy = [1, 2, 3].map((n) {
  print('mapping $n');
  return n * 2;
});
print('created');       // nothing mapped yet
print(lazy.first);      // maps ONLY the first element
```

Output:

```
created
mapping 1
2
```

Consequences:

1. Efficient: can work with infinite sequences.
2. **Re-evaluated every time you iterate it:**

```dart
final it = [1, 2, 3].map((n) { print('work'); return n; });
it.length;   // runs the function 3 times
it.length;   // runs again 3 times!
```

Fix: call `.toList()` once if you need to reuse results.

3. If the source list changes, the lazy view changes too.

### Generators with `sync*`

```dart
Iterable<int> naturals() sync* {
  var i = 0;
  while (true) {
    yield i++;
  }
}

print(naturals().where((n) => n.isOdd).take(5).toList()); // [1, 3, 5, 7, 9]
```

---

## 4. Functional thinking

| Idea | Meaning |
|------|---------|
| Immutability | Don't change data, produce new data |
| Pure functions | Output depends only on input |
| Higher-order functions | Functions that take/return functions |
| Declarative | Say *what*, not *how* |

Compare:

```dart
// Imperative
final names = <String>[];
for (final u in users) {
  if (u.age >= 18) names.add(u.name);
}

// Declarative
final names2 = users.where((u) => u.age >= 18).map((u) => u.name).toList();
```

Both are fine. Pick the one that is **clearest**. Don't build 10-step chains no one can read; break them into named variables or small functions.

---

## 5. Building maps and sets from iterables

```dart
final byId = {for (final u in users) u.id: u};            // Map<int, User>
final names = {for (final u in users) u.name};            // Set<String>
final lookup = Map.fromEntries(users.map((u) => MapEntry(u.id, u)));
```

Map helpers:

```dart
map.map((k, v) => MapEntry(k, v * 2));
map.entries.where((e) => e.value > 1);
map.removeWhere((k, v) => v == 0);
```

---

## 6. Sorting by multiple fields

```dart
people.sort((a, b) {
  final byAge = a.age.compareTo(b.age);
  return byAge != 0 ? byAge : a.name.compareTo(b.name);
});
```

`package:collection` adds `sortedBy`, `groupListsBy`, `maxBy`, `minBy`, `equals`, `flattened`, etc.

---

## Common mistakes

- Using `.map(...)` just for side effects (use `for` / `forEach`). Because it is lazy, nothing happens!
- Forgetting `.toList()` and re-running expensive lambdas.
- `reduce` on an empty list → error. Use `fold`.
- Long unreadable chains.

---

## Code challenge

Given:

```dart
final words = ['apple', 'banana', 'avocado', 'blueberry', 'cherry', 'apricot'];
```

1. Return the words starting with `'a'`, uppercased.
2. Return a `Map<String, int>` of first letter → number of words.
3. Return the longest word.
4. Write `Iterable<int> evens()` as an infinite generator, take the first 5.
5. Flatten `[[1,2],[3],[4,5]]` into `[1,2,3,4,5]`.

<details>
<summary>Solution</summary>

```dart
void main() {
  final words = ['apple', 'banana', 'avocado', 'blueberry', 'cherry', 'apricot'];

  print(words.where((w) => w.startsWith('a')).map((w) => w.toUpperCase()).toList());

  final counts = <String, int>{};
  for (final w in words) {
    counts.update(w[0], (c) => c + 1, ifAbsent: () => 1);
  }
  print(counts);

  print(words.reduce((a, b) => a.length >= b.length ? a : b));

  print(evens().take(5).toList());

  print([[1, 2], [3], [4, 5]].expand((l) => l).toList());
}

Iterable<int> evens() sync* {
  for (var i = 0;; i += 2) {
    yield i;
  }
}
```
</details>

---

## Interview questions

1. What does it mean that `map` and `where` are lazy?
2. Difference between `reduce` and `fold`?
3. Why can calling `.length` twice on a mapped iterable be expensive?
4. `map` vs `forEach` vs `for-in`: when to use each?
5. What is `sync*` and when would you use it?
6. How do you convert a list of objects into a map keyed by ID?

<details>
<summary>Hints</summary>

1. They compute elements only when you iterate, on demand.
2. `fold` has an initial value (works for empty and can change type); `reduce` uses first element and fails when empty.
3. The lazy pipeline re-runs on each iteration; cache with `toList()`.
4. `map` to transform, `forEach` for simple side effects, `for-in` when you need `break`, `await` or clarity.
5. Generator producing a lazy `Iterable` with `yield`.
6. Collection-for: `{for (final x in xs) x.id: x}`.
</details>
