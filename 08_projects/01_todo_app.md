# Project 1 — Todo App

## Goal

Build a complete small app end-to-end, applying Dart, widgets, state, storage, and tests.
Estimated time: 1–2 weekends.

---

## Requirements

### Must have

1. Add, edit, delete, and complete todos.
2. Filter: All / Active / Completed.
3. Todos persist after closing the app.
4. Empty state ("Nothing to do!") and error state.
5. Swipe to delete with **Undo** snackbar.
6. Light/dark theme following the system.

### Should have

7. Due date with a date picker; overdue items highlighted.
8. Search by title.
9. Reorder by drag and drop.
10. Counter: "3 items left".

### Stretch

11. Categories / tags with colors.
12. Local notifications for due todos.
13. Cloud sync (Firebase/Supabase) with offline-first design.
14. Responsive layout (list + detail on tablets).

---

## Architecture (follow it)

```
lib/
  main.dart
  app/            app.dart, router.dart, theme.dart
  features/
    todos/
      domain/       todo.dart, todo_repository.dart
      data/         local_todo_repository.dart, todo_dto.dart
      presentation/ todos_page.dart, todos_view_model.dart, widgets/
  core/             utils, widgets
test/
```

Use a state management approach of your choice (start with **Riverpod** or **Cubit**).

---

## Step-by-step plan

### Step 1 — Model (domain)

```dart
class Todo {
  const Todo({
    required this.id,
    required this.title,
    this.done = false,
    this.dueDate,
  });

  final String id;
  final String title;
  final bool done;
  final DateTime? dueDate;

  bool isOverdue(DateTime now) => !done && dueDate != null && dueDate!.isBefore(now);

  Todo copyWith({String? title, bool? done, DateTime? dueDate}) => Todo(
        id: id,
        title: title ?? this.title,
        done: done ?? this.done,
        dueDate: dueDate ?? this.dueDate,
      );
}
```

Write unit tests for `isOverdue` first (inject `now`).

### Step 2 — Repository interface + in-memory fake

```dart
abstract interface class TodoRepository {
  Future<List<Todo>> getAll();
  Future<void> upsert(Todo todo);
  Future<void> delete(String id);
}
```

### Step 3 — View model

- State: `sealed` → loading / error / data(todos, filter, query).
- Derived list: apply filter + search in a **pure function** and unit test it.
- Methods: `add`, `toggle`, `edit`, `remove`, `undoRemove`, `setFilter`, `setQuery`, `reorder`.

```dart
List<Todo> applyFilter(List<Todo> all, Filter filter, String query) {
  final q = query.trim().toLowerCase();
  return all.where((t) {
    final matchesFilter = switch (filter) {
      Filter.all => true,
      Filter.active => !t.done,
      Filter.completed => t.done,
    };
    return matchesFilter && (q.isEmpty || t.title.toLowerCase().contains(q));
  }).toList();
}
```

### Step 4 — UI

- `TodosPage` with `ListView.builder`, `Dismissible` (with `ValueKey(todo.id)`), `CheckboxListTile`.
- `AddTodoSheet` (bottom sheet with a `Form`).
- `FilterChips` and a search field.

### Step 5 — Persistence

Implement `LocalTodoRepository` with **Drift** (SQLite) or `shared_preferences` + JSON (simple). Expose a `Stream` if you can.

### Step 6 — Polish

Empty/error states, animations (`AnimatedList` / `AnimatedSwitcher`), haptics, accessibility labels, large text.

### Step 7 — Tests

| Test | Type |
|------|------|
| `applyFilter` combos | Unit |
| View model add/toggle/remove/undo with fake repo | Unit |
| Add a todo via UI and see it in the list | Widget |
| Repository persistence round trip | Unit/integration |

### Step 8 — Ship checklist

`flutter analyze` clean, tests pass, icons/name set, README with screenshots, GitHub repo with CI.

---

## Algorithm corner

1. **Reorder:** when a user drags an item from index `old` to `new`, how do you update the list **and** persist the order? (Hint: `ReorderableListView` gives `newIndex` that is off by one when moving down. Explain why.)
2. **Search ranking:** results starting with the query appear before results that merely contain it. Implement the comparator.
3. **Undo:** keep a stack of actions (command pattern). Implement `undo` for add, delete, and toggle.
4. Complexity: if there are 100,000 todos, which operations are `O(n)`? How would you speed up search (index/trie/debounce/isolate)?

<details>
<summary>Hint — reorder</summary>

```dart
void onReorder(int oldIndex, int newIndex) {
  if (newIndex > oldIndex) newIndex -= 1;       // removing the item shifts later indexes left
  final item = list.removeAt(oldIndex);
  list.insert(newIndex, item);
}
```
</details>

---

## Self-review rubric

| Area | Question |
|------|----------|
| Correctness | Do all requirements work, including edge cases (empty title, very long text)? |
| Architecture | Does any widget import storage code? |
| State | Can the UI show an impossible state? |
| Tests | Could you refactor safely? |
| UX | Loading/empty/error, undo, accessibility? |
| Performance | Smooth with 10,000 items? |
| Code quality | Names, small widgets, no duplication, lints clean? |

---

## Interview questions

1. Walk me through your app's architecture. Why did you choose that state management?
2. How does your app persist data? What happens when the schema changes?
3. How did you handle undo after a swipe-to-delete?
4. How do you keep `ReorderableListView` and your stored order consistent?
5. How would you add cloud sync without rewriting the UI?
6. What would you do differently if you started again?
7. How did you test it, and what would you not test?

<details>
<summary>Hints</summary>

1. UI → view model → repository → storage; explain trade-offs.
2. Repository behind an interface; migrations versioned.
3. Keep removed item and index; re-insert on undo; delay actual persistence deletion or delete then restore.
4. Store an `order` field/position and update after reorder; adjust `newIndex` when moving down.
5. Repository becomes offline-first: local DB as source of truth + sync queue.
6. Reflect honestly: earlier tests, simpler state, better naming.
7. Logic and view model thoroughly; widgets for key flows; skip trivial getters.
</details>
