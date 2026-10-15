# 05 — Collections: List, Set, Map

## Goal

Master the three core data structures. Algorithms are mostly "choose the right collection".

---

## 1. List — ordered, allows duplicates

```dart
final nums = [5, 3, 8];            // List<int>
final empty = <String>[];          // typed empty list

nums.add(1);                       // [5, 3, 8, 1]
nums.addAll([7, 9]);
nums.insert(0, 100);
nums.remove(8);                    // removes first match
nums.removeAt(0);
print(nums[0]);                    // access by index (RangeError if out of range!)
print(nums.length);
print(nums.first);
print(nums.last);
print(nums.contains(3));
print(nums.indexOf(3));
```

### Fixed vs growable vs unmodifiable

```dart
final a = List.filled(3, 0);               // [0, 0, 0] fixed length
final b = List.generate(5, (i) => i * i);  // [0, 1, 4, 9, 16]
final c = List.unmodifiable([1, 2]);       // cannot change
const d = [1, 2];                          // const -> cannot change
```

### Sorting

```dart
final names = ['Zoe', 'Adam', 'Mia'];
names.sort();                                      // in place
names.sort((a, b) => b.length.compareTo(a.length)); // custom: longest first
```

`sort` changes the list. `sorted` does not exist in core Dart (it is in `package:collection`).

### Slicing

```dart
print(nums.sublist(1, 3));   // items at index 1 and 2
print(nums.take(2).toList());
print(nums.skip(2).toList());
```

### Copy vs reference

```dart
final x = [1, 2];
final y = x;           // SAME list (reference)
y.add(3);
print(x);              // [1, 2, 3]

final z = [...x];      // copy (shallow)
final w = List.of(x);  // copy
```

**Shallow copy:** nested objects are still shared. Remember this in interviews.

---

## 2. Set — unique values, fast lookup

```dart
final tags = {'dart', 'flutter', 'dart'};  // {dart, flutter}
tags.add('mobile');
print(tags.contains('dart'));   // O(1) average

final a = {1, 2, 3};
final b = {3, 4};
print(a.union(b));          // {1, 2, 3, 4}
print(a.intersection(b));   // {3}
print(a.difference(b));     // {1, 2}
```

Use a `Set` when you ask "have I seen this?" or need to remove duplicates:

```dart
final unique = [1, 2, 2, 3, 3].toSet().toList(); // [1, 2, 3]
```

**Note:** `{}` alone is an empty **Map**, not a Set. Write `<int>{}` for an empty Set.

---

## 3. Map — key → value

```dart
final ages = <String, int>{'Ada': 36, 'Alan': 41};

ages['Grace'] = 85;             // add or update
print(ages['Ada']);             // 36
print(ages['Nobody']);          // null  (so the type is int?)
print(ages['Nobody'] ?? 0);     // 0

ages.remove('Alan');
print(ages.containsKey('Ada'));
print(ages.keys);
print(ages.values);

for (final entry in ages.entries) {
  print('${entry.key}: ${entry.value}');
}

// Update or insert in one step
ages.update('Ada', (v) => v + 1, ifAbsent: () => 1);
ages.putIfAbsent('Linus', () => 55);
```

Lookup is `O(1)` on average (hash table). This is the most powerful tool in problem solving.

### Counting with a Map (very common pattern)

```dart
Map<String, int> countWords(List<String> words) {
  final counts = <String, int>{};
  for (final w in words) {
    counts[w] = (counts[w] ?? 0) + 1;
  }
  return counts;
}
```

---

## 4. Choose the right collection

| Need | Use | Lookup cost |
|------|-----|-------------|
| Order matters, index access | `List` | by index O(1), by value O(n) |
| "Have I seen this?" | `Set` | O(1) avg |
| Find value by a key | `Map` | O(1) avg |
| Keep insertion order | default `Map`/`Set` (LinkedHashMap/Set) | — |
| Sorted keys | `SplayTreeMap` / `SplayTreeSet` (`dart:collection`) | O(log n) |
| FIFO queue | `Queue` (`dart:collection`) | O(1) |

---

## 5. Spread, collection-if, collection-for

```dart
final base = [1, 2];
final more = [0, ...base, 3];                 // [0, 1, 2, 3]
final maybe = [...?possiblyNullList];         // null-aware spread
final merged = {...mapA, ...mapB};            // later wins
```

---

## 6. Immutability

Mutable collections cause surprising bugs when shared.
Prefer creating **new** collections instead of changing old ones (especially in Flutter state).

```dart
final next = [...items, newItem];   // instead of items.add(newItem)
```

---

## Common mistakes

- Reading `list[i]` without checking length → `RangeError`.
- Expecting `list2 = list1` to copy.
- Using `List.contains` inside a loop (O(n²)). Use a `Set`.
- Writing `{}` and expecting a Set.
- Modifying a `const` list → runtime error.

---

## Code challenge

1. `bool hasDuplicates(List<int> xs)` — fastest way you can.
2. `List<int> intersect(List<int> a, List<int> b)` — values in both, without duplicates.
3. `Map<String, List<String>> groupByFirstLetter(List<String> words)` — `['apple','avocado','banana']` → `{a: [apple, avocado], b: [banana]}`.

<details>
<summary>Solution</summary>

```dart
bool hasDuplicates(List<int> xs) {
  final seen = <int>{};
  for (final x in xs) {
    if (!seen.add(x)) return true; // add returns false if already present
  }
  return false;
}

List<int> intersect(List<int> a, List<int> b) =>
    a.toSet().intersection(b.toSet()).toList();

Map<String, List<String>> groupByFirstLetter(List<String> words) {
  final out = <String, List<String>>{};
  for (final w in words) {
    out.putIfAbsent(w[0], () => []).add(w);
  }
  return out;
}
```

`hasDuplicates`: `O(n)` time. The naive double loop is `O(n²)`.
</details>

---

## Interview questions

1. List vs Set vs Map: when do you pick each?
2. What is the average time complexity of `Map` lookup and why?
3. What is the difference between `final list = []` and `const list = []`?
4. What is a shallow copy? Show how it can bite you.
5. How can you remove duplicates from a list while keeping the order?
6. Why is `list.contains(x)` inside a loop a performance smell?

<details>
<summary>Hints</summary>

1. Order/index → List; uniqueness/membership → Set; association → Map.
2. O(1) average via hashing; O(n) worst case on heavy collisions.
3. `final` — variable can't be reassigned but list contents can change. `const` — list itself is immutable.
4. Copying the outer list only; inner objects/lists are shared, so changing one changes both.
5. `LinkedHashSet` (default `toSet()` keeps insertion order): `list.toSet().toList()`.
6. `contains` is O(n); in a loop that gives O(n²). Convert to a Set first → O(n).
</details>
