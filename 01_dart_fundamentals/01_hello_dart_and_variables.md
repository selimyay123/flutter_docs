# 01 — Hello Dart and Variables

## Goal

Write your first Dart program. Understand variables, constants, and printing.

---

## 1. The smallest program

```dart
void main() {
  print('Hello, Dart!');
}
```

- `main()` is where every Dart program starts.
- `void` means "returns nothing".
- `print(...)` writes text to the console.
- Statements end with `;`.

---

## 2. Variables

A variable is a **named box** that holds a value.

```dart
var name = 'Ada';        // type inferred: String
int age = 36;            // type written explicitly
double height = 1.65;
bool isStudent = false;
```

Dart is **statically typed**: every variable has a type known at compile time.
`var` does **not** mean "any type". Dart *infers* the type once, then it is fixed.

```dart
var x = 10;
x = 'hello'; // ERROR: String can't be assigned to int
```

### `dynamic` (avoid it)

```dart
dynamic y = 10;
y = 'hello'; // allowed, but you lose all safety
```

Use `dynamic` only when you truly cannot know the type (e.g. raw JSON). Prefer real types.

---

## 3. `final` and `const`

| Keyword | Value fixed when? | Can change later? |
|---------|-------------------|-------------------|
| `var` | — | Yes |
| `final` | At **runtime**, once | No |
| `const` | At **compile time** | No (and deeply immutable) |

```dart
final now = DateTime.now();   // known only when program runs -> final
const pi = 3.14159;           // known now -> const
const List<int> primes = [2, 3, 5]; // const list: cannot be modified at all
```

**Rule of thumb (senior habit):** use `const` if you can, else `final`, else `var`.
Fewer changing things = fewer bugs.

In Flutter, `const` widgets are reused and **skip rebuilds** → real performance gains.

### `late`

`late` means "I promise I will set this before I read it."

```dart
late final String config;

void init() {
  config = 'loaded';
}
```

Reading before setting throws an error at runtime. Use rarely.

---

## 4. String interpolation

```dart
final name = 'Ada';
final age = 36;
print('Name: $name, next year: ${age + 1}');
```

- `$name` — insert a variable.
- `${expression}` — insert any expression.

---

## 5. Comments

```dart
// single line

/* multi
   line */

/// Documentation comment. Shows up in IDE tooltips.
int add(int a, int b) => a + b;
```

---

## 6. Naming rules (Dart style)

| Thing | Style | Example |
|-------|-------|---------|
| Variables, functions | `lowerCamelCase` | `userName` |
| Classes, enums, typedefs | `UpperCamelCase` | `UserProfile` |
| Files, packages | `snake_case` | `user_profile.dart` |
| Constants | `lowerCamelCase` | `maxRetries` (not `MAX_RETRIES`) |
| Private | start with `_` | `_cache` |

The underscore `_` makes something private to its **library (file)**, not to the class.

---

## Common mistakes

- Using `var` then expecting to switch types.
- Using `dynamic` everywhere "to make errors go away".
- Forgetting `;`.
- Using `final` when `const` is possible.

---

## Code challenge

Write a program that stores a product name, price (double), and quantity (int), then prints:

```
Total for 3 x Coffee: 13.5
```

Use `final`/`const` where possible and string interpolation.

<details>
<summary>Solution</summary>

```dart
void main() {
  const name = 'Coffee';
  const price = 4.5;
  const quantity = 3;
  final total = price * quantity;
  print('Total for $quantity x $name: $total');
}
```
</details>

---

## Interview questions

1. `var` vs `dynamic` vs `Object`: what is the difference?
2. `final` vs `const`: explain with an example where only `final` works.
3. What does `late` do and what is its risk?
4. Is Dart statically or dynamically typed? Is it strongly typed?
5. How do you make a member private in Dart?

<details>
<summary>Hints</summary>

1. `var` infers one fixed type. `dynamic` disables type checks. `Object` is the root type but you can only call `Object` methods without a cast.
2. `final now = DateTime.now();` (only known at runtime). `const` needs a compile-time value.
3. Delays initialization; reading before assignment throws `LateInitializationError`.
4. Statically typed (sound type system), with type inference.
5. Prefix with `_`. Privacy is per library (file), not per class.
</details>
