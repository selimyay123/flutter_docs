# 09 — Gestures and Animations

## Goal

Make apps feel alive. Detect touches, then animate with confidence — from the easy way to the full control way.

---

## 1. Gestures

### Tap and friends

```dart
GestureDetector(
  onTap: () => print('tap'),
  onDoubleTap: () => print('double'),
  onLongPress: () => print('long'),
  child: const Text('Tap me'),
)
```

Drag and scale:

```dart
GestureDetector(
  onPanUpdate: (details) => setState(() => _offset += details.delta),
  onScaleUpdate: (d) => setState(() => _scale = d.scale),
  child: ...,
)
```

### InkWell (tap with ripple effect)

```dart
InkWell(
  borderRadius: BorderRadius.circular(12),
  onTap: () {},
  child: const Padding(padding: EdgeInsets.all(16), child: Text('Ripple')),
)
```

`InkWell` needs a `Material` ancestor (e.g. `Card`, `Material`, `Scaffold` body).

Prefer real buttons (`IconButton`, `TextButton`, ...) when something is a button: they bring accessibility, focus, and tap-target size for free.

### How gestures are decided

Flutter has a **gesture arena**: when multiple detectors compete (a list scroll vs. a tap), one wins. This is why a horizontal drag inside a vertical list can behave unexpectedly.

---

## 2. Three levels of animation

| Level | Tool | Control | Effort |
|-------|------|---------|--------|
| 1. Implicit | `AnimatedContainer`, `AnimatedOpacity`, `TweenAnimationBuilder` | Low | Tiny |
| 2. Explicit | `AnimationController` + `AnimatedBuilder` | High | Medium |
| 3. Physics/Custom | `SpringSimulation`, `CustomPainter`, Rive/Lottie | Total | High |

**Start at level 1. Move up only when you need to.**

---

## 3. Implicit animations — just change the value

```dart
class _BoxState extends State<Box> {
  bool _big = false;

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: () => setState(() => _big = !_big),
      child: AnimatedContainer(
        duration: const Duration(milliseconds: 300),
        curve: Curves.easeInOut,
        width: _big ? 200 : 100,
        height: _big ? 200 : 100,
        decoration: BoxDecoration(
          color: _big ? Colors.orange : Colors.indigo,
          borderRadius: BorderRadius.circular(_big ? 40 : 8),
        ),
      ),
    );
  }
}
```

You change values with `setState`; Flutter **interpolates** between old and new automatically.

More: `AnimatedOpacity`, `AnimatedAlign`, `AnimatedPadding`, `AnimatedPositioned`, `AnimatedSwitcher` (cross-fades when its child changes), `AnimatedCrossFade`, `AnimatedSize`, `AnimatedScale`, `AnimatedRotation`.

### TweenAnimationBuilder (any value)

```dart
TweenAnimationBuilder<double>(
  tween: Tween(begin: 0, end: progress),
  duration: const Duration(milliseconds: 600),
  builder: (context, value, child) => LinearProgressIndicator(value: value),
)
```

### AnimatedSwitcher

```dart
AnimatedSwitcher(
  duration: const Duration(milliseconds: 250),
  child: Text('$count', key: ValueKey(count)),   // key change triggers the animation
)
```

---

## 4. Explicit animations — you hold the controller

```dart
class PulseLogo extends StatefulWidget {
  const PulseLogo({super.key});
  @override
  State<PulseLogo> createState() => _PulseLogoState();
}

class _PulseLogoState extends State<PulseLogo> with SingleTickerProviderStateMixin {
  late final AnimationController _controller;
  late final Animation<double> _scale;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(
      vsync: this,                                   // `this` is the TickerProvider
      duration: const Duration(seconds: 1),
    )..repeat(reverse: true);

    _scale = Tween<double>(begin: 0.9, end: 1.1).animate(
      CurvedAnimation(parent: _controller, curve: Curves.easeInOut),
    );
  }

  @override
  void dispose() {
    _controller.dispose();                           // never forget
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return ScaleTransition(
      scale: _scale,
      child: const FlutterLogo(size: 100),
    );
  }
}
```

Key pieces:

- **`AnimationController`**: produces a value 0.0 → 1.0 over time, driven by the **Ticker** (once per frame). Methods: `forward()`, `reverse()`, `repeat()`, `stop()`, `reset()`, `animateTo()`.
- **`vsync` / `TickerProvider`**: syncs with the screen refresh and pauses when off-screen to save battery. Use `SingleTickerProviderStateMixin` (one controller) or `TickerProviderStateMixin` (several).
- **`Tween`**: maps 0..1 to any range (`Color`, `Offset`, `double`).
- **`Curve`**: changes the rhythm (`easeIn`, `bounceOut`, `elasticOut`...).
- **`*Transition` widgets** (`FadeTransition`, `SlideTransition`, `ScaleTransition`, `RotationTransition`) rebuild **only the necessary part**.

### AnimatedBuilder (custom animation, rebuild only a subtree)

```dart
AnimatedBuilder(
  animation: _controller,
  child: const Icon(Icons.star, size: 48),            // built once, passed in
  builder: (context, child) {
    return Transform.rotate(angle: _controller.value * 2 * math.pi, child: child);
  },
)
```

Passing the static `child` avoids rebuilding it 60 times per second.

### Staggered animations

Use `Interval` curves so parts animate at different times with one controller:

```dart
final fade = CurvedAnimation(parent: _controller, curve: const Interval(0.0, 0.5));
final slide = CurvedAnimation(parent: _controller, curve: const Interval(0.3, 1.0, curve: Curves.easeOut));
```

---

## 5. Hero & page transitions

See navigation lesson. `Hero(tag: ...)` flies a widget between routes.

---

## 6. Performance of animations

- Animate **transform / opacity**, not layout, when possible (`Transform`, `FadeTransition` are cheap; `AnimatedContainer` changing size relayouts).
- Use `AnimatedBuilder`'s `child` parameter.
- Wrap heavy static content in `RepaintBoundary` so it isn't repainted each frame.
- Use `const` for static pieces.
- Check performance in **profile mode** with DevTools (not debug!).
- Target: stay under 16 ms per frame (8 ms on 120 Hz devices).

---

## 7. Packages worth knowing

| Package | Use |
|---------|-----|
| `flutter_animate` | Chainable declarative effects |
| `lottie` | After Effects animations (JSON) |
| `rive` | Interactive vector animations |
| `animations` | Material motion patterns (container transform, shared axis) |

---

## 8. Motion design principles

- Motion should **explain** (where something came from/went), not decorate.
- 150–400 ms for most transitions. Faster for small elements.
- Ease-out for entering, ease-in for leaving.
- Respect accessibility: check `MediaQuery.disableAnimationsOf(context)` and reduce motion.

---

## Common mistakes

- Forgetting `dispose()` on controllers (leaks, tickers errors).
- Calling `setState` every frame to animate (use `AnimatedBuilder`).
- Using `AnimatedContainer` where a cheap `Transform` is enough.
- Animations tested only in debug mode.
- Long animations that block users from acting.

---

## Code challenge

1. **Like button:** tap toggles a heart; it scales up with `Curves.elasticOut` and changes color (implicit or explicit).
2. **Expandable card:** tap to expand/collapse details using `AnimatedSize` or `AnimatedCrossFade`.
3. **Loading dots:** three dots pulsing one after another with a **staggered** explicit animation.
4. **Draggable box** that snaps back to the center with an animation when released.

<details>
<summary>Hint — snap back</summary>

Use an `AnimationController` + `Tween<Offset>` from the dragged position to `Offset.zero` in `onPanEnd`; call `controller.forward(from: 0)` and update the position in a listener / `AnimatedBuilder`.
</details>

---

## Interview questions

1. Implicit vs explicit animations: when to choose each?
2. What does a `Ticker` do? What is `vsync` for?
3. Why is `AnimatedBuilder`'s `child` parameter important?
4. What does `dispose()` need to do for animations?
5. How would you animate several properties in sequence with one controller?
6. How do you keep animations at 60/120 FPS?
7. What is the gesture arena?

<details>
<summary>Hints</summary>

1. Implicit for simple property changes; explicit for repeating, sequencing, or interactive control.
2. Calls back each frame; `vsync` ties the controller to screen refresh and pauses off-screen.
3. Builds static subtree once so it isn't rebuilt every frame.
4. Dispose the controller (stops the ticker).
5. `Interval` curves on one controller.
6. Animate transforms/opacity, `RepaintBoundary`, `const`, profile mode.
7. The system that resolves which gesture recognizer wins when several compete.
</details>
