# 03 — Platform Channels and Native Integration

## Goal

Call native Android/iOS code from Dart when Flutter alone is not enough. Know the options and their trade-offs.

---

## 1. When do you need native code?

Flutter covers UI. For device features you use **plugins** (packages with native code): camera, location, notifications, biometrics, Bluetooth, payments.

Write your own native bridge when:

- A plugin doesn't exist or is poorly maintained.
- You need a native SDK (payment, maps, ML) with no Flutter wrapper.
- You must integrate with an existing native app/module.

Always check pub.dev **first**.

---

## 2. Options

| Mechanism | Use |
|-----------|-----|
| **Plugins** (pub.dev) | 95% of cases |
| **MethodChannel** | Call a native method, get a result (async) |
| **EventChannel** | Native → Dart **stream** of events (sensors, connectivity) |
| **BasicMessageChannel** | Custom message codec |
| **Pigeon** | **Generates type-safe** channel code (recommended) |
| **FFI** (`dart:ffi`) | Call C/C++/Rust libraries directly, fast, synchronous |
| **Platform views** | Embed a native view (map, webview) inside Flutter UI |
| **Add-to-app** | Flutter module inside an existing native app |

---

## 3. MethodChannel — how it works

```
Dart  ──(binary message, async)──►  Native handler
Dart  ◄──────────(result)─────────  Native handler
```

Messages are serialized by a **codec** (StandardMessageCodec): `null`, `bool`, `int`, `double`, `String`, `Uint8List`, `List`, `Map`.

### Dart side

```dart
class BatteryService {
  static const _channel = MethodChannel('com.example.app/battery');

  Future<int> getBatteryLevel() async {
    try {
      final level = await _channel.invokeMethod<int>('getBatteryLevel');
      return level ?? -1;
    } on PlatformException catch (e) {
      throw BatteryException(e.message ?? 'Unknown error');
    } on MissingPluginException {
      throw const BatteryException('Not supported on this platform');
    }
  }
}
```

### Android (Kotlin) — `MainActivity.kt`

```kotlin
class MainActivity : FlutterActivity() {
    private val channel = "com.example.app/battery"

    override fun configureFlutterEngine(flutterEngine: FlutterEngine) {
        super.configureFlutterEngine(flutterEngine)
        MethodChannel(flutterEngine.dartExecutor.binaryMessenger, channel)
            .setMethodCallHandler { call, result ->
                if (call.method == "getBatteryLevel") {
                    val bm = getSystemService(BATTERY_SERVICE) as BatteryManager
                    val level = bm.getIntProperty(BatteryManager.BATTERY_PROPERTY_CAPACITY)
                    if (level >= 0) result.success(level)
                    else result.error("UNAVAILABLE", "Battery level not available", null)
                } else {
                    result.notImplemented()
                }
            }
    }
}
```

### iOS (Swift) — `AppDelegate.swift`

```swift
let controller = window?.rootViewController as! FlutterViewController
let channel = FlutterMethodChannel(name: "com.example.app/battery",
                                   binaryMessenger: controller.binaryMessenger)
channel.setMethodCallHandler { call, result in
  guard call.method == "getBatteryLevel" else { result(FlutterMethodNotImplemented); return }
  UIDevice.current.isBatteryMonitoringEnabled = true
  let level = Int(UIDevice.current.batteryLevel * 100)
  if level < 0 { result(FlutterError(code: "UNAVAILABLE", message: "Battery info unavailable", details: nil)) }
  else { result(level) }
}
```

Rules:

- Use a **unique channel name** (reverse-domain).
- Handlers run on the **main (platform) thread**. Do heavy native work on a background thread, then call `result` (thread rules differ per platform).
- Always handle `PlatformException` and `MissingPluginException` in Dart.
- Wrap channels in a Dart **service class** behind an interface; widgets never touch channels.

---

## 4. EventChannel — streams from native

```dart
const _events = EventChannel('com.example.app/compass');

Stream<double> get heading =>
    _events.receiveBroadcastStream().map((e) => e as double);
```

Native side registers a `StreamHandler` with `onListen` (start sensor) and `onCancel` (stop sensor). Always stop native work in `onCancel` to avoid leaks/battery drain.

---

## 5. Pigeon — type-safe, generated

Define an interface in Dart:

```dart
// pigeons/messages.dart
@HostApi()
abstract class BatteryApi {
  int getBatteryLevel();
}
```

Run Pigeon → generates Dart, Kotlin, Swift code. You implement `BatteryApi` natively and call it from Dart with real types: no string method names, no manual casting. Use it for anything beyond one or two methods.

---

## 6. FFI — talk to C directly

```dart
import 'dart:ffi';

typedef NativeAdd = Int32 Function(Int32, Int32);
typedef DartAdd = int Function(int, int);

final lib = DynamicLibrary.open('libmath.so');       // platform specific name
final add = lib.lookupFunction<NativeAdd, DartAdd>('add');
print(add(2, 3));
```

Use when you need: performance, existing C/C++/Rust libs (crypto, codecs, ML), no async overhead. Tools: `ffigen` generates bindings from C headers; `flutter_rust_bridge` for Rust.

Calls are **synchronous** on the calling isolate: run heavy ones in a background isolate.

---

## 7. Platform views

Embed native UI (Google Maps, WebView, ads) inside Flutter via `AndroidView` / `UiKitView` (or `HybridComposition`). They have **performance and gesture trade-offs**, so use well-maintained plugins (`webview_flutter`, `google_maps_flutter`).

---

## 8. Writing a plugin package

```bash
flutter create --template=plugin --platforms=android,ios my_plugin
```

Structure: `lib/` (Dart API + platform interface), `android/`, `ios/`, `example/` app. Follow the **federated plugin** pattern for multi-platform support: `my_plugin` (app-facing), `my_plugin_platform_interface`, `my_plugin_android`, `my_plugin_ios`.

---

## 9. Permissions, lifecycle, and safety

- Ask permissions **at the moment of need**, explain why (`permission_handler`), and handle "denied" and "permanently denied" gracefully (guide to Settings).
- Declare permissions: `AndroidManifest.xml`, `Info.plist` (usage description strings are mandatory on iOS).
- Native code must respect the **app lifecycle** (stop sensors in background).
- Treat data from channels as **untrusted**: validate types.
- Test the Dart side with a mock channel:

```dart
TestWidgetsFlutterBinding.ensureInitialized();

setUp(() {
  TestDefaultBinaryMessengerBinding.instance.defaultBinaryMessenger
      .setMockMethodCallHandler(const MethodChannel('com.example.app/battery'), (call) async {
    if (call.method == 'getBatteryLevel') return 87;
    return null;
  });
});

test('reads battery', () async => expect(await BatteryService().getBatteryLevel(), 87));
```

---

## 10. Decision guide

```
Need a device feature?
 ├─ A good plugin exists?            → use it
 ├─ Simple one-off native call?      → MethodChannel (+ Pigeon if >2 methods)
 ├─ Continuous native events?        → EventChannel
 ├─ C/C++/Rust library, performance? → FFI
 └─ Embed native UI?                 → Platform view (prefer existing plugin)
```

---

## Common mistakes

- Reinventing a plugin that already exists.
- Stringly-typed method names scattered everywhere (use Pigeon / one wrapper).
- Forgetting `notImplemented` / error results (Dart call hangs or throws `MissingPluginException`).
- Doing heavy work on the platform main thread.
- Not stopping native streams on cancel.
- Missing iOS `Info.plist` usage strings → crash/rejection.
- Putting channel calls directly in widgets (untestable).

---

## Code challenge

1. Implement a **`DeviceInfoService`** with a MethodChannel returning the device model on Android and iOS (you may use Kotlin and Swift snippets from the docs). Wrap it in an interface, add a fake, and unit test with a mock channel.
2. (Optional) Replace it with **Pigeon**.
3. (Optional, advanced) Write a tiny C function `int add(int, int)` and call it via **FFI** on desktop.
4. Read the source of a small plugin on pub.dev (e.g. `url_launcher`) and draw how its platform interface is organized.

<details>
<summary>Hint — interface + fake</summary>

```dart
abstract interface class DeviceInfoService {
  Future<String> model();
}
class FakeDeviceInfoService implements DeviceInfoService {
  @override
  Future<String> model() async => 'Test Phone';
}
```
</details>

---

## Interview questions

1. How does Flutter communicate with native code?
2. `MethodChannel` vs `EventChannel`?
3. What types can pass through a standard channel?
4. What is Pigeon and why use it?
5. When would you pick FFI over a MethodChannel?
6. What is a platform view and what are its costs?
7. How do you structure and test code that uses a channel?
8. What errors must you handle on the Dart side?
9. How should permissions be handled?

<details>
<summary>Hints</summary>

1. Asynchronous binary messages over named channels with codecs (plus FFI for C).
2. One-shot calls with results vs continuous streams from native.
3. null, bool, numbers, String, byte/num lists, List, Map.
4. Generates type-safe interfaces; removes string names and casts.
5. Native library access, performance, synchronous calls without message overhead.
6. Embeds native UI; gesture/composition/performance trade-offs.
7. Wrap in a service behind an interface; mock the channel in tests.
8. `PlatformException`, `MissingPluginException`.
9. Request in context, handle denial/permanent denial, declare in manifests, explain usage.
</details>
