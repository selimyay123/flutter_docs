# 05 — Release and CI/CD

## Goal

Ship your app: build, sign, test automatically, and publish to the stores safely.

---

## 1. Build modes

| Mode | Command | Use |
|------|---------|-----|
| **Debug** | `flutter run` | Development; hot reload; asserts; slow |
| **Profile** | `flutter run --profile` | Performance testing; close to release |
| **Release** | `flutter build ...` | Production; AOT-compiled, optimized, small |

---

## 2. Before release: checklist

- [ ] App name, **bundle ID / application ID**, version, icons, splash set.
- [ ] Permissions: only the ones you use; **usage strings** for iOS.
- [ ] Remove debug banners/logs/test endpoints.
- [ ] Environment config points to **production** (flavors / `--dart-define`).
- [ ] Error reporting + analytics set up.
- [ ] Privacy policy & data safety forms.
- [ ] Tested on real low-end and high-end devices, different OS versions.
- [ ] Accessibility checks.
- [ ] Deep links, push notifications, in-app purchases verified.
- [ ] Dependencies updated; no known vulnerabilities.

---

## 3. Versioning

`pubspec.yaml`:

```yaml
version: 1.4.2+27      # versionName 1.4.2, build number (versionCode) 27
```

- `1.4.2` — **semantic version** `MAJOR.MINOR.PATCH`: breaking / feature / fix.
- `+27` — **build number**; must **increase** with every upload to a store.

Override at build time: `flutter build apk --build-name=1.4.2 --build-number=27`.

---

## 4. Android release

### App identity & icons

- `applicationId` in `android/app/build.gradle` (cannot change after publishing).
- Launcher icons: `flutter_launcher_icons`. Splash: `flutter_native_splash`.

### Signing

1. Create an upload keystore:

```bash
keytool -genkey -v -keystore ~/upload-keystore.jks -keyalg RSA -keysize 2048 -validity 10000 -alias upload
```

2. Reference it in `android/key.properties` (**never commit it or the keystore**) and configure `signingConfigs` in Gradle.

### Build

```bash
flutter build appbundle --release                  # .aab for Google Play (preferred)
flutter build apk --release --split-per-abi        # smaller APKs per CPU architecture
```

### Publish

Google Play Console → create app → upload `.aab` → store listing, content rating, data safety → testing tracks (internal → closed → open) → production with **staged rollout** (e.g. 5% → 20% → 100%).

Enroll in **Play App Signing** (Google holds the app signing key; you keep an upload key that can be reset).

---

## 5. iOS release

Requires a Mac, Xcode, and an **Apple Developer Program** membership.

1. Set bundle identifier, team, signing (automatic signing is easiest) in Xcode.
2. Add icons, launch screen, `Info.plist` usage descriptions (`NSCameraUsageDescription`, etc.).
3. Build:

```bash
flutter build ipa --release
```

4. Upload via Xcode Organizer / `xcrun altool` / **Transporter** / Fastlane.
5. **TestFlight** for beta testing → App Store Connect: metadata, screenshots, privacy → submit for **review**.

Common rejection reasons: crashes, broken links, missing privacy info, using private APIs, login required without demo account, misleading metadata, not enough "app-like" functionality.

---

## 6. Web, desktop

```bash
flutter build web --release          # outputs build/web (host anywhere: Firebase Hosting, Netlify, S3...)
flutter build macos | windows | linux
```

Web specifics: pick a renderer strategy (`--wasm` support for performance where available), set base href (`--base-href /app/`), caching headers, SEO is limited (Flutter web is app-oriented).

---

## 7. Obfuscation and symbols

```bash
flutter build appbundle --obfuscate --split-debug-info=build/symbols
```

- Makes reverse engineering harder (not impossible).
- **Keep the `symbols` folder** for each release; you need it to de-obfuscate crash stack traces. Upload symbols to Crashlytics/Sentry in CI.
- Never rely on obfuscation for secrets.

---

## 8. Flavors / environments

Separate **dev / staging / prod** with different app IDs, icons, API URLs, Firebase projects.

```bash
flutter run --flavor dev -t lib/main_dev.dart
flutter build appbundle --flavor prod -t lib/main_prod.dart --dart-define-from-file=env/prod.json
```

Install dev and prod side by side on one device (different app IDs). Reduces "I tested against production by accident" incidents.

---

## 9. CI/CD — automate it

**CI (Continuous Integration):** on every push/PR, automatically verify the code.
**CD (Continuous Delivery/Deployment):** automatically build and distribute.

### GitHub Actions example (`.github/workflows/ci.yml`)

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: subosito/flutter-action@v2
        with:
          channel: stable
          cache: true
      - run: flutter pub get
      - run: dart format --output=none --set-exit-if-changed .
      - run: flutter analyze
      - run: flutter test --coverage
```

### Delivery pipeline sketch

```
PR → format + analyze + test → merge to main → build (android + ios) → sign
   → upload to Firebase App Distribution / TestFlight / Play internal track
   → manual promote → staged production rollout
```

Tools: **GitHub Actions**, **Codemagic** (Flutter-focused), **Bitrise**, **Fastlane** (store uploads, signing automation).

**Secrets** (keystore, API keys, certificates) go in the CI secret store, never in the repo.

---

## 10. Release strategy

| Practice | Why |
|----------|-----|
| **Staged rollout** | Limit blast radius of a bad release |
| **Feature flags / remote config** | Turn features on/off without a store update |
| **Crash-free-rate monitoring** | Halt rollout if it drops |
| **Force-update mechanism** | Retire dangerously old versions (min supported version from remote config) |
| **Changelog** (`CHANGELOG.md`) | Communicate changes |
| **Git tags** (`v1.4.2`) | Reproducible builds |
| **Hotfix branch process** | Fast, safe fixes |
| **Over-the-air updates** | Not available for compiled Flutter code (store policies); use remote config for behavior |

---

## 11. After release

- Watch crash rates, ANRs, startup time, store reviews.
- Respond to reviews.
- Keep up with new OS versions & store policy deadlines (target SDK requirements!).
- Keep dependencies updated regularly (small steps beat big-bang upgrades).
- Run `flutter upgrade` on a branch, check breaking changes.

---

## Common mistakes

- Losing the signing keystore or committing it.
- Forgetting to bump the build number → upload rejected.
- Shipping with debug/test endpoints.
- No staged rollout, no monitoring.
- Not saving obfuscation symbols.
- Releasing without testing the **release** build (it can behave differently from debug).
- Secrets in the repository.

---

## Code challenge

1. Set up a GitHub repo for your capstone app with a CI workflow that runs **format check, analyze, and tests** on each PR (make it fail on purpose, then fix).
2. Create **dev** and **prod** flavors with different app names and a visible environment banner.
3. Produce a signed Android release `.aab` and install the release APK on a real phone; compare performance with debug.
4. Write a `RELEASING.md` with the exact steps from "version bump" to "100% rollout" and rollback plan.
5. Implement a **minimum supported version** check using a (fake) remote config JSON, and show a blocking "Please update" screen.

<details>
<summary>Hint — version comparison</summary>

Use `package_info_plus` for the current version, and compare semantic versions numerically (`1.10.0 > 1.9.0` — don't compare as strings!):

```dart
List<int> parse(String v) => v.split('.').map(int.parse).toList();
bool isOlder(String a, String b) {
  final x = parse(a), y = parse(b);
  for (var i = 0; i < 3; i++) {
    if (x[i] != y[i]) return x[i] < y[i];
  }
  return false;
}
```
</details>

---

## Interview questions

1. Debug vs profile vs release builds?
2. What do `version: 1.4.2+27` mean? Which must increase on each store upload?
3. How do you sign an Android app? What happens if you lose the key?
4. How do you protect secrets in the CI pipeline and in the app?
5. What is a staged rollout and why use it?
6. What are flavors and why use them?
7. What does obfuscation do? What must you keep?
8. Describe a good CI pipeline for a Flutter project.
9. How do you roll back a bad release?
10. What is a force-update mechanism?

<details>
<summary>Hints</summary>

1. Debug: JIT/hot reload; profile: AOT with tooling for perf; release: optimized, no tooling.
2. Version name 1.4.2, build number 27; build number.
3. Upload keystore + Play App Signing; losing an upload key can be reset via Google; without Play App Signing you can't update the app.
4. CI secret store, not in the repo; the app can't keep true secrets, so use a backend.
5. Gradually expose to users, halt if metrics worsen.
6. Environment-specific app variants (IDs, icons, endpoints).
7. Renames symbols to make decompilation harder; keep symbol files for crash reports.
8. Format, analyze, test, build, sign, distribute, with caching and secrets management.
9. Halt rollout, release a hotfix with a higher build number, or disable via feature flags (stores don't allow instant rollbacks).
10. Remote minimum version triggers a blocking update screen.
</details>
