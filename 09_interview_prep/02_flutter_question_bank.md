# 02 — Flutter Senior Interview Question Bank

## Goal

Equip you to answer core architectural and internal Flutter questions with the clarity and depth expected of senior mobile engineers.

---

## 1. Flutter Engine & Rendering Pipeline

### Q: What is Impeller and why did Flutter transition away from Skia?
**Answer:**
Skia relies on runtime just-in-time shader compilation. On iOS and modern graphics APIs (Metal, Vulkan), compiling shaders at the moment a visual effect appears creates noticeable frame drops known as **shader compilation jank**.
**Impeller** solves this by:
1. Pre-compiling all shaders ahead of time (AOT) during build.
2. Utilizing modern low-level APIs (Metal on iOS, Vulkan on Android) directly.
3. Guaranteeing deterministic, smooth 60/120 FPS frame delivery.

### Q: Walk through the phases of a frame in Flutter.
1. **VSync Signal**: Display refresh trigger starts the frame.
2. **Animation / Tickers**: Active controllers advance their value.
3. **Build Phase**: Dirty `Element` instances re-invoke their `Widget.build()` methods.
4. **Layout Phase**: Constraints pass down, sizes pass up through `RenderObject`s.
5. **Compositing / Paint**: Render objects paint onto separate layer trees.
6. **Raster Phase**: The engine rasterizes layers on the GPU thread.

---

## 2. The Three Trees & Element Tree Mechanics

### Q: Why is `BuildContext` needed in Flutter? What is it under the hood?
**Answer:**
`BuildContext` is an interface implemented by the `Element` itself. It represents the widget's location in the element tree. 
It is required because:
- Widgets are immutable configurations with no reference to parents or children.
- The `Element` tracks ancestor relationships, enabling `Theme.of(context)`, `MediaQuery.of(context)`, and state preservation across rebuilds.

### Q: When is an Element updated vs destroyed and replaced?
**Answer:**
Flutter evaluates `Widget.canUpdate(oldWidget, newWidget)`:
```dart
static bool canUpdate(Widget oldWidget, Widget newWidget) {
  return oldWidget.runtimeType == newWidget.runtimeType
      && oldWidget.key == newWidget.key;
}
```
- If **both** `runtimeType` and `key` match: The existing `Element` is retained, assigned the new widget configuration, and updated.
- If **either** differs: The old `Element` (and its attached `State` and `RenderObject`) is unmounted and discarded, and a new element is instantiated.

---

## 3. Constraints & Layout Pitfalls

### Q: Why does `Container(width: 200, height: 200)` take the entire screen if it is the root widget?
**Answer:**
The layout rule dictates: **Constraints go down, sizes go up.**
The root of the app receives **tight constraints** matching the exact physical screen dimensions (e.g., width: [screen_width, screen_width]). When the child `Container` requests 200x200, the tight parent constraint overrides it, forcing it to fill the entire screen.
**Fix**: Introduce a widget that loosens constraints (e.g., `Center`, `Align`, or `UnconstrainedBox`).

### Q: Explain the cause and fix of "A RenderFlex overflowed by X pixels".
**Answer:**
A `Row` or `Column` (`RenderFlex`) calculates the total non-flexible children's size along its main axis. If the sum exceeds the incoming bounded constraint, it overflows.
**Fixes**:
1. Wrap the expanding child in `Expanded` or `Flexible`.
2. Wrap the `Row` in a `Wrap` widget to allow line breaks.
3. Wrap the content in a scroll view (`SingleChildScrollView`).

---

## 4. State Architecture & Performance

### Q: How do you identify whether a performance issue is on the UI thread or Raster thread?
**Answer:**
In DevTools Performance view:
- **UI Thread Spikes (Blue Bars)**: Excessive Dart execution time. Caused by heavy computations in `build()`, non-lazy list building, or complex synchronous parsing. Fix: Move work to `Isolate.run()`, use `const`, split widgets.
- **Raster Thread Spikes (Green Bars)**: Excessive GPU rendering time. Caused by heavy layer composition (`saveLayer`, `BackdropFilter`, excessive shadows, nested clipping). Fix: Reduce off-screen rendering, use `RepaintBoundary`.

---

## Code Challenge

Create a custom `StatelessWidget` named `DebouncedTapDetector` that:
1. Accepts a `VoidCallback onTap` and a `Duration duration` (default 500ms).
2. Prevents multiple rapid clicks (double taps / spamming) by ignoring taps within the duration window.
3. Does not use external packages.

<details>
<summary>Solution</summary>

```dart
import 'package:flutter/material.dart';

class DebouncedTapDetector extends StatefulWidget {
  const DebouncedTapDetector({
    super.key,
    required this.child,
    required this.onTap,
    this.duration = const Duration(milliseconds: 500),
  });

  final Widget child;
  final VoidCallback onTap;
  final Duration duration;

  @override
  State<DebouncedTapDetector> createState() => _DebouncedTapDetectorState();
}

class _DebouncedTapDetectorState extends State<DebouncedTapDetector> {
  DateTime? _lastTapTime;

  void _handleTap() {
    final now = DateTime.now();
    if (_lastTapTime == null || now.difference(_lastTapTime!) > widget.duration) {
      _lastTapTime = now;
      widget.onTap();
    }
  }

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _handleTap,
      child: widget.child,
    );
  }
}
```
</details>

---

## Rapid-Fire Interview Questions

1. What is the difference between `Navigator.push` and `Navigator.pushReplacement`?
2. Why should `ScrollController` and `AnimationController` be disposed in `dispose()`?
3. What is the purpose of `InheritedWidget.updateShouldNotify()`?
4. When should you choose `RepaintBoundary`?
5. Why is `shrinkWrap: true` on a large `ListView` an anti-pattern?

<details>
<summary>Hints & Answers</summary>

1. `push` adds a route to the stack; `pushReplacement` swaps the top route, preventing back navigation to the previous screen.
2. Failure to dispose keeps tickers and event listeners active, causing memory leaks and crashes when attempting to update unmounted elements.
3. It returns a boolean indicating whether dependent widgets that subscribed to this inherited widget need to be scheduled for a rebuild.
4. When an animated or frequently repainting widget sits alongside static content; isolating it prevents re-painting the entire tree.
5. `shrinkWrap: true` forces the list to measure every item eagerly to calculate its total height, completely nullifying virtualization benefits.
</details>
