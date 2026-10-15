# 02 — Big-O: Measuring Cost

## Goal

Compare algorithms **without** running them. Predict how code behaves when data grows.

---

## 1. The idea

Big-O describes how **work grows** as input size `n` grows. We ignore constants and small terms.

- `3n + 10` → `O(n)`
- `n² + 5n` → `O(n²)`
- `500` → `O(1)`

We care about the **shape** of the growth, not the exact number of steps.

---

## 2. The common classes (fast → slow)

| Big-O | Name | Example | n = 1,000,000 |
|-------|------|---------|---------------|
| `O(1)` | Constant | list index, map lookup | 1 |
| `O(log n)` | Logarithmic | binary search | ~20 |
| `O(n)` | Linear | loop once | 1,000,000 |
| `O(n log n)` | Linearithmic | good sorting | ~20,000,000 |
| `O(n²)` | Quadratic | nested loops | 1,000,000,000,000 (too slow) |
| `O(2ⁿ)` | Exponential | all subsets | impossible |
| `O(n!)` | Factorial | all permutations | impossible |

**Rule of thumb:** a computer does ~10⁸–10⁹ simple operations per second.

| n up to | Acceptable complexity |
|---------|----------------------|
| 10–12 | `O(n!)` |
| 20–25 | `O(2ⁿ)` |
| 500 | `O(n³)` |
| 5,000–10,000 | `O(n²)` |
| 10⁶ | `O(n log n)` |
| 10⁸ | `O(n)` |
| anything | `O(log n)`, `O(1)` |

Use this to **guess the intended solution** from the constraints.

---

## 3. How to calculate it

### Single loop → `O(n)`

```dart
int sum(List<int> xs) {
  var s = 0;
  for (final x in xs) s += x;   // n steps
  return s;
}
```

### Nested loops → multiply

```dart
for (var i = 0; i < n; i++) {
  for (var j = 0; j < n; j++) {   // n * n = O(n²)
    ...
  }
}
```

### Sequential blocks → add, keep the biggest

```dart
for (...) { }          // O(n)
for (...) { for (...) { } }   // O(n²)
// total O(n + n²) = O(n²)
```

### Halving → `O(log n)`

```dart
while (n > 1) {
  n = n ~/ 2;    // n, n/2, n/4, ... 1  -> log₂(n) steps
}
```

### Different inputs → different letters

```dart
void f(List<int> a, List<int> b) {
  for (final x in a) {}   // O(a)
  for (final y in b) {}   // O(b)
}
// O(a + b), NOT O(n)
```

Nested loops over two lists → `O(a * b)`.

### Hidden costs

| Operation | Cost |
|-----------|------|
| `list.add` | `O(1)` amortized |
| `list.insert(0, x)`, `removeAt(0)` | `O(n)` (shifts items) |
| `list.contains`, `indexOf` | `O(n)` |
| `set.contains`, `map[key]` | `O(1)` avg |
| `list.sort()` | `O(n log n)` |
| `string + string` | `O(n)` (copy) |
| `list.sublist` | `O(k)` (copy) |
| `[...a, ...b]` | `O(a + b)` |

A loop with `list.contains` inside = `O(n²)` even though it *looks* like one loop.

---

## 4. Space complexity

Extra memory your algorithm uses (not counting the input).

```dart
// O(1) space
int sum(List<int> xs) { var s = 0; for (final x in xs) s += x; return s; }

// O(n) space
List<int> doubled(List<int> xs) => xs.map((x) => x * 2).toList();
```

**Recursion uses stack memory:** depth of recursion = space. A recursive function that goes `n` deep uses `O(n)` space. Too deep → stack overflow.

---

## 5. Best, average, worst case

Quick sort: average `O(n log n)`, worst `O(n²)`. Hash map: average `O(1)`, worst `O(n)`.
By default "Big-O" in interviews means **worst case**. Mention others when relevant.

### Amortized cost

`list.add` is usually `O(1)`, but sometimes the list must grow (copy everything, `O(n)`).
Because growth doubles capacity, the **average** over many adds is `O(1)`. This is *amortized* `O(1)`.

---

## 6. Trading time for space

| Technique | Costs more | Saves |
|-----------|-----------|-------|
| Hash map for lookups | memory | time (`O(n²)` → `O(n)`) |
| Memoization (cache) | memory | time (exponential → polynomial) |
| Sorting first | `O(n log n)` | enables `O(n)` / `O(log n)` steps |
| Precompute prefix sums | `O(n)` memory | `O(1)` range queries |

---

## 7. Practical advice

- Big-O isn't everything. For `n = 20`, a "slow" algorithm may be faster due to simplicity.
- Constants matter in real apps: a UI list scroll is about **16 ms per frame**.
- **Measure** with `Stopwatch` and DevTools before optimizing.
- Readability first, unless profiling proves a problem.

```dart
final sw = Stopwatch()..start();
doWork();
print('took ${sw.elapsedMicroseconds} µs');
```

---

## Code challenge

Find the time and space complexity of each. Then check.

```dart
// A
bool f1(List<int> xs) => xs.contains(5);

// B
int f2(List<int> xs) {
  var c = 0;
  for (var i = 0; i < xs.length; i++) {
    for (var j = i; j < xs.length; j++) {
      c++;
    }
  }
  return c;
}

// C
int f3(int n) {
  var c = 0;
  for (var i = 1; i < n; i *= 2) c++;
  return c;
}

// D
int f4(int n) => n <= 1 ? 1 : f4(n - 1) + f4(n - 1);

// E
List<int> f5(List<int> xs) {
  final out = <int>[];
  for (final x in xs) {
    if (!out.contains(x)) out.add(x);
  }
  return out;
}
```

<details>
<summary>Solution</summary>

- **A:** `O(n)` time, `O(1)` space.
- **B:** `n + (n-1) + ... + 1 = n(n+1)/2` → `O(n²)` time, `O(1)` space.
- **C:** `i` doubles → `O(log n)` time, `O(1)` space.
- **D:** each call makes 2 calls, depth `n` → `O(2ⁿ)` time, `O(n)` space (stack depth).
- **E:** loop `O(n)` × `contains` `O(n)` → `O(n²)` time, `O(n)` space. Fix with a `Set` → `O(n)`.
</details>

---

## Interview questions

1. What does `O(n log n)` mean? Give an algorithm.
2. Why do we drop constants and lower-order terms?
3. What is amortized complexity? Example?
4. What is the time complexity of `list.removeAt(0)` and why?
5. What is the space complexity of a recursive function that goes `n` levels deep?
6. You have 1,000,000 items. Is an `O(n²)` solution acceptable?
7. Trade-off: how can you make a slow algorithm faster by spending memory?

<details>
<summary>Hints</summary>

1. Work grows slightly faster than linear; merge sort / Dart's `sort`.
2. We study growth for large `n`; constants depend on hardware.
3. Average cost per operation over a sequence; `list.add` with capacity doubling.
4. `O(n)`; every element shifts one slot left.
5. `O(n)` stack space (the call stack).
6. No. 10¹² operations is far too slow. Aim for `O(n log n)` or better.
7. Cache results (memoization), hash maps, precomputed tables.
</details>
