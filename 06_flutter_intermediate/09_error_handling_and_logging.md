# 09 — Error Handling and Logging in Flutter

## Goal

Make the app **fail gracefully**, and make failures **visible to you** in production.

---

## 1. Where errors come from in Flutter

| Source | Caught by default? |
|--------|--------------------|
| Exceptions during **build/layout/paint** | Yes: `FlutterError.onError` (red screen in debug, grey box in release) |
| **Event handler** exceptions (`onPressed`) | Reported via `FlutterError.onError` |
| **Async** errors not caught (unawaited futures, timers, streams) | **No** — go to the `Zone` / `PlatformDispatcher.onError` |
| **Native** crashes (platform code) | Not by Dart handlers; need Crashlytics native SDK |
| Errors in **other isolates** | Must be forwarded manually |

---

## 2. Global error hooks

```dart
void main() {
  // 1) Errors inside the Flutter framework
  FlutterError.onError = (FlutterErrorDetails details) {
    FlutterError.presentError(details);               // still prints in debug
    reportError(details.exception, details.stack);
  };

  // 2) Uncaught async errors (also covers errors outside the framework)
  PlatformDispatcher.instance.onError = (error, stack) {
    reportError(error, stack);
    return true;                                      // handled
  };

  runApp(const MyApp());
}
```

Crash reporting services: **Firebase Crashlytics**, **Sentry**, etc. They give stack traces, device info, breadcrumbs, and release health.

### Custom error widget

```dart
ErrorWidget.builder = (details) => const Material(
      child: Center(child: Text('Something went wrong')),
    );
```

(Show details only in debug.)

---

## 3. Layered handling strategy

```
Service/API   → throws low-level exceptions
Repository    → catches, converts to AppException / Result
ViewModel     → converts to UI state (message, canRetry)
View          → shows friendly text + retry
Global hooks  → catch what everything else missed → report
```

**Rule:** catch an error **where you can do something useful** about it. Otherwise let it bubble.

---

## 4. Result type vs exceptions (revisited)

For **expected** failures in the data layer, prefer returning a `Result`:

```dart
sealed class Result<T> {
  const Result();
}
final class Success<T> extends Result<T> { const Success(this.value); final T value; }
final class Failure<T> extends Result<T> { const Failure(this.error); final AppError error; }

Future<Result<User>> login(String email, String password) async {
  try {
    final user = await _api.login(email, password);
    return Success(user);
  } on ApiException catch (e) {
    return Failure(AppError.fromApi(e));
  }
}
```

The compiler forces callers to handle both branches. Use exceptions for **truly exceptional** and programmer errors.

---

## 5. Showing errors to users

| Situation | UI |
|-----------|----|
| Whole screen failed to load | Full-screen message + **Retry** |
| A refresh failed but old data exists | Keep data, show **snackbar** |
| Field validation | Inline error text under the field |
| Action failed (save) | Snackbar/dialog + keep user input |
| Fatal/unknown | Friendly generic message + report |

Message guidelines:

1. Say **what happened** in plain language.
2. Say **what the user can do**.
3. Don't expose stack traces, URLs, SQL, or internal codes (log them instead).
4. Keep the user's work (never clear a form on failure).
5. Localize messages.

---

## 6. Logging

`print` is for quick experiments only (avoid in production code; the `avoid_print` lint flags it).

```dart
import 'dart:developer' as dev;

dev.log('Fetching user', name: 'UserRepo', error: e, stackTrace: st, level: 1000);
```

Or use `package:logging` / `logger`:

```dart
final _log = Logger('UserRepository');
_log.info('Loading user $id');
_log.warning('Retrying request', error, stack);
_log.severe('Failed to parse', error, stack);
```

### Logging levels

| Level | Use |
|-------|-----|
| `fine` / `debug` | Developer details (off in prod) |
| `info` | Normal milestones |
| `warning` | Something odd but handled |
| `severe` / `error` | Failures to investigate |

### What to log (and not)

Do log: operation, ids, durations, error types, app version, **breadcrumbs** (last user actions).
**Never** log: passwords, tokens, full credit card numbers, personal data (privacy laws like GDPR/KVKK).

---

## 7. Assertions and invariants

```dart
assert(items.isNotEmpty, 'List must not be empty');   // debug only
```

For conditions that must **never** be wrong in production too, throw `StateError`/`ArgumentError` explicitly.

---

## 8. Observability: metrics, tracing

Beyond crashes, monitor:

- **Startup time**, **frame times** (jank), memory.
- **Network** failures by endpoint.
- **Funnel** events (analytics): where users abandon.
- **Non-fatal errors** (caught but important): `recordError(e, st, fatal: false)`.

Use feature flags/remote config to disable a broken feature without a store release.

---

## 9. Debugging tools

- **DevTools**: Inspector, Performance, Memory, Network, Logging, CPU profiler.
- `debugPrint`, `debugger()`, breakpoints, conditional breakpoints, watch expressions.
- `flutter run --profile` for realistic performance.
- `flutter analyze` and lints.
- Reproduce with the **exact** device/OS/build mode.

---

## 10. Unhandled async error — example and fix

```dart
void onTap() {
  repo.save(data);               // returns a Future; if it throws → unhandled error
}

// Fix 1: await in an async handler and handle
Future<void> onTap() async {
  try {
    await repo.save(data);
  } on AppException catch (e) {
    if (mounted) showError(e);
  }
}

// Fix 2: intentionally fire-and-forget, but safe
unawaited(repo.save(data).catchError((e, st) => reportError(e, st)));
```

---

## Common mistakes

- Empty `catch` blocks.
- Catching `Exception` everywhere and showing `e.toString()` to users.
- Logging secrets.
- No crash reporting in production.
- Testing only the happy path.
- Clearing user input after an error.
- Letting errors in streams silently kill the subscription (handle `onError`; use `cancelOnError: false` if appropriate).

---

## Code challenge

1. Set up `FlutterError.onError` and `PlatformDispatcher.instance.onError` to send errors to a **fake** `ErrorReporter` (interface) that prints a structured report. Trigger errors from a build method, a button, a timer, and an unawaited future; verify each reaches the reporter.
2. Create `AppError` (sealed) with `network`, `timeout`, `unauthorized`, `server`, `unknown`, and a function mapping any exception to a **user message**.
3. Add a `Retry` button in an error view that re-runs the failed action.
4. Write a tiny `Logger` wrapper with levels and a redaction function that masks anything that looks like `Bearer xxx` or an email.

<details>
<summary>Hint — redaction</summary>

```dart
String redact(String s) => s
    .replaceAll(RegExp(r'Bearer\s+[A-Za-z0-9\-._~+/]+=*'), 'Bearer ***')
    .replaceAll(RegExp(r'[\w.+-]+@[\w-]+\.[\w.-]+'), '***@***');
```
</details>

---

## Interview questions

1. Which errors does `FlutterError.onError` catch, and which does it not?
2. How do you catch uncaught asynchronous errors globally?
3. Where in the architecture do you catch exceptions, and why there?
4. How do you present errors to users?
5. What should you never log?
6. `assert` vs throwing exceptions?
7. How do you find the cause of a crash that only happens in production?
8. What is a non-fatal error and why record it?
9. How do you handle errors in streams?

<details>
<summary>Hints</summary>

1. Framework errors (build/layout/paint/handlers); not uncaught async errors or native crashes.
2. `PlatformDispatcher.instance.onError` (or `runZonedGuarded`).
3. Repositories convert; view models map to UI state; global hooks as last resort.
4. Friendly, actionable, no internals, keep user's input, offer retry.
5. Secrets, tokens, PII.
6. Asserts are debug-only checks; exceptions run in production.
7. Crash reporting with stack traces (symbols/obfuscation maps), breadcrumbs, device info, reproduce with the same build.
8. Handled problems that still indicate bugs or degraded experiences.
9. `onError` callbacks, `handleError`, try/catch with `await for`, decide on cancel behavior.
</details>
