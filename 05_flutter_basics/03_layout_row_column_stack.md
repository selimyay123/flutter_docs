# 03 — Layout: Row, Column, Stack

## Goal

Arrange widgets on screen. Layout is where most Flutter beginners struggle. We fix that here.

---

## 1. Row and Column

- `Row` — children left → right (**horizontal**, main axis = horizontal).
- `Column` — children top → bottom (**vertical**, main axis = vertical).

```dart
Column(
  mainAxisAlignment: MainAxisAlignment.center,        // along the main axis
  crossAxisAlignment: CrossAxisAlignment.start,       // across the main axis
  mainAxisSize: MainAxisSize.min,                     // shrink to children
  children: const [
    Text('One'),
    Text('Two'),
    Text('Three'),
  ],
)
```

### The two axes

```
Column:                      Row:
 main axis ↓                 main axis →
 cross axis →                cross axis ↓
```

### `MainAxisAlignment`

| Value | Effect |
|-------|--------|
| `start` | Pack at start |
| `center` | Center |
| `end` | Pack at end |
| `spaceBetween` | Equal gaps between, none at the edges |
| `spaceAround` | Equal space around each (half at the edges) |
| `spaceEvenly` | Equal gaps everywhere |

### `CrossAxisAlignment`

`start`, `center`, `end`, `stretch` (fill the cross axis), `baseline` (align text baselines in a Row).

### `mainAxisSize`

- `MainAxisSize.max` (default) — take **all** available main-axis space.
- `MainAxisSize.min` — only as big as children.

---

## 2. Expanded, Flexible, Spacer

These control **how children share the main-axis space** inside a Row/Column/Flex.

```dart
Row(
  children: [
    const Icon(Icons.person),
    const SizedBox(width: 8),
    Expanded(child: Text('A very long name that must wrap or ellipsize')),  // takes the rest
    const Text('12:30'),
  ],
)
```

- `Expanded` — child **must** fill its share. `flex` sets the ratio.
- `Flexible` — child **may** be smaller than its share (`fit: FlexFit.loose`).
- `Spacer` — empty `Expanded`.

```dart
Row(
  children: const [
    Expanded(flex: 2, child: ColoredBox(color: Colors.red, child: SizedBox(height: 40))),
    Expanded(flex: 1, child: ColoredBox(color: Colors.blue, child: SizedBox(height: 40))),
  ],
)  // red is twice as wide as blue
```

### The classic error: text overflow in a Row

```
A RenderFlex overflowed by 123 pixels on the right.   (yellow-black stripes)
```

Cause: children are wider than the Row. Fix: wrap the long child in `Expanded` or `Flexible` (and set `overflow: TextOverflow.ellipsis` if needed).

---

## 3. Padding, Align, Center, SizedBox

```dart
Padding(padding: const EdgeInsets.fromLTRB(16, 8, 16, 8), child: child)
Align(alignment: Alignment.bottomRight, child: child)
Center(child: child)
SizedBox(width: 100, height: 50, child: child)
SizedBox.expand(child: child)                      // fill the parent
const SizedBox(height: 12)                         // a gap
```

`EdgeInsets`: `all(8)`, `symmetric(horizontal: 16, vertical: 8)`, `only(top: 4)`, `fromLTRB`.

For gaps in modern Flutter you can also use `Column(spacing: 8, children: [...])` (Flutter 3.27+) instead of many `SizedBox`es.

---

## 4. Stack — layers

Children are drawn **on top of each other** (first child at the bottom).

```dart
Stack(
  alignment: Alignment.center,
  children: [
    Image.network('https://picsum.photos/300/200'),
    Container(color: Colors.black38, width: 300, height: 200),
    const Text('Overlay', style: TextStyle(color: Colors.white, fontSize: 24)),
  ],
)
```

### Positioned

Place a child at exact edges **of the Stack**.

```dart
Stack(
  children: [
    const SizedBox(width: 200, height: 200),
    Positioned(top: 8, right: 8, child: Icon(Icons.close)),
    Positioned.fill(child: ColoredBox(color: Colors.black12)),
  ],
)
```

Use `Stack` for badges, overlays, floating labels. Don't use it for normal flow layout.

---

## 5. Wrap, ListTile, and friends

`Wrap` — like `Row` but wraps to the next line:

```dart
Wrap(
  spacing: 8,
  runSpacing: 8,
  children: [for (final t in tags) Chip(label: Text(t))],
)
```

`ListTile` — a ready-made row (leading, title, subtitle, trailing):

```dart
ListTile(
  leading: const CircleAvatar(child: Text('A')),
  title: const Text('Ada'),
  subtitle: const Text('Online'),
  trailing: const Icon(Icons.chevron_right),
  onTap: () {},
)
```

---

## 6. Layout recipes

### Centered card with spacing

```dart
Center(
  child: ConstrainedBox(
    constraints: const BoxConstraints(maxWidth: 400),
    child: Card(
      child: Padding(
        padding: const EdgeInsets.all(24),
        child: Column(
          mainAxisSize: MainAxisSize.min,
          children: const [Text('Title'), SizedBox(height: 12), Text('Body')],
        ),
      ),
    ),
  ),
)
```

### Space between two items

```dart
Row(
  mainAxisAlignment: MainAxisAlignment.spaceBetween,
  children: const [Text('Total'), Text('\$42.00')],
)
```

### Full-width button

```dart
SizedBox(
  width: double.infinity,
  child: FilledButton(onPressed: () {}, child: const Text('Continue')),
)
```

### Column in a scrollable page

```dart
SingleChildScrollView(
  padding: const EdgeInsets.all(16),
  child: Column(children: [...]),
)
```

---

## 7. A debugging method for layout

1. Read the **error message**. Flutter's layout errors are descriptive.
2. Turn on the **Flutter Inspector** in DevTools → "Select widget mode" → "Show guidelines".
3. Add temporary colors: `ColoredBox(color: Colors.red.withOpacity(.2), child: ...)`.
4. Ask: "**What constraints does each widget get from its parent?**" (next lesson).

---

## Common mistakes

- Row/Column children too big → overflow. Use `Expanded`/`Flexible`/scroll.
- `Expanded` outside a Row/Column/Flex → error ("Incorrect use of ParentDataWidget").
- `ListView` inside a `Column` without `Expanded` → "unbounded height" error (see next lesson).
- Nesting `Container` inside `Container` when `padding` or `decoration` would do.
- Using `Stack` where `Row`/`Column` is correct.

---

## Code challenge

Build these three layouts without looking at a solution:

1. A **chat bubble row:** avatar at the left, a bubble with a name and message, time at the far right. Text must not overflow with very long messages.
2. A **product card:** image on top, title, price on the left and a "heart" icon on the right of the same row, a full-width "Add to cart" button.
3. A **photo with badge:** an image with a red circular badge showing "3" in its top-right corner (use `Stack` + `Positioned`).

<details>
<summary>Hint for #1</summary>

```dart
Row(
  crossAxisAlignment: CrossAxisAlignment.start,
  children: [
    const CircleAvatar(child: Text('A')),
    const SizedBox(width: 12),
    Expanded(
      child: Container(
        padding: const EdgeInsets.all(12),
        decoration: BoxDecoration(
          color: Colors.grey.shade200,
          borderRadius: BorderRadius.circular(12),
        ),
        child: Column(
          crossAxisAlignment: CrossAxisAlignment.start,
          children: const [
            Text('Ada', style: TextStyle(fontWeight: FontWeight.bold)),
            Text('This message can be really, really long and will wrap properly.'),
          ],
        ),
      ),
    ),
    const SizedBox(width: 8),
    const Text('12:30'),
  ],
)
```
</details>

---

## Interview questions

1. What are the main axis and cross axis in `Row` and `Column`?
2. `Expanded` vs `Flexible`: what is the difference?
3. Why does "RenderFlex overflowed" happen and how do you fix it?
4. When do you use `Stack` and `Positioned`?
5. `MainAxisSize.min` vs `max`?
6. How would you place a button at the bottom of the screen with content scrolling above it?
7. `Wrap` vs `Row`?

<details>
<summary>Hints</summary>

1. Row: main = horizontal, cross = vertical. Column: opposite.
2. `Expanded` forces the child to fill the share; `Flexible` lets it be smaller.
3. Children exceed available space; use `Expanded`/`Flexible`, scrolling, or `Wrap`.
4. Overlapping layers: badges, overlays; `Positioned` anchors to Stack edges.
5. `min` shrinks to content; `max` takes all available space on the main axis.
6. `Column` with `Expanded(child: ListView(...))` and the button below, or `Scaffold.bottomNavigationBar`.
7. `Wrap` moves overflowing children to a new line.
</details>
