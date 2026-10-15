# 04 — Functions

## Goal

Write clean, reusable functions. Understand parameters, closures, and functions as values.

---

## 1. Basics

```dart
int add(int a, int b) {
  return a + b;
}

// Arrow syntax for one expression
int multiply(int a, int b) => a * b;
```

Return type first, then name, then parameters.

---

## 2. Parameters

### Positional (required)

```dart
int sum(int a, int b) => a + b;
sum(1, 2);
```

### Optional positional `[ ]`

```dart
String greet(String name, [String? title]) =>
    title == null ? 'Hi $name' : 'Hi $title $name';
```

### Named `{ }`

Readable at the call site. Very common in Flutter.

```dart
void createUser({required String name, int age = 18, String? email}) {}

createUser(name: 'Ada', email: 'a@x.com');
```

- `required` — must pass it.
- Default values: `age = 18`.
- Named params can be passed in any order.

**Senior tip:** if a function takes more than 2–3 parameters, or boolean flags, use named parameters.
`draw(true, false)` is unreadable. `draw(filled: true, rounded: false)` is clear.

---

## 3. Functions are objects (first-class)

You can store a function in a variable, pass it, and return it.

```dart
int square(int x) => x * x;

void main() {
  final f = square;          // store
  print(f(4));               // 16

  print([1, 2, 3].map(square).toList()); // pass: [1, 4, 9]
}
```

### Function types

```dart
int Function(int) op = (x) => x + 1;

typedef Predicate<T> = bool Function(T);

List<T> where<T>(List<T> list, Predicate<T> test) {
  final result = <T>[];
  for (final item in list) {
    if (test(item)) result.add(item);
  }
  return result;
}
```

### Anonymous functions (lambdas)

```dart
final isEven = (int n) => n % 2 == 0;
[1, 2, 3, 4].where((n) => n.isEven).forEach(print);
```

---

## 4. Closures

A closure is a function that **remembers variables** from where it was created.

```dart
Function makeCounter() {
  var count = 0;
  return () {
    count++;
    return count;
  };
}

void main() {
  final next = makeCounter();
  print(next()); // 1
  print(next()); // 2
}
```

`count` lives on, even after `makeCounter` finished. Flutter callbacks (`onPressed`) are closures.

---

## 5. Return multiple values — records (Dart 3)

```dart
(int min, int max) minMax(List<int> xs) {
  var lo = xs.first, hi = xs.first;
  for (final x in xs) {
    if (x < lo) lo = x;
    if (x > hi) hi = x;
  }
  return (lo, hi);
}

void main() {
  final (lo, hi) = minMax([3, 1, 9]);
  print('$lo $hi'); // 1 9
}
```

---

## 6. Good function design (senior mindset)

1. **Do one thing.** If you need "and" to describe it, split it.
2. **Small.** Ideally fits on one screen.
3. **Good name.** `calculateTax` not `doStuff`.
4. **Few parameters.** Group related data into a class.
5. **Pure when possible.** Same input → same output, no side effects. Easy to test.
6. **Fail early.** Validate inputs at the top.

Pure vs impure:

```dart
int addPure(int a, int b) => a + b;      // pure

var total = 0;
void addImpure(int a) => total += a;     // impure: changes outside state
```

---

## 7. Recursion preview

A function that calls itself. Needs a **base case** and a **step toward it**.

```dart
int factorial(int n) => n <= 1 ? 1 : n * factorial(n - 1);
```

Deep dive in `04_problem_solving/04_recursion_and_backtracking.md`.

---

## Common mistakes

- Boolean flag parameters that change behavior (split into two functions).
- Forgetting `required` on named params that must be given.
- Functions that both compute and print (hard to test).
- Mutating arguments (lists are passed by reference to the same object).

---

## Code challenge

1. Write `List<int> filterAndMap(List<int> xs, bool Function(int) keep, int Function(int) transform)`.
   Example: keep even numbers, then double them: `[1,2,3,4]` → `[4, 8]`.
2. Write `Function makeMultiplier(int factor)` that returns a function multiplying its input by `factor`.

<details>
<summary>Solution</summary>

```dart
List<int> filterAndMap(
  List<int> xs,
  bool Function(int) keep,
  int Function(int) transform,
) {
  final out = <int>[];
  for (final x in xs) {
    if (keep(x)) out.add(transform(x));
  }
  return out;
}

int Function(int) makeMultiplier(int factor) => (x) => x * factor;

void main() {
  print(filterAndMap([1, 2, 3, 4], (x) => x.isEven, (x) => x * 2)); // [4, 8]
  final triple = makeMultiplier(3);
  print(triple(5)); // 15
}
```
</details>

---

## Interview questions

1. Positional vs optional positional vs named parameters: when do you use each?
2. What is a closure? Give a real Flutter example.
3. What does "functions are first-class" mean?
4. What is a pure function and why is it easy to test?
5. What is a record and how does it differ from a class or a `List`?
6. Why can too many boolean parameters be a code smell?

<details>
<summary>Hints</summary>

1. Positional for 1–2 obvious args; optional rarely; named for clarity and many options.
2. A function capturing outer variables. `onPressed: () => setState(() => count++)` captures `count`.
3. Functions can be assigned, passed and returned like any value.
4. No side effects, deterministic. Just assert on output.
5. Anonymous immutable aggregate with fixed, typed fields; no boilerplate; type-safe unlike a `List`.
6. Call sites become unreadable (`f(true, false)`); the function probably does more than one thing.
</details>
