# 01 — Dart Senior Interview Question Bank

## Goal

Master the foundational and subtle aspects of Dart that interviewers use to distinguish junior developers from senior engineers.

---

## Core Focus Areas

In senior interviews, Dart questions test whether you understand **how the language works under the hood** (memory, typing, concurrency, runtime semantics), not just surface syntax.

---

## 1. Type System & Sound Null Safety

### Q: What is sound null safety and how does it optimize compilation?
**Answer:**
Sound null safety means if a variable is declared as non-nullable (`T`), the compiler **guarantees** it can never be `null` at runtime. 
- In unsound languages, every object dereference needs defensive null checks in machine code.
- In Dart, the AOT compiler skips null checks for non-nullable types, generating smaller binaries and faster native machine code.

### Q: Compare `dynamic`, `Object`, `Object?`, and `var`.
| Type | Static Type Checks | Can Be Null? | Available Members |
| :--- | :--- | :--- | :--- |
| `var x = 1;` | Static (inferred `int`) | No | Methods on inferred type |
| `Object` | Static compile-time | No | Only methods on `Object` (`toString`, `==`, etc.) |
| `Object?` | Static compile-time | Yes | Need null-check before calling `Object` methods |
| `dynamic` | Disabled (deferred to runtime)| Yes | Any method (throws `NoSuchMethodError` if missing)|

---

## 2. Concurrency, Event Loop & Isolates

### Q: Why does an infinite loop freeze Flutter even with `async`?
```dart
Future<void> runForever() async {
  while (true) {} // Freezes the app
}
```
**Answer:**
Dart is single-threaded per isolate. An `async` function does not run on a separate thread; it simply yields when executing an `await` on an uncompleted `Future`. Synchronous CPU work (like a tight loop) starves the event loop, preventing microtasks, events, and UI rendering from ever running.

### Q: How do Isolates communicate? Can they share memory?
**Answer:**
Isolates have completely separate heaps and memory spaces. Communication occurs exclusively through message passing using `SendPort` and `ReceivePort`. Since memory is isolated:
- No mutexes, locks, or race conditions.
- Objects passed between isolates are deep-copied, or ownership is transferred in specialized buffer cases.

---

## 3. Object Orientation & Dart 3 Modifiers

### Q: Explain class modifiers: `sealed`, `base`, `interface`, and `final`.
- **`sealed`**: Cannot be extended or implemented outside the defining library. Enables compile-time exhaustive pattern matching in `switch`.
- **`base`**: Allows inheritance (`extends`), but forbids implementation (`implements`) outside the library to guarantee private contract integrity.
- **`interface`**: Allows `implements`, but forbids `extends` outside the library.
- **`final`**: Completely closes the class to both `extends` and `implements` outside the library.

### Q: What is the contract between `operator ==` and `hashCode`?
**Answer:**
If two objects are equal according to `==`, their `hashCode` values **must** be identical. 
If violated, hash-based collections (`HashSet`, `HashMap`) will place identical objects into different buckets, causing lookup failures and memory leaks.

---

## 4. Collections & Performance

### Q: Why is `List.contains()` in a loop dangerous, and what is the fix?
```dart
// Anti-pattern: O(N * M)
for (var id in targetIds) {
  if (items.map((e) => e.id).contains(id)) { ... }
}
```
**Answer:**
`List.contains()` is an `O(N)` linear search. Doing this inside a loop creates quadratic time complexity `O(N * M)`.
**Fix:** Convert the lookup collection into a `Set` (`O(1)` average lookup time), reducing total execution to `O(N + M)`.

---

## Code Challenge

Write a memoization wrapper function `memoize1` that caches the result of any single-argument pure function `R Function(T)`. Ensure:
1. Lookup is `O(1)`.
2. Handles `null` inputs and outputs properly.
3. Clean, generic Dart implementation.

<details>
<summary>Solution</summary>

```dart
R Function(T) memoize1<T, R>(R Function(T) fn) {
  final cache = <T, R>{};
  return (T arg) {
    if (cache.containsKey(arg)) {
      return cache[arg] as R;
    }
    final result = fn(arg);
    cache[arg] = result;
    return result;
  };
}

void main() {
  int expensive(int n) {
    print('Computing $n');
    return n * 2;
  }

  final fast = memoize1(expensive);
  print(fast(5)); // Prints "Computing 5" then "10"
  print(fast(5)); // Prints only "10"
}
```
</details>

---

## Rapid-Fire Interview Questions

1. What is the difference between `const []` and `final list = []`?
2. What are extension types (Dart 3.3+) and what is their runtime overhead?
3. Why does integer division in Dart use `~/` instead of `/`?
4. When does `identical(a, b)` return true?
5. What is the difference between a broadcast stream and a single-subscription stream?

<details>
<summary>Hints & Answers</summary>

1. `const []` is an immutable compile-time canonical instance. `final list = []` creates a mutable list whose reference cannot be reassigned.
2. Extension types provide zero-cost static wrapping at compile time; runtime overhead is zero.
3. `/` returns a `double` to represent true fractional division; `~/` explicitly truncates to an `int`.
4. When `a` and `b` reference the exact same memory address (identical instance).
5. Single-subscription streams buffer events for one listener; broadcast streams allow multiple simultaneous listeners without historical replay.
</details>
