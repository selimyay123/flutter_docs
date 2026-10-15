# 03 — Array and String Patterns

## Goal

Master the 6 patterns that solve most array/string problems.

Patterns:
1. Hash map / set (frequency, seen)
2. Two pointers
3. Sliding window
4. Prefix sums
5. Sorting + scan
6. Fast & slow pointers (see linked lists later)

---

## Pattern 1 — Hash map / set

Use when: counting, finding duplicates, "have I seen x?", pairs.

**Problem:** Group anagrams. `["eat","tea","tan","ate","nat","bat"]` → `[["eat","tea","ate"],["tan","nat"],["bat"]]`

Idea: anagrams have the same letters sorted → use sorted word as the **key**.

```dart
List<List<String>> groupAnagrams(List<String> words) {
  final groups = <String, List<String>>{};
  for (final w in words) {
    final key = (w.split('')..sort()).join();
    groups.putIfAbsent(key, () => []).add(w);
  }
  return groups.values.toList();
}
// Time O(n * k log k) where k = word length, Space O(n*k)
```

---

## Pattern 2 — Two pointers

Use when: **sorted** data, pairs, palindromes, in-place edits.

Two indexes move toward each other (or the same direction).

**Problem:** In a sorted list, find two numbers summing to `target`.

```dart
List<int>? twoSumSorted(List<int> xs, int target) {
  var left = 0, right = xs.length - 1;
  while (left < right) {
    final sum = xs[left] + xs[right];
    if (sum == target) return [xs[left], xs[right]];
    if (sum < target) {
      left++;          // need bigger sum
    } else {
      right--;         // need smaller sum
    }
  }
  return null;
}
// Time O(n), Space O(1)
```

Why does it work? If the sum is too small, the **only** way to increase it is to move `left` up. We never miss the answer.

**Problem:** Remove duplicates from a sorted list in place; return new length.

```dart
int removeDuplicates(List<int> xs) {
  if (xs.isEmpty) return 0;
  var write = 1;                           // slow pointer = where to write
  for (var read = 1; read < xs.length; read++) {   // fast pointer = scanning
    if (xs[read] != xs[read - 1]) {
      xs[write++] = xs[read];
    }
  }
  return write;
}
```

**Problem:** Reverse a list in place.

```dart
void reverse<T>(List<T> xs) {
  for (var i = 0, j = xs.length - 1; i < j; i++, j--) {
    final tmp = xs[i];
    xs[i] = xs[j];
    xs[j] = tmp;
  }
}
```

---

## Pattern 3 — Sliding window

Use when: **contiguous** subarray/substring and a condition (max sum, longest unique, at most k...).

A window `[left, right]` grows on the right and shrinks on the left. Each element enters once and leaves once → `O(n)`.

### Fixed-size window

**Problem:** Max sum of any `k` consecutive numbers.

```dart
int maxSumK(List<int> xs, int k) {
  var window = 0;
  for (var i = 0; i < k; i++) window += xs[i];
  var best = window;
  for (var i = k; i < xs.length; i++) {
    window += xs[i] - xs[i - k];     // add new, remove old
    if (window > best) best = window;
  }
  return best;
}
```

### Variable-size window

**Problem:** Length of the longest substring with no repeated characters. `"abcabcbb"` → `3`.

```dart
int longestUnique(String s) {
  final lastSeen = <String, int>{};   // char -> last index
  var left = 0, best = 0;
  for (var right = 0; right < s.length; right++) {
    final ch = s[right];
    final prev = lastSeen[ch];
    if (prev != null && prev >= left) {
      left = prev + 1;                // jump left past the duplicate
    }
    lastSeen[ch] = right;
    final len = right - left + 1;
    if (len > best) best = len;
  }
  return best;
}
// Time O(n), Space O(min(n, alphabet))
```

**Template:**

```
left = 0
for right in 0..n-1:
    add item[right] to window
    while window is invalid:
        remove item[left] from window
        left++
    update the answer
```

---

## Pattern 4 — Prefix sums

Use when: many **range sum** queries.

`prefix[i]` = sum of the first `i` elements. Sum of range `[l, r]` = `prefix[r+1] - prefix[l]` in `O(1)`.

```dart
class RangeSum {
  RangeSum(List<int> xs) : _prefix = List.filled(xs.length + 1, 0) {
    for (var i = 0; i < xs.length; i++) {
      _prefix[i + 1] = _prefix[i] + xs[i];
    }
  }
  final List<int> _prefix;

  int sum(int l, int r) => _prefix[r + 1] - _prefix[l];   // inclusive
}
```

Build once `O(n)`, query many times `O(1)`. Trade memory for speed.

**Problem:** Count subarrays that sum to `k` (prefix sums + hash map).

```dart
int subarraySum(List<int> xs, int k) {
  final seen = <int, int>{0: 1};   // prefix sum -> how many times
  var sum = 0, count = 0;
  for (final x in xs) {
    sum += x;
    count += seen[sum - k] ?? 0;   // earlier prefix that makes a subarray = k
    seen[sum] = (seen[sum] ?? 0) + 1;
  }
  return count;
}
```

---

## Pattern 5 — Sort, then scan

Sorting costs `O(n log n)` but often makes the rest simple.

**Problem:** Merge overlapping intervals. `[[1,3],[2,6],[8,10]]` → `[[1,6],[8,10]]`.

```dart
List<List<int>> mergeIntervals(List<List<int>> intervals) {
  if (intervals.isEmpty) return [];
  final sorted = [...intervals]..sort((a, b) => a[0].compareTo(b[0]));
  final merged = <List<int>>[sorted.first];
  for (final cur in sorted.skip(1)) {
    final last = merged.last;
    if (cur[0] <= last[1]) {
      if (cur[1] > last[1]) last[1] = cur[1];   // extend
    } else {
      merged.add([...cur]);
    }
  }
  return merged;
}
```

(We copy `cur` to avoid mutating the caller's lists. Think about side effects!)

---

## Pattern 6 — In-place tricks and bit of math

- **Kadane's algorithm:** max subarray sum in one pass.

```dart
int maxSubarray(List<int> xs) {
  var current = xs.first, best = xs.first;
  for (final x in xs.skip(1)) {
    current = current + x > x ? current + x : x;   // extend or restart
    if (current > best) best = current;
  }
  return best;
}
```

- **Rotate list by k:** reverse whole, reverse first k, reverse the rest.
- **Missing number** in `0..n`: `n(n+1)/2 - sum(list)`.
- **XOR trick:** `a ^ a = 0`; find the single non-duplicate: XOR everything.

```dart
int singleNumber(List<int> xs) => xs.fold(0, (a, b) => a ^ b);
```

---

## Choosing the pattern

```
Is the answer about a contiguous range? ──yes──► sliding window / prefix sum
        │no
Is the data sorted (or can be)? ──yes──► two pointers / binary search
        │no
Need counts / membership / pairs? ──yes──► hash map / set
```

---

## Common mistakes

- Off-by-one on window edges.
- Forgetting to handle empty input.
- Mutating the input list unintentionally.
- Using sliding window with **negative** numbers when the "shrink when too big" logic assumes positives.

---

## Code challenge

Solve each using the best pattern. State the Big-O.

1. `bool validPalindromeOne(String s)` — palindrome if you may delete at most one char. `"abca"` → `true`.
2. `int longestOnes(List<int> bits, int k)` — longest run of 1s if you may flip at most `k` zeros. `[1,1,0,0,1,1,1,0], 2` → `7`.
3. `List<int> productExceptSelf(List<int> xs)` — result[i] = product of all others, **no division**, `O(n)`. `[1,2,3,4]` → `[24,12,8,6]`.
4. `int minSubArrayLen(int target, List<int> positives)` — shortest subarray with sum ≥ target.

<details>
<summary>Solution</summary>

```dart
bool validPalindromeOne(String s) {
  bool check(int i, int j) {
    while (i < j) {
      if (s[i] != s[j]) return false;
      i++;
      j--;
    }
    return true;
  }

  var i = 0, j = s.length - 1;
  while (i < j) {
    if (s[i] != s[j]) return check(i + 1, j) || check(i, j - 1);
    i++;
    j--;
  }
  return true;
}

int longestOnes(List<int> bits, int k) {
  var left = 0, zeros = 0, best = 0;
  for (var right = 0; right < bits.length; right++) {
    if (bits[right] == 0) zeros++;
    while (zeros > k) {
      if (bits[left] == 0) zeros--;
      left++;
    }
    best = best > right - left + 1 ? best : right - left + 1;
  }
  return best;
}
// [1,1,0,0,1,1,1,0], k=2 -> best window is indexes 0..6 (two zeros) = 7

List<int> productExceptSelf(List<int> xs) {
  final n = xs.length;
  final result = List.filled(n, 1);
  var left = 1;
  for (var i = 0; i < n; i++) {
    result[i] = left;
    left *= xs[i];
  }
  var right = 1;
  for (var i = n - 1; i >= 0; i--) {
    result[i] *= right;
    right *= xs[i];
  }
  return result;
}

int minSubArrayLen(int target, List<int> xs) {
  var left = 0, sum = 0, best = 1 << 62;
  for (var right = 0; right < xs.length; right++) {
    sum += xs[right];
    while (sum >= target) {
      best = best < right - left + 1 ? best : right - left + 1;
      sum -= xs[left++];
    }
  }
  return best == 1 << 62 ? 0 : best;
}
```

Complexities: all `O(n)` time. Palindrome `O(1)` space, others `O(1)` except `productExceptSelf` which is `O(1)` extra beyond the output.
</details>

---

## Interview questions

1. When do you use two pointers? Why is it `O(n)` and not `O(n²)`?
2. Fixed vs variable sliding window: describe the template.
3. What are prefix sums and what trade-off do they make?
4. Why do we sort intervals before merging?
5. Why might sliding window fail with negative numbers?
6. Solve "two sum" three ways. Compare cost.

<details>
<summary>Hints</summary>

1. Sorted/symmetric data; each pointer only moves one direction, so total moves ≤ n.
2. Expand right; while invalid, shrink left; record answer. Fixed: add new, remove old.
3. Precompute cumulative sums to answer range queries in `O(1)` using `O(n)` memory.
4. After sorting by start, overlaps are always adjacent.
5. Shrinking doesn't necessarily reduce the sum, so the monotonic logic breaks. Use prefix sums + hash map.
6. Brute `O(n²)`; hash map `O(n)` time/`O(n)` space; sort + two pointers `O(n log n)`/`O(1)` (but loses original indices).
</details>
