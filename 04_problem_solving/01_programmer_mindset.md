# 01 — The Programmer Mindset

## Goal

Learn to **think** like a problem solver. This lesson matters more than any syntax.

---

## 1. What is an algorithm?

An algorithm is a **finite list of clear steps** that turns input into output.

A recipe is an algorithm. Finding a word in a dictionary is an algorithm.
Programming = designing algorithms + choosing data structures to hold the data.

> Programs = Algorithms + Data Structures. — Niklaus Wirth

---

## 2. The Problem-Solving Framework (use it every time)

### Step 1 — Understand

Ask until there is no doubt:

- What is the **input**? (type, size, range, can it be empty/null/negative?)
- What is the **output**?
- What are the **constraints**? (n = 10 or n = 10 million?)
- What are the **edge cases**?

### Step 2 — Examples

Write small examples by hand, including a tricky one. Solve them **manually** and notice what *you* do.
Your brain already knows an algorithm. Observe it.

### Step 3 — Brute force first

Find **any** correct solution, even slow. Say its cost out loud.

```
Problem: Do two numbers in the list add up to target?
Brute force: try every pair -> O(n²). It's correct. Now improve.
```

### Step 4 — Optimize

Ask the **optimization questions**:

1. Am I **repeating work**? → store results (hash map, memo)
2. Am I **searching** a lot? → sort, binary search, hash
3. Is the data **sorted** or can it be? → two pointers, binary search
4. Do I need **all** combos or just best? → greedy / DP
5. Is it **self-similar**? → recursion
6. Is it a **network** of things? → graph / BFS / DFS
7. Do I need **min/max** of a moving range? → heap, deque, sliding window

### Step 5 — Plan in pseudocode, then code

### Step 6 — Test

Run your examples by hand through the code. Test edge cases. State the final Big-O.

---

## 3. Pattern recognition cheat sheet

| If you see... | Think... |
|--------------|----------|
| "Pair/triple that sums to..." | Hash map, two pointers |
| "Sorted array" | Binary search, two pointers |
| "Substring / subarray with a condition" | Sliding window |
| "Count/frequency/duplicates" | Hash map / set |
| "Next greater / matching brackets" | Stack |
| "Shortest path (unweighted)" | BFS |
| "All combinations/permutations" | Backtracking |
| "Optimal with overlapping subproblems" | Dynamic programming |
| "Top K / k-th largest" | Heap |
| "Tree" | Recursion (DFS), BFS by levels |
| "Intervals" | Sort by start, merge |
| "Linked list cycle / middle" | Fast & slow pointers |
| "Connected components / grouping" | Union-Find or DFS |

You build this table in your head by **solving many problems** and writing down the pattern each time.

---

## 4. Worked example — walk through the full framework

**Problem:** *Given a list of integers and a target, return the indices of two numbers that add up to the target.*

**Understand:** Exactly one solution? Can I reuse an element? (No.) Negatives? (Yes.)

**Examples:**

- `[2, 7, 11, 15], 9` → `[0, 1]`
- `[3, 3], 6` → `[0, 1]`

**Brute force:**

```dart
List<int> twoSumBrute(List<int> nums, int target) {
  for (var i = 0; i < nums.length; i++) {
    for (var j = i + 1; j < nums.length; j++) {
      if (nums[i] + nums[j] == target) return [i, j];
    }
  }
  return [];
}
// Time O(n²), space O(1)
```

**Optimize:** I keep *searching* for `target - nums[i]`. Searching is slow in a list but `O(1)` in a map.
So remember each number I've seen with its index.

```dart
List<int> twoSum(List<int> nums, int target) {
  final seen = <int, int>{};                 // value -> index
  for (var i = 0; i < nums.length; i++) {
    final need = target - nums[i];
    final j = seen[need];
    if (j != null) return [j, i];
    seen[nums[i]] = i;
  }
  return [];
}
// Time O(n), space O(n)
```

Trade-off: we spent memory (`O(n)`) to save time. **Trade-offs are the heart of engineering.**

---

## 5. Techniques to unstick yourself

1. **Solve a smaller version.** (n = 1, 2, 3)
2. **Draw it.** Boxes and arrows. Always.
3. **Work backwards** from the answer.
4. **Change the data** (sort it, index it, group it).
5. **Simplify constraints** then add them back.
6. **Explain it to a rubber duck.**
7. **Look for what is repeated.**
8. **Walk away for 10 minutes.** The background brain works.

---

## 6. How to practice (the 20-minute rule)

1. Try for **20 minutes** with the framework.
2. Still stuck? Read **only a hint**, not the whole solution.
3. After solving, write down: *pattern, trick, Big-O, mistakes*.
4. **Re-solve** it from scratch after 3 days, then after 2 weeks (spaced repetition).
5. Keep a `problems_log.md`:

```
| Date | Problem | Pattern | Time | Mistake | Redo date |
```

Quality over quantity: 100 problems **understood** beats 500 problems copied.

---

## 7. Communicating (interview skill)

Think out loud:

1. Restate the problem.
2. Ask clarifying questions.
3. Give a brute-force idea and its cost.
4. Improve it, explain the trade-off.
5. Code cleanly (good names, small helpers).
6. Test with an example, then edge cases.
7. State the Big-O.

Interviewers hire for **how you think**, not just the final answer.

---

## 8. Mindset habits

| Habit | Why |
|-------|-----|
| Be curious about *why* it works | Real understanding transfers |
| Embrace being stuck | It is the learning, not a failure |
| Simplest thing that works | Complexity is the enemy |
| Make it work → make it right → make it fast | Order matters |
| Never optimize without measuring | Guessing wastes time |
| Read other people's code | You learn styles and tricks |

---

## Code challenge

Apply the **full framework** (write each step in comments) to:

1. *Given a list of integers, return `true` if any value appears at least twice.* (You solved something like this earlier. Now follow the steps.)
2. *Given a string, return the first character that does not repeat, or `null`.* `"swiss"` → `'w'`.
3. *Given a list of prices by day, find the max profit from one buy then one sell.* `[7,1,5,3,6,4]` → `5`.

<details>
<summary>Solution</summary>

```dart
bool containsDuplicate(List<int> xs) {
  final seen = <int>{};
  for (final x in xs) {
    if (!seen.add(x)) return true;
  }
  return false;
}

String? firstUnique(String s) {
  final counts = <String, int>{};
  for (final ch in s.split('')) {
    counts[ch] = (counts[ch] ?? 0) + 1;
  }
  for (final ch in s.split('')) {
    if (counts[ch] == 1) return ch;
  }
  return null;
}

int maxProfit(List<int> prices) {
  var minPrice = 1 << 62;       // very large
  var best = 0;
  for (final p in prices) {
    if (p < minPrice) minPrice = p;
    if (p - minPrice > best) best = p - minPrice;
  }
  return best;
}
```

Idea for #3: for each day, the best profit selling today = today - cheapest day so far. One pass: `O(n)`.
</details>

---

## Interview questions

1. What are the first three things you do when given a new problem?
2. Why start with brute force?
3. Name 5 "optimization questions" you ask yourself.
4. What is a trade-off? Give a time-vs-memory example.
5. How do you recognize a sliding-window problem? A DP problem?
6. What do you do when you are completely stuck in an interview?

<details>
<summary>Hints</summary>

1. Restate, ask about input/constraints/edge cases, write examples.
2. Gives a correct baseline, reveals what's slow, shows the interviewer your reasoning.
3. Repeating work? Searching a lot? Sorted? Overlapping subproblems? Self-similar?
4. Hash map for `O(1)` lookup costs `O(n)` extra memory but removes an inner loop.
5. Sliding window: contiguous subarray/substring + condition. DP: optimal/count over choices with overlapping subproblems.
6. Simplify (smaller input), draw, test an example by hand, say your thoughts, ask for a hint.
</details>
