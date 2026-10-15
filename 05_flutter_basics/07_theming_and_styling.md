# 07 — Theming and Styling

## Goal

Make apps look consistent and support light/dark mode without repeating colors and sizes everywhere.

---

## 1. Why themes?

Hard-coded styles (`Color(0xFF123456)` in 50 places) are painful to change. A **theme** is a central place that defines how your app looks. Widgets read from it.

---

## 2. ThemeData and Material 3

```dart
MaterialApp(
  theme: ThemeData(
    useMaterial3: true,
    colorSchemeSeed: const Color(0xFF6750A4),   // generates a full palette
    brightness: Brightness.light,
  ),
  darkTheme: ThemeData(
    useMaterial3: true,
    colorSchemeSeed: const Color(0xFF6750A4),
    brightness: Brightness.dark,
  ),
  themeMode: ThemeMode.system,                  // follow the OS
  home: const HomePage(),
)
```

A **seed color** generates a harmonious `ColorScheme` (primary, secondary, surface, error, and their "on" colors). Material 3 is the default in recent Flutter versions.

### Reading the theme

```dart
final theme = Theme.of(context);
final colors = theme.colorScheme;
final text = theme.textTheme;

Text('Title', style: text.titleLarge?.copyWith(color: colors.primary))
Container(color: colors.surfaceContainerHighest)
```

**Rule:** colors from `colorScheme`, text styles from `textTheme`. Then dark mode works automatically.

### The "on" colors

| Color | Content color on top of it |
|-------|---------------------------|
| `primary` | `onPrimary` |
| `secondary` | `onSecondary` |
| `surface` | `onSurface` |
| `error` | `onError` |

Always pair them to keep text readable.

---

## 3. Customize component themes

Override once; every widget picks it up.

```dart
ThemeData(
  useMaterial3: true,
  colorSchemeSeed: Colors.teal,
  appBarTheme: const AppBarTheme(centerTitle: true),
  cardTheme: CardTheme(
    elevation: 0,
    shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(16)),
  ),
  inputDecorationTheme: InputDecorationTheme(
    border: OutlineInputBorder(borderRadius: BorderRadius.circular(12)),
    filled: true,
  ),
  filledButtonTheme: FilledButtonThemeData(
    style: FilledButton.styleFrom(
      minimumSize: const Size.fromHeight(48),
      shape: RoundedRectangleBorder(borderRadius: BorderRadius.circular(12)),
    ),
  ),
)
```

> Class names for component themes (e.g. `CardTheme` vs `CardThemeData`) have changed across Flutter versions. If the analyzer complains, check the docs for your version. That's normal. Reading docs and API changes is part of the job.

---

## 4. Typography

```dart
Text('Headline', style: Theme.of(context).textTheme.headlineMedium)
```

| Style group | Typical use |
|-------------|-------------|
| `displayLarge/Medium/Small` | Huge numbers, hero text |
| `headlineLarge...` | Screen titles |
| `titleLarge/Medium/Small` | Section titles, app bars |
| `bodyLarge/Medium/Small` | Paragraphs |
| `labelLarge/Medium/Small` | Buttons, captions |

### Custom fonts

1. Add font files to `assets/fonts/`.
2. Declare in `pubspec.yaml`:

```yaml
flutter:
  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
```

3. `ThemeData(fontFamily: 'Inter')`. Or use the `google_fonts` package.

---

## 5. Design tokens: spacing, radius, sizes

Avoid "magic numbers". Create constants.

```dart
abstract final class Gaps {
  static const xs = SizedBox(height: 4, width: 4);
  static const sm = SizedBox(height: 8, width: 8);
  static const md = SizedBox(height: 16, width: 16);
  static const lg = SizedBox(height: 24, width: 24);
}

abstract final class Radii {
  static const sm = Radius.circular(8);
  static const md = Radius.circular(16);
}
```

Same idea as a design system: **consistency by construction**.

---

## 6. ThemeExtension — your own tokens

```dart
@immutable
class BrandColors extends ThemeExtension<BrandColors> {
  const BrandColors({required this.success, required this.warning});
  final Color success;
  final Color warning;

  @override
  BrandColors copyWith({Color? success, Color? warning}) =>
      BrandColors(success: success ?? this.success, warning: warning ?? this.warning);

  @override
  BrandColors lerp(ThemeExtension<BrandColors>? other, double t) {
    if (other is! BrandColors) return this;
    return BrandColors(
      success: Color.lerp(success, other.success, t)!,
      warning: Color.lerp(warning, other.warning, t)!,
    );
  }
}

// register
ThemeData(extensions: const [BrandColors(success: Colors.green, warning: Colors.orange)])

// use
final brand = Theme.of(context).extension<BrandColors>()!;
```

---

## 7. Dark mode toggle

```dart
class _AppState extends State<App> {
  ThemeMode _mode = ThemeMode.system;

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      themeMode: _mode,
      theme: lightTheme,
      darkTheme: darkTheme,
      home: SettingsPage(onModeChanged: (m) => setState(() => _mode = m)),
    );
  }
}
```

Persist the choice (see `06_flutter_intermediate/06_local_storage.md`).

---

## 8. Local overrides

Change style for a **subtree** only:

```dart
Theme(
  data: Theme.of(context).copyWith(
    colorScheme: Theme.of(context).colorScheme.copyWith(primary: Colors.red),
  ),
  child: const DangerZone(),
)
```

---

## 9. Responsive text and accessibility

- Users can scale text up to 200%+. **Don't use fixed heights** for text containers.
- Keep contrast ratio ≥ 4.5:1 for body text.
- Test with large fonts and dark mode.
- Add `Semantics(label: ...)` for screen readers where needed.
- Minimum tap target: 48×48 dp.

---

## Common mistakes

- Hard-coded colors and `TextStyle(fontSize: 14)` everywhere.
- Only designing in light mode.
- Fixed-height text boxes that overflow with large font scale.
- Using `Colors.white`/`Colors.black` instead of `onSurface` etc.
- Re-creating `ThemeData` in `build` every time (it's costly and prevents `const` benefits). Build it once, e.g. in a static field.

---

## Code challenge

1. Create `AppTheme.light` and `AppTheme.dark` as static getters in `theme/app_theme.dart`.
2. Build a settings screen with a `SegmentedButton<ThemeMode>` (System / Light / Dark) that switches the whole app live.
3. Add a `ThemeExtension` with a `success` color and show a green "Saved" banner that adapts to dark mode.
4. Audit your screen: no hex colors, no raw font sizes in widgets.

<details>
<summary>Hint — segmented button</summary>

```dart
SegmentedButton<ThemeMode>(
  segments: const [
    ButtonSegment(value: ThemeMode.system, label: Text('System')),
    ButtonSegment(value: ThemeMode.light, label: Text('Light')),
    ButtonSegment(value: ThemeMode.dark, label: Text('Dark')),
  ],
  selected: {mode},
  onSelectionChanged: (s) => onChanged(s.first),
)
```
</details>

---

## Interview questions

1. What is a `ColorScheme` and why use a seed color?
2. Why read colors/styles from `Theme.of(context)` instead of hard-coding?
3. What are `onPrimary`, `onSurface` for?
4. How do you override a theme for only one part of the tree?
5. What is a `ThemeExtension`?
6. How do you support dark mode properly?
7. Name three accessibility considerations for styling.

<details>
<summary>Hints</summary>

1. A set of semantic colors; the seed generates an accessible palette.
2. Consistency, dark mode support, and one place to change.
3. Content colors guaranteed to be readable on their paired background.
4. Wrap in `Theme(data: ..., child: ...)`.
5. A way to add custom tokens to `ThemeData` that animate between themes.
6. `darkTheme` + `themeMode`, semantic colors only, test contrast.
7. Contrast, text scaling, tap target size, semantic labels.
</details>
