# 03 — The Big Picture

## Goal

Understand how Dart and Flutter work **before** writing code. A map makes the journey easier.

---

## 1. How Flutter is different

Most frameworks (React Native, native) use the platform's own buttons and text fields.
**Flutter draws every pixel itself** using its own rendering engine.

```
 Your Dart code (widgets)
        │
   Flutter Framework (Dart)      <- widgets, layout, animation, gestures
        │
   Flutter Engine (C++)          <- Impeller/Skia graphics, text, Dart runtime
        │
   Platform Embedder             <- iOS / Android / Web / Desktop
```

Result: same look everywhere, high speed (60/120 FPS), full control.

---

## 2. How Dart runs

Dart has two modes:

| Mode | Used when | Compiler |
|------|-----------|----------|
| **JIT** (Just-In-Time) | Development (`flutter run`) | Compiles while running → enables **hot reload** |
| **AOT** (Ahead-Of-Time) | Release builds | Compiles to fast machine code before shipping |

You get fast developer feedback **and** fast apps. That is Dart's superpower.

---

## 3. The core idea of Flutter: UI = f(state)

```
UI = function(state)
```

You do **not** say "change the label text to 5".
You say "the label shows `count`" and change `count`. Flutter rebuilds what is needed.

```dart
// Mental model
Widget build() => Text('Count: $count');   // describe the UI for the current state
```

This is called **declarative UI**. It removes a whole class of bugs ("I forgot to update that label").

---

## 4. Everything is a widget

A widget is a **description** of a part of the screen. Widgets nest like boxes in boxes:

```
MaterialApp
 └─ Scaffold
     ├─ AppBar
     │   └─ Text('Home')
     └─ Center
         └─ Text('Hello')
```

Layout, padding, alignment, themes, even gestures are widgets.
You will learn them in folder `05`.

---

## 5. The three trees (preview)

Flutter keeps three trees internally. You will study them in depth in `06/01`.

| Tree | Role | Lifetime |
|------|------|----------|
| Widget tree | Cheap, immutable configuration | Rebuilt often |
| Element tree | The "living" instances, holds state links | Long-lived |
| RenderObject tree | Does layout & painting | Long-lived |

Why it matters: rebuilding widgets is cheap because the heavy objects are reused.

---

## 6. Course learning path visual

```
Dart basics ─► Dart OOP ─► Async ─┐
                                   ├─► Flutter basics ─► State & Architecture ─► Advanced ─► Projects
Problem solving (every day!) ──────┘
```

---

## 7. What "senior level" means

A senior engineer is not someone who knows more syntax. They:

- Choose the **simplest design** that works.
- Know the **trade-offs** (speed vs memory, simple vs flexible).
- Write code others can **read and change**.
- **Test** and **measure** before optimizing.
- Explain complex things in simple words.

Every lesson here trains these habits.

---

## Code challenge

Draw (on paper) the widget tree for a screen with: a top bar titled "Profile", a circle avatar in the middle, and under it the user's name and a "Follow" button.
No code. Just boxes and arrows.

<details>
<summary>Possible answer</summary>

```
Scaffold
 ├─ AppBar(title: Text('Profile'))
 └─ Center
     └─ Column
         ├─ CircleAvatar
         ├─ Text('Ada Lovelace')
         └─ ElevatedButton(child: Text('Follow'))
```
</details>

---

## Interview questions

1. What does "declarative UI" mean? Compare it with imperative UI.
2. Why does Flutter not use native platform widgets?
3. JIT vs AOT: where does each appear in Flutter's workflow?
4. What does `UI = f(state)` mean in practice?
5. Name the three trees in Flutter and say which one is rebuilt most often.

<details>
<summary>Hints</summary>

1. Declarative: describe *what* UI looks like for a state. Imperative: issue commands to mutate widgets step by step.
2. It draws with its own engine → pixel-perfect consistency, no platform bridge for UI, full control.
3. JIT in debug (hot reload). AOT in release (speed, small startup).
4. UI is a pure result of current data. Change data → UI updates.
5. Widget, Element, RenderObject. The widget tree is rebuilt most often.
</details>
