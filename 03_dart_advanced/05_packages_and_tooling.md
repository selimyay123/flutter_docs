# 05 — Packages, Libraries, and Tooling

## Goal

Organize code into files and packages, use community code wisely, and keep code clean with tools.

---

## 1. Libraries and imports

Every `.dart` file is a **library**.

```dart
import 'dart:math';                          // Dart SDK library
import 'package:http/http.dart' as http;     // external package
import 'package:my_app/models/user.dart';    // your own package file (preferred)
import '../utils/helpers.dart';              // relative (OK inside lib/)
```

Options:

```dart
import 'package:foo/foo.dart' show Bar;      // import only Bar
import 'package:foo/foo.dart' hide Baz;      // everything except Baz
import 'package:foo/foo.dart' deferred as foo; // lazy loading (web code splitting)
```

### `export` — make barrel files

```dart
// lib/models/models.dart
export 'user.dart';
export 'product.dart';
```

One import gives you all models.

### `part` / `part of` (rarely, mostly for code-gen)

```dart
part 'user.g.dart';
```

---

## 2. Project layout

```
my_package/
  lib/
    my_package.dart       <- public API (exports)
    src/                  <- private implementation (convention)
      parser.dart
  test/
  bin/                    <- command-line entrypoints
  pubspec.yaml
```

Convention: everything under `lib/src/` is private to the package. Users import only the top-level library.

---

## 3. pub.dev and `pubspec.yaml`

Packages live on <https://pub.dev>. Add one:

```bash
dart pub add collection
flutter pub add provider
```

Version constraints:

| Constraint | Meaning |
|------------|---------|
| `^1.2.3` | `>=1.2.3 <2.0.0` (caret, most common) |
| `any` | Anything (avoid) |
| `1.2.3` | Exactly (avoid) |

`pubspec.lock` pins exact versions. **Commit it for apps, not for libraries.**

Commands:

```bash
flutter pub get          # download deps
flutter pub upgrade      # newest allowed
flutter pub outdated     # what's old
dart pub deps            # dependency tree
```

### Choosing a package (senior checklist)

- Maintained? (recent releases, open issues handled)
- Popular? (likes, downloads, pub points)
- Null-safe & Dart 3 ready?
- Platform support matches yours?
- License compatible?
- Do you really need it? Each dependency is a long-term cost.

---

## 4. The Dart toolchain

| Command | What it does |
|---------|--------------|
| `dart run` | Run a program |
| `dart analyze` | Static analysis (finds bugs and style issues) |
| `dart format .` | Auto-format (ends style debates) |
| `dart test` | Run tests |
| `dart fix --apply` | Auto-fix lint warnings and migrate APIs |
| `dart compile exe` | Native executable |
| `dart doc` | Generate API docs |

### Lints — `analysis_options.yaml`

```yaml
include: package:flutter_lints/flutter.yaml

linter:
  rules:
    prefer_const_constructors: true
    avoid_print: true
    unawaited_futures: true
```

Treat warnings as bugs waiting to happen. Keep the analyzer at **zero issues**.

---

## 5. Useful standard libraries

| Library | Use |
|---------|-----|
| `dart:core` | Auto-imported: `String`, `List`, `DateTime`, `Duration`... |
| `dart:math` | `Random`, `max`, `min`, `sqrt`, `pi` |
| `dart:async` | `Future`, `Stream`, `Timer`, `Completer` |
| `dart:convert` | `jsonEncode`, `jsonDecode`, `utf8`, `base64` |
| `dart:collection` | `Queue`, `LinkedHashMap`, `SplayTreeMap`, `HashSet` |
| `dart:io` | Files, sockets, processes (not on web) |
| `dart:isolate` | Isolates |

```dart
import 'dart:math';
final rnd = Random();
print(rnd.nextInt(10));        // 0..9
```

---

## 6. Documentation comments

```dart
/// Returns the sum of [a] and [b].
///
/// Throws [ArgumentError] if either is negative.
///
/// ```dart
/// add(1, 2); // 3
/// ```
int add(int a, int b) { ... }
```

Good docs explain **why** and **contracts**, not what the code obviously does.

---

## Common mistakes

- Adding a dependency for 5 lines of code.
- Importing from another package's `src/` folder.
- Not committing `pubspec.lock` in an app.
- Ignoring analyzer warnings.
- Circular imports between files.

---

## Code challenge

1. Create a package-style folder `mini_lib/` with `lib/src/stats.dart` containing `mean`, `median`, `mode` for `List<num>` and `lib/mini_lib.dart` exporting it. Import it from a `bin/main.dart` and print results for `[1, 2, 2, 3, 4]`.
2. Add `dart analyze` and `dart format` to your routine. Break a rule on purpose and see the warnings.

<details>
<summary>Solution (core of step 1)</summary>

```dart
// lib/src/stats.dart
double mean(List<num> xs) => xs.fold<num>(0, (a, b) => a + b) / xs.length;

double median(List<num> xs) {
  final s = [...xs]..sort();
  final mid = s.length ~/ 2;
  return s.length.isOdd ? s[mid].toDouble() : (s[mid - 1] + s[mid]) / 2;
}

num mode(List<num> xs) {
  final counts = <num, int>{};
  for (final x in xs) {
    counts[x] = (counts[x] ?? 0) + 1;
  }
  return counts.entries.reduce((a, b) => a.value >= b.value ? a : b).key;
}
```
</details>

---

## Interview questions

1. What is a library in Dart? How does `_` privacy relate to it?
2. What does `^1.2.3` mean in `pubspec.yaml`?
3. Should `pubspec.lock` be committed? For apps or libraries?
4. How do you decide whether to add a third-party package?
5. What is a barrel file?
6. What are lints for, and how do they help a team?

<details>
<summary>Hints</summary>

1. A file; underscore makes a name private to its library.
2. Compatible versions: `>=1.2.3 <2.0.0`.
3. Apps yes (reproducible builds); libraries no.
4. Maintenance, popularity, platform support, license, size, whether you can write it simply yourself.
5. A file that re-exports others, simplifying imports.
6. Automatic detection of bugs and style problems; consistent code with no review debates.
</details>
