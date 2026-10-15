# 07 — Error Handling

## Goal

Handle failure like a professional. Real apps fail all the time: no network, bad input, missing file.

---

## 1. Exceptions vs Errors

| | `Exception` | `Error` |
|--|------------|---------|
| Meaning | Expected problem the program can recover from | Programmer bug |
| Examples | `FormatException`, `IOException`, `TimeoutException` | `RangeError`, `StateError`, `TypeError`, `AssertionError` |
| Should you catch it? | Yes | Usually **no**. Fix the code. |

Dart lets you throw **anything**, but throw `Exception` / `Error` subclasses.

---

## 2. try / catch / finally

```dart
void main() {
  try {
    final n = int.parse('abc');   // throws FormatException
    print(n);
  } on FormatException catch (e) {
    print('Not a number: ${e.message}');
  } catch (e, stackTrace) {
    print('Something else: $e');
    print(stackTrace);
  } finally {
    print('Always runs (cleanup).');
  }
}
```

- `on Type` — catch only that type.
- `catch (e, st)` — `e` is the object, `st` the stack trace.
- `finally` — always runs. Close files, stop spinners.
- `rethrow` — catch, log, and pass it on **keeping the stack trace**.

```dart
try {
  riskyCall();
} catch (e) {
  log('failed');
  rethrow;          // better than `throw e;`
}
```

---

## 3. Custom exceptions

```dart
class InsufficientFundsException implements Exception {
  InsufficientFundsException(this.needed);
  final double needed;

  @override
  String toString() => 'Need $needed more';
}

void withdraw(double balance, double amount) {
  if (amount > balance) {
    throw InsufficientFundsException(amount - balance);
  }
}
```

---

## 4. Choose how to report failure

Four ways. Pick deliberately.

| Style | Example | When |
|-------|---------|------|
| Throw | `throw FormatException()` | Truly exceptional, caller can't continue |
| Nullable return | `int? tryParse(...)` | Absence is a normal outcome |
| Result type | `Result<T>` (success or failure) | Expected failures you want the caller to handle |
| Assert | `assert(x > 0)` | Developer-only checks (removed in release) |

### A simple Result type with sealed classes (Dart 3)

```dart
sealed class Result<T> {
  const Result();
}

final class Ok<T> extends Result<T> {
  const Ok(this.value);
  final T value;
}

final class Err<T> extends Result<T> {
  const Err(this.message);
  final String message;
}

Result<int> parseAge(String input) {
  final n = int.tryParse(input);
  if (n == null) return const Err('Not a number');
  if (n < 0) return const Err('Negative age');
  return Ok(n);
}

void main() {
  switch (parseAge('42')) {
    case Ok(:final value):
      print('Age: $value');
    case Err(:final message):
      print('Error: $message');
  }
}
```

The compiler checks your `switch` covers every case (**exhaustive**). You cannot forget to handle errors.

---

## 5. Best practices

1. **Fail fast:** validate early, throw clear messages.
2. **Don't swallow errors:** an empty `catch {}` hides bugs.
3. **Catch specific types** when you can.
4. **Add context:** "Failed to load user 42" beats "null".
5. **Clean up** in `finally`.
6. **Do not use exceptions for normal control flow** (slow and confusing).
7. At app boundaries (UI, network), convert exceptions into user-friendly states.

---

## Common mistakes

- `catch (e) {}` and nothing else.
- Catching `Error` types to hide bugs.
- `throw e;` instead of `rethrow;` (loses the stack trace).
- Showing raw exception text to the user.

---

## Code challenge

Write `int safeDivide(int a, int b)` that throws a custom `DivisionByZeroException` when `b == 0`.
Then write `Result<double> divide(double a, double b)` using the `Result` type above that never throws.

<details>
<summary>Solution</summary>

```dart
class DivisionByZeroException implements Exception {
  @override
  String toString() => 'Cannot divide by zero';
}

int safeDivide(int a, int b) {
  if (b == 0) throw DivisionByZeroException();
  return a ~/ b;
}

Result<double> divide(double a, double b) =>
    b == 0 ? const Err('Cannot divide by zero') : Ok(a / b);
```
</details>

---

## Interview questions

1. `Exception` vs `Error`: what is the difference and which do you catch?
2. `rethrow` vs `throw e`: why does it matter?
3. When would you return `null`, throw, or return a `Result`?
4. What does `finally` guarantee?
5. What are `assert`s and are they active in release builds?
6. What is wrong with an empty `catch` block?

<details>
<summary>Hints</summary>

1. Exceptions = recoverable conditions; Errors = programming bugs. Catch exceptions, fix errors.
2. `rethrow` keeps the original stack trace.
3. Null: absence normal. Result: failure expected, caller must decide. Throw: unexpected, can't proceed.
4. The block runs whether the `try` succeeds, throws, or returns.
5. Dev-time checks; disabled in release (and profile) builds.
6. Hides bugs, makes debugging nearly impossible.
</details>
