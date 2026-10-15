# 04 — Custom Painting

## Goal

Draw anything yourself: charts, gauges, signatures, games. Understand the `Canvas` and `CustomPainter`.

---

## 1. Why custom painting?

When no widget draws what you need: chart, progress ring, drawing pad, custom shape, particles. Painting is also how Flutter itself draws every widget (via `RenderObject.paint`), so learning this demystifies the framework.

---

## 2. CustomPaint and CustomPainter

```dart
class CirclePainter extends CustomPainter {
  const CirclePainter({required this.progress, required this.color});
  final double progress;       // 0..1
  final Color color;

  @override
  void paint(Canvas canvas, Size size) {
    final center = size.center(Offset.zero);
    final radius = size.shortestSide / 2 - 8;

    final track = Paint()
      ..color = color.withOpacity(0.2)
      ..style = PaintingStyle.stroke
      ..strokeWidth = 12;

    final arc = Paint()
      ..color = color
      ..style = PaintingStyle.stroke
      ..strokeWidth = 12
      ..strokeCap = StrokeCap.round;

    canvas.drawCircle(center, radius, track);
    canvas.drawArc(
      Rect.fromCircle(center: center, radius: radius),
      -math.pi / 2,                    // start at the top
      2 * math.pi * progress,          // sweep angle
      false,
      arc,
    );
  }

  @override
  bool shouldRepaint(CirclePainter old) =>
      old.progress != progress || old.color != color;
}

// usage
SizedBox(
  width: 120,
  height: 120,
  child: CustomPaint(painter: CirclePainter(progress: 0.7, color: Colors.teal)),
)
```

Key facts:

- `paint(canvas, size)`: draw within `size`. (`size` comes from layout: constraints → `CustomPaint`'s `size`, or its child's size.)
- `shouldRepaint`: return `true` only if inputs changed. **Critical for performance.**
- Use `import 'dart:math' as math;`
- `CustomPaint` sizes: gives the painter the given `size`; if no child, it uses `size:` param or 0×0 (so wrap it in `SizedBox`/`Expanded`).

---

## 3. Canvas and Paint

### Canvas methods

```dart
canvas.drawLine(p1, p2, paint);
canvas.drawRect(rect, paint);
canvas.drawRRect(RRect.fromRectAndRadius(rect, const Radius.circular(12)), paint);
canvas.drawCircle(center, r, paint);
canvas.drawOval(rect, paint);
canvas.drawArc(rect, start, sweep, useCenter, paint);
canvas.drawPath(path, paint);
canvas.drawImage(image, offset, paint);
canvas.drawShadow(path, color, elevation, transparentOccluder);
```

Transforms:

```dart
canvas.save();                 // push state
canvas.translate(dx, dy);
canvas.rotate(radians);
canvas.scale(sx, sy);
canvas.clipRect(rect);
// ... draw ...
canvas.restore();              // pop state
```

Always pair `save`/`restore`.

### Paint properties

```dart
final paint = Paint()
  ..color = Colors.red
  ..style = PaintingStyle.fill        // or stroke
  ..strokeWidth = 4
  ..strokeCap = StrokeCap.round
  ..strokeJoin = StrokeJoin.round
  ..isAntiAlias = true
  ..blendMode = BlendMode.srcOver
  ..shader = const LinearGradient(colors: [Colors.blue, Colors.purple])
      .createShader(Rect.fromLTWH(0, 0, size.width, size.height))
  ..maskFilter = const MaskFilter.blur(BlurStyle.normal, 8);
```

Create `Paint` objects once if possible (final fields), not on each paint call for hot paths.

---

## 4. Path — arbitrary shapes

```dart
final path = Path()
  ..moveTo(0, size.height)
  ..lineTo(size.width / 2, 0)
  ..lineTo(size.width, size.height)
  ..close();                           // triangle
canvas.drawPath(path, paint);

// Curves
path.quadraticBezierTo(cx, cy, x, y);
path.cubicTo(c1x, c1y, c2x, c2y, x, y);
path.arcToPoint(Offset(x, y), radius: const Radius.circular(10));
```

Use as clipper: `ClipPath(clipper: MyClipper(), child: ...)` to create wave headers, ticket shapes.

---

## 5. Example: a simple line chart

```dart
class LineChartPainter extends CustomPainter {
  LineChartPainter(this.values, {required this.color});
  final List<double> values;
  final Color color;

  @override
  void paint(Canvas canvas, Size size) {
    if (values.length < 2) return;
    final minV = values.reduce(math.min);
    final maxV = values.reduce(math.max);
    final range = (maxV - minV) == 0 ? 1 : (maxV - minV);

    Offset point(int i) => Offset(
          size.width * i / (values.length - 1),
          size.height - (values[i] - minV) / range * size.height,
        );

    final path = Path()..moveTo(point(0).dx, point(0).dy);
    for (var i = 1; i < values.length; i++) {
      path.lineTo(point(i).dx, point(i).dy);
    }

    canvas.drawPath(
      path,
      Paint()
        ..color = color
        ..style = PaintingStyle.stroke
        ..strokeWidth = 3
        ..strokeJoin = StrokeJoin.round,
    );

    for (var i = 0; i < values.length; i++) {
      canvas.drawCircle(point(i), 4, Paint()..color = color);
    }
  }

  @override
  bool shouldRepaint(LineChartPainter old) =>
      !listEquals(old.values, values) || old.color != color;
}
```

This mixes **math** (mapping data space → pixel space: linear interpolation) with drawing. That mapping `y = size.height - (v - min) / range * size.height` is a typical algorithmic step.

---

## 6. Animated painting

Pass an `Animation` as `repaint` so only painting reruns (no rebuild):

```dart
class WavePainter extends CustomPainter {
  WavePainter(this.animation) : super(repaint: animation);
  final Animation<double> animation;

  @override
  void paint(Canvas canvas, Size size) {
    final path = Path()..moveTo(0, size.height / 2);
    for (double x = 0; x <= size.width; x++) {
      final y = size.height / 2 +
          math.sin((x / size.width * 2 * math.pi) + animation.value * 2 * math.pi) * 20;
      path.lineTo(x, y);
    }
    canvas.drawPath(path, Paint()..color = Colors.blue..style = PaintingStyle.stroke..strokeWidth = 3);
  }

  @override
  bool shouldRepaint(WavePainter old) => false;     // repaint is driven by the listenable
}
```

With `super(repaint: animation)`, each animation tick **repaints without rebuilding the widget tree**. Efficient.

---

## 7. Interaction: a drawing pad

```dart
class _PadState extends State<Pad> {
  final List<List<Offset>> _strokes = [];

  @override
  Widget build(BuildContext context) {
    return GestureDetector(
      onPanStart: (d) => setState(() => _strokes.add([d.localPosition])),
      onPanUpdate: (d) => setState(() => _strokes.last.add(d.localPosition)),
      child: CustomPaint(painter: _PadPainter(_strokes), size: Size.infinite),
    );
  }
}

class _PadPainter extends CustomPainter {
  _PadPainter(this.strokes);
  final List<List<Offset>> strokes;

  @override
  void paint(Canvas canvas, Size size) {
    final paint = Paint()..color = Colors.black..strokeWidth = 3..style = PaintingStyle.stroke..strokeCap = StrokeCap.round;
    for (final s in strokes) {
      final path = Path()..moveTo(s.first.dx, s.first.dy);
      for (final p in s.skip(1)) { path.lineTo(p.dx, p.dy); }
      canvas.drawPath(path, paint);
    }
  }

  @override
  bool shouldRepaint(_PadPainter old) => true;     // list mutated in place
}
```

Better: keep strokes immutable and compare, or use a `ChangeNotifier` as the `repaint` listenable.

---

## 8. Hit testing

Override `hitTest` in the painter to define tappable regions:

```dart
@override
bool? hitTest(Offset position) => path.contains(position);
```

---

## 9. Performance tips

- Return `false` from `shouldRepaint` when nothing changed.
- Use `repaint:` listenable instead of `setState` for animation.
- Wrap in `RepaintBoundary` if siblings repaint often.
- Avoid allocating objects in `paint` (cache `Paint`, `Path`, `TextPainter`).
- Draw only what is visible.
- Avoid `saveLayer` and large blurs.
- For very large scenes: `Picture` caching (`PictureRecorder`) or `canvas.drawVertices`/`drawAtlas`.

### Text in a painter

```dart
final tp = TextPainter(
  text: const TextSpan(text: 'Hello', style: TextStyle(color: Colors.black, fontSize: 16)),
  textDirection: TextDirection.ltr,
)..layout();
tp.paint(canvas, const Offset(10, 10));
```

---

## 10. Shaders (preview)

Fragment shaders (GLSL, `.frag` files, `FragmentProgram`) enable GPU effects like ripples, gradients, and noise. Declare in `pubspec.yaml` under `flutter: shaders:` and use `Paint()..shader = program.fragmentShader()`. Advanced topic.

---

## Common mistakes

- `shouldRepaint` always `true` (wasted work) or always `false` (stale UI).
- Forgetting a size (`CustomPaint` is 0×0).
- Missing `save`/`restore` → transforms leak into later drawing.
- Allocating many objects per frame.
- Drawing outside the given `size` (use `clipRect` if needed).
- Mutating data in place and expecting `shouldRepaint` to detect it.

---

## Code challenge

1. **Gauge:** a half-circle speedometer with ticks and a needle driven by a value (animated).
2. **Bar chart** with labels (using `TextPainter`) and gridlines. Map data → pixels with a helper.
3. **Signature pad** with undo, clear, and exporting to PNG (hint: `PictureRecorder` → `Picture.toImage` → `toByteData`).
4. **Particles:** 200 dots moving and bouncing off the edges, driven by a `Ticker`. Keep 60 FPS — check in profile mode.
5. Use `ClipPath` with a custom clipper to build a wave-shaped header.

<details>
<summary>Hint — angle for a gauge</summary>

Map the value to an angle: `angle = startAngle + (value - min) / (max - min) * sweepAngle`. Then rotate the canvas by `angle` and draw the needle as a line from the center.
</details>

---

## Interview questions

1. What is `CustomPainter` and how does `CustomPaint` use it?
2. What does `shouldRepaint` do and why is it important?
3. How do you animate a painter without rebuilding widgets?
4. What are `Canvas.save` and `restore` for?
5. How does `CustomPaint` get its size?
6. How would you make a custom shape tappable?
7. How do you keep a complex painter fast?
8. What is the difference between a `Path` clip and a `Rect` clip in cost and quality?

<details>
<summary>Hints</summary>

1. A class with `paint(Canvas, Size)`; `CustomPaint` calls it during the paint phase.
2. Decides if repaint is necessary when the painter instance changes; avoids wasted work.
3. Pass the animation to `super(repaint: animation)`.
4. Save/restore canvas state (transforms, clips) to isolate changes.
5. From the parent's constraints/`size` parameter or its child.
6. Override `hitTest` using `Path.contains`.
7. Cache objects, minimal repaint scope, repaint boundaries, avoid layers.
8. Rect clips are cheap; path clips with anti-aliasing cost more.
</details>
