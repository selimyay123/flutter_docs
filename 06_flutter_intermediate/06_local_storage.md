# 06 — Local Storage

## Goal

Save data on the device: settings, tokens, offline content. Choose the right tool.

---

## 1. Options at a glance

| Need | Tool | Notes |
|------|------|-------|
| Small settings (theme, onboarding done) | `shared_preferences` | Key-value, not secure, async |
| Secrets (tokens, passwords) | `flutter_secure_storage` | Keychain (iOS) / Keystore (Android) |
| Structured data, queries, relations | **SQLite**: `drift` or `sqflite` | SQL, migrations |
| Fast object store, no SQL | `isar` / `hive` / `objectbox` | Check each package's maintenance status |
| Files (images, documents, cache) | `path_provider` + `dart:io` | App documents / temp dirs |
| Cloud + offline sync | Firebase Firestore, Supabase, etc. | Server-managed |

Choose by **data shape and size**, not by popularity.

---

## 2. shared_preferences

`flutter pub add shared_preferences`

```dart
final prefs = await SharedPreferences.getInstance();

await prefs.setBool('onboarded', true);
await prefs.setString('theme', 'dark');
await prefs.setStringList('recent', ['a', 'b']);

final onboarded = prefs.getBool('onboarded') ?? false;   // values may be null
await prefs.remove('theme');
```

Only for **small** primitives. Not for lists of objects, not for secrets, not for large data.

### Wrap it in a repository (testable)

```dart
abstract interface class SettingsRepository {
  Future<ThemeMode> loadThemeMode();
  Future<void> saveThemeMode(ThemeMode mode);
}

class PrefsSettingsRepository implements SettingsRepository {
  PrefsSettingsRepository(this._prefs);
  final SharedPreferences _prefs;

  @override
  Future<ThemeMode> loadThemeMode() async {
    final name = _prefs.getString('themeMode');
    return ThemeMode.values.asNameMap()[name] ?? ThemeMode.system;
  }

  @override
  Future<void> saveThemeMode(ThemeMode mode) => _prefs.setString('themeMode', mode.name);
}
```

Widgets never touch `SharedPreferences` directly.

---

## 3. Secure storage

```dart
const storage = FlutterSecureStorage();
await storage.write(key: 'access_token', value: token);
final token = await storage.read(key: 'access_token');
await storage.delete(key: 'access_token');
```

Rules: never log tokens; delete on logout; don't store secrets in `shared_preferences`.

---

## 4. SQL with Drift (type-safe SQLite)

SQL basics you should know:

```sql
CREATE TABLE todos (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  done INTEGER NOT NULL DEFAULT 0,
  created_at INTEGER NOT NULL
);

INSERT INTO todos (title, created_at) VALUES ('Learn Dart', 1700000000);
SELECT * FROM todos WHERE done = 0 ORDER BY created_at DESC LIMIT 20;
UPDATE todos SET done = 1 WHERE id = 3;
DELETE FROM todos WHERE id = 3;
CREATE INDEX idx_todos_done ON todos(done);          -- speed up filters
```

Drift (sketch):

```dart
class Todos extends Table {
  IntColumn get id => integer().autoIncrement()();
  TextColumn get title => text().withLength(min: 1, max: 100)();
  BoolColumn get done => boolean().withDefault(const Constant(false))();
  DateTimeColumn get createdAt => dateTime()();
}

@DriftDatabase(tables: [Todos])
class AppDatabase extends _$AppDatabase {
  AppDatabase(super.e);
  @override
  int get schemaVersion => 1;

  Stream<List<Todo>> watchOpenTodos() =>
      (select(todos)..where((t) => t.done.equals(false))).watch();     // live stream!

  Future<int> add(String title) =>
      into(todos).insert(TodosCompanion.insert(title: title, createdAt: DateTime.now()));
}
```

**Reactive queries** (`.watch()` returns a `Stream`) make the UI update automatically when data changes. That plugs straight into `StreamBuilder`, Riverpod `StreamProvider`, or Bloc.

### Migrations

When your schema changes (new column), existing users have the old database. You **must** migrate:

```dart
@override
MigrationStrategy get migration => MigrationStrategy(
  onUpgrade: (m, from, to) async {
    if (from < 2) await m.addColumn(todos, todos.dueDate);
  },
);
```

A broken migration = users lose data or crash at startup. **Test migrations.**

---

## 5. Files

```dart
final dir = await getApplicationDocumentsDirectory();   // persistent, private to the app
final file = File('${dir.path}/notes.json');
await file.writeAsString(jsonEncode(notes));
final text = await file.readAsString();

final cacheDir = await getTemporaryDirectory();         // OS may clear this
```

Write atomically for important files (write to temp, then rename) to avoid corruption if the app dies mid-write.

---

## 6. Offline-first pattern

```
UI ◄── observes ── Local DB (single source of truth) ◄── sync ── Remote API
```

1. UI **only** reads from the local database (via a stream).
2. A repository refreshes from the network and **writes into** the DB.
3. The UI updates automatically.
4. Writes go to the DB immediately (optimistic), plus a queue to sync later.
5. Handle **conflicts** (last-write-wins, server-wins, merge by field).

This gives instant UI, offline support, and one data path.

---

## 7. Design rules

- **Single source of truth** for each piece of data.
- **Version** your stored data format.
- Keep storage behind **interfaces** (repositories) for tests.
- Do I/O asynchronously, off the UI path; heavy DB work in a background isolate (Drift supports this).
- Never store sensitive data in plain text.
- Plan **data deletion** (logout, GDPR "delete my data").
- Cache with **expiry** (time-to-live) and **size limits**.

---

## Common mistakes

- Storing big lists as JSON strings in `shared_preferences`.
- Storing tokens in `shared_preferences`.
- Forgetting migrations → crashes on update.
- Reading storage synchronously in `build`.
- Multiple sources of truth (memory, prefs, DB all diverging).
- Not awaiting writes before reading.

---

## Code challenge

1. Persist the theme mode with `shared_preferences` behind a `SettingsRepository` and restore it at startup (before `runApp`, so there is no flash of the wrong theme).
2. Build an **offline notes app** with Drift/SQLite: add, edit, delete, search, live-updating list via a stream.
3. Add a schema migration (v1 → v2: add a `pinned` column) and verify existing data survives.
4. Write a cache with TTL: `Cache<K, V>(ttl: Duration)` with `get`, `put`, and expiry (inject a clock function to test it).

<details>
<summary>Hint — TTL cache</summary>

```dart
class TtlCache<K, V> {
  TtlCache(this.ttl, {DateTime Function()? now}) : _now = now ?? DateTime.now;
  final Duration ttl;
  final DateTime Function() _now;
  final _map = <K, (V, DateTime)>{};

  V? get(K key) {
    final entry = _map[key];
    if (entry == null) return null;
    if (_now().difference(entry.$2) > ttl) {
      _map.remove(key);
      return null;
    }
    return entry.$1;
  }

  void put(K key, V value) => _map[key] = (value, _now());
}
```
</details>

---

## Interview questions

1. When do you use `shared_preferences` vs a database vs secure storage?
2. Why is `shared_preferences` bad for tokens?
3. What is a database migration and why does it matter?
4. What is offline-first? What is the "single source of truth"?
5. How do reactive queries (`watch`) simplify UI code?
6. How would you handle sync conflicts?
7. How do you make storage code testable?
8. What is an index and when do you add one?

<details>
<summary>Hints</summary>

1. Prefs: small settings; DB: structured/queryable/large; secure: secrets.
2. Stored in plain text; readable on rooted/backup-extracted devices.
3. Updating the schema for existing installs without losing data.
4. UI reads from local DB; network updates the DB; one canonical copy.
5. DB changes push new data via a Stream; UI just listens.
6. Last-write-wins, server-wins, timestamps/versions, field-level merge, user prompt.
7. Interface + in-memory fake; in-memory DB for integration tests.
8. A structure speeding up lookups on a column; for frequent filters/joins; costs write speed/space.
</details>
