# 06 — Stacks, Queues, Linked Lists, Heaps

## Goal

Learn the data structures behind many algorithms and understand their trade-offs.

---

## 1. Stack — LIFO (Last In, First Out)

Like a pile of plates: you add and remove from the **top**.

| Operation | Cost |
|-----------|------|
| push | `O(1)` |
| pop | `O(1)` |
| peek | `O(1)` |

In Dart a `List` works as a stack: `add` = push, `removeLast` = pop, `last` = peek.

```dart
final stack = <int>[];
stack.add(1);
stack.add(2);
print(stack.removeLast()); // 2
print(stack.last);          // 1
```

**Where used:** undo/redo, browser back button, call stack, expression parsing, DFS.

### Classic: valid parentheses

`"({[]})"` → true, `"(]"` → false.

```dart
bool isValid(String s) {
  const pairs = {')': '(', ']': '[', '}': '{'};
  final stack = <String>[];
  for (final ch in s.split('')) {
    if (pairs.containsKey(ch)) {
      if (stack.isEmpty || stack.removeLast() != pairs[ch]) return false;
    } else {
      stack.add(ch);
    }
  }
  return stack.isEmpty;
}
```

### Classic: monotonic stack — "next greater element"

For each temperature, how many days until a warmer one? `[73,74,75,71,69,72,76,73]` → `[1,1,4,2,1,1,0,0]`.

```dart
List<int> dailyTemperatures(List<int> t) {
  final answer = List.filled(t.length, 0);
  final stack = <int>[];                       // indexes with unresolved days
  for (var i = 0; i < t.length; i++) {
    while (stack.isNotEmpty && t[i] > t[stack.last]) {
      final j = stack.removeLast();
      answer[j] = i - j;
    }
    stack.add(i);
  }
  return answer;
}
// O(n): each index is pushed once and popped once
```

Pattern: **"next greater/smaller"** → monotonic stack.

---

## 2. Queue — FIFO (First In, First Out)

Like a line at a shop: add at the **back**, remove from the **front**.

Use `Queue` from `dart:collection` (`List.removeAt(0)` is `O(n)`!).

```dart
import 'dart:collection';

final q = Queue<int>();
q.add(1);            // enqueue
q.add(2);
print(q.removeFirst()); // 1  dequeue, O(1)
print(q.first);         // 2  peek
```

`Queue` also supports `addFirst`/`removeLast` → it is a **deque** (double-ended queue).

**Where used:** BFS, task scheduling, print queue, message queues, sliding window max.

### Sliding window maximum with a deque — `O(n)`

```dart
List<int> maxSlidingWindow(List<int> nums, int k) {
  final dq = Queue<int>();                     // stores indexes, values decreasing
  final out = <int>[];
  for (var i = 0; i < nums.length; i++) {
    if (dq.isNotEmpty && dq.first <= i - k) dq.removeFirst();     // out of window
    while (dq.isNotEmpty && nums[dq.last] < nums[i]) {
      dq.removeLast();                          // smaller values can never be max
    }
    dq.addLast(i);
    if (i >= k - 1) out.add(nums[dq.first]);
  }
  return out;
}
```

---

## 3. Linked List

Nodes where each points to the next. **No index access**, but cheap insert/remove when you hold the node.

```dart
class ListNode {
  ListNode(this.value, [this.next]);
  int value;
  ListNode? next;
}
```

```
head -> [1] -> [2] -> [3] -> null
```

| Operation | Array/List | Linked list |
|-----------|-----------|-------------|
| Access by index | `O(1)` | `O(n)` |
| Insert/remove at front | `O(n)` | `O(1)` |
| Insert/remove in middle (have node) | `O(n)` | `O(1)` |
| Memory | compact | extra pointer per item |
| CPU cache | friendly | unfriendly |

In practice, arrays/lists win almost always. But linked lists teach **pointer thinking** and are a famous interview topic.

### Build, traverse

```dart
ListNode? fromList(List<int> xs) {
  ListNode? head;
  for (final x in xs.reversed) {
    head = ListNode(x, head);
  }
  return head;
}

void printList(ListNode? head) {
  final parts = <int>[];
  for (var n = head; n != null; n = n.next) parts.add(n.value);
  print(parts.join(' -> '));
}
```

### Reverse a linked list (must know!)

```dart
ListNode? reverseList(ListNode? head) {
  ListNode? prev;
  var cur = head;
  while (cur != null) {
    final next = cur.next;     // 1. remember next
    cur.next = prev;           // 2. flip arrow
    prev = cur;                // 3. move prev
    cur = next;                // 4. move cur
  }
  return prev;
}
```

Draw it: `1 → 2 → 3` becomes `null ← 1 ← 2 ← 3`. `O(n)` time, `O(1)` space.

### Fast & slow pointers

- **Middle of list:** slow moves 1 step, fast moves 2. When fast ends, slow is in the middle.
- **Cycle detection (Floyd):** if fast catches slow, there is a cycle.

```dart
bool hasCycle(ListNode? head) {
  var slow = head, fast = head;
  while (fast != null && fast.next != null) {
    slow = slow!.next;
    fast = fast.next!.next;
    if (identical(slow, fast)) return true;
  }
  return false;
}

ListNode? middle(ListNode? head) {
  var slow = head, fast = head;
  while (fast != null && fast.next != null) {
    slow = slow!.next;
    fast = fast.next!.next;
  }
  return slow;
}
```

### Merge two sorted lists — the **dummy head** trick

```dart
ListNode? mergeTwo(ListNode? a, ListNode? b) {
  final dummy = ListNode(0);
  var tail = dummy;
  while (a != null && b != null) {
    if (a.value <= b.value) {
      tail.next = a;
      a = a.next;
    } else {
      tail.next = b;
      b = b.next;
    }
    tail = tail.next!;
  }
  tail.next = a ?? b;
  return dummy.next;
}
```

A dummy node removes special cases for "the head".

---

## 4. Heap / Priority Queue

A structure that always gives the **smallest** (min-heap) or **largest** (max-heap) item quickly.

| Operation | Cost |
|-----------|------|
| insert | `O(log n)` |
| get min | `O(1)` |
| remove min | `O(log n)` |
| build from n items | `O(n)` |

Dart's core library has `PriorityQueue` in `package:collection`:

```dart
import 'package:collection/collection.dart';

final heap = PriorityQueue<int>();             // min-heap by default
heap.add(5);
heap.add(1);
heap.add(3);
print(heap.removeFirst());  // 1

final maxHeap = PriorityQueue<int>((a, b) => b.compareTo(a));
```

### Top K frequent elements

```dart
List<int> topK(List<int> nums, int k) {
  final counts = <int, int>{};
  for (final n in nums) counts[n] = (counts[n] ?? 0) + 1;

  // min-heap of size k keyed by frequency
  final heap = PriorityQueue<MapEntry<int, int>>((a, b) => a.value.compareTo(b.value));
  for (final e in counts.entries) {
    heap.add(e);
    if (heap.length > k) heap.removeFirst();   // drop the least frequent
  }
  return heap.toList().map((e) => e.key).toList();
}
// O(n log k)
```

Pattern: **"top K / k-th largest"** → heap of size k.

### How a heap works (concept)

A binary tree stored in an array where each parent ≤ its children. Insert at the end and **bubble up**; remove the root, move the last to the root and **sink down**. Index math: children of `i` are `2i+1` and `2i+2`; parent is `(i-1) ~/ 2`.

---

## 5. Choose the structure

| Need | Structure |
|------|-----------|
| Undo / matching / next greater | Stack |
| BFS / order of arrival | Queue |
| Both ends | Deque (`Queue`) |
| Fast min or max repeatedly | Heap |
| Lookup by key | Map |
| Uniqueness | Set |
| Frequent insert/remove in the middle with node refs | Linked list |

---

## Common mistakes

- Using `removeAt(0)` as queue dequeue (`O(n)`).
- Popping from an empty stack.
- Losing the rest of a linked list by overwriting `next` before saving it.
- Forgetting null checks at list ends.
- Using a heap when sorting once is enough.

---

## Code challenge

1. Implement `MinStack` with `push`, `pop`, `top`, and `getMin` all in `O(1)`.
2. Implement a `Queue` using **two stacks**.
3. Remove the n-th node from the end of a linked list in **one pass** (hint: two pointers n apart).
4. Merge `k` sorted lists using a heap.
5. Evaluate Reverse Polish Notation: `["2","1","+","3","*"]` → `9`.

<details>
<summary>Solution</summary>

```dart
class MinStack {
  final _stack = <int>[];
  final _mins = <int>[];                // mins[i] = min of stack[0..i]

  void push(int x) {
    _stack.add(x);
    _mins.add(_mins.isEmpty || x < _mins.last ? x : _mins.last);
  }

  int pop() {
    _mins.removeLast();
    return _stack.removeLast();
  }

  int get top => _stack.last;
  int get min => _mins.last;
}

class QueueFromStacks<T> {
  final _in = <T>[];
  final _out = <T>[];

  void enqueue(T x) => _in.add(x);

  T dequeue() {
    if (_out.isEmpty) {
      while (_in.isNotEmpty) {
        _out.add(_in.removeLast());      // reverse order
      }
    }
    return _out.removeLast();            // amortized O(1)
  }
}

ListNode? removeNthFromEnd(ListNode? head, int n) {
  final dummy = ListNode(0, head);
  var fast = dummy as ListNode?, slow = dummy as ListNode?;
  for (var i = 0; i < n; i++) fast = fast!.next;
  while (fast!.next != null) {
    fast = fast.next;
    slow = slow!.next;
  }
  slow!.next = slow.next!.next;
  return dummy.next;
}

int evalRPN(List<String> tokens) {
  final st = <int>[];
  for (final t in tokens) {
    switch (t) {
      case '+': st.add(st.removeLast() + st.removeLast());
      case '*': st.add(st.removeLast() * st.removeLast());
      case '-': final b = st.removeLast(), a = st.removeLast(); st.add(a - b);
      case '/': final b = st.removeLast(), a = st.removeLast(); st.add(a ~/ b);
      default: st.add(int.parse(t));
    }
  }
  return st.single;
}
```

For #4: push the head of each list into a min-heap keyed by value; pop the smallest, append, push its `next`. `O(N log k)`.
</details>

---

## Interview questions

1. Stack vs queue: give a real use for each.
2. Why is `List.removeAt(0)` a poor dequeue? What do you use instead?
3. Array vs linked list: compare access, insertion, memory, cache behavior.
4. How do you detect a cycle in a linked list in `O(1)` space?
5. What is a monotonic stack and what problems does it solve?
6. How does a heap give `O(log n)` insert and remove-min?
7. How would you find the k-th largest element in a stream?

<details>
<summary>Hints</summary>

1. Stack: undo, DFS. Queue: BFS, task scheduling.
2. Shifts all elements → `O(n)`. Use `Queue` from `dart:collection`.
3. Array: `O(1)` index, cache friendly. Linked list: `O(1)` splice with node, pointer overhead, poor cache.
4. Floyd: slow and fast pointers.
5. Stack keeping elements in sorted order; "next greater/smaller", histograms, temperatures.
6. Tree height is log n; bubble up/sink down along one path.
7. Min-heap of size k; the root is the k-th largest.
</details>
