# 07 — Trees and Graphs

## Goal

Model hierarchies and networks, then explore them with DFS and BFS.

Why this matters for Flutter: the **widget tree** is a tree. Navigation is a graph. Dependencies, folders, and social networks are graphs.

---

## 1. Binary trees

Each node has at most two children: `left` and `right`.

```dart
class TreeNode {
  TreeNode(this.value, [this.left, this.right]);
  int value;
  TreeNode? left;
  TreeNode? right;
}
```

```
        1
       / \
      2   3
     / \
    4   5
```

Vocabulary: **root** (top), **leaf** (no children), **height** (longest path down), **depth** (distance from root), **subtree**.

### Build the tree above

```dart
final tree = TreeNode(1, TreeNode(2, TreeNode(4), TreeNode(5)), TreeNode(3));
```

---

## 2. Depth-First Search (DFS) on trees — recursion

A tree is **self-similar**: a node plus two smaller trees. So recursion fits perfectly.

Think: "What do I need from the left subtree and right subtree, and how do I combine them?"

### Height

```dart
int height(TreeNode? n) {
  if (n == null) return 0;                         // base case
  return 1 + max(height(n.left), height(n.right));
}
```

(Use `import 'dart:math';` for `max`.)

### Traversals

```dart
void preorder(TreeNode? n, List<int> out) {        // node, left, right
  if (n == null) return;
  out.add(n.value);
  preorder(n.left, out);
  preorder(n.right, out);
}

void inorder(TreeNode? n, List<int> out) {         // left, node, right
  if (n == null) return;
  inorder(n.left, out);
  out.add(n.value);
  inorder(n.right, out);
}

void postorder(TreeNode? n, List<int> out) {       // left, right, node
  if (n == null) return;
  postorder(n.left, out);
  postorder(n.right, out);
  out.add(n.value);
}
```

For the tree above: preorder `1 2 4 5 3`, inorder `4 2 5 1 3`, postorder `4 5 2 3 1`.

When to use: **preorder** copies/serializes a tree; **inorder** gives sorted order in a BST; **postorder** deletes/evaluates children first.

### Common tree questions

```dart
bool isSameTree(TreeNode? a, TreeNode? b) {
  if (a == null && b == null) return true;
  if (a == null || b == null) return false;
  return a.value == b.value && isSameTree(a.left, b.left) && isSameTree(a.right, b.right);
}

TreeNode? invert(TreeNode? n) {
  if (n == null) return null;
  final left = invert(n.left);
  final right = invert(n.right);
  n.left = right;
  n.right = left;
  return n;
}

bool hasPathSum(TreeNode? n, int target) {
  if (n == null) return false;
  if (n.left == null && n.right == null) return n.value == target;
  return hasPathSum(n.left, target - n.value) || hasPathSum(n.right, target - n.value);
}

// Diameter: longest path between any two nodes
int diameter(TreeNode? root) {
  var best = 0;
  int depth(TreeNode? n) {
    if (n == null) return 0;
    final l = depth(n.left), r = depth(n.right);
    if (l + r > best) best = l + r;               // path through this node
    return 1 + (l > r ? l : r);
  }
  depth(root);
  return best;
}
```

The `diameter` shows an important trick: a helper returns **one thing** (height) while updating **another** (best answer).

---

## 3. Breadth-First Search (BFS) on trees — queue

Visit **level by level**.

```dart
import 'dart:collection';

List<List<int>> levelOrder(TreeNode? root) {
  if (root == null) return [];
  final result = <List<int>>[];
  final q = Queue<TreeNode>()..add(root);
  while (q.isNotEmpty) {
    final size = q.length;                  // nodes in this level
    final level = <int>[];
    for (var i = 0; i < size; i++) {
      final n = q.removeFirst();
      level.add(n.value);
      if (n.left != null) q.add(n.left!);
      if (n.right != null) q.add(n.right!);
    }
    result.add(level);
  }
  return result;
}
// [[1],[2,3],[4,5]]
```

The `size` trick (process exactly the current level) is reused in many BFS problems.

---

## 4. Binary Search Tree (BST)

Rule: for every node, **left subtree < node < right subtree**.

```
      8
     / \
    3   10
   / \    \
  1   6    14
```

Search, insert: go left or right. Average `O(log n)`, worst `O(n)` (if degenerate like a linked list). Balanced trees (AVL, Red-Black) guarantee `O(log n)`. Dart's `SplayTreeMap`/`SplayTreeSet` are self-balancing.

```dart
TreeNode insert(TreeNode? n, int v) {
  if (n == null) return TreeNode(v);
  if (v < n.value) {
    n.left = insert(n.left, v);
  } else if (v > n.value) {
    n.right = insert(n.right, v);
  }
  return n;
}

bool contains(TreeNode? n, int v) {
  while (n != null) {
    if (v == n.value) return true;
    n = v < n.value ? n.left : n.right;
  }
  return false;
}

// Validate a BST: pass allowed bounds down
bool isValidBST(TreeNode? n, [int? lo, int? hi]) {
  if (n == null) return true;
  if (lo != null && n.value <= lo) return false;
  if (hi != null && n.value >= hi) return false;
  return isValidBST(n.left, lo, n.value) && isValidBST(n.right, n.value, hi);
}
```

Common bug: checking only the immediate children. Every node in the left subtree must be smaller, not just the left child.

---

## 5. Graphs

A graph = **nodes (vertices)** + **connections (edges)**. Edges can be directed or undirected, weighted or not.

### Representing a graph

**Adjacency list** (best for most cases): map node → neighbors.

```dart
final graph = <int, List<int>>{
  0: [1, 2],
  1: [0, 3],
  2: [0, 3],
  3: [1, 2, 4],
  4: [3],
};
```

| Representation | Memory | Check edge | List neighbors |
|----------------|--------|-----------|----------------|
| Adjacency list | `O(V + E)` | `O(degree)` | `O(degree)` |
| Adjacency matrix | `O(V²)` | `O(1)` | `O(V)` |

### DFS on a graph

Graphs can have **cycles**, so keep a `visited` set.

```dart
List<int> dfs(Map<int, List<int>> g, int start) {
  final visited = <int>{};
  final order = <int>[];

  void go(int node) {
    if (!visited.add(node)) return;
    order.add(node);
    for (final next in g[node] ?? const <int>[]) {
      go(next);
    }
  }

  go(start);
  return order;
}
```

### BFS on a graph — **shortest path in an unweighted graph**

```dart
int? shortestPath(Map<int, List<int>> g, int start, int goal) {
  final visited = <int>{start};
  final q = Queue<(int, int)>()..add((start, 0));      // (node, distance)
  while (q.isNotEmpty) {
    final (node, dist) = q.removeFirst();
    if (node == goal) return dist;
    for (final next in g[node] ?? const <int>[]) {
      if (visited.add(next)) q.add((next, dist + 1));
    }
  }
  return null;                                         // unreachable
}
```

BFS explores in rings (distance 1, then 2, ...), so the first time you reach the goal it is the shortest route.

Time `O(V + E)`, space `O(V)`.

### Grid problems = graphs

A grid cell's neighbors are up/down/left/right.

**Number of islands** (`1` = land, `0` = water):

```dart
int numIslands(List<List<String>> grid) {
  final rows = grid.length, cols = grid[0].length;
  var count = 0;

  void sink(int r, int c) {
    if (r < 0 || c < 0 || r >= rows || c >= cols || grid[r][c] != '1') return;
    grid[r][c] = '0';                    // mark visited
    sink(r + 1, c);
    sink(r - 1, c);
    sink(r, c + 1);
    sink(r, c - 1);
  }

  for (var r = 0; r < rows; r++) {
    for (var c = 0; c < cols; c++) {
      if (grid[r][c] == '1') {
        count++;
        sink(r, c);
      }
    }
  }
  return count;
}
```

### Topological sort — order tasks with dependencies

"Course A must come before B". Works only on **directed acyclic graphs (DAG)**.

```dart
List<int>? topoSort(int n, List<List<int>> edges) {     // edge [a, b]: a before b
  final next = List.generate(n, (_) => <int>[]);
  final indegree = List.filled(n, 0);
  for (final [a, b] in edges) {
    next[a].add(b);
    indegree[b]++;
  }
  final q = Queue<int>()..addAll([for (var i = 0; i < n; i++) if (indegree[i] == 0) i]);
  final order = <int>[];
  while (q.isNotEmpty) {
    final x = q.removeFirst();
    order.add(x);
    for (final y in next[x]) {
      if (--indegree[y] == 0) q.add(y);
    }
  }
  return order.length == n ? order : null;               // null => cycle
}
```

Real use: build systems, package managers (`pub get` resolving dependencies!).

### Dijkstra (weighted shortest path) — concept

BFS + a **min-heap** that always expands the cheapest known node. `O((V + E) log V)`. Requires non-negative weights.

```dart
import 'package:collection/collection.dart';

Map<int, int> dijkstra(Map<int, List<(int, int)>> g, int src) {   // node -> [(neighbor, weight)]
  final dist = <int, int>{src: 0};
  final heap = PriorityQueue<(int, int)>((a, b) => a.$1.compareTo(b.$1))
    ..add((0, src));
  while (heap.isNotEmpty) {
    final (d, node) = heap.removeFirst();
    if (d > (dist[node] ?? 1 << 62)) continue;          // stale entry
    for (final (next, w) in g[node] ?? const <(int, int)>[]) {
      final nd = d + w;
      if (nd < (dist[next] ?? 1 << 62)) {
        dist[next] = nd;
        heap.add((nd, next));
      }
    }
  }
  return dist;
}
```

### Union-Find (Disjoint Set) — "are these connected?"

```dart
class UnionFind {
  UnionFind(int n) : _parent = List.generate(n, (i) => i), _rank = List.filled(n, 0);
  final List<int> _parent;
  final List<int> _rank;

  int find(int x) => _parent[x] == x ? x : _parent[x] = find(_parent[x]);   // path compression

  bool union(int a, int b) {
    final ra = find(a), rb = find(b);
    if (ra == rb) return false;                      // already connected
    if (_rank[ra] < _rank[rb]) {
      _parent[ra] = rb;
    } else if (_rank[ra] > _rank[rb]) {
      _parent[rb] = ra;
    } else {
      _parent[rb] = ra;
      _rank[ra]++;
    }
    return true;
  }
}
```

Near `O(1)` per operation. Used for connected components, cycle detection, Kruskal's MST.

---

## 6. DFS vs BFS

| | DFS | BFS |
|--|-----|-----|
| Data structure | Stack (or recursion) | Queue |
| Explores | Deep first | Level by level |
| Finds shortest path (unweighted) | No | **Yes** |
| Memory | Depth | Width of widest level |
| Good for | Paths, cycles, components, backtracking, trees | Shortest path, levels, nearest |

---

## Common mistakes

- Forgetting `visited` in graph search → infinite loop.
- Marking visited **when popped** instead of **when pushed** in BFS → duplicates/slowdown.
- Checking BST validity locally.
- Not handling `null` (empty) trees.
- Recursion too deep on a skewed tree (stack overflow) → use iteration.

---

## Code challenge

1. `int maxDepth(TreeNode? root)` (iterative with BFS).
2. `TreeNode? lowestCommonAncestor(TreeNode? root, int p, int q)` for a **BST**, then for a general tree.
3. `bool canFinish(int numCourses, List<List<int>> prerequisites)` — can all courses be taken? (cycle detection).
4. Rotting oranges: grid with `0` empty, `1` fresh, `2` rotten. Each minute rot spreads to 4 neighbors. Minutes until none fresh, or `-1`. (Multi-source BFS.)
5. Clone a graph / detect a cycle in an undirected graph with Union-Find.

<details>
<summary>Solution (1–3)</summary>

```dart
int maxDepth(TreeNode? root) {
  if (root == null) return 0;
  var depth = 0;
  final q = Queue<TreeNode>()..add(root);
  while (q.isNotEmpty) {
    depth++;
    for (var i = q.length; i > 0; i--) {
      final n = q.removeFirst();
      if (n.left != null) q.add(n.left!);
      if (n.right != null) q.add(n.right!);
    }
  }
  return depth;
}

// BST: split point
TreeNode? lcaBST(TreeNode? root, int p, int q) {
  var n = root;
  while (n != null) {
    if (p < n.value && q < n.value) {
      n = n.left;
    } else if (p > n.value && q > n.value) {
      n = n.right;
    } else {
      return n;                      // paths diverge here
    }
  }
  return null;
}

// General tree
TreeNode? lca(TreeNode? n, int p, int q) {
  if (n == null || n.value == p || n.value == q) return n;
  final l = lca(n.left, p, q), r = lca(n.right, p, q);
  if (l != null && r != null) return n;      // p and q are on different sides
  return l ?? r;
}

bool canFinish(int n, List<List<int>> prereqs) =>
    topoSort(n, [for (final [a, b] in prereqs) [b, a]]) != null;
```

Multi-source BFS (#4): push **all** rotten oranges into the queue at minute 0, then BFS by levels; each level = 1 minute.
</details>

---

## Interview questions

1. Explain preorder, inorder, postorder. Which gives sorted output for a BST?
2. DFS vs BFS: when do you use each?
3. Why does BFS find the shortest path in an unweighted graph?
4. What is the time complexity of BFS/DFS on an adjacency list?
5. How do you detect a cycle in a directed graph? In an undirected graph?
6. What is topological sorting and where is it used in real software?
7. When does a BST degrade to `O(n)`, and how do balanced trees fix it?
8. What does Union-Find do and what are its two optimizations?

<details>
<summary>Hints</summary>

1. Node position relative to children: before / between / after. Inorder is sorted.
2. DFS for exhaustive exploration; BFS for shortest path or levels.
3. It visits nodes in order of distance from the start.
4. `O(V + E)`.
5. Directed: DFS with recursion-stack coloring, or topo sort failing. Undirected: DFS with parent tracking, or Union-Find.
6. Ordering with dependencies: build systems, task schedulers, `pub get`.
7. Inserting sorted data creates a chain; rotations (AVL / Red-Black) keep height `O(log n)`.
8. Tracks connected groups; path compression and union by rank.
</details>
