# 04 — Recursion and Backtracking

## Goal

Solve problems by **breaking them into smaller copies of themselves**. Then explore all possibilities systematically.

---

## 1. Recursion in one sentence

A function that calls itself on a **smaller** problem, until the problem is so small the answer is obvious.

Two parts, always:

1. **Base case** — the stopping point (answer known).
2. **Recursive case** — reduce the problem and call yourself.

```dart
int factorial(int n) {
  if (n <= 1) return 1;            // base case
  return n * factorial(n - 1);     // smaller problem
}
```

### Trace it (do this on paper!)

```
factorial(4)
 = 4 * factorial(3)
 = 4 * (3 * factorial(2))
 = 4 * (3 * (2 * factorial(1)))
 = 4 * (3 * (2 * 1))
 = 24
```

Each call waits on the **call stack** until the deeper call returns.

---

## 2. The recursive thinking recipe

1. **Define** exactly what the function returns. ("`sum(xs, i)` returns the sum from index `i` to the end.")
2. **Trust** it works for smaller input (the "leap of faith").
3. Solve the current step **using** the smaller answer.
4. Add the base case.

Example — sum of a list:

```dart
int sum(List<int> xs, [int i = 0]) {
  if (i == xs.length) return 0;          // nothing left
  return xs[i] + sum(xs, i + 1);         // this item + sum of the rest
}
```

Example — reverse a string:

```dart
String reverse(String s) =>
    s.isEmpty ? '' : reverse(s.substring(1)) + s[0];
```

Example — power (fast, `O(log n)`):

```dart
double power(double x, int n) {
  if (n == 0) return 1;
  if (n < 0) return 1 / power(x, -n);
  final half = power(x, n ~/ 2);
  return n.isEven ? half * half : half * half * x;
}
```

Halving the problem each time gives logarithmic cost.

---

## 3. The dangers

### Stack overflow

Too deep → crash. Dart's stack is limited (roughly thousands to ~10,000+ frames).

```dart
int bad(int n) => bad(n + 1);   // no base case -> StackOverflowError
```

If depth could be huge (like 100,000), convert to a **loop** or use an explicit stack.

### Exponential blow-up

```dart
int fib(int n) => n < 2 ? n : fib(n - 1) + fib(n - 2);   // O(2ⁿ)!
```

`fib(40)` makes ~300 million calls. Why? It re-solves the same subproblems again and again:

```
            fib(5)
          /        \
      fib(4)       fib(3)
      /    \       /    \
  fib(3) fib(2) fib(2) fib(1)    <- fib(3), fib(2) computed twice+
```

**Fix 1 — memoization (cache):**

```dart
int fibMemo(int n, [Map<int, int>? memo]) {
  memo ??= {};
  if (n < 2) return n;
  return memo[n] ??= fibMemo(n - 1, memo) + fibMemo(n - 2, memo);
}
// O(n) time, O(n) space
```

**Fix 2 — iterate:**

```dart
int fibIter(int n) {
  var a = 0, b = 1;
  for (var i = 0; i < n; i++) {
    final next = a + b;
    a = b;
    b = next;
  }
  return a;
}
// O(n) time, O(1) space
```

This is the doorway to dynamic programming (lesson 08).

---

## 4. Recursion on structures

Many structures are naturally recursive: folders contain folders, trees contain trees, JSON contains JSON.

```dart
// Count all nested list items
int countAll(List<dynamic> items) {
  var total = 0;
  for (final item in items) {
    total += item is List ? countAll(item) : 1;
  }
  return total;
}

countAll([1, [2, 3, [4]], 5]); // 5
```

In Flutter, the widget tree and `build` itself are recursive.

---

## 5. Backtracking

**Idea:** build a solution **step by step**. At each step, **choose**, **explore**, and **un-choose** (undo) if it leads nowhere. It is a **depth-first search over the decision tree**.

Template:

```
void backtrack(state):
    if state is a complete solution:
        save it
        return
    for each choice available now:
        if choice is valid:
            make choice          (choose)
            backtrack(new state) (explore)
            undo choice          (un-choose)
```

### Subsets

`[1,2,3]` → `[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]`

```dart
List<List<int>> subsets(List<int> nums) {
  final result = <List<int>>[];
  final current = <int>[];

  void go(int start) {
    result.add([...current]);                 // every path is a valid subset
    for (var i = start; i < nums.length; i++) {
      current.add(nums[i]);                   // choose
      go(i + 1);                              // explore
      current.removeLast();                   // un-choose
    }
  }

  go(0);
  return result;
}
// Time O(n * 2ⁿ), Space O(n) recursion depth (excluding output)
```

### Permutations

```dart
List<List<int>> permutations(List<int> nums) {
  final result = <List<int>>[];
  final current = <int>[];
  final used = List.filled(nums.length, false);

  void go() {
    if (current.length == nums.length) {
      result.add([...current]);
      return;
    }
    for (var i = 0; i < nums.length; i++) {
      if (used[i]) continue;
      used[i] = true;
      current.add(nums[i]);
      go();
      current.removeLast();
      used[i] = false;
    }
  }

  go();
  return result;
}
// Time O(n * n!)
```

### N-Queens style: **prune** early

Pruning = stopping a branch as soon as it can't succeed. It turns "impossible" into "fast enough".

```dart
// Generate valid parentheses with n pairs
List<String> generateParens(int n) {
  final out = <String>[];

  void go(String s, int open, int close) {
    if (s.length == 2 * n) {
      out.add(s);
      return;
    }
    if (open < n) go('$s(', open + 1, close);
    if (close < open) go('$s)', open, close + 1);   // prune: never close more than open
  }

  go('', 0, 0);
  return out;
}

print(generateParens(3)); // ((())), (()()), (())(), ()(()), ()()()
```

### Combination sum (reuse allowed)

```dart
List<List<int>> combinationSum(List<int> candidates, int target) {
  final out = <List<int>>[];
  final cur = <int>[];

  void go(int start, int remaining) {
    if (remaining == 0) {
      out.add([...cur]);
      return;
    }
    for (var i = start; i < candidates.length; i++) {
      if (candidates[i] > remaining) continue;
      cur.add(candidates[i]);
      go(i, remaining - candidates[i]);   // i, not i+1: reuse allowed
      cur.removeLast();
    }
  }

  go(0, target);
  return out;
}
```

---

## 6. Recursion vs iteration

| | Recursion | Iteration |
|--|-----------|-----------|
| Clarity for trees/backtracking | Great | Awkward |
| Memory | Stack per call | Constant |
| Risk | Stack overflow | None |
| Speed | Slight call overhead | Faster |

Any recursion can become iteration using your **own stack** (`List` used with `add`/`removeLast`).

---

## Common mistakes

- Missing or wrong base case.
- Not making the problem smaller each call.
- Forgetting to copy `current` before saving it (`result.add(current)` saves a reference that later changes!).
- Forgetting to undo the choice.
- Recomputing overlapping subproblems without memoization.

---

## Code challenge

1. `bool isPalindrome(String s)` recursively.
2. `int sumDigits(int n)` recursively: `1234` → `10`.
3. `List<String> letterCombinations(String digits)` — phone keypad (`"23"` → `ad, ae, af, bd, be, bf, cd, ce, cf`).
4. `int countPaths(int rows, int cols)` — paths from top-left to bottom-right moving only right/down. First recursive, then add memoization.
5. Tower of Hanoi: print the moves for `n` disks.

<details>
<summary>Solution</summary>

```dart
bool isPal(String s) =>
    s.length < 2 || (s[0] == s[s.length - 1] && isPal(s.substring(1, s.length - 1)));

int sumDigits(int n) => n < 10 ? n : n % 10 + sumDigits(n ~/ 10);

List<String> letterCombinations(String digits) {
  if (digits.isEmpty) return [];
  const map = {
    '2': 'abc', '3': 'def', '4': 'ghi', '5': 'jkl',
    '6': 'mno', '7': 'pqrs', '8': 'tuv', '9': 'wxyz',
  };
  final out = <String>[];
  void go(int i, String path) {
    if (i == digits.length) {
      out.add(path);
      return;
    }
    for (final ch in map[digits[i]]!.split('')) {
      go(i + 1, path + ch);
    }
  }
  go(0, '');
  return out;
}

int countPaths(int r, int c, [Map<(int, int), int>? memo]) {
  memo ??= {};
  if (r == 1 || c == 1) return 1;
  return memo[(r, c)] ??= countPaths(r - 1, c, memo) + countPaths(r, c - 1, memo);
}

void hanoi(int n, String from, String to, String via) {
  if (n == 0) return;
  hanoi(n - 1, from, via, to);
  print('Move disk $n: $from -> $to');
  hanoi(n - 1, via, to, from);
}
// Hanoi takes 2ⁿ - 1 moves.
```
</details>

---

## Interview questions

1. What are the two essential parts of a recursive function?
2. Why is naive Fibonacci `O(2ⁿ)` and how do you fix it?
3. What is a stack overflow and how do you avoid it?
4. Explain backtracking with the choose / explore / un-choose template.
5. What is pruning and why does it matter?
6. How do you convert a recursive solution to an iterative one?
7. Time complexity of generating all subsets? All permutations?

<details>
<summary>Hints</summary>

1. Base case and a recursive step that moves toward it.
2. Repeated overlapping subproblems; memoize or iterate.
3. Too many nested calls exhaust the call stack; limit depth, loop, or use an explicit stack.
4. Add a choice, recurse, remove it, so the next choice starts from a clean state.
5. Abandon partial solutions that cannot succeed, cutting the search tree.
6. Use your own stack data structure to hold the pending work.
7. Subsets `O(n·2ⁿ)`, permutations `O(n·n!)`.
</details>
