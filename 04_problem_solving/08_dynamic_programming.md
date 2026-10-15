# 08 — Dynamic Programming (DP)

## Goal

Turn exponential recursion into fast solutions by **remembering answers**. Learn a reliable way to *find* the DP, not memorize it.

---

## 1. What is DP?

DP solves a problem by combining answers to **overlapping subproblems**, solving each only once.

Two conditions:

1. **Optimal substructure:** the best answer is built from best answers of smaller pieces.
2. **Overlapping subproblems:** the same small problem appears many times.

DP = **recursion + memory**.

---

## 2. The DP recipe (use every time)

1. **Define the state:** "`f(i)` means ___". (This is the hardest and most important step.)
2. **Find the transition:** how does `f(i)` depend on smaller states?
3. **Base cases.**
4. **Order of computation** (top-down with memo, or bottom-up table).
5. **Answer:** which state(s) hold it?
6. **Optimize space** if only the last few states are needed.

---

## 3. Three ways to solve the same problem

**Climbing stairs:** you can climb 1 or 2 steps. How many ways to reach step `n`?

State: `f(n)` = number of ways to reach step `n`.
Transition: last move was 1 step or 2 steps → `f(n) = f(n-1) + f(n-2)`.
Base: `f(0) = 1`, `f(1) = 1`.

### A) Plain recursion — `O(2ⁿ)` (too slow)

```dart
int ways(int n) => n <= 1 ? 1 : ways(n - 1) + ways(n - 2);
```

### B) Top-down with memoization — `O(n)`

```dart
int waysMemo(int n, [Map<int, int>? memo]) {
  memo ??= {};
  if (n <= 1) return 1;
  return memo[n] ??= waysMemo(n - 1, memo) + waysMemo(n - 2, memo);
}
```

### C) Bottom-up table — `O(n)`

```dart
int waysTable(int n) {
  final dp = List.filled(n + 2, 0);
  dp[0] = 1;
  dp[1] = 1;
  for (var i = 2; i <= n; i++) {
    dp[i] = dp[i - 1] + dp[i - 2];
  }
  return dp[n];
}
```

### D) Space-optimized — `O(1)` memory

```dart
int waysFast(int n) {
  var a = 1, b = 1;
  for (var i = 2; i <= n; i++) {
    final c = a + b;
    a = b;
    b = c;
  }
  return b;
}
```

Same idea, four levels of polish. In an interview, start from A, evolve to B/C, mention D.

---

## 4. 1D DP examples

### House robber

Can't rob two adjacent houses. Maximize money. `[2,7,9,3,1]` → `12` (2 + 9 + 1).

State: `f(i)` = best money using houses `0..i`.
Transition: skip house `i` → `f(i-1)`; rob it → `f(i-2) + nums[i]`. Take the max.

```dart
int rob(List<int> nums) {
  var prev2 = 0, prev1 = 0;
  for (final x in nums) {
    final cur = prev1 > prev2 + x ? prev1 : prev2 + x;
    prev2 = prev1;
    prev1 = cur;
  }
  return prev1;
}
```

### Coin change (minimum coins)

Coins `[1, 5, 10]`, amount `12` → `3` (10 + 1 + 1).

State: `f(a)` = fewest coins to make amount `a`.
Transition: try each coin `c`: `f(a) = 1 + min(f(a - c))`.
Base: `f(0) = 0`.

```dart
int coinChange(List<int> coins, int amount) {
  const inf = 1 << 30;
  final dp = List.filled(amount + 1, inf);
  dp[0] = 0;
  for (var a = 1; a <= amount; a++) {
    for (final c in coins) {
      if (c <= a && dp[a - c] + 1 < dp[a]) {
        dp[a] = dp[a - c] + 1;
      }
    }
  }
  return dp[amount] >= inf ? -1 : dp[amount];
}
// O(amount * coins)
```

Why not **greedy** (always take the biggest coin)? With coins `[1, 3, 4]` and amount `6`: greedy gives `4+1+1 = 3` coins, the best is `3+3 = 2` coins. **Greedy fails; DP is correct.**

### Longest increasing subsequence (LIS)

`[10,9,2,5,3,7,101,18]` → `4` (`2,3,7,101`).

State: `f(i)` = length of LIS ending at index `i`.
Transition: `f(i) = 1 + max(f(j))` for all `j < i` where `nums[j] < nums[i]`.

```dart
int lis(List<int> nums) {
  if (nums.isEmpty) return 0;
  final dp = List.filled(nums.length, 1);
  var best = 1;
  for (var i = 1; i < nums.length; i++) {
    for (var j = 0; j < i; j++) {
      if (nums[j] < nums[i] && dp[j] + 1 > dp[i]) dp[i] = dp[j] + 1;
    }
    if (dp[i] > best) best = dp[i];
  }
  return best;
}
// O(n²). An O(n log n) version uses binary search on "tails".
```

---

## 5. 2D DP examples

### Unique paths in a grid

State: `f(r, c)` = ways to reach cell `(r, c)` moving right/down.
Transition: `f(r, c) = f(r-1, c) + f(r, c-1)`.

```dart
int uniquePaths(int rows, int cols) {
  final dp = List.generate(rows, (_) => List.filled(cols, 1));
  for (var r = 1; r < rows; r++) {
    for (var c = 1; c < cols; c++) {
      dp[r][c] = dp[r - 1][c] + dp[r][c - 1];
    }
  }
  return dp[rows - 1][cols - 1];
}
```

### Longest common subsequence (LCS)

`"abcde"` and `"ace"` → `3` (`ace`).

State: `f(i, j)` = LCS length of `a[0..i)` and `b[0..j)`.
Transition: if `a[i-1] == b[j-1]` → `1 + f(i-1, j-1)`; else `max(f(i-1, j), f(i, j-1))`.

```dart
int lcs(String a, String b) {
  final dp = List.generate(a.length + 1, (_) => List.filled(b.length + 1, 0));
  for (var i = 1; i <= a.length; i++) {
    for (var j = 1; j <= b.length; j++) {
      dp[i][j] = a[i - 1] == b[j - 1]
          ? dp[i - 1][j - 1] + 1
          : (dp[i - 1][j] > dp[i][j - 1] ? dp[i - 1][j] : dp[i][j - 1]);
    }
  }
  return dp[a.length][b.length];
}
// O(a * b)
```

### Edit distance (how spell checkers work)

Minimum insert/delete/replace operations to turn `a` into `b`.

```dart
int editDistance(String a, String b) {
  final dp = List.generate(a.length + 1, (_) => List.filled(b.length + 1, 0));
  for (var i = 0; i <= a.length; i++) dp[i][0] = i;     // delete all
  for (var j = 0; j <= b.length; j++) dp[0][j] = j;     // insert all
  for (var i = 1; i <= a.length; i++) {
    for (var j = 1; j <= b.length; j++) {
      if (a[i - 1] == b[j - 1]) {
        dp[i][j] = dp[i - 1][j - 1];
      } else {
        final replace = dp[i - 1][j - 1], delete = dp[i - 1][j], insert = dp[i][j - 1];
        dp[i][j] = 1 + [replace, delete, insert].reduce((x, y) => x < y ? x : y);
      }
    }
  }
  return dp[a.length][b.length];
}
```

### 0/1 Knapsack

Items with weight and value; capacity `W`. Each item once. Maximize value.

State: `f(i, w)` = best value using the first `i` items with capacity `w`.
Transition: skip → `f(i-1, w)`; take (if fits) → `value[i] + f(i-1, w - weight[i])`.

```dart
int knapsack(List<int> weights, List<int> values, int capacity) {
  final dp = List.filled(capacity + 1, 0);               // 1D optimized
  for (var i = 0; i < weights.length; i++) {
    for (var w = capacity; w >= weights[i]; w--) {       // backwards: each item once
      final take = values[i] + dp[w - weights[i]];
      if (take > dp[w]) dp[w] = take;
    }
  }
  return dp[capacity];
}
```

Looping `w` **backwards** prevents using the same item twice. Looping forwards gives *unbounded* knapsack.

---

## 6. DP vs Greedy vs Backtracking

| | Idea | Use when |
|--|------|---------|
| Greedy | Take the best local choice, never reconsider | Proven that local = global (intervals, Huffman) |
| DP | Try all choices, reuse subresults | Overlapping subproblems |
| Backtracking | Try all choices, no reuse | Need **all** solutions, no overlap |

Greedy example: **maximum non-overlapping intervals** → sort by end time, always pick the interval that ends first.

```dart
int maxNonOverlapping(List<List<int>> intervals) {
  final s = [...intervals]..sort((a, b) => a[1].compareTo(b[1]));
  var count = 0, end = -(1 << 62);
  for (final [start, finish] in s) {
    if (start >= end) {
      count++;
      end = finish;
    }
  }
  return count;
}
```

---

## 7. How to recognize a DP problem

Keywords: *"minimum / maximum / number of ways / is it possible"* + choices at each step + the same sub-situation reached by different paths.

Ask: "If I knew the answer for all smaller cases, could I build this one?" If yes → DP.

---

## 8. How to *discover* the state

- What changes as I make decisions? (index, remaining amount, remaining capacity, last choice)
- Those changing things are your state dimensions.
- Start with plain recursion; the parameters of the recursive function **are** the state. Memoize on them.

This trick works for almost every DP problem: **write the recursion first, then add a cache.**

---

## Common mistakes

- Defining the state vaguely.
- Wrong base cases / indexing off by one (use `n + 1` sized tables).
- Iterating in the wrong order (a state must be computed after the ones it needs).
- Using greedy when it isn't provably correct.
- Memoizing with mutable keys.

---

## Code challenge

1. **Min cost climbing stairs:** cost array; pay to step; start at 0 or 1; reach the top.
2. **Partition equal subset sum:** can `[1,5,11,5]` split into two equal-sum halves? (Knapsack variant.)
3. **Word break:** `s = "leetcode"`, dict `["leet","code"]` → `true`.
4. **Longest palindromic substring** (expand around center — it is `O(n²)`).
5. **Decode ways:** `"226"` → `3` (`2 2 6`, `22 6`, `2 26`).

<details>
<summary>Solution</summary>

```dart
int minCostClimbing(List<int> cost) {
  var a = 0, b = 0;                         // cost to reach step i-2 and i-1
  for (var i = 2; i <= cost.length; i++) {
    final c = (b + cost[i - 1]) < (a + cost[i - 2]) ? b + cost[i - 1] : a + cost[i - 2];
    a = b;
    b = c;
  }
  return b;
}

bool canPartition(List<int> nums) {
  final total = nums.fold(0, (a, b) => a + b);
  if (total.isOdd) return false;
  final target = total ~/ 2;
  final dp = List.filled(target + 1, false);
  dp[0] = true;
  for (final x in nums) {
    for (var s = target; s >= x; s--) {
      if (dp[s - x]) dp[s] = true;
    }
  }
  return dp[target];
}

bool wordBreak(String s, List<String> dict) {
  final words = dict.toSet();
  final dp = List.filled(s.length + 1, false);
  dp[0] = true;                             // empty prefix
  for (var i = 1; i <= s.length; i++) {
    for (var j = 0; j < i; j++) {
      if (dp[j] && words.contains(s.substring(j, i))) {
        dp[i] = true;
        break;
      }
    }
  }
  return dp[s.length];
}

String longestPalindrome(String s) {
  var start = 0, len = 0;
  void expand(int l, int r) {
    while (l >= 0 && r < s.length && s[l] == s[r]) {
      l--;
      r++;
    }
    if (r - l - 1 > len) {
      len = r - l - 1;
      start = l + 1;
    }
  }
  for (var i = 0; i < s.length; i++) {
    expand(i, i);          // odd length
    expand(i, i + 1);      // even length
  }
  return s.substring(start, start + len);
}

int numDecodings(String s) {
  if (s.isEmpty || s[0] == '0') return 0;
  var prev2 = 1, prev1 = 1;                 // ways for prefix lengths i-2 and i-1
  for (var i = 2; i <= s.length; i++) {
    var cur = 0;
    if (s[i - 1] != '0') cur += prev1;
    final two = int.parse(s.substring(i - 2, i));
    if (two >= 10 && two <= 26) cur += prev2;
    prev2 = prev1;
    prev1 = cur;
  }
  return prev1;
}
```
</details>

---

## Interview questions

1. What two properties must a problem have for DP to apply?
2. Memoization (top-down) vs tabulation (bottom-up): trade-offs?
3. How do you decide the DP state? Describe your process.
4. Why does greedy fail for coin change with `[1,3,4]`?
5. How do you reduce space from `O(n²)` to `O(n)` in LCS?
6. When is greedy the right choice over DP?
7. Explain 0/1 vs unbounded knapsack and the loop direction difference.

<details>
<summary>Hints</summary>

1. Optimal substructure and overlapping subproblems.
2. Top-down: easy to write from recursion, computes only needed states, recursion depth risk. Bottom-up: no recursion, easier space optimization, computes all states.
3. Write the plain recursion; its changing parameters are the state; define in words what `f(...)` means.
4. Greedy picks 4 first and yields 4+1+1; DP finds 3+3.
5. Each row depends only on the previous row; keep two rows.
6. When a local-optimal choice is provably globally optimal (exchange argument), like interval scheduling.
7. 0/1: each item once, iterate capacity downward. Unbounded: reuse allowed, iterate upward.
</details>
