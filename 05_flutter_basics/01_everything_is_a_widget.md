# 01 — Everything Is a Widget

## Goal

Build your first Flutter screen. Understand what a widget is and how to compose them.

---

## 1. Your first app

```dart
import 'package:flutter/material.dart';

void main() {
  runApp(const MyApp());
}

class MyApp extends StatelessWidget {
  const MyApp({super.key});

  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Hello Flutter',
      theme: ThemeData(colorSchemeSeed: Colors.indigo, useMaterial3: true),
      home: Scaffold(
        appBar: AppBar(title: const Text('Home')),
        body: const Center(
          child: Text('Hello, Flutter!'),
        ),
      ),
    );
  }
}
```

Read it top to bottom:

- `main()` → `runApp()` puts a widget at the **root** of the screen.
- `MyApp` is a widget (a class that extends `StatelessWidget`).
- `build()` **describes** the UI for that widget.
- `MaterialApp` sets up theme, navigation, and localization.
- `Scaffold` gives you a standard page layout (app bar, body, floating button, drawer...).

---

## 2. What is a widget?

A widget is an **immutable description** of part of the UI. It is not the pixels themselves, just a blueprint.

Flutter widgets fall in a few families:

| Family | Examples |
|--------|----------|
| Structure | `Scaffold`, `AppBar`, `SafeArea` |
| Layout | `Row`, `Column`, `Stack`, `Padding`, `Center`, `Align`, `Expanded` |
| Display | `Text`, `Image`, `Icon`, `Card`, `Divider` |
| Input | `TextField`, `Checkbox`, `Switch`, `Slider`, buttons |
| Scrolling | `ListView`, `GridView`, `SingleChildScrollView` |
| Styling | `Theme`, `DecoratedBox`, `Container` |
| Behavior | `GestureDetector`, `Dismissible`, `AnimatedFoo` |
| Invisible helpers | `InheritedWidget`, `Provider`, `MediaQuery` |

Even **padding is a widget**: `Padding(padding: ..., child: ...)`.

---

## 3. Composition — the main skill

You build big UIs by nesting small widgets. Every widget has either:

- no children (`Text`, `Icon`)
- one `child` (`Padding`, `Center`, `Container`)
- many `children` (`Row`, `Column`, `Stack`, `ListView`)

```dart
Card(
  child: Padding(
    padding: const EdgeInsets.all(16),
    child: Column(
      crossAxisAlignment: CrossAxisAlignment.start,
      children: const [
        Text('Ada Lovelace', style: TextStyle(fontSize: 20, fontWeight: FontWeight.bold)),
        SizedBox(height: 4),
        Text('First programmer'),
      ],
    ),
  ),
)
```

### Prefer small widgets (classes) over giant `build` methods

```dart
class ProfileCard extends StatelessWidget {
  const ProfileCard({super.key, required this.name, required this.bio});

  final String name;
  final String bio;

  @override
  Widget build(BuildContext context) {
    return Card(
      child: Padding(
        padding: const EdgeInsets.all(16),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text(name, style: Theme.of(context).textTheme.titleLarge),
            const SizedBox(height: 4),
            Text(bio),
          ],
        ),
      ),
    );
  }
}
```

Why classes instead of helper methods like `Widget _buildCard()`?

- A widget **class** can be `const` and Flutter can skip rebuilding it.
- It has its own `BuildContext`, and can be tested alone.
- Rebuilds stay local.

---

## 4. Essential widgets — a tour

### Text

```dart
const Text(
  'Hello',
  style: TextStyle(fontSize: 24, color: Colors.teal, fontWeight: FontWeight.w600),
  maxLines: 2,
  overflow: TextOverflow.ellipsis,
  textAlign: TextAlign.center,
)
```

### Container (use it wisely)

```dart
Container(
  width: 120,
  height: 60,
  padding: const EdgeInsets.all(8),
  margin: const EdgeInsets.symmetric(vertical: 8),
  decoration: BoxDecoration(
    color: Colors.indigo,
    borderRadius: BorderRadius.circular(12),
    boxShadow: const [BoxShadow(blurRadius: 8, color: Colors.black26)],
  ),
  child: const Text('Box', style: TextStyle(color: Colors.white)),
)
```

Don't use `Container` when a simpler widget does the job: `SizedBox` (spacing/size), `Padding`, `ColoredBox`, `DecoratedBox`, `Align`.

### Buttons

```dart
ElevatedButton(onPressed: () {}, child: const Text('Elevated'))
FilledButton(onPressed: () {}, child: const Text('Filled'))
OutlinedButton(onPressed: () {}, child: const Text('Outlined'))
TextButton(onPressed: () {}, child: const Text('Text'))
IconButton(onPressed: () {}, icon: const Icon(Icons.favorite))
```

`onPressed: null` disables a button.

### Image

```dart
Image.network('https://picsum.photos/200', width: 100, height: 100, fit: BoxFit.cover)
Image.asset('assets/logo.png')   // declare in pubspec.yaml
```

### Icon, Divider, SizedBox

```dart
const Icon(Icons.star, color: Colors.amber, size: 32)
const Divider()
const SizedBox(height: 16)       // vertical gap
```

---

## 5. The `BuildContext`

`BuildContext` is the widget's **location in the tree**. You use it to look **up** the tree:

```dart
Theme.of(context)           // nearest Theme
MediaQuery.of(context)      // screen size, padding, text scale
Navigator.of(context)       // navigation
ScaffoldMessenger.of(context).showSnackBar(...)
```

Rule: the context you use must be **below** the widget you're looking for. A famous bug:

```dart
// ERROR: this context is ABOVE the Scaffold, so Scaffold.of can't find it
Widget build(BuildContext context) {
  return Scaffold(
    body: ElevatedButton(
      onPressed: () => Scaffold.of(context).openDrawer(),  // fails
      child: const Text('x'),
    ),
  );
}
// Fix: wrap the button in Builder(...) or move it into its own widget class.
```

---

## 6. `const` — a free performance win

```dart
const Text('Static');            // created once, reused
const SizedBox(height: 8);
```

If a widget and **all** its arguments are compile-time constants, mark it `const`. Flutter then skips rebuilding it. The `prefer_const_constructors` lint helps.

---

## 7. Hot reload practice

Run the app, change the text color and padding in `ProfileCard`, press `r`. Notice the speed. That fast loop is how you learn Flutter.

---

## Common mistakes

- One giant `build` method with 300 lines of nesting. Split into widgets.
- Using `Container` for everything.
- Forgetting `const`.
- Using `Scaffold.of(context)` with the wrong context.
- Hard-coding colors/sizes everywhere instead of using the theme.

---

## Code challenge

Build a **profile screen** (no state yet):

- App bar titled "Profile".
- A `CircleAvatar` with initials, a name, a short bio.
- A row with three stats (Posts, Followers, Following), each a number over a label.
- A "Follow" `FilledButton` and a "Message" `OutlinedButton` side by side.

Requirements: at least **3 custom widget classes** (e.g. `StatItem`, `ProfileHeader`, `ActionButtons`) and as many `const` as possible.

<details>
<summary>Hint — StatItem</summary>

```dart
class StatItem extends StatelessWidget {
  const StatItem({super.key, required this.value, required this.label});
  final String value;
  final String label;

  @override
  Widget build(BuildContext context) {
    return Column(
      children: [
        Text(value, style: Theme.of(context).textTheme.titleLarge),
        Text(label, style: Theme.of(context).textTheme.bodySmall),
      ],
    );
  }
}
```

Use it in a `Row(mainAxisAlignment: MainAxisAlignment.spaceEvenly, children: [...])`.
</details>

---

## Interview questions

1. What is a widget in Flutter? Is it the thing drawn on screen?
2. What is `BuildContext`?
3. Why prefer small widget classes over helper methods returning widgets?
4. What does `const` do for a widget?
5. What is `MaterialApp` vs `Scaffold`?
6. `Container` vs `SizedBox` vs `Padding`: when do you choose each?
7. Why does `Scaffold.of(context)` sometimes fail inside the `build` method of the same widget that creates the `Scaffold`?

<details>
<summary>Hints</summary>

1. An immutable configuration/description; rendering is done by RenderObjects.
2. A handle to the widget's position in the tree, used to look up ancestors.
3. `const` instances, independent rebuilds, element reuse, testability.
4. Canonical compile-time instance; Flutter can skip rebuilding it.
5. `MaterialApp` configures the app (theme, routes); `Scaffold` is a page skeleton.
6. Use the most specific lightweight widget: `SizedBox` for size/gap, `Padding` for padding, `Container` when you need several features.
7. That `context` belongs to a widget above the `Scaffold`; lookups go upward only.
</details>
