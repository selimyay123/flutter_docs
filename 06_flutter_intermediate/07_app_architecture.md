# 07 — App Architecture

## Goal

Structure an app so it stays understandable at 10,000 lines and with 10 developers.

---

## 1. Why architecture?

Without structure, apps become "spaghetti": widgets call APIs, parse JSON, hold business rules. Every change breaks something. Architecture gives:

- **Separation of concerns** — each piece has one job.
- **Testability** — logic testable without UI/network.
- **Replaceability** — swap API, database, or UI.
- **Team scalability** — people can work in parallel.

Principle: **depend on abstractions; dependencies point inward** (UI → logic → data; never the opposite).

---

## 2. The layered architecture (recommended by the Flutter team)

```
┌─────────────────────────────────────────┐
│ UI layer                                │
│   Views (widgets)  ◄──  ViewModels      │  ← presentation logic & UI state
├─────────────────────────────────────────┤
│ Domain layer (optional)                 │
│   Use cases / entities (pure Dart)      │  ← complex business rules
├─────────────────────────────────────────┤
│ Data layer                              │
│   Repositories  ◄──  Services           │  ← repositories = source of truth
│   (cache, mapping)   (API, DB, device)  │
└─────────────────────────────────────────┘
```

### Roles

| Piece | Responsibility | Knows about |
|-------|----------------|-------------|
| **View** | Render state, forward user events | ViewModel |
| **ViewModel** (Notifier/Cubit/ChangeNotifier) | Hold UI state, react to events, call repositories | Repositories |
| **Repository** | Single source of truth for one kind of data; combine services, cache, map DTOs → domain models | Services |
| **Service** | Wrap one external thing (HTTP client, DB, GPS) | The outside world |
| **Domain model** | Plain immutable Dart objects | Nothing |
| **Use case** (optional) | One business operation spanning repos | Repositories |

Use the domain/use-case layer **only** when logic is complex or reused. Don't add ceremony for CRUD.

### Data flow

```
User taps → View calls viewModel.addTodo() → ViewModel calls repo.add() → repo writes DB / API
                                                                │
View ◄── new state ◄── ViewModel ◄── Stream/Future ◄────────────┘
```

---

## 3. Folder structure

### By layer (small apps)

```
lib/
  main.dart
  ui/ ... data/ ... domain/ ...
```

### By feature (recommended as the app grows)

```
lib/
  main.dart
  app/                  ← app widget, router, theme, DI setup
  core/                 ← shared: errors, utils, widgets, constants
  features/
    todos/
      data/
        todo_api.dart
        todo_repository_impl.dart
        todo_dto.dart
      domain/
        todo.dart                 ← entity
        todo_repository.dart      ← interface
      presentation/
        todos_page.dart
        todos_view_model.dart
        widgets/
    auth/
      ...
```

**Feature-first** keeps related code together, enables deleting a feature cleanly, and reduces merge conflicts.

Rule: a feature must **not** import another feature's internals; talk through public interfaces or the shared `core`.

---

## 4. A complete (tiny) example with Riverpod

```dart
// domain/todo.dart
class Todo {
  const Todo({required this.id, required this.title, this.done = false});
  final String id;
  final String title;
  final bool done;
  Todo copyWith({String? title, bool? done}) =>
      Todo(id: id, title: title ?? this.title, done: done ?? this.done);
}

// domain/todo_repository.dart
abstract interface class TodoRepository {
  Future<List<Todo>> getAll();
  Future<void> save(Todo todo);
  Future<void> delete(String id);
}

// data/todo_repository_impl.dart
class InMemoryTodoRepository implements TodoRepository {
  final _items = <String, Todo>{};
  @override Future<List<Todo>> getAll() async => _items.values.toList();
  @override Future<void> save(Todo t) async => _items[t.id] = t;
  @override Future<void> delete(String id) async => _items.remove(id);
}

// app/providers.dart  (the DI composition root)
final todoRepositoryProvider = Provider<TodoRepository>((ref) => InMemoryTodoRepository());

// presentation/todos_view_model.dart
class TodosViewModel extends AsyncNotifier<List<Todo>> {
  @override
  Future<List<Todo>> build() => ref.watch(todoRepositoryProvider).getAll();

  Future<void> toggle(Todo todo) async {
    final repo = ref.read(todoRepositoryProvider);
    await repo.save(todo.copyWith(done: !todo.done));
    ref.invalidateSelf();                      // reload
    await future;
  }
}
final todosProvider = AsyncNotifierProvider<TodosViewModel, List<Todo>>(TodosViewModel.new);

// presentation/todos_page.dart
class TodosPage extends ConsumerWidget {
  const TodosPage({super.key});
  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final todos = ref.watch(todosProvider);
    return todos.when(
      loading: () => const Center(child: CircularProgressIndicator()),
      error: (e, _) => Center(child: Text('$e')),
      data: (list) => ListView(
        children: [
          for (final t in list)
            CheckboxListTile(
              value: t.done,
              title: Text(t.title),
              onChanged: (_) => ref.read(todosProvider.notifier).toggle(t),
            ),
        ],
      ),
    );
  }
}
```

Notice: the **UI never knows** if data comes from memory, SQLite, or HTTP. Swap `InMemoryTodoRepository` for `ApiTodoRepository` in **one place**.

---

## 5. Dependency Injection (DI)

DI = give an object its dependencies **from outside** instead of creating them inside.

```dart
// Bad: hard-wired
class TodoViewModel { final repo = ApiTodoRepository(); }

// Good: injected
class TodoViewModel { TodoViewModel(this.repo); final TodoRepository repo; }
```

Benefits: swap fakes in tests, configure per environment (dev/staging/prod).

Tools: Riverpod providers, `provider`, `get_it` (service locator), `injectable` (codegen). Or plain **constructor injection** wired in `main()` (the *composition root*) — perfectly fine.

**Service locator vs DI container:** `get_it` lets any code pull dependencies (`getIt<Repo>()`) — convenient but hides dependencies. Prefer constructor injection and keep `get_it` calls only at the composition root.

---

## 6. DTO vs domain model

- **DTO** (data transfer object): mirrors JSON/DB rows exactly (snake_case, nullable fields).
- **Domain model**: clean, valid, UI-friendly.
- The repository **maps** DTO → domain.

This protects the app from backend changes. A renamed JSON field changes only the DTO and the mapper.

---

## 7. Principles to remember

| Principle | Meaning |
|-----------|---------|
| **SRP** | One reason to change per class |
| **DIP** | Depend on interfaces |
| **KISS** | Keep it simple |
| **YAGNI** | Don't build it until needed |
| **DRY** | Don't repeat knowledge (but don't over-abstract too early) |
| **Law of Demeter** | Don't reach through objects (`a.b.c.d()`) |
| **Composition over inheritance** | Combine small parts |
| **Unidirectional data flow** | State down, events up |

> "Duplication is far cheaper than the wrong abstraction." — Sandi Metz

Architecture is about **trade-offs**. An over-engineered 3-screen app is a failure too.

---

## 8. Error handling in architecture

- Services throw low-level exceptions.
- **Repositories** catch and convert them to **domain errors** (or `Result`).
- ViewModels convert them to UI state (`error: 'Check your connection'`).
- Views show messages and retry buttons.

Each layer speaks its own language.

---

## 9. Environment & configuration

- Use `--dart-define` / `--dart-define-from-file=env/dev.json` for environment values; read with `String.fromEnvironment`.
- **Flavors** (dev/staging/prod) for different bundle ids, icons, and endpoints.
- Do **not** commit secrets.

```bash
flutter run --dart-define=API_URL=https://dev.api.example.com
```

```dart
const apiUrl = String.fromEnvironment('API_URL', defaultValue: 'https://api.example.com');
```

---

## 10. Smells that tell you architecture is breaking

- Widgets import `http` or `sqflite`.
- `setState` calls inside a 600-line `State`.
- Business rules (discounts, validation) in `onPressed`.
- A single `utils.dart` with 50 unrelated functions.
- Circular imports between features.
- Tests require a running emulator for plain logic.

---

## Common mistakes

- Copying "Clean Architecture" diagrams for a to-do app.
- Fat ViewModels doing everything → extract repositories/use cases.
- Leaking DTOs into the UI.
- Singletons everywhere.
- No composition root; objects created randomly.

---

## Code challenge

1. Take your Posts app from the previous lesson and reorganize it **feature-first** with `data/domain/presentation`.
2. Introduce a `PostRepository` interface; create `ApiPostRepository` and `FakePostRepository`. Use a `--dart-define` flag to switch.
3. Add a DTO (`PostDto`) and a mapper to the domain `Post`.
4. Write unit tests of your ViewModel with the fake repository: loading → data, and loading → error.
5. Draw your dependency graph. Do all arrows point inward?

<details>
<summary>Hint — mapper</summary>

```dart
extension PostDtoX on PostDto {
  Post toDomain() => Post(id: id.toString(), title: title.trim(), body: body);
}
```
</details>

---

## Interview questions

1. Describe the architecture of your last app. Why those choices?
2. What does each layer (UI, domain, data) do? What must it not know?
3. Repository vs service?
4. What is dependency injection and what does it buy you?
5. Feature-first vs layer-first folder structure?
6. DTO vs domain model: why separate them?
7. Where does business logic belong? Where does validation belong?
8. How do you handle errors across layers?
9. What is a composition root?
10. When is Clean Architecture overkill?
11. How would you design the architecture for an offline-capable app?

<details>
<summary>Hints</summary>

1. Be concrete: layers, state management, DI, testing, trade-offs.
2. UI renders/forwards events; domain has rules; data talks to the outside. Inner layers don't know outer details.
3. Repository = source of truth for a data type (combines services/caches); service = wrapper for one external resource.
4. Pass dependencies in → testing, flexibility, clear dependencies.
5. Feature-first scales: cohesion, isolation, easier deletion.
6. Protect the app from backend/DB shape changes; keep models clean.
7. In domain/ViewModels/use cases, not in widgets; client validation for UX, server for security.
8. Convert to the language of each layer; show friendly messages at UI.
9. The single place where object graph is assembled (usually `main`/root providers).
10. Simple CRUD, small apps, short-lived prototypes.
11. Local DB as source of truth, repository syncs, queue writes, conflict policy, streams to UI.
</details>
