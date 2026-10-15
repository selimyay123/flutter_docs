# 08 — Responsive and Adaptive Design

## Goal

Build UIs that look right on phones, tablets, foldables, desktop, and web.

---

## 1. Responsive vs Adaptive

| | Responsive | Adaptive |
|--|-----------|----------|
| Meaning | Layout **flexes** with available space | App **changes** its UI/behavior per platform or device class |
| Example | Grid has more columns on wider screens | iOS-style dialogs on iOS; mouse hover and keyboard shortcuts on desktop |

Good apps do both.

---

## 2. Tools for measuring

```dart
// Device/window info
final size = MediaQuery.sizeOf(context);            // prefer sizeOf: rebuilds only on size change
final padding = MediaQuery.paddingOf(context);      // safe areas (notch, home indicator)
final textScale = MediaQuery.textScalerOf(context);
final orientation = MediaQuery.orientationOf(context);

// Space available to THIS widget
LayoutBuilder(builder: (context, constraints) { ... constraints.maxWidth ... })
```

Use `MediaQuery.xOf(context)` (e.g. `sizeOf`) instead of `MediaQuery.of(context).size` to avoid unnecessary rebuilds when unrelated properties (like keyboard insets) change.

### SafeArea

```dart
SafeArea(child: ...)       // avoid notches, status bars, home indicators
```

---

## 3. Breakpoints

A common set (Material guidance):

| Class | Width | Typical layout |
|-------|-------|----------------|
| Compact | < 600 | Phone: single column, bottom nav |
| Medium | 600–839 | Tablet/foldable: navigation rail, 2 panes |
| Expanded | ≥ 840 | Desktop/large tablet: drawer/rail + multi-pane |

```dart
enum ScreenSize { compact, medium, expanded }

ScreenSize screenSizeOf(double width) =>
    width < 600 ? ScreenSize.compact : width < 840 ? ScreenSize.medium : ScreenSize.expanded;
```

```dart
LayoutBuilder(
  builder: (context, c) => switch (screenSizeOf(c.maxWidth)) {
    ScreenSize.compact => const PhoneLayout(),
    ScreenSize.medium => const TabletLayout(),
    ScreenSize.expanded => const DesktopLayout(),
  },
)
```

---

## 4. Layout patterns

### 1) Adaptive navigation

```dart
Scaffold(
  body: Row(
    children: [
      if (width >= 600)
        NavigationRail(
          selectedIndex: index,
          onDestinationSelected: onSelect,
          labelType: NavigationRailLabelType.all,
          destinations: const [
            NavigationRailDestination(icon: Icon(Icons.home), label: Text('Home')),
            NavigationRailDestination(icon: Icon(Icons.settings), label: Text('Settings')),
          ],
        ),
      Expanded(child: pages[index]),
    ],
  ),
  bottomNavigationBar: width < 600
      ? NavigationBar(
          selectedIndex: index,
          onDestinationSelected: onSelect,
          destinations: const [
            NavigationDestination(icon: Icon(Icons.home), label: 'Home'),
            NavigationDestination(icon: Icon(Icons.settings), label: 'Settings'),
          ],
        )
      : null,
)
```

### 2) List–detail (master–detail)

- Compact: list → tap → push detail screen.
- Expanded: list on the left, detail on the right, no navigation.

### 3) Grid with flexible columns

```dart
GridView.builder(
  gridDelegate: const SliverGridDelegateWithMaxCrossAxisExtent(
    maxCrossAxisExtent: 220,            // as many columns as fit
    mainAxisSpacing: 12,
    crossAxisSpacing: 12,
  ),
  ...
)
```

### 4) Max content width

On wide screens, stretched text lines are unreadable. Constrain content (`ConstrainedBox(maxWidth: 720)` + `Center`).

### 5) Wrap & Flexible

`Wrap`, `Flexible`, `FittedBox`, `AspectRatio`, `FractionallySizedBox` handle proportional layouts.

---

## 5. Don't size with raw screen pixels

Bad: `width: MediaQuery.of(context).size.width * 0.3` everywhere.

Better:

- Let layout widgets distribute space (`Expanded`, `Flexible`).
- Use constraints (`LayoutBuilder`, `ConstrainedBox`).
- Use fixed sizes for fixed things (icons, paddings), flexible sizing for containers.
- Test with different **text scales** and **window sizes**.

---

## 6. Platform adaptation

```dart
import 'package:flutter/foundation.dart';

final isIOS = defaultTargetPlatform == TargetPlatform.iOS;
// kIsWeb tells you if running on web

Theme.of(context).platform
```

Adaptive widgets: `Switch.adaptive`, `Slider.adaptive`, `CircularProgressIndicator.adaptive`, `showAdaptiveDialog`, `CupertinoApp` for full iOS look.

Platform differences to respect:

| Aspect | Mobile | Desktop/Web |
|--------|--------|-------------|
| Input | Touch | Mouse hover, right-click, keyboard shortcuts, scroll wheel |
| Scrollbars | Hidden | Visible |
| Navigation | Back gesture | URL bar, browser back |
| Tap target | ≥ 48 dp | Can be smaller but still accessible |
| Text selection | Long-press | Mouse drag |

Add hover feedback (`InkWell`, `MouseRegion`), keyboard shortcuts (`Shortcuts`/`Actions`, `CallbackShortcuts`), focus traversal.

---

## 7. Orientation, foldables, and window changes

- Don't lock orientation unless needed.
- Foldables: check `MediaQuery.displayFeaturesOf(context)` / `DisplayFeature` hinges.
- Desktop windows can be resized continuously: layouts must not break at **any** width.

---

## 8. Images and assets

- Provide `2.0x`/`3.0x` assets (Flutter picks by device pixel ratio).
- Use `cacheWidth`/`cacheHeight` to decode large network images at the display size and save memory.
- Prefer vector (SVG via `flutter_svg`) for icons/illustrations.

---

## 9. Accessibility = adaptive too

- Large text: no fixed-height text boxes; use `Flexible`, scrolling.
- Screen readers: `Semantics`, meaningful labels.
- Color contrast, don't use color alone to convey meaning.
- Respect "reduce motion" and "bold text".
- Keyboard/focus support.

Test with TalkBack/VoiceOver and font scale 200%.

---

## 10. Testing responsive layouts

```dart
testWidgets('shows rail on wide screens', (tester) async {
  tester.view.physicalSize = const Size(1200, 800);
  tester.view.devicePixelRatio = 1.0;
  addTearDown(tester.view.reset);

  await tester.pumpWidget(const MaterialApp(home: HomeShell()));
  expect(find.byType(NavigationRail), findsOneWidget);
  expect(find.byType(NavigationBar), findsNothing);
});
```

Also use **golden tests** for visual regression at several sizes.

---

## Common mistakes

- Designing only for one phone.
- Hard-coded widths/heights.
- `MediaQuery.of(context)` in many places (excess rebuilds).
- Ignoring safe areas.
- Stretched layouts on tablets/desktop.
- Using device type (`isTablet`) rather than **available width**.
- Assuming touch only.

---

## Code challenge

Build an **email-like app shell**:

1. Compact: bottom `NavigationBar`, list → push detail.
2. Medium: `NavigationRail` + list.
3. Expanded: rail + list + detail panes side by side (selected email shown at right).
4. Content max width 720 px for the detail.
5. Add keyboard shortcuts (↑/↓ to move selection, Delete to remove) for desktop.
6. Write widget tests at 3 sizes.

<details>
<summary>Hint — responsive switch</summary>

Extract `bool isExpanded(BoxConstraints c) => c.maxWidth >= 840;` and use a single `LayoutBuilder` at the top of the shell. Keep the **selected email id** in state outside the layout so rotating/resizing keeps the selection.
</details>

---

## Interview questions

1. Responsive vs adaptive design?
2. `MediaQuery` vs `LayoutBuilder`: when each?
3. Why `MediaQuery.sizeOf(context)` instead of `MediaQuery.of(context).size`?
4. How do you design for tablets and desktop, not just phones?
5. What is a breakpoint? Give typical values.
6. How do you handle very large text sizes?
7. How do you test responsive behavior?
8. What are some desktop/web-specific UX considerations?

<details>
<summary>Hints</summary>

1. Responsive flexes; adaptive changes structure/behavior per platform/device.
2. `MediaQuery` for window/device; `LayoutBuilder` for the space given to a widget.
3. The specific accessor only rebuilds when that property changes.
4. Navigation rail, multi-pane, max content widths, hover/keyboard.
5. Width thresholds where layout changes: 600 / 840.
6. Flexible layouts, no fixed heights, scrolling, test at 200%.
7. Widget tests with custom view sizes, golden tests, manual resizing.
8. Mouse hover, shortcuts, scrollbars, focus, right-click menus, window resizing, URLs.
</details>
