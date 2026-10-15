# 05 — Sorting and Searching

## Goal

Understand the classic algorithms, not to memorize them, but to learn the **ideas** behind them
(divide and conquer, invariants, halving).

---

## 1. Searching

### Linear search — `O(n)`

```dart
int linearSearch(List<int> xs, int target) {
  for (var i = 0; i < xs.length; i++) {
    if (xs[i] == target) return i;
  }
  return -1;
}
```

Works on anything. Slow on big data.

### Binary search — `O(log n)` on **sorted** data

Look at the middle. Throw away half. Repeat.

```dart
int binarySearch(List<int> xs, int target) {
  var lo = 0, hi = xs.length - 1;
  while (lo <= hi) {
    final mid = lo + (hi - lo) ~/ 2;       // avoids overflow in other languages
    if (xs[mid] == target) return mid;
    if (xs[mid] < target) {
      lo = mid + 1;
    } else {
      hi = mid - 1;
    }
  }
  return -1;
}
```

1,000,000 items → ~20 steps. 1,000,000,000 items → ~30 steps.

**Invariant:** "If the target exists, it is in `[lo, hi]`." Each step keeps this true while shrinking the range. Thinking in invariants prevents off-by-one bugs.

### Lower bound (first position where `xs[i] >= target`)

The most useful variant. Many problems reduce to this.

```dart
int lowerBound(List<int> xs, int target) {
  var lo = 0, hi = xs.length;              // hi is exclusive
  while (lo < hi) {
    final mid = lo + (hi - lo) ~/ 2;
    if (xs[mid] < target) {
      lo = mid + 1;
    } else {
      hi = mid;
    }
  }
  return lo;                               // insertion point
}
```

### Binary search on the **answer**

Not only arrays! If you can test "is `x` big enough?" and the result flips from false to true exactly once, binary search the answer.

**Problem:** Integer square root of `n` (floor).

```dart
int isqrt(int n) {
  var lo = 0, hi = n;
  while (lo < hi) {
    final mid = lo + (hi - lo + 1) ~/ 2;   // upper mid to avoid infinite loop
    if (mid * mid <= n) {
      lo = mid;
    } else {
      hi = mid - 1;
    }
  }
  return lo;
}
```

**Problem:** Minimum eating speed. Koko has piles `[3,6,7,11]` and `h = 8` hours. Find the minimum speed `k` (bananas/hour) so she finishes in time.

```dart
int minSpeed(List<int> piles, int h) {
  bool canFinish(int k) {
    var hours = 0;
    for (final p in piles) {
      hours += (p + k - 1) ~/ k;           // ceil(p / k)
    }
    return hours <= h;
  }

  var lo = 1, hi = piles.reduce((a, b) => a > b ? a : b);
  while (lo < hi) {
    final mid = lo + (hi - lo) ~/ 2;
    if (canFinish(mid)) {
      hi = mid;
    } else {
      lo = mid + 1;
    }
  }
  return lo;
}
// O(n log max)
```

**Recognize it:** "minimum X such that ..." or "maximum X such that ..." with a monotonic yes/no check.

---

## 2. Simple sorts (know them, don't use them)

### Bubble sort — `O(n²)`

Repeatedly swap neighbors that are out of order.

```dart
void bubbleSort(List<int> xs) {
  for (var i = 0; i < xs.length - 1; i++) {
    var swapped = false;
    for (var j = 0; j < xs.length - 1 - i; j++) {
      if (xs[j] > xs[j + 1]) {
        final t = xs[j]; xs[j] = xs[j + 1]; xs[j + 1] = t;
        swapped = true;
      }
    }
    if (!swapped) break;                   // already sorted -> O(n) best case
  }
}
```

### Insertion sort — `O(n²)`, great for tiny/nearly-sorted data

Like sorting playing cards in your hand.

```dart
void insertionSort(List<int> xs) {
  for (var i = 1; i < xs.length; i++) {
    final key = xs[i];
    var j = i - 1;
    while (j >= 0 && xs[j] > key) {
      xs[j + 1] = xs[j];
      j--;
    }
    xs[j + 1] = key;
  }
}
```

---

## 3. Fast sorts — `O(n log n)`

### Merge sort — divide and conquer, **stable**, guaranteed `O(n log n)`

1. Split in half.
2. Sort each half (recursion).
3. **Merge** two sorted halves.

```dart
List<int> mergeSort(List<int> xs) {
  if (xs.length <= 1) return xs;
  final mid = xs.length ~/ 2;
  final left = mergeSort(xs.sublist(0, mid));
  final right = mergeSort(xs.sublist(mid));
  return _merge(left, right);
}

List<int> _merge(List<int> a, List<int> b) {
  final out = <int>[];
  var i = 0, j = 0;
  while (i < a.length && j < b.length) {
    out.add(a[i] <= b[j] ? a[i++] : b[j++]);   // <= keeps it stable
  }
  while (i < a.length) out.add(a[i++]);
  while (j < b.length) out.add(b[j++]);
  return out;
}
```

Time `O(n log n)`; space `O(n)`.

### Quick sort — divide and conquer, in-place, average `O(n log n)`

1. Pick a **pivot**.
2. Partition: smaller items left, larger right.
3. Recurse on both sides.

```dart
void quickSort(List<int> xs, [int lo = 0, int? hiOpt]) {
  final hi = hiOpt ?? xs.length - 1;
  if (lo >= hi) return;
  final p = _partition(xs, lo, hi);
  quickSort(xs, lo, p - 1);
  quickSort(xs, p + 1, hi);
}

int _partition(List<int> xs, int lo, int hi) {
  final pivot = xs[hi];
  var i = lo;
  for (var j = lo; j < hi; j++) {
    if (xs[j] < pivot) {
      final t = xs[i]; xs[i] = xs[j]; xs[j] = t;
      i++;
    }
  }
  final t = xs[i]; xs[i] = xs[hi]; xs[hi] = t;
  return i;
}
```

Worst case `O(n²)` (bad pivots on sorted data). Real implementations pick random/median pivots.

### Comparison

| Algorithm | Best | Average | Worst | Space | Stable |
|-----------|------|---------|-------|-------|--------|
| Bubble | n | n² | n² | 1 | Yes |
| Insertion | n | n² | n² | 1 | Yes |
| Merge | n log n | n log n | n log n | n | Yes |
| Quick | n log n | n log n | n² | log n | No |
| Counting | n + k | n + k | n + k | k | Yes |

**Stable** = equal elements keep their original order. Important when sorting by one field after another.

**Lower bound:** comparison sorts can't beat `O(n log n)`. Counting sort beats it by not comparing (works for small integer ranges).

### Dart's `list.sort()`

Uses a hybrid (a mix of insertion sort and a pivot-based approach; **not guaranteed stable**). Use it. Don't hand-write sorts in production.

---

## 4. Quick select — k-th smallest in average `O(n)`

Same partition as quick sort, but recurse into only the side that holds the k-th item.

```dart
int kthSmallest(List<int> xs, int k) {      // k is 0-based
  final a = [...xs];
  var lo = 0, hi = a.length - 1;
  while (lo <= hi) {
    final p = _partition(a, lo, hi);
    if (p == k) return a[p];
    if (p < k) {
      lo = p + 1;
    } else {
      hi = p - 1;
    }
  }
  throw StateError('k out of range');
}
```

---

## 5. Counting sort — no comparisons

```dart
List<int> countingSort(List<int> xs, int maxValue) {
  final counts = List.filled(maxValue + 1, 0);
  for (final x in xs) counts[x]++;
  final out = <int>[];
  for (var v = 0; v <= maxValue; v++) {
    for (var c = 0; c < counts[v]; c++) out.add(v);
  }
  return out;
}
```

---

## Common mistakes

- Binary search on **unsorted** data.
- `mid = (lo + hi) / 2` (use `~/`, and `lo + (hi - lo) ~/ 2` as habit).
- Infinite loops from wrong `lo`/`hi` updates. Test with 1 and 2 elements.
- Forgetting sort stability when it matters.
- Using `hi = xs.length` with `lo <= hi` (mixing exclusive and inclusive styles).

---

## Code challenge

1. Find the **first and last** position of a value in a sorted list with duplicates (`O(log n)`).
2. Search in a **rotated** sorted array: `[4,5,6,7,0,1,2]`, target `0` → `4`.
3. Find the **peak** element: an index where `xs[i] > neighbors` (`O(log n)`).
4. Sort a list of people by age, then by name, using `sort` with a comparator.

<details>
<summary>Solution</summary>

```dart
(int, int) firstLast(List<int> xs, int t) {
  final first = lowerBound(xs, t);
  if (first == xs.length || xs[first] != t) return (-1, -1);
  final last = lowerBound(xs, t + 1) - 1;
  return (first, last);
}

int searchRotated(List<int> xs, int t) {
  var lo = 0, hi = xs.length - 1;
  while (lo <= hi) {
    final mid = lo + (hi - lo) ~/ 2;
    if (xs[mid] == t) return mid;
    if (xs[lo] <= xs[mid]) {                    // left half sorted
      if (xs[lo] <= t && t < xs[mid]) {
        hi = mid - 1;
      } else {
        lo = mid + 1;
      }
    } else {                                    // right half sorted
      if (xs[mid] < t && t <= xs[hi]) {
        lo = mid + 1;
      } else {
        hi = mid - 1;
      }
    }
  }
  return -1;
}

int findPeak(List<int> xs) {
  var lo = 0, hi = xs.length - 1;
  while (lo < hi) {
    final mid = lo + (hi - lo) ~/ 2;
    if (xs[mid] < xs[mid + 1]) {
      lo = mid + 1;                             // a peak lies to the right
    } else {
      hi = mid;
    }
  }
  return lo;
}

// people.sort((a, b) {
//   final c = a.age.compareTo(b.age);
//   return c != 0 ? c : a.name.compareTo(b.name);
// });
```
</details>

---

## Interview questions

1. Why does binary search need sorted data? What is its complexity?
2. What is an invariant and how does it help you write binary search?
3. Merge sort vs quick sort: speed, memory, stability, worst case?
4. What does "stable sort" mean and when does it matter?
5. Why can't comparison sorting beat `O(n log n)`?
6. What is "binary search on the answer"? Give an example.
7. How does quick select find the k-th element in `O(n)` average?

<details>
<summary>Hints</summary>

1. Halving relies on order; `O(log n)`.
2. A statement true at every loop step ("target is in [lo, hi]"), used to prove correctness and boundaries.
3. Merge: guaranteed `n log n`, stable, `O(n)` memory. Quick: in-place, worst `n²`, unstable.
4. Equal keys keep original order; matters for multi-key sorting (sort by name, then stable sort by age).
5. A decision tree with n! leaves needs height ≥ log₂(n!) ≈ n log n.
6. Monotonic feasibility check, e.g. minimum eating speed or capacity.
7. Partition and recurse into one side only, so work halves on average: n + n/2 + n/4 ... = 2n.
</details>
