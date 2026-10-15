# 04 — Constraints: The Rule That Explains All Layout

## Goal

Understand Flutter's single layout rule. After this, layout errors stop being mysterious.

---

## 1. The golden rule

> **Constraints go down. Sizes go up. The parent sets the position.**

Three steps, for every widget:

1. The **parent** gives the child **constraints**: min/max width and height.
2. The **child** picks a size **within** those constraints and tells the parent.
3. The **parent** decides **where** to place the child.

```
Parent: "You may be 0–300 wide and 0–500 tall."
Child:  "OK, I will be 120 × 40."
Parent: "Fine. I put you at (90, 230)."
```

A child **cannot** choose a size outside its constraints, and it **doesn't know** where it will be placed.

---

## 2. BoxConstraints

```dart
BoxConstraints(minWidth: 0, maxWidth: 300, minHeight: 0, maxHeight: 500)
```

| Type | Meaning |
|------|---------|
| **Tight** | min == max → exactly one size allowed (child has no choice) |
| **Loose** | min = 0, max > 0 → child can be anything up to max |
| **Unbounded** | max = infinity → no limit (common in scrolling directions) |

The root: the screen gives the app **tight** constraints = the screen size.

---

## 3. Why "my width doesn't work"

```dart
Center(
  child: Container(width: 100, height: 100, color: Colors.red),
)
```

`Center` gives its child **loose** constraints → the Container gets 100×100. ✔

```dart
Container(
  width: 100, height: 100, color: Colors.red,   // directly under the screen root
)
```

The screen gives **tight** constraints (full screen). A tight constraint **overrides** the Container's request: it fills the screen! The red box ignores 100×100.

Fix: put it in `Center`, `Align`, or `UnconstrainedBox` — widgets that **loosen** constraints.

---

## 4. How common widgets treat constraints

| Widget | Behavior |
|--------|----------|
| `Center`, `Align` | Loosens constraints for the child; takes all available space itself |
| `Padding` | Shrinks constraints by the padding, then adds padding to the size |
| `SizedBox(w, h)` | Gives the child **tight** constraints of that size (within its own limits) |
| `ConstrainedBox` | Adds extra constraints |
| `LimitedBox` | Limits size **only when unbounded** |
| `Container` (no child, no size) | Fills available space |
| `Container` (has child, no size) | Wraps the child |
| `Text` | As wide as the text needs, up to max width (then wraps) |
| `Row`/`Column` | Pass **unbounded** constraints on the main axis to non-flex children |
| `Expanded` | Gives the child **tight** constraints on the main axis (its share) |
| `ListView` | Unbounded in the scroll direction |
| `Stack` | Loosens for non-positioned children |
| `FittedBox` | Scales the child to fit |
| `AspectRatio` | Picks the biggest size with the ratio |

---

## 5. The famous errors and their causes

### "Vertical viewport was given unbounded height"

```dart
Column(
  children: [
    ListView(...),     // ERROR
  ],
)
```

`Column` gives its children **unbounded height**. A `ListView` wants to fill all the height it's given → infinite. Fix: **give it a size**.

```dart
Column(
  children: [
    Expanded(child: ListView(...)),                 // fill the remaining space
    // or SizedBox(height: 200, child: ListView(...))
    // or ListView(shrinkWrap: true, physics: NeverScrollableScrollPhysics())  <- slow for big lists
  ],
)
```

### "RenderFlex overflowed by N pixels"

A Row/Column was given **bounded** space but its children needed more. Fix: `Expanded`/`Flexible`, scroll, or shrink content.

### "BoxConstraints forces an infinite width"

A widget that wants to fill width (e.g. `TextField`, `Expanded` misuse) is inside a Row (unbounded width). Fix: wrap in `Expanded` or `SizedBox(width: ...)`.

```dart
Row(
  children: [
    Expanded(child: TextField()),     // ✔
    IconButton(onPressed: () {}, icon: const Icon(Icons.send)),
  ],
)
```

### "Incorrect use of ParentDataWidget"

`Expanded` must be a **direct** child of Row/Column/Flex. `Positioned` must be a direct child of Stack.

---

## 6. LayoutBuilder — read the constraints

Make decisions based on the **space available** (not just the screen size).

```dart
LayoutBuilder(
  builder: (context, constraints) {
    if (constraints.maxWidth > 600) {
      return const WideLayout();
    }
    return const NarrowLayout();
  },
)
```

Prefer `LayoutBuilder` over `MediaQuery` for components, because a widget may live in a small panel on a big screen.

---

## 7. Intrinsic sizes (use sparingly)

`IntrinsicHeight`/`IntrinsicWidth` make a widget ask children for their natural size — it costs an **extra layout pass** (potentially `O(n²)` in deep nesting). Use rarely.

---

## 8. How Flutter layout is fast

Layout is a **single pass** down and up the tree (`O(n)`), thanks to the constraint rule. This is why Flutter can lay out thousands of widgets in a 16 ms frame. Intrinsics break this, hence the warnings.

---

## Mental checklist when layout looks wrong

1. What are the **constraints** my widget receives? (Inspector shows them.)
2. Is the parent giving **tight** or **loose** constraints?
3. Is the direction **unbounded** (inside Row/Column/ListView)?
4. Do I need to **loosen** (Center/Align), **bound** (SizedBox/ConstrainedBox/Expanded) or **scroll**?

---

## Common mistakes

- Setting `width` on a `Container` directly under a tight parent.
- `ListView` inside `Column` without `Expanded`.
- `shrinkWrap: true` on huge lists (kills performance).
- Using `MediaQuery` width for component sizing instead of `LayoutBuilder`.
- Putting `Expanded` inside a `Padding` inside a `Row` (not a direct child).

---

## Code challenge

**Predict first, then run:**

1. What size is the red box?
   ```dart
   Center(child: Container(width: double.infinity, height: 50, color: Colors.red))
   ```
2. What size is the blue box?
   ```dart
   Container(
     color: Colors.green,
     child: Center(
       child: SizedBox(width: 10, height: 10, child: ColoredBox(color: Colors.blue)),
     ),
   )
   ```
3. Why does this crash, and how do you fix it?
   ```dart
   Row(children: [TextField(), const Text('hi')])
   ```
4. Build a widget `MaxWidthBox` that centers its child and limits it to 600 px width on large screens, but full width on small screens.

<details>
<summary>Solution</summary>

1. `Center` loosens constraints to `0..screenWidth`; `double.infinity` is clamped to the max → the box is **full screen width** × 50.
2. The green Container has no size → with a child it wraps it, but `Center` expands to fill the parent's constraints (tight screen) → green fills the screen. The blue box is 10×10 (SizedBox gives tight 10×10 inside Center's loose constraints).
3. `TextField` wants infinite width inside an unbounded Row. Fix: `Expanded(child: TextField())`.
4.
```dart
class MaxWidthBox extends StatelessWidget {
  const MaxWidthBox({super.key, required this.child, this.maxWidth = 600});
  final Widget child;
  final double maxWidth;

  @override
  Widget build(BuildContext context) => Center(
        child: ConstrainedBox(
          constraints: BoxConstraints(maxWidth: maxWidth),
          child: child,
        ),
      );
}
```
</details>

---

## Interview questions

1. State Flutter's layout rule in one sentence.
2. What is the difference between tight and loose constraints?
3. Why does a `Container(width: 100)` sometimes ignore its width?
4. Why does `ListView` inside `Column` cause an unbounded height error? Three fixes?
5. What does `Expanded` actually do to constraints?
6. `LayoutBuilder` vs `MediaQuery`: when do you use each?
7. Why is Flutter's layout algorithm efficient? What can break that?

<details>
<summary>Hints</summary>

1. Constraints go down, sizes go up, parent sets position.
2. Tight: min == max (exact size forced); loose: child can choose up to max.
3. A parent with tight constraints forces its own size; use `Center`/`Align` to loosen.
4. `Column` gives unbounded height; `Expanded`, fixed `SizedBox` height, or `shrinkWrap` (careful).
5. Gives the child tight main-axis constraints equal to its flex share.
6. `LayoutBuilder` for local available space; `MediaQuery` for device/window info (size, padding, text scale, orientation).
7. Single-pass layout O(n); intrinsic widgets add extra passes.
</details>
