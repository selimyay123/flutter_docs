# 03 — Coding Round Playbook

## Goal

Excel in live technical coding rounds. Technical proficiency alone is not enough; interviewers evaluate your **problem-solving framework**, **communication**, and **edge-case handling**.

---

## 1. The 45-Minute Live Interview Roadmap

```
[00-05 min] Clarify requirements & agree on constraints
[05-12 min] Manual examples & discuss Brute Force (Time/Space)
[12-20 min] Propose Optimized Solution & write Pseudocode
[20-35 min] Clean, modular implementation
[35-42 min] Dry-run with test cases & edge cases
[42-45 min] Wrap-up, final Big-O verification, trade-offs
```

---

## 2. Phase 1: Clarification Checklist

**Never start coding immediately.** Ask clarifying questions:
1. **Inputs/Outputs**: "Can the input list be empty or null?"
2. **Bounds**: "What is the maximum size of `N`? Can numbers be negative?"
3. **Duplicates**: "Can elements contain duplicate values? Should the output preserve order?"
4. **Memory constraints**: "Are we optimizing primarily for time, or is memory constrained?"

---

## 3. Communication Strategy: Thinking Out Loud

Interviewers cannot grade thoughts you keep to yourself.

| Situation | Poor Response | Senior Response |
| :--- | :--- | :--- |
| **Facing a problem** | Silence for 4 minutes while writing code. | "Let me first walk through a brute-force approach. We could check all pairs in $O(N^2)$, but looking at the constraints $N = 10^5$, we need an $O(N)$ or $O(N \log N)$ approach." |
| **Getting stuck** | "I don't know how to do this." | "My current approach is struggling with repeated keys. Let me draw an example on paper to see if a two-pointer approach works better than a hash map here." |
| **Finding an edge case** | Silently patch lines in code. | "I noticed that if `target` is 0, this check will fail. Let me add a guard clause at the start to handle this cleanly." |

---

## 4. Live Coding Walkthrough: Real Problem

### The Prompt
*Given an array of integers representing stock prices per day, find the maximum profit you can achieve with at most **two** transactions (buy then sell, then buy then sell again). You cannot engage in multiple transactions simultaneously.*

### Step 1: Clarify & Examples
- Example: `prices = [3, 3, 5, 0, 0, 3, 1, 4]`
- Best: Buy at 0, sell at 3 (profit = 3); buy at 1, sell at 4 (profit = 3). Total = 6.
- Edge case: `prices` has fewer than 2 elements $\rightarrow$ profit = 0.

### Step 2: Formulate Algorithm (State Machine DP)
Instead of an $O(N^2)$ scan, track 4 variables representing the maximum balance at each stage of the two transactions:
1. `buy1`: Max balance after first buy.
2. `sell1`: Max balance after first sell.
3. `buy2`: Max balance after second buy.
4. `sell2`: Max balance after second sell.

### Step 3: Implement

```dart
int maxProfitTwoTransactions(List<int> prices) {
  if (prices.length < 2) return 0;

  var buy1 = -prices[0];
  var sell1 = 0;
  var buy2 = -prices[0];
  var sell2 = 0;

  for (var i = 1; i < prices.length; i++) {
    final price = prices[i];

    buy1 = buy1 > -price ? buy1 : -price;
    sell1 = sell1 > buy1 + price ? sell1 : buy1 + price;
    buy2 = buy2 > sell1 - price ? buy2 : sell1 - price;
    sell2 = sell2 > buy2 + price ? sell2 : buy2 + price;
  }

  return sell2;
}

void main() {
  print(maxProfitTwoTransactions([3, 3, 5, 0, 0, 3, 1, 4])); // 6
  print(maxProfitTwoTransactions([1, 2, 3, 4, 5]));          // 4
  print(maxProfitTwoTransactions([7, 6, 4, 3, 1]));          // 0
}
```

### Step 4: Verify Complexity
- **Time Complexity**: $O(N)$ single-pass.
- **Space Complexity**: $O(1)$ constant variables.

---

## Code Challenge

Solve the **Container With Most Water** problem:
*Given $N$ non-negative integers $a_1, a_2, \dots, a_n$ where each represents a point at coordinate $(i, a_i)$, $n$ vertical lines are drawn. Find two lines that together with the x-axis form a container that contains the most water.*
- Constraints: $N \ge 2$. Target: $O(N)$ time, $O(1)$ space.

<details>
<summary>Solution & Walkthrough</summary>

```dart
int maxArea(List<int> height) {
  var left = 0;
  var right = height.length - 1;
  var maxWater = 0;

  while (left < right) {
    final width = right - left;
    final h = height[left] < height[right] ? height[left] : height[right];
    final currentWater = width * h;

    if (currentWater > maxWater) {
      maxWater = currentWater;
    }

    // Move the shorter line inward
    if (height[left] < height[right]) {
      left++;
    } else {
      right--;
    }
  }

  return maxWater;
}

void main() {
  print(maxArea([1, 8, 6, 2, 5, 4, 8, 3, 7])); // 49
}
```
</details>

---

## Coding Round Checklist

1. Did you restate the problem and confirm your assumptions?
2. Did you state the Big-O of the brute force before jumping to optimization?
3. Did you check boundaries (empty list, single element, negative numbers)?
4. Did you run through at least one manual test case before saying "I am done"?
5. Is your code clean, with descriptive variable names and no dead code?
