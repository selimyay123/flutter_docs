# 05 — Networking and JSON

## Goal

Talk to a server, parse the response safely, and handle every failure mode.

---

## 1. HTTP in 2 minutes

| Method | Meaning | Example |
|--------|---------|---------|
| `GET` | Read | `GET /users/1` |
| `POST` | Create | `POST /users` |
| `PUT` / `PATCH` | Replace / partial update | `PATCH /users/1` |
| `DELETE` | Remove | `DELETE /users/1` |

Status codes: `2xx` success, `3xx` redirect, `4xx` **client** error (400 bad request, 401 unauthenticated, 403 forbidden, 404 not found, 429 rate limit), `5xx` **server** error.

JSON example:

```json
{ "id": 1, "name": "Ada", "email": "ada@x.com", "tags": ["math", "code"] }
```

---

## 2. The `http` package

`flutter pub add http`

```dart
import 'dart:convert';
import 'package:http/http.dart' as http;

Future<Map<String, dynamic>> fetchUser(int id) async {
  final response = await http
      .get(Uri.parse('https://jsonplaceholder.typicode.com/users/$id'))
      .timeout(const Duration(seconds: 10));

  if (response.statusCode != 200) {
    throw HttpException('Failed: ${response.statusCode}');
  }
  return jsonDecode(response.body) as Map<String, dynamic>;
}
```

POST:

```dart
final res = await http.post(
  Uri.parse('https://example.com/users'),
  headers: {'Content-Type': 'application/json'},
  body: jsonEncode({'name': 'Ada'}),
);
```

Build URIs safely (handles encoding):

```dart
Uri.https('api.example.com', '/search', {'q': 'flutter dart', 'page': '2'});
```

**`dio`** is a popular alternative: interceptors (auth tokens, logging, retry), cancellation, uploads with progress, base options.

---

## 3. Model classes: `fromJson` / `toJson`

```dart
class User {
  const User({required this.id, required this.name, required this.email});

  final int id;
  final String name;
  final String email;

  factory User.fromJson(Map<String, dynamic> json) {
    return User(
      id: json['id'] as int,
      name: json['name'] as String,
      email: json['email'] as String,
    );
  }

  Map<String, dynamic> toJson() => {'id': id, 'name': name, 'email': email};
}
```

Parse lists:

```dart
final list = (jsonDecode(body) as List<dynamic>)
    .map((e) => User.fromJson(e as Map<String, dynamic>))
    .toList();
```

### Safer parsing with patterns (Dart 3)

```dart
factory User.fromJson(Map<String, dynamic> json) {
  if (json case {'id': int id, 'name': String name, 'email': String email}) {
    return User(id: id, name: name, email: email);
  }
  throw FormatException('Invalid User JSON: $json');
}
```

### Real-world JSON problems

- Missing/`null` fields → use nullable types or defaults.
- Numbers arriving as `int` **or** `double` → `(json['price'] as num).toDouble()`.
- Dates as strings → `DateTime.parse` / `tryParse`.
- Snake_case keys vs camelCase fields.
- Server adds new fields → ignore unknown ones.
- Never trust the server: validate.

### Code generation

For big models use `json_serializable` + `freezed`:

```dart
@freezed
class User with _$User {
  const factory User({required int id, required String name}) = _User;
  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}
// dart run build_runner build
```

Know how to write it by hand first.

---

## 4. Show it in the UI

### FutureBuilder (simple screens)

```dart
class _UserPageState extends State<UserPage> {
  late final Future<User> _future = repo.getUser(1);      // created once!

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<User>(
      future: _future,
      builder: (context, snapshot) {
        if (snapshot.connectionState != ConnectionState.done) {
          return const Center(child: CircularProgressIndicator());
        }
        if (snapshot.hasError) {
          return Center(child: Text('Error: ${snapshot.error}'));
        }
        final user = snapshot.requireData;
        return Text(user.name);
      },
    );
  }
}
```

For anything more complex (refresh, retry, caching) use your state-management layer: `AsyncValue` in Riverpod, states in Bloc.

---

## 5. Layered networking architecture

```
Widget  →  ViewModel/Notifier/Bloc  →  Repository  →  DataSource (API client)  →  http/dio
                                            │
                                            └── LocalDataSource (cache / DB)
```

```dart
// 1. Low-level API client (knows HTTP & JSON)
class UserApi {
  UserApi(this._client);
  final http.Client _client;

  Future<User> getUser(int id) async {
    final res = await _client.get(Uri.parse('$baseUrl/users/$id'));
    if (res.statusCode != 200) throw ApiException(res.statusCode);
    return User.fromJson(jsonDecode(res.body) as Map<String, dynamic>);
  }
}

// 2. Repository (the app's view of data; can combine API + cache)
abstract interface class UserRepository {
  Future<User> getUser(int id);
}

class UserRepositoryImpl implements UserRepository {
  UserRepositoryImpl(this._api, this._cache);
  final UserApi _api;
  final Map<int, User> _cache;

  @override
  Future<User> getUser(int id) async {
    final cached = _cache[id];
    if (cached != null) return cached;
    final user = await _api.getUser(id);
    return _cache[id] = user;
  }
}
```

Why layers? Swap the API for a fake in tests, add caching without touching the UI, and keep widgets free from HTTP details.

---

## 6. Failure handling (be thorough)

Things that go wrong: no internet, timeout, server error, 401 expired token, bad JSON, rate limit.

```dart
sealed class AppException implements Exception {
  const AppException(this.message);
  final String message;
}
class NetworkException extends AppException { const NetworkException() : super('No connection'); }
class TimeoutAppException extends AppException { const TimeoutAppException() : super('Request timed out'); }
class ServerException extends AppException { const ServerException(this.code) : super('Server error'); final int code; }
class ParseException extends AppException { const ParseException() : super('Unexpected response'); }

Future<T> guard<T>(Future<T> Function() call) async {
  try {
    return await call();
  } on SocketException {
    throw const NetworkException();
  } on TimeoutException {
    throw const TimeoutAppException();
  } on FormatException {
    throw const ParseException();
  }
}
```

(`SocketException` is from `dart:io`; on web it's different — packages like `dio` normalize errors.)

### Retry with exponential back-off

Retry only **idempotent** requests (GET) or those with an idempotency key. Wait 0.5s, 1s, 2s... with jitter. Never hammer a failing server.

### Authentication

- Send `Authorization: Bearer <token>` header.
- Store tokens in **secure storage** (`flutter_secure_storage`), never in plain prefs or source code.
- On `401`: refresh the token once, retry the request; if refresh fails → logout. (An interceptor in `dio` does this elegantly.)
- Never hard-code API keys in the app. They can be extracted. Put secrets on a server.

### Cancellation & stale responses

If a user types quickly, an older request can return **after** a newer one. Keep a request id or use cancellation tokens (`dio` CancelToken) and ignore stale results.

---

## 7. Caching and offline

| Strategy | Idea |
|----------|------|
| In-memory cache | Fast, lost on restart |
| HTTP cache headers (ETag, Cache-Control) | Server-driven |
| Disk cache (DB / files) | Survives restart |
| **Stale-while-revalidate** | Show cached data instantly, refresh in the background |
| Offline queue | Store writes locally; sync when online |

---

## 8. Other protocols

- **WebSockets** (`web_socket_channel`) — real-time bidirectional (chat, live prices). Exposes a `Stream`.
- **GraphQL** (`graphql_flutter`, `ferry`), **gRPC**, **Firebase SDKs**.
- **Server-Sent Events** — one-way live updates.

---

## 9. Testing network code

Never call the real internet in tests. Inject `http.Client` and use `MockClient`:

```dart
import 'package:http/testing.dart';

test('parses user', () async {
  final client = MockClient((request) async {
    return http.Response('{"id":1,"name":"Ada","email":"a@x.com"}', 200);
  });
  final api = UserApi(client);
  final user = await api.getUser(1);
  expect(user.name, 'Ada');
});
```

---

## Common mistakes

- Parsing JSON in the UI layer.
- Creating a future inside `build`.
- Ignoring status codes (parsing an error body as a User).
- No timeout.
- Hard-coded secrets or tokens in code/logs.
- Not handling `null`/missing fields → runtime crashes.
- Parsing huge JSON on the UI isolate (use `compute`/`Isolate.run`).
- Showing raw exceptions to users.

---

## Code challenge

Build a **Posts app** using `https://jsonplaceholder.typicode.com/posts`:

1. `Post` model with `fromJson`/`toJson` (hand-written, pattern-based).
2. `PostApi` (HTTP only) → `PostRepository` (interface) → state holder (your choice).
3. UI with loading / error (+ Retry button) / empty / data states, pull-to-refresh.
4. Pagination with `?_page=1&_limit=20`.
5. Unit test the repository with a `MockClient` for: success, 500 error, malformed JSON.
6. Extra: add an in-memory cache with stale-while-revalidate.

<details>
<summary>Hint — pagination guard</summary>

```dart
Future<void> loadMore() async {
  if (_loading || !_hasMore) return;
  _loading = true;
  try {
    final page = await repo.getPosts(page: _page, limit: 20);
    _items.addAll(page);
    _hasMore = page.length == 20;
    _page++;
  } finally {
    _loading = false;
  }
}
```
</details>

---

## Interview questions

1. Walk me through making an API call and showing the result in a Flutter screen.
2. Why use a repository layer between UI and API?
3. How do you parse JSON safely? What goes wrong with `as` casts?
4. How do you handle different failures (offline, timeout, 401, 500)?
5. What is exponential back-off? Which requests are safe to retry?
6. How do you store an auth token securely? Can you hide an API key inside the app?
7. How do you avoid showing stale results from an older request?
8. What is stale-while-revalidate?
9. How would you test code that calls the network?
10. Why parse big JSON in an isolate?

<details>
<summary>Hints</summary>

1. Widget → state holder → repository → API client → parse into model → state → UI states.
2. Decouple, testable, central caching, swap data sources.
3. Validate structure (patterns/try-catch); invalid casts throw at runtime.
4. Map to typed app exceptions/states; offer retry; refresh tokens on 401.
5. Increasing delays + jitter; idempotent (GET, or with idempotency keys).
6. Secure storage (Keychain/Keystore). No: client apps can be inspected; keep secrets on a server.
7. Cancel or ignore using request identity/latest-only transformers.
8. Show cache immediately, update from network afterward.
9. Inject the client and mock responses.
10. Prevent UI jank from CPU-heavy parsing.
</details>
