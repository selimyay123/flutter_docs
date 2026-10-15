# 04 — Provider, Riverpod, and Bloc

## Goal

Learn the three most-used state management tools. Understand each one's **philosophy**, so you can pick and defend a choice.

> Package APIs evolve. Always check the official docs for current syntax. The **concepts** here are stable.

---

## 1. Provider (simple, widely used)

Add: `flutter pub add provider`

```dart
class CartModel extends ChangeNotifier { ... }   // from the previous lesson

void main() {
  runApp(
    ChangeNotifierProvider(
      create: (_) => CartModel(),
      child: const MyApp(),
    ),
  );
}
```

Reading:

```dart
// Rebuilds when CartModel notifies
final cart = context.watch<CartModel>();

// Read once (no rebuild) — use in callbacks
context.read<CartModel>().add(product);

// Rebuild only if this selected value changes
final count = context.select<CartModel, int>((c) => c.count);
```

Or widgets: `Consumer<CartModel>(builder: (context, cart, child) => ...)`, `Selector`.

Multiple providers: `MultiProvider(providers: [...], child: ...)`.

Rules:

- `watch` inside `build`; `read` inside event handlers.
- Never call `read` in `build` for something you need to react to.

**Pros:** easy, minimal boilerplate. **Cons:** runtime `ProviderNotFoundException`, depends on `BuildContext`, weak for async/derived state.

> The Provider author created Riverpod to fix these issues. Provider is in maintenance mode: fine for existing apps, but for new apps prefer Riverpod or Bloc.

---

## 2. Riverpod (compile-safe, context-free, async-first)

Add: `flutter pub add flutter_riverpod` (and optionally `riverpod_annotation` + generator).

```dart
void main() => runApp(const ProviderScope(child: MyApp()));
```

### Core idea: providers are global *declarations*, state lives in the scope

```dart
// Simple value
final greetingProvider = Provider<String>((ref) => 'Hello');

// Mutable state with logic (Notifier)
class CounterNotifier extends Notifier<int> {
  @override
  int build() => 0;                 // initial state
  void increment() => state++;      // assigning state notifies listeners
}

final counterProvider = NotifierProvider<CounterNotifier, int>(CounterNotifier.new);

// Async data (loading / error / data handled for you)
final userProvider = FutureProvider.family<User, int>((ref, id) {
  final repo = ref.watch(userRepositoryProvider);   // dependency injection
  return repo.fetch(id);
});

// Streams
final messagesProvider = StreamProvider<List<Message>>((ref) => ref.watch(chatRepoProvider).watch());
```

### In widgets: `ConsumerWidget`

```dart
class CounterPage extends ConsumerWidget {
  const CounterPage({super.key});

  @override
  Widget build(BuildContext context, WidgetRef ref) {
    final count = ref.watch(counterProvider);

    return Scaffold(
      body: Center(child: Text('$count')),
      floatingActionButton: FloatingActionButton(
        onPressed: () => ref.read(counterProvider.notifier).increment(),
        child: const Icon(Icons.add),
      ),
    );
  }
}
```

### Handling async with `AsyncValue`

```dart
final user = ref.watch(userProvider(42));

return user.when(
  loading: () => const CircularProgressIndicator(),
  error: (e, st) => Text('Error: $e'),
  data: (u) => Text(u.name),
);
```

### Key features

- **`ref.watch`** — rebuild on change. **`ref.read`** — one-shot. **`ref.listen`** — side effects (snackbar, navigation).
- **`.select`** — rebuild only for a part of the state.
- **`.family`** — parametrized providers (cache per argument).
- **`autoDispose`** — free state when nobody listens.
- **Overrides** — swap implementations in tests: `ProviderScope(overrides: [repoProvider.overrideWithValue(FakeRepo())])`.
- Providers can depend on other providers → automatic recomputation (derived state).

**Pros:** compile-time safety, testability, no context needed, great async/caching. **Cons:** learning curve, code generation optional but common, ecosystem churn.

---

## 3. Bloc / Cubit (explicit, event-driven)

Add: `flutter pub add flutter_bloc`

### Cubit — methods emit states

```dart
class CounterCubit extends Cubit<int> {
  CounterCubit() : super(0);
  void increment() => emit(state + 1);
}
```

### Bloc — events in, states out

```dart
sealed class CartEvent {}
class ProductAdded extends CartEvent { ProductAdded(this.product); final Product product; }
class ProductRemoved extends CartEvent { ProductRemoved(this.id); final String id; }

class CartState {
  const CartState({this.items = const []});
  final List<Product> items;
}

class CartBloc extends Bloc<CartEvent, CartState> {
  CartBloc() : super(const CartState()) {
    on<ProductAdded>((event, emit) => emit(CartState(items: [...state.items, event.product])));
    on<ProductRemoved>((event, emit) =>
        emit(CartState(items: state.items.where((p) => p.id != event.id).toList())));
  }
}
```

### In widgets

```dart
BlocProvider(
  create: (_) => CartBloc(),
  child: const CartPage(),
)

BlocBuilder<CartBloc, CartState>(
  builder: (context, state) => Text('${state.items.length} items'),
)

// Side effects (navigation, snackbars):
BlocListener<CartBloc, CartState>(
  listener: (context, state) { ... },
  child: ...,
)

context.read<CartBloc>().add(ProductAdded(product));
```

Also: `BlocSelector`, `BlocConsumer` (builder + listener), `MultiBlocProvider`.

### Async with Bloc

```dart
on<LoadRequested>((event, emit) async {
  emit(Loading());
  try {
    final data = await repo.load();
    emit(Loaded(data));
  } catch (e) {
    emit(Failure(e.toString()));
  }
});
```

Event **transformers** (`debounce`, `droppable`, `restartable`) from `bloc_concurrency` handle search-as-you-type and double taps elegantly.

**Pros:** very predictable, every change is an event (great logging/debugging), consistent structure across a big team, easy to test (`bloc_test`). **Cons:** more boilerplate.

---

## 4. Comparison

| | Provider | Riverpod | Bloc / Cubit |
|--|----------|----------|--------------|
| Learning curve | Low | Medium | Medium |
| Boilerplate | Low | Low–Medium | Higher |
| Needs `BuildContext` | Yes | No | Yes (to find bloc) |
| Compile-time safety | No | Yes | Partial |
| Async/caching | Manual | Excellent | Manual (via events) |
| Testing | Good | Excellent (overrides) | Excellent (`bloc_test`) |
| Debuggability | OK | Good (observers) | Excellent (events stream) |
| Typical team | Small/simple | Modern, flexible | Large, process-heavy |

### Decision shortcut

- Learning or small app → **Provider** or **Riverpod**.
- New production app, async-heavy → **Riverpod**.
- Large team wanting strict structure → **Bloc**.
- Local effect only → **`setState`**.

---

## 5. What stays the same, whatever you pick

1. **State = immutable data**, UI = f(state).
2. **Logic outside widgets**, depends on a **repository interface** (injectable).
3. **Model states explicitly** (loading/error/data).
4. **Listen narrowly**, select what you need.
5. **Dispose** what you create.
6. **Test** the logic with fakes.

If you can implement the same feature in all three, you understand state management, not a package.

---

## 6. Testing a notifier (Riverpod) — preview

```dart
test('increments', () {
  final container = ProviderContainer();
  addTearDown(container.dispose);

  expect(container.read(counterProvider), 0);
  container.read(counterProvider.notifier).increment();
  expect(container.read(counterProvider), 1);
});
```

No widgets, no emulator, runs in milliseconds.

---

## Common mistakes

- Using `context.read` in `build` and wondering why the UI doesn't update.
- Using `ref.watch` inside callbacks (use `read`).
- Creating a provider/bloc inside `build` (recreated every time).
- Keeping UI-only state (animation, text controllers) in global state.
- Calling `emit` after a Bloc is closed.
- Large monolithic state objects that rebuild everything.

---

## Code challenge

Implement the **same feature three times** (Provider, Riverpod, Bloc): a **todo list** with add, toggle, delete, and a filter (all/active/done). Then answer: which felt simplest? Where was the boilerplate? Where were bugs easiest to avoid?

Bonus: add a **debounced search** (Bloc `restartable` transformer / Riverpod with `Timer` in `ref.onDispose`).

<details>
<summary>Hint — derived state in Riverpod</summary>

```dart
final filteredTodosProvider = Provider<List<Todo>>((ref) {
  final todos = ref.watch(todosProvider);
  final filter = ref.watch(filterProvider);
  return switch (filter) {
    Filter.all => todos,
    Filter.active => todos.where((t) => !t.done).toList(),
    Filter.done => todos.where((t) => t.done).toList(),
  };
});
```
</details>

---

## Interview questions

1. `watch` vs `read` vs `select` (Provider) — when each?
2. Why was Riverpod created? What problems of Provider does it solve?
3. What is a `Notifier`/`AsyncNotifier` in Riverpod?
4. Explain `ref.watch`, `ref.read`, `ref.listen`.
5. Cubit vs Bloc: when do you add events?
6. How do you handle side effects (navigation, snackbar) without causing them in `build`?
7. How do you test a Bloc / a Riverpod provider?
8. How do you prevent unnecessary rebuilds?
9. How would you choose a state management approach for a new app?
10. What is `autoDispose` and why is it useful?

<details>
<summary>Hints</summary>

1. watch in build to rebuild; read in handlers; select for a narrow subset.
2. Context-independence, compile safety, async/caching, testing overrides.
3. Class that holds state and exposes methods that update it.
4. Rebuild on change / one-time read / run side effects on change.
5. Events when you need traceability, transformers, or complex flows; Cubit for simplicity.
6. `BlocListener` / `ref.listen` — reactions outside `build`.
7. Bloc: `bloc_test` with expected state sequence. Riverpod: `ProviderContainer` with overrides.
8. Selectors, smaller widgets, const, equality/immutability.
9. Team skills, app complexity, testing needs, async patterns, consistency.
10. Disposes state when no listeners remain → frees memory and cancels work.
</details>
