# 02 — Setup Your Environment

## Goal

Run Dart and Flutter code on your Mac.

> You can start the Dart lessons with **no install** using [DartPad](https://dartpad.dev). Install Flutter when you reach folder `05`.

---

## 1. What are Dart and Flutter?

| Name | What it is |
|------|------------|
| **Dart** | A programming language (by Google). Typed, fast, easy to read. |
| **Flutter** | A UI toolkit. You write Dart, Flutter draws apps for iOS, Android, web, desktop. |

Flutter *needs* Dart. Dart does *not* need Flutter (you can write servers and scripts too).

---

## 2. Install Flutter (macOS)

Follow the official guide, it is always the freshest: <https://docs.flutter.dev/get-started/install>

Summary:

1. Install **Xcode** from the App Store (for iOS simulator). Run once and accept the license.
2. Install **Android Studio** (for Android emulator and SDK).
3. Install **VS Code** (or use Android Studio) + the **Flutter** extension.
4. Install the Flutter SDK (the guide gives you a download or you can use VS Code to install it).
5. Add Flutter to your `PATH`.
6. Run:

```bash
flutter doctor
```

`flutter doctor` tells you what is missing. Fix each item until you see green checks.

Dart is **included** inside Flutter. Check:

```bash
dart --version
flutter --version
```

---

## 3. Your first commands

```bash
# Run a single Dart file
dart run hello.dart

# Create a Flutter app
flutter create my_first_app
cd my_first_app

# Run it (pick a device)
flutter devices
flutter run
```

### Hot reload vs hot restart

| Action | Key (terminal) | What happens |
|--------|----------------|--------------|
| Hot reload | `r` | Injects new code, **keeps** app state. Very fast. |
| Hot restart | `R` | Restarts app, **resets** state. |
| Quit | `q` | Stops the app. |

Use hot reload for UI tweaks. Use hot restart when you change `main()` or global state.

---

## 4. Project structure (Flutter app)

```
my_first_app/
  lib/            <- YOUR code lives here (main.dart is the entry)
  test/           <- tests
  android/        <- Android native project
  ios/            <- iOS native project
  web/ macos/ ... <- other platforms
  pubspec.yaml    <- project config: name, dependencies, assets
  analysis_options.yaml <- lint rules
```

### `pubspec.yaml` in 30 seconds

```yaml
name: my_first_app
environment:
  sdk: ^3.5.0        # Dart version range
dependencies:
  flutter:
    sdk: flutter
  http: ^1.2.0       # a package you added
dev_dependencies:
  flutter_test:
    sdk: flutter
  flutter_lints: ^4.0.0
flutter:
  uses-material-design: true
```

Add a package:

```bash
flutter pub add http
```

---

## 5. Good habits from day one

- Use **Git** from the start: `git init`, commit often.
- Keep the analyzer happy: `dart analyze` (or watch the red/yellow lines in the editor).
- Format automatically: `dart format .`
- Learn 3 shortcuts in your editor: *quick fix*, *go to definition*, *rename symbol*.

---

## Code challenge

1. Create `hello.dart` with a `main()` that prints your name and today's year.
2. Run it with `dart run hello.dart`.
3. Run `flutter create challenge_app`, start it, and change the title text on the screen using hot reload.

<details>
<summary>Solution (step 1)</summary>

```dart
void main() {
  final now = DateTime.now();
  print('Hi, I am learning Dart in ${now.year}!');
}
```
</details>

---

## Interview questions

1. What is the difference between Dart and Flutter?
2. What does `flutter doctor` do?
3. Hot reload vs hot restart: when do you need each?
4. What is `pubspec.yaml`?
5. Where does your app code live in a Flutter project, and what is the entry point?

<details>
<summary>Hints</summary>

1. Dart = language. Flutter = UI framework written in Dart.
2. Checks your toolchain (SDK, IDEs, devices, licenses) and reports problems.
3. Reload keeps state and is for UI changes; restart resets state and is needed for `main()`, global variables, `initState` logic changes.
4. The manifest: name, SDK constraints, dependencies, assets, fonts.
5. `lib/`, entry is `lib/main.dart` → `main()`.
</details>
