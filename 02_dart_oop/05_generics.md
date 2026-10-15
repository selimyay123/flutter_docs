# 05 — Generics

## Goal

Write code that works with **many types** while staying type-safe.

---

## 1. The problem

```dart
int firstInt(List<int> xs) => xs.first;
String firstString(List<String> xs) => xs.first;
// ... same code again for every type!
```

---

## 2. Generic functions

```dart
T first<T>(List<T> xs) => xs.first;

void main() {
  final a = first([1, 2, 3]);     // T = int
  final b = first(['x', 'y']);    // T = String
  print(a + 1);                   // works: compiler knows a is int
}
```

`T` is a **type parameter**: a placeholder filled in at use time.

---

## 3. Generic classes

```dart
class Box<T> {
  Box(this.value);
  T value;
  Box<R> map<R>(R Function(T) f) => Box(f(value));
}

void main() {
  final b = Box(5);                 // Box<int>
  final s = b.map((n) => 'n=$n');   // Box<String>
}
```

Without generics you would use `Object`/`dynamic` and need unsafe casts.

### A generic Stack

```dart
class Stack<T> {
  final _items = <T>[];

  void push(T item) => _items.add(item);
  T pop() => _items.removeLast();
  T? get peek => _items.isEmpty ? null : _items.last;
  bool get isEmpty => _items.isEmpty;
  int get length => _items.length;
}
```

---

## 4. Bounded generics (`extends`)

Limit what `T` can be.

```dart
T maxOf<T extends Comparable<T>>(List<T> xs) {
  var best = xs.first;
  for (final x in xs) {
    if (x.compareTo(best) > 0) best = x;
  }
  return best;
}

print(maxOf([3, 9, 2]));        // 9
print(maxOf(['pear', 'zebra'])); // zebra
// maxOf([Object()]);           // ERROR: Object is not Comparable
```

`extends Comparable<T>` lets you call `compareTo` safely.

---

## 5. Generics are **reified** in Dart

Unlike Java, Dart keeps type arguments at runtime.

```dart
print(<int>[] is List<int>);     // true
print(<int>[] is List<String>);  // false
```

---

## 6. Variance — an important gotcha

Dart generics are **covariant**: `List<int>` is a subtype of `List<num>`.

```dart
List<num> nums = <int>[1, 2];
nums.add(1.5);   // compiles but throws a runtime error!
```

Because the real list only accepts `int`. Lesson: don't upcast mutable collections.
Use `Iterable<num>` (read-only view) when you only need to read.

---

## 7. Real-world generics in Flutter

- `Future<T>`, `Stream<T>`
- `List<Widget>`
- `ValueNotifier<T>`, `ValueListenable<T>`
- `State<T extends StatefulWidget>`
- `Navigator.push<T>` returns `Future<T?>`
- A `Result<T>` or `ApiResponse<T>` wrapper

```dart
class ApiResponse<T> {
  ApiResponse.success(this.data) : error = null;
  ApiResponse.failure(this.error) : data = null;
  final T? data;
  final String? error;
}
```

---

## 8. Type inference helps

```dart
final list = <String>[];      // explicit
final list2 = ['a', 'b'];     // inferred List<String>
final map = {'a': 1};         // inferred Map<String, int>
final empty = [];             // inferred List<dynamic>  <- usually a mistake!
```

Always specify the type for empty collections: `<String>[]`.

---

## Common mistakes

- Empty `[]`/`{}` without type → `dynamic`.
- Using `dynamic` where a type parameter would work.
- Over-engineering: not every class needs generics.

---

## Code challenge

1. Write `Pair<A, B>` with `swap()` returning `Pair<B, A>`.
2. Write a generic `Queue<T>` (`enqueue`, `dequeue`, `isEmpty`) using a `List`. What is the cost of `dequeue`? Can you make it `O(1)`?
3. Write `Map<K, List<T>> groupBy<T, K>(Iterable<T> items, K Function(T) key)`.

<details>
<summary>Solution</summary>

```dart
class Pair<A, B> {
  Pair(this.first, this.second);
  final A first;
  final B second;
  Pair<B, A> swap() => Pair(second, first);
}

// O(1) dequeue using a head index (amortized)
class MyQueue<T> {
  final _items = <T>[];
  var _head = 0;

  void enqueue(T x) => _items.add(x);

  T dequeue() {
    if (isEmpty) throw StateError('Queue is empty');
    final x = _items[_head++];
    if (_head > 32 && _head * 2 > _items.length) {
      _items.removeRange(0, _head);   // compact occasionally
      _head = 0;
    }
    return x;
  }

  bool get isEmpty => _head >= _items.length;
}

Map<K, List<T>> groupBy<T, K>(Iterable<T> items, K Function(T) key) {
  final out = <K, List<T>>{};
  for (final item in items) {
    out.putIfAbsent(key(item), () => []).add(item);
  }
  return out;
}
```

`list.removeAt(0)` is `O(n)` because everything shifts. The head-index trick avoids that.
(Dart's built-in `Queue` from `dart:collection` is simpler.)
</details>

---

## Interview questions

1. What are generics and why use them instead of `dynamic`?
2. What does `T extends Comparable<T>` mean?
3. Are Dart generics reified? What does that allow?
4. Explain the covariance problem with `List<num> = <int>[]`.
5. Why is `final x = [];` a code smell?

<details>
<summary>Hints</summary>

1. Reusable code with compile-time type safety, no casts.
2. Upper bound: `T` must support comparison with its own type.
3. Yes; `is List<int>` works at runtime.
4. It compiles, but adding a `double` fails at runtime because the real list is `List<int>`.
5. Becomes `List<dynamic>` and loses type safety.
</details>
