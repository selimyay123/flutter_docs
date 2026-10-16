# 01 — How to Study (and How to Think)

## Goal

Learn *how to learn* programming. Tools change. Thinking does not.

---

## 1. Programming is problem solving

Code is just the way you write the solution down. The real work is:

1. **Understand** the problem.
2. **Break it** into small pieces.
3. **Find a pattern** you already know.
4. **Write** the simplest solution.
5. **Test** it. Break it. Improve it.

Beginners jump straight to step 4. Seniors spend most time on steps 1–3.

---

## 2. The 5-step problem solving loop

Use this loop for **every** challenge in this course.

```
1. RESTATE   -> Say the problem in your own words.
2. EXAMPLES  -> Write 3 inputs and the expected outputs (normal, edge, weird).
3. PLAN      -> Write steps in plain English (pseudocode). No code yet.
4. CODE      -> Translate the plan into Dart.
5. REVIEW    -> Test examples. Check speed (Big-O). Simplify.
```

### Example

Problem: *"Find the largest number in a list."*

1. **Restate:** Look at every number, remember the biggest one.
2. **Examples:**
   - `[3, 9, 2]` → `9`
   - `[-5, -1]` → `-1`
   - `[]` → ? (edge case: decide — error or `null`)
3. **Plan:** Start with the first item as "best". For each next item, if bigger, replace "best". Return "best".
4. **Code:**
   ```dart
   int? largest(List<int> numbers) {
     if (numbers.isEmpty) return null;
     var best = numbers.first;
     for (final n in numbers) {
       if (n > best) best = n;
     }
     return best;
   }
   ```
5. **Review:** One pass over the list → time `O(n)`, extra memory `O(1)`.

---

## 3. Rules of a good learner

| Rule | Why |
|------|-----|
| Type code by hand | Your fingers build memory |
| Predict output before running | Trains your mental compiler |
| Struggle for 20 min before hints | Struggle creates learning |
| Read error messages fully | The answer is usually in them |
| Explain out loud | Finds gaps in understanding |
| Build small things often | Skill = repetition |
| Review yesterday first | Spaced repetition beats cramming |

---

## 4. The debugging mindset

When code fails, do **not** guess randomly. Do this:

1. **Reproduce** — make the bug happen every time.
2. **Isolate** — remove code until the bug is small.
3. **Hypothesize** — "I think X happens because Y."
4. **Test** one hypothesis (print, debugger, unit test).
5. **Fix** the root cause, not the symptom.

> Bugs are not failures. They are the computer telling you your mental model is wrong.

---

## 5. How to use the challenges

- Create a `my_solutions/` folder.
- One file per challenge: `my_solutions/01_largest.dart`.
- Solutions in the lessons are hidden in `<details>` blocks. Open them **only after** you tried.
- After solving, ask: *"Can I make it simpler? Faster? Safer?"*

---

## Code challenge

Without using `list.reduce`, `max`, or `sort`, write `int? secondLargest(List<int> numbers)`.

- `[4, 9, 7]` → `7`
- `[5, 5, 5]` → `null` (no second distinct value) — your choice, but be consistent.
- `[]` → `null`

Follow the 5-step loop and write your plan in a comment first.

<details>
<summary>Solution</summary>

```dart
int? secondLargest(List<int> numbers) {
  int? first;
  int? second;
  for (final n in numbers) {
    if (first == null || n > first) {
      second = first;
      first = n;
    } else if (n != first && (second == null || n > second)) {
      second = n;
    }
  }
  return second;
}

My Solution

int? secondLargest(List<int> numbers) {
    if (numbers.isEmpty) return null;
    var largestNum = numbers.first;
    var secondLargestNum = numbers.first;
    for (final num in numbers) {
      if (num > largestNum) largestNum = num;
    }
    for (final num in numbers) {
      if (num == largestNum) continue;
      if (num > secondLargestNum) secondLargestNum = num;
    }
    return secondLargestNum;
  }
```

One pass, `O(n)` time, `O(1)` memory.
</details>

---

## Interview questions

1. Walk me through how you approach a problem you have never seen before.
2. What is an edge case? Give three for a function that finds the largest number in a list.
3. Why should you write examples *before* code?
4. How do you debug a bug that only happens sometimes?
5. What does "root cause" mean, and why is fixing a symptom dangerous?

<details>
<summary>Hints</summary>

1. Restate → examples → plan → code → review. Ask clarifying questions out loud.
2. A rare input at the border of valid input: empty list, one item, all equal, negatives, huge values.
3. Examples reveal misunderstandings cheaply; they become your first tests.
4. Add logging, find what differs between good and bad runs (timing, data, state), write a test that loops until it fails.
5. The underlying reason the bug exists. Fixing the symptom hides it and it returns somewhere else.
</details>
