# 01 — Performance

## Goal

Find and fix slow spots using **measurement**, not guessing.

---

## 1. Golden rules

1. **Measure first.** Profile, then optimize the real bottleneck.
2. **Profile mode only.** Debug mode is slow on purpose (`flutter run --profile` on a **real device**). Never judge performance from debug or the simulator.
3. Make it **correct → clear → fast**, in that order.

---

## 2. What "fast" means

| Metric | Target |
|--------|--------|
| Frame build + raster time | < 16 ms (60 Hz) / < 8 ms (120 Hz) |
| Startup (cold start to first frame) | As low as possible (< 2 s feels good) |
| Memory | Stable; no growth over time (leaks) |
| App size | Reasonable download |
| Jank | No dropped frames while scrolling/animating |

Two threads matter: **UI thread** (Dart: build/layout/paint) and **raster thread** (GPU). A frame is dropped if either is late.

---

## 3. DevTools — your instrument panel

Run: `flutter run --profile`, then open DevTools.

| Tab | Use |
|-----|-----|
| **Performance** | Frame chart (blue = UI, green = raster); red bars = janky frames. Timeline events. |
| **CPU Profiler** | Which Dart functions take the time (flame chart) |
| **Memory** | Heap usage, allocation tracking, leaks, snapshots |
| **Widget Inspector** | Tree, "Track widget rebuilds" |
| **Network** | Request timing |
| **App Size** | What bloats the bundle |

Overlays: `showPerformanceOverlay: true` in `MaterialApp`, or the inspector toggles ("Highlight repaints", "Track widget builds").

### Reading the frame chart

- **UI bar tall** → too much Dart work: expensive `build`, layout, or logic. Fix code.
- **Raster bar tall** → too much GPU work: heavy effects (`saveLayer`, big blur, clips, opacity on large trees, huge images). Simplify visuals.

---

## 4. Build-phase optimizations

### Keep `build` cheap and pure

- No network, no sorting a big list, no JSON parsing, no regex compiling inside `build`.
- Precompute outside; cache in state.

### Rebuild less

```dart
// 1. const
const Padding(padding: EdgeInsets.all(8), child: Text('Static'))

// 2. Smaller widgets: only the part that changes is rebuilt
// 3. Move setState/listeners down the tree
// 4. Narrow subscriptions
final name = ref.watch(userProvider.select((u) => u.name));
final count = context.select((Cart c) => c.items.length);
// 5. Pass `child` into builders
AnimatedBuilder(animation: a, child: const HeavyStatic(), builder: (_, child) => Transform.scale(scale: a.value, child: child));
```

### Avoid helper methods that return big widget trees

A `Widget _buildHeader()` rebuilds every time the parent does. A `const HeaderWidget()` class doesn't.

---

## 5. Lists and scrolling

- `ListView.builder` / `GridView.builder` / slivers; never build all children eagerly.
- `itemExtent` / `prototypeItem` for uniform rows.
- Avoid `shrinkWrap: true` on long lists, and nested scrollables in the same direction.
- Keep item widgets cheap; avoid `Opacity`, `ClipRRect`, shadows, and `BackdropFilter` per item when possible.
- Use keys for dynamic lists.
- Add `addAutomaticKeepAlives: false` if items don't need to persist.
- Paginate data.

---

## 6. Images

Images are the #1 source of memory problems.

```dart
Image.network(
  url,
  cacheWidth: 400,               // decode at display size, not full 4000 px
  fit: BoxFit.cover,
)
```

- Resize images on the server or use thumbnails for lists.
- Use `cached_network_image` (disk + memory cache).
- Precache hero images: `precacheImage(...)`.
- Use WebP/AVIF where possible; vector for icons.
- Free big images when leaving a screen.

---

## 7. Expensive painting & compositing

| Costly | Cheaper alternative |
|--------|---------------------|
| `Opacity` widget over big subtree | `FadeTransition`, `AnimatedOpacity`, or apply alpha to colors |
| `ClipRRect` + `saveLayer` (`Clip.antiAliasWithSaveLayer`) | Default `Clip.hardEdge`/`antiAlias`; use `borderRadius` in `BoxDecoration` |
| Large `BackdropFilter` / blur | Smaller area or static blurred image |
| Many shadows | Fewer or pre-rendered |
| Repainting static content with animations | `RepaintBoundary` |

Check `debugProfileBuildsEnabled`, `debugRepaintRainbowEnabled` to see repaints.

### RepaintBoundary

```dart
RepaintBoundary(child: ExpensiveStaticBackground())
```

Isolates a subtree into its own layer so animations elsewhere don't force it to repaint. Don't sprinkle everywhere: extra layers cost memory.

---

## 8. Heavy computation → isolates

```dart
final parsed = await Isolate.run(() => parseHugeJson(text));
```

Anything taking > ~8 ms on the UI isolate is a candidate: big JSON, image processing, encryption, sorting 100k items, regex over large text.

---

## 9. Startup time

- Do the minimum before `runApp`. Show the first frame ASAP.
- Defer non-critical init (analytics, remote config) until after the first frame.
- Lazy-load features (deferred imports on web, lazy providers).
- Avoid big synchronous work in `main()`.
- Native splash screen (`flutter_native_splash`) hides the delay but fix the cause.
- Measure: `flutter run --profile --trace-startup`.

---

## 10. Memory

Common leaks:

- Not cancelling `StreamSubscription`/`Timer`s.
- Not disposing controllers, focus nodes, animation controllers.
- Listeners added but never removed.
- Static/global references to widgets/contexts.
- Unbounded caches.
- Large images decoded at full size.

Detect: DevTools Memory → take snapshots before/after repeating an action (open/close a screen 10 times); diff the retained objects.

Use `dispose` religiously; use `WeakReference`/size-limited LRU caches.

---

## 11. App size

- `flutter build apk --analyze-size` / DevTools "App Size".
- `--split-per-abi`, App Bundles (`.aab`).
- Remove unused packages, assets, fonts; compress images.
- Tree-shaking of icons is automatic in release (avoid dynamic `IconData`).
- Obfuscation (`--obfuscate --split-debug-info`) reduces size slightly and hides symbols (keep symbols for crash reports!).

---

## 12. Performance checklist

- [ ] Tested in **profile** mode on a **low-end** physical device.
- [ ] No jank (< 16 ms frames) in main flows: scroll, navigation, animations.
- [ ] `const` used; widgets small; `setState` low.
- [ ] Lists are lazy; images are resized/cached.
- [ ] No heavy work in `build`.
- [ ] Heavy CPU work in isolates.
- [ ] No leaks: everything disposed.
- [ ] Startup measured; non-critical init deferred.

---

## Common mistakes

- Optimizing based on a debug run.
- Premature optimization.
- Caching everything (memory blowup).
- Overusing `RepaintBoundary`/`Opacity`.
- Ignoring the raster thread.
- Loading full-resolution images in lists.

---

## Code challenge

1. Make a screen with a `ListView` of 10,000 items where each item runs a deliberately slow `build` (e.g. computing primes). Profile it, find the hot spot in the CPU profiler, then fix it by caching/precomputing.
2. Make a list of network images at full size; measure memory; add `cacheWidth` and compare.
3. Use "Track widget rebuilds" to find an over-eager `setState`; fix by splitting widgets/using `const`.
4. Intentionally leak a `Timer` in a screen; show the leak in DevTools memory; fix it.
5. Move a heavy JSON parse into `Isolate.run` and compare frame charts.

<details>
<summary>Hint — #1 caching in item</summary>

Precompute results once when data is loaded (in your view model), not per item in `build`. Store the computed value inside the item model.
</details>

---

## Interview questions

1. How do you investigate a janky scroll?
2. Why test performance in profile mode on a real device?
3. UI thread vs raster thread: what causes a slowdown in each?
4. How does `const` help performance?
5. How do you reduce memory from images?
6. When do you use `RepaintBoundary`? When does it hurt?
7. How do you detect and fix a memory leak?
8. What would you move to an isolate?
9. How do you improve app startup time?
10. Why are `Opacity` and `saveLayer` expensive?

<details>
<summary>Hints</summary>

1. Profile mode → frame chart → UI or raster? CPU profiler → fix the hot function; check rebuilds, image sizes, effects.
2. Debug has asserts and JIT; simulators don't have real GPU/CPU characteristics.
3. UI: expensive build/layout/logic. Raster: heavy effects, clipping, blur, huge images.
4. Instances are reused; subtree is not rebuilt.
5. `cacheWidth/Height`, thumbnails, caching, formats, dispose.
6. Isolate frequent repaints of static content; too many layers cost memory.
7. Memory tab snapshots/diffs; look for undisposed controllers/subscriptions/listeners.
8. Large JSON, image processing, crypto, large sorts.
9. Minimal `main`, defer init, lazy load, measure.
10. They render a subtree to an offscreen buffer and composite it, using extra GPU memory/time.
</details>
