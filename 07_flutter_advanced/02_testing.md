# 02 — Testing

## Goal

Write tests that give you **confidence to change code**. Know the three kinds of Flutter tests and when to use each.

---

## 1. Why test?

- Catch bugs **before** users do.
- **Refactor safely.**
- Tests are **executable documentation**.
- Good tests force good design (testable code = loosely coupled code).

---

## 2. The test pyramid

```
        /\         Integration / E2E   — few, slow, real device/app
       /  \
      /----\       Widget tests        — some, fast, simulated UI
     /      \
    /--------\     Unit tests          — many, very fast, pure Dart
```

| Type | Tests | Speed | Needs device? |
|------|-------|-------|---------------|
| **Unit** | A function/class/ViewModel/repository | ms | No |
| **Widget** | A widget or screen in a simulated environment | ~100 ms | No |
| **Integration** | The full app on a device/emulator | seconds | Yes |
| **Golden** | Pixel comparison with a stored image | fast | No |

Write most tests at the bottom. Don't test only through the UI.

---

## 3. Unit tests

`test/` folder; run `flutter test` (or `dart test` for pure Dart).

```dart
import 'package:test/test.dart';        // or flutter_test

int add(int a, int b) => a + b;

void main() {
  group('add', () {
    test('adds positives', () => expect(add(2, 3), 5));
    test('handles negatives', () => expect(add(-2, -3), -5));
  });
}
```

### Anatomy: Arrange – Act – Assert

```dart
test('cart total sums item prices', () {
  // Arrange
  final cart = Cart()..add(Product('a', 10))..add(Product('b', 5));
  // Act
  final total = cart.total;
  // Assert
  expect(total, 15);
});
```

### Useful matchers

```dart
expect(x, equals(3));
expect(list, containsAll([1, 2]));
expect(list, isEmpty);
expect(() => parse('x'), throwsA(isA<FormatException>()));
expect(future, completion(equals(5)));
expect(future, throwsA(isA<ApiException>()));
expect(stream, emitsInOrder([1, 2, emitsDone]));
expect(value, closeTo(0.3, 1e-9));           // doubles!
```

### Test names say behavior

"`login fails with 401 shows invalid credentials`" beats "`test1`".

### Edge cases to include

Empty, one item, boundaries, null/absent, invalid input, duplicate, large input, error from a dependency.

---

## 4. Test doubles: fakes, stubs, mocks

| Kind | What | Example |
|------|------|---------|
| **Fake** | Working simplified implementation | `InMemoryTodoRepository` |
| **Stub** | Returns canned answers | API returning a fixed JSON |
| **Mock** | Records calls so you can verify interaction | `verify(repo.save(any))` |

Prefer **fakes** (simple, less brittle). Use mocks (`mocktail`) when you must verify interactions or simulate failures.

```dart
class FakeUserRepository implements UserRepository {
  FakeUserRepository({this.users = const [], this.failWith});
  final List<User> users;
  final Exception? failWith;

  @override
  Future<List<User>> getUsers() async {
    if (failWith != null) throw failWith!;
    return users;
  }
}

test('shows error when repository fails', () async {
  final vm = UsersViewModel(FakeUserRepository(failWith: const NetworkException()));
  await vm.load();
  expect(vm.state, isA<UsersError>());
});
```

With `mocktail`:

```dart
class MockRepo extends Mock implements UserRepository {}

final repo = MockRepo();
when(() => repo.getUsers()).thenAnswer((_) async => [user]);
await vm.load();
verify(() => repo.getUsers()).called(1);
```

### Time and randomness are dependencies

Don't call `DateTime.now()` or `Random()` deep inside logic. Inject a **clock** or a seeded random, so tests are deterministic.

```dart
class Greeter {
  Greeter(this._now);
  final DateTime Function() _now;
  String greet() => _now().hour < 12 ? 'Good morning' : 'Hello';
}
```

---

## 5. Widget tests

```dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';

testWidgets('counter increments', (tester) async {
  await tester.pumpWidget(const MaterialApp(home: CounterPage()));

  expect(find.text('0'), findsOneWidget);

  await tester.tap(find.byIcon(Icons.add));
  await tester.pump();                          // rebuild one frame

  expect(find.text('1'), findsOneWidget);
});
```

### Key APIs

| API | Meaning |
|-----|---------|
| `pumpWidget(w)` | Build `w` |
| `pump([duration])` | Advance one frame (optionally time) |
| `pumpAndSettle()` | Pump until animations finish (fails on infinite animations!) |
| `find.text`, `find.byType`, `find.byKey`, `find.byIcon`, `find.byTooltip`, `find.descendant` | Locate widgets |
| `tester.tap`, `enterText`, `drag`, `fling`, `longPress` | Simulate user |
| `expect(finder, findsOneWidget / findsNothing / findsNWidgets(n))` | Assert |
| `tester.widget<T>(finder)` | Read widget properties |

Give important widgets a `Key` (`Key('submit_button')`) for stable finding.

### Testing with providers / DI

```dart
await tester.pumpWidget(
  ProviderScope(
    overrides: [userRepositoryProvider.overrideWithValue(FakeUserRepository(users: [ada]))],
    child: const MaterialApp(home: UsersPage()),
  ),
);
await tester.pumpAndSettle();
expect(find.text('Ada'), findsOneWidget);
```

### Async in widget tests

Futures with real timers: use `tester.runAsync`, or fake timers by `pump(Duration(...))`. Pending timers at test end cause failures. Always finish or cancel them.

### Golden tests (visual regression)

```dart
await expectLater(find.byType(ProfileCard), matchesGoldenFile('goldens/profile_card.png'));
// update goldens:  flutter test --update-goldens
```

Fonts/platforms can differ; keep goldens in CI on one platform, or use `golden_toolkit`/`alchemist`.

---

## 6. Integration tests

`integration_test` package; runs the **real app** on a device/emulator.

```dart
import 'package:integration_test/integration_test.dart';

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  testWidgets('user can log in and see home', (tester) async {
    app.main();
    await tester.pumpAndSettle();

    await tester.enterText(find.byKey(const Key('email')), 'ada@x.com');
    await tester.enterText(find.byKey(const Key('password')), 'secret123');
    await tester.tap(find.byKey(const Key('login')));
    await tester.pumpAndSettle();

    expect(find.text('Welcome, Ada'), findsOneWidget);
  });
}
```

Run: `flutter test integration_test`. Keep few and focused on **critical user journeys** (login, checkout). Use a test backend or fakes.

---

## 7. Testing state management

### Bloc (`bloc_test`)

```dart
blocTest<CounterCubit, int>(
  'emits [1] on increment',
  build: CounterCubit.new,
  act: (cubit) => cubit.increment(),
  expect: () => [1],
);
```

### Riverpod

```dart
final container = ProviderContainer(overrides: [repoProvider.overrideWithValue(FakeRepo())]);
addTearDown(container.dispose);
expect(await container.read(usersProvider.future), hasLength(2));
```

---

## 8. Design for testability

1. **Inject dependencies** (constructor injection).
2. **Pure functions** for logic.
3. **Interfaces** around I/O (network, DB, clock, platform).
4. **Small widgets** with explicit inputs.
5. No hidden globals/singletons.
6. Keep logic out of `build` and event handlers.

If a test is hard to write, the design is trying to tell you something.

---

## 9. Coverage & CI

```bash
flutter test --coverage        # creates coverage/lcov.info
```

- Coverage shows what is **not** tested; it doesn't prove tests are **good**. 100% coverage with weak asserts is meaningless.
- Run `flutter analyze` + `flutter test` on **every pull request** (CI).
- Aim for high coverage on **business logic**, less on pure UI glue.
- Flaky tests are poison: fix or delete them.

---

## 10. TDD in a nutshell

1. **Red** — write a failing test.
2. **Green** — write minimal code to pass.
3. **Refactor** — clean up, tests stay green.

Great for algorithms and business rules, and a fantastic way to **practice problem solving**.

---

## Common mistakes

- Testing implementation details (private methods) instead of behavior.
- Brittle tests tied to exact text/layout everywhere.
- Using real network/time/random in tests.
- Giant tests doing many things; one behavior per test.
- Over-mocking → tests that only prove the mocks work.
- `pumpAndSettle` with infinite animations (e.g. a spinner) → timeout. Use `pump(duration)`.
- Ignoring failing/flaky tests.

---

## Code challenge

1. **TDD:** write tests first, then implement `String fizzBuzz(int n)` and `bool isValidPassword(String)` (≥ 8 chars, 1 digit, 1 uppercase).
2. Unit-test your `Stack<T>` and `LruCache` from earlier lessons: empty cases, order, capacity eviction.
3. Write a widget test for the registration form: invalid email shows an error; valid inputs call `onSubmit` once.
4. Test a ViewModel with a `FakeRepository`: loading → data; loading → error; retry after error succeeds.
5. Add a golden test for a `ProfileCard` at two text scales.

<details>
<summary>Hint — password test list</summary>

```dart
group('isValidPassword', () {
  test('rejects short', () => expect(isValidPassword('Ab1'), isFalse));
  test('rejects no digit', () => expect(isValidPassword('Abcdefgh'), isFalse));
  test('rejects no uppercase', () => expect(isValidPassword('abcdefg1'), isFalse));
  test('accepts valid', () => expect(isValidPassword('Abcdefg1'), isTrue));
});
```
</details>

---

## Interview questions

1. Explain the test pyramid for a Flutter app.
2. Unit vs widget vs integration tests: what and when?
3. Fake vs stub vs mock?
4. How do you make code testable?
5. `pump` vs `pumpAndSettle`?
6. How do you test code that depends on time or randomness?
7. What are golden tests and what are their downsides?
8. Does 100% coverage mean the app is bug-free?
9. How would you test a screen that fetches data?
10. What makes a test flaky? How do you fix it?
11. What is TDD?

<details>
<summary>Hints</summary>

1. Many unit, some widget, few integration; speed vs realism.
2. Logic / UI behavior in isolation / full flows on a device.
3. Fake: working simplified impl; stub: canned values; mock: verifies interactions.
4. DI, interfaces for I/O, pure functions, small widgets.
5. `pump` = one frame; `pumpAndSettle` = until no more frames scheduled.
6. Inject clock/seeded random.
7. Compare rendered pixels to a baseline; platform/font differences, maintenance.
8. No; asserts must be meaningful and behaviors covered.
9. Override repository with a fake, pump, assert loading/data/error states.
10. Real time/network/randomness, shared state, order dependence; inject fakes, isolate state.
11. Red-green-refactor development.
</details>
