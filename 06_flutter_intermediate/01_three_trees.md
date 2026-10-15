# 01 — The Three Trees

## Goal

Understand **how Flutter really works inside**. This is the most common "senior" Flutter interview topic, and it explains keys, performance, and `setState`.

---

## 1. The three trees

When you write widgets, Flutter builds three parallel trees:

| Tree | Class | Role | Created | Lifetime |
|------|-------|------|---------|----------|
| **Widget tree** | `Widget` | Immutable configuration ("what do you want?") | Every `build` | Very short |
| **Element tree** | `Element` | The **instance** in the tree; links widget ↔ render object; owns `State` | When first inserted | Long (reused) |
| **Render tree** | `RenderObject` | **Layout, painting, hit-testing** | By elements | Long (reused) |

```
 Widget tree           Element tree            RenderObject tree
 (blueprints)          (live instances)        (does the work)

 Padding  ───────────► SingleChildRenderObjectElement ──► RenderPadding
   │                         │                                │
 Text     ───────────► LeafRenderObjectElement ─────────► RenderParagraph
```

Not every widget has a render object. `StatelessWidget` and `StatefulWidget` are **composition**: their elements just call `build` and manage children. Only `RenderObjectWidget`s (like `Padding`, `Text`, `Row`) create render objects.

### Analogy

- **Widget** = an architectural drawing (cheap, throw away).
- **Element** = the construction manager (remembers which drawing is currently built where).
- **RenderObject** = the actual building (expensive; you don't rebuild it if only the paint color changes).

---

## 2. What happens on a rebuild?

`setState` marks an **Element dirty**. On the next frame:

1. Flutter calls `build()` → creates a **new widget** (cheap).
2. For each child slot, the element asks: **"Can I reuse my current element for this new widget?"**

   `Widget.canUpdate(old, new)`:
   ```dart
   static bool canUpdate(Widget oldWidget, Widget newWidget) {
     return oldWidget.runtimeType == newWidget.runtimeType &&
         oldWidget.key == newWidget.key;
   }
   ```

3. - **Same type and same key** → **update**: keep the element (and its `State` and `RenderObject`), give it the new widget configuration. Render object properties update only if changed.
   - **Different** → throw the old element away, **create** a new one (State lost!).

That is why rebuilding the widget tree is cheap: heavy objects are **reused**.

### Consequence: State lives in the Element

The `State` object is attached to the element. Replacing a widget with another of the same type & key keeps the state. Changing type or key **destroys** the state.

---

## 3. Why keys exist

Without keys, matching is by **position and type**. Reordering stateful items breaks:

```dart
// Two colored boxes with state; swap them in a Row
Row(children: [Box(color: red), Box(color: blue)])
→ after swap
Row(children: [Box(color: blue), Box(color: red)])
```

If each `Box` is stateful (holding, say, a counter), Flutter matches by **position**: element 1 gets the new widget #1 (blue config) but **keeps its old State** (red box's counter). The counters stay in place; colors move. Bug!

With keys: `Box(key: ValueKey('red'))` → element matched by key, so State follows the item.

See next lesson for key types.

---

## 4. The frame pipeline

For each frame (16 ms @ 60 Hz):

```
1. Animate / ticker callbacks
2. Build        — dirty elements call build()
3. Layout       — constraints down, sizes up (RenderObjects)
4. Paint        — render objects record drawing commands (layers)
5. Composite    — layers sent to the GPU thread
6. Rasterize    — GPU draws pixels (engine, Impeller)
```

Two main threads: the **UI thread** (Dart: build/layout/paint) and the **raster thread** (GPU work). Jank can come from either.

### Why `const` widgets are fast

A `const` widget instance is **identical** to the previous one. During update, Flutter checks `identical(oldWidget, newWidget)` and **skips rebuilding that subtree** entirely.

---

## 5. Types of widgets and elements

| Widget kind | Examples | Element |
|-------------|----------|---------|
| `StatelessWidget` | `Text` (composes), your widgets | `StatelessElement` |
| `StatefulWidget` | `Checkbox`, your widgets | `StatefulElement` (holds `State`) |
| `InheritedWidget` | `Theme`, `MediaQuery`, `Provider` | `InheritedElement` (tracks dependents) |
| `RenderObjectWidget` | `Padding`, `Row`, `Opacity` | `RenderObjectElement` |
| `ProxyWidget` | `Expanded`, `Positioned` (set ParentData) | `ParentDataElement` |

---

## 6. InheritedWidget — how data flows down efficiently

```dart
class CounterScope extends InheritedWidget {
  const CounterScope({super.key, required this.count, required super.child});
  final int count;

  static CounterScope of(BuildContext context) =>
      context.dependOnInheritedWidgetOfExactType<CounterScope>()!;

  @override
  bool updateShouldNotify(CounterScope old) => count != old.count;
}
```

- `dependOnInheritedWidgetOfExactType` registers the caller as a **dependent**. When `updateShouldNotify` returns true, **only dependents rebuild**.
- Lookup is `O(1)` (each element keeps a map of inherited widgets above it).
- `Theme.of(context)`, `MediaQuery.of(context)`, `Provider`, `Riverpod`'s scope — all based on this.

Lookup without subscribing: `context.getInheritedWidgetOfExactType<T>()`.

---

## 7. BuildContext is the Element

```dart
abstract class Element implements BuildContext { ... }
```

The `context` passed to `build` **is** that widget's element. `Theme.of(context)` walks up **element** ancestors. This explains:

- Why a context from *above* a widget can't find that widget.
- Why using a context after the element is **unmounted** is a bug.

---

## 8. RenderObject basics

```dart
class RenderPadding extends RenderShiftedBox {
  @override
  void performLayout() {
    child!.layout(constraints.deflate(padding), parentUsesSize: true); // constraints down
    size = constraints.constrain(child!.size + padding);               // size up
    // position child via parentData
  }

  @override
  void paint(PaintingContext context, Offset offset) { ... }
}
```

- Layout is **constraints-down, sizes-up**.
- `markNeedsLayout()` / `markNeedsPaint()` cause only the necessary work.
- **Relayout boundaries** and **repaint boundaries** limit how far changes propagate. `RepaintBoundary` makes a subtree paint into its own layer so changes elsewhere don't repaint it.

You rarely write RenderObjects, but you must know that they exist and why rebuilds are cheap.

---

## 9. Practical implications (cheat sheet)

| Fact | What it means for you |
|------|-----------------------|
| Widgets are cheap | Don't fear rebuilding them; fear **expensive work inside** `build` |
| Elements/State are reused when type+key match | Use keys when order/identity changes |
| `const` stops rebuild at that node | Use `const` everywhere possible |
| Rebuilds start at the dirty element and go down | Keep `setState` low in the tree |
| `InheritedWidget` rebuilds only dependents | State-management packages use it for efficiency |
| `build` can run at any time (every frame in animations) | No side effects/IO in `build` |
| `RenderObject`s are expensive | Avoid needless layout changes (animate transforms) |

---

## Common mistakes

- Believing widgets are "heavy" and trying to cache them manually.
- Doing network calls or creating controllers in `build`.
- Using `Container` + `GlobalKey` hacks instead of understanding keys.
- Putting `setState` at the top of the tree.
- Confusing "the widget is rebuilt" with "the screen is repainted". Different stages!

---

## Code challenge

1. Create two stateful boxes each with its own tap counter. Add a button that swaps their order in a `Row`. Observe the bug. Fix it with `ValueKey`s.
2. Print inside `build` of a child to observe rebuilds: make the child `const` and see the print disappear after a parent `setState`.
3. Write an `InheritedWidget` named `UserScope` exposing a `User`, and two widgets: one depends on it (rebuilds on change), one doesn't (use `getInheritedWidgetOfExactType`). Prove the difference with print statements.
4. In DevTools, turn on **"Highlight repaints"** and see which parts repaint when an animation runs. Wrap the static part in a `RepaintBoundary`.

<details>
<summary>Hint — #1</summary>

```dart
Row(children: [
  for (final c in colors) CounterBox(key: ValueKey(c), color: c),
])
```
Reordering `colors` now moves the **elements with their state**.
</details>

---

## Interview questions

1. Name Flutter's three trees and each one's job.
2. What does Flutter check to decide to reuse or replace an element?
3. Where does the `State` object live? When is it lost?
4. Why are rebuilds cheap in Flutter?
5. Explain how `const` widgets improve performance.
6. How does `InheritedWidget` notify dependents? Why is `Theme.of(context)` efficient?
7. What is `BuildContext`, really?
8. What are the phases of a frame?
9. What does `RepaintBoundary` do?
10. Why do you need keys when reordering a list of stateful widgets?

<details>
<summary>Hints</summary>

1. Widget (config), Element (instance/glue), RenderObject (layout/paint).
2. `runtimeType` and `key` equal (`Widget.canUpdate`).
3. In the Element (`StatefulElement`); lost when the element is removed/replaced (type or key mismatch).
4. Widgets are light descriptions; elements and render objects are reused.
5. Identical instance → subtree skipped.
6. Dependents are registered via `dependOn...`; `updateShouldNotify` decides; map lookup O(1).
7. The Element for that widget.
8. Animation/ticker, build, layout, paint, composite, raster.
9. Isolates a subtree into its own paint layer to limit repaint scope.
10. Matching by position would attach states to wrong items; keys match by identity.
</details>
