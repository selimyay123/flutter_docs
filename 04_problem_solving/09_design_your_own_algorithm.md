# 09 — Design Your Own Algorithm

## Goal

Move from "I recognize known problems" to "I can **invent** a solution for a problem I have never seen."
This is the skill that separates strong engineers from people who memorize.

---

## 1. Algorithm design is a skill with a method

The method:

```
1. Model the problem (data + rules)
2. Find a correct (even slow) solution
3. Find the bottleneck
4. Apply a design strategy to remove it
5. Prove it (examples, invariants)
6. Measure it
```

---

## 2. The 8 design strategies

| Strategy | Core idea | Example |
|----------|-----------|---------|
| **Brute force** | Try everything | Check all pairs |
| **Divide & conquer** | Split, solve parts, combine | Merge sort, binary search |
| **Greedy** | Best local choice | Interval scheduling |
| **Dynamic programming** | Reuse overlapping subresults | Knapsack |
| **Backtracking** | Build, undo, prune | Sudoku, N-Queens |
| **Transform** | Change the data first | Sort, then scan |
| **Hashing / indexing** | Precompute for fast lookup | Two-sum map |
| **Graph modeling** | See states as nodes, moves as edges | Word ladder |

The most creative one: **modeling.** The same problem can be easy or impossible depending on how you represent it.

---

## 3. Modeling example — Word Ladder

*Change `hit` to `cog` one letter at a time, each step must be a word in the dictionary. Fewest steps?*

New problem? No. Model it:

- **Node** = a word.
- **Edge** = two words that differ by one letter.
- **Shortest path in an unweighted graph** = **BFS**.

```dart
import 'dart:collection';

int ladderLength(String begin, String end, List<String> dict) {
  final words = dict.toSet();
  if (!words.contains(end)) return 0;
  final q = Queue<(String, int)>()..add((begin, 1));
  final seen = <String>{begin};
  while (q.isNotEmpty) {
    final (word, steps) = q.removeFirst();
    if (word == end) return steps;
    for (var i = 0; i < word.length; i++) {
      for (var c = 97; c <= 122; c++) {                  // 'a'..'z'
        final next = word.substring(0, i) + String.fromCharCode(c) + word.substring(i + 1);
        if (words.contains(next) && seen.add(next)) {
          q.add((next, steps + 1));
        }
      }
    }
  }
  return 0;
}
```

Lesson: **state-space search.** Many puzzles (sliding tile, water jug, lock combinations) = BFS over states.

---

## 4. Invent-an-algorithm exercise: step by step

**Problem:** *Meeting rooms II: given meeting time intervals, what is the minimum number of rooms needed?*
`[[0,30],[5,10],[15,20]]` → `2`.

### Step 1 — Brute force thinking

For every moment in time, count how many meetings are happening. The answer is the max count. Too slow if times are huge.

### Step 2 — Observe

Only the **start** and **end** moments change the count. Everything between is constant.

### Step 3 — Transform

Turn intervals into **events**: `+1` at start, `-1` at end. Sort events by time. Sweep through, keep a running count.

```dart
int minMeetingRooms(List<List<int>> meetings) {
  final events = <(int time, int delta)>[];
  for (final [start, end] in meetings) {
    events.add((start, 1));
    events.add((end, -1));
  }
  events.sort((a, b) {
    final byTime = a.$1.compareTo(b.$1);
    return byTime != 0 ? byTime : a.$2.compareTo(b.$2);   // ends (-1) before starts (+1) at same time
  });
  var rooms = 0, best = 0;
  for (final e in events) {
    rooms += e.$2;
    if (rooms > best) best = rooms;
  }
  return best;
}
// O(n log n)
```

**Technique learned:** the *sweep line*. You invented it by asking "what actually changes?"

### Alternative idea: a min-heap of end times

Sort by start. Keep a heap of end times of rooms in use. For each meeting: if the earliest-ending room is free, reuse it; else add a room. Different strategy, same complexity. Having **two** solutions shows depth.

---

## 5. Prove it: invariants

An **invariant** is something that stays true during the algorithm. If you can state it, you can trust it.

- Binary search: "the target is within `[lo, hi]`."
- Kadane: "`current` is the best sum of a subarray ending here."
- BFS: "queue contains nodes at distance `d` then `d+1`."
- Insertion sort: "the left part is sorted."

**Habit:** before coding a loop, write its invariant as a comment.

```dart
// Invariant: xs[0..i) is sorted
for (var i = 1; i < xs.length; i++) { ... }
```

Also practice **proof by exchange** for greedy: "if there were a better solution that does not make my choice, I can swap and not make it worse."

---

## 6. Test like a scientist

1. **Edge cases:** empty, one item, two items, all equal, sorted, reverse sorted, negatives, huge.
2. **Brute-force comparison:** write the slow, obviously correct version; run both on thousands of random inputs; compare.

```dart
import 'dart:math';

void stressTest() {
  final rnd = Random(42);
  for (var t = 0; t < 10000; t++) {
    final xs = List.generate(rnd.nextInt(8), (_) => rnd.nextInt(21) - 10);
    final fast = maxSubarrayFast(xs);
    final slow = maxSubarrayBrute(xs);
    if (fast != slow) {
      print('MISMATCH on $xs: fast=$fast slow=$slow');
      return;
    }
  }
  print('All good');
}
```

This **stress testing** technique finds bugs humans miss. Use it for any tricky algorithm.

---

## 7. Design thinking in real apps (Flutter)

Algorithm skill is not only for interviews.

| Real problem | Algorithm idea |
|--------------|----------------|
| Autocomplete suggestions | Trie / prefix search + debounce |
| Diffing a list to animate changes | LCS / edit distance (`DiffUtil`) |
| Infinite scroll with caching | Queue + LRU cache |
| Undo / redo | Two stacks |
| Offline sync conflict resolution | Merge, timestamps, version vectors |
| Route/navigation guards | Graph |
| Search in a big list | Index + binary search |
| Drag-and-drop reorder | Array splice, index math |
| Layout (Flutter itself) | Tree recursion: constraints down, sizes up |

### LRU cache — a classic that is also genuinely useful (image cache!)

Idea: a Map that keeps insertion order. On access, remove and re-insert to make the key "newest". When full, evict the oldest.

```dart
class LruCache<K, V> {
  LruCache(this.capacity);
  final int capacity;
  final _map = <K, V>{};                 // Dart's default Map keeps insertion order

  V? get(K key) {
    final value = _map.remove(key);
    if (value == null) return null;
    _map[key] = value;                   // re-insert -> now the most recent
    return value;
  }

  void put(K key, V value) {
    _map.remove(key);
    _map[key] = value;
    if (_map.length > capacity) {
      _map.remove(_map.keys.first);      // evict least recently used
    }
  }
}
```

All operations `O(1)` on average.

---

## 8. A 30-day training plan

| Days | Topic | Goal |
|------|-------|------|
| 1–5 | Arrays, hash maps | 10 problems |
| 6–10 | Two pointers, sliding window | 10 problems |
| 11–14 | Stack, queue, linked list | 8 problems |
| 15–18 | Binary search, sorting | 8 problems |
| 19–22 | Trees, BFS/DFS | 10 problems |
| 23–26 | Graphs, heap | 8 problems |
| 27–30 | Recursion/backtracking, DP | 10 problems |

Then: mixed problems, timed (30 min), and **explain each aloud**.

Good sources: LeetCode ("Top Interview 150", "NeetCode 150"), HackerRank, Exercism (Dart track), Advent of Code.

---

## Common mistakes

- Coding before understanding.
- Jumping to advanced structures when a map and a loop would do.
- Only reading solutions.
- Not testing edge cases.
- Never stress-testing.

---

## Code challenge

Design and implement (explain your strategy first):

1. **Trie** with `insert`, `search`, `startsWith`. Then use it to implement autocomplete returning all words with a prefix.
2. **Two-stack undo/redo** manager for a text editor with `type(String)`, `undo()`, `redo()`.
3. **Task scheduler:** tasks with dependencies and durations; compute the **minimum total time** with unlimited workers (hint: topological order + longest path in a DAG).
4. **Stress test** your `maxSubarray` against a brute-force version.

<details>
<summary>Solution (1 and 2)</summary>

```dart
class TrieNode {
  final children = <String, TrieNode>{};
  bool isWord = false;
}

class Trie {
  final _root = TrieNode();

  void insert(String word) {
    var n = _root;
    for (final ch in word.split('')) {
      n = n.children.putIfAbsent(ch, TrieNode.new);
    }
    n.isWord = true;
  }

  TrieNode? _find(String prefix) {
    var n = _root;
    for (final ch in prefix.split('')) {
      final next = n.children[ch];
      if (next == null) return null;
      n = next;
    }
    return n;
  }

  bool search(String w) => _find(w)?.isWord ?? false;
  bool startsWith(String p) => _find(p) != null;

  List<String> autocomplete(String prefix) {
    final start = _find(prefix);
    if (start == null) return [];
    final out = <String>[];
    void walk(TrieNode n, String path) {
      if (n.isWord) out.add(path);
      n.children.forEach((ch, child) => walk(child, path + ch));
    }
    walk(start, prefix);
    return out;
  }
}

class Editor {
  var _text = '';
  final _undo = <String>[];
  final _redo = <String>[];

  String get text => _text;

  void type(String s) {
    _undo.add(_text);
    _redo.clear();                 // a new action invalidates the redo history
    _text += s;
  }

  void undo() {
    if (_undo.isEmpty) return;
    _redo.add(_text);
    _text = _undo.removeLast();
  }

  void redo() {
    if (_redo.isEmpty) return;
    _undo.add(_text);
    _text = _redo.removeLast();
  }
}
```

Trie cost: insert/search `O(L)` where `L` = word length — independent of how many words are stored.
</details>

---

## Interview questions

1. Walk me through how you would design an algorithm for a problem you have never seen.
2. What is an invariant? Give an example from a loop you wrote.
3. How do you test an algorithm beyond the given examples?
4. How can modeling a problem as a graph simplify it? Example?
5. How would you implement an LRU cache? Complexity?
6. What is the sweep-line technique?
7. When would you choose a trie over a hash map?

<details>
<summary>Hints</summary>

1. Model, brute force, find bottleneck, apply strategy, prove with invariants, test, measure.
2. Statement always true at the loop start, e.g. "prefix is sorted".
3. Edge cases, brute-force comparison with random inputs (stress test), property checks.
4. Word ladder: words = nodes, one-letter changes = edges → BFS.
5. Hash map + order (insertion-ordered map or doubly linked list); `O(1)` get/put.
6. Convert intervals to start/end events, sort, and sweep keeping a running state.
7. Prefix queries (autocomplete); hash maps can only do exact lookup.
</details>
