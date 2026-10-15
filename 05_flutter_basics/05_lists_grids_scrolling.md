# 05 — Lists, Grids, and Scrolling

## Goal

Show large amounts of data efficiently. Lists are in almost every app.

---

## 1. ListView

### Simple (small, fixed list)

```dart
ListView(
  padding: const EdgeInsets.all(16),
  children: const [
    ListTile(title: Text('One')),
    ListTile(title: Text('Two')),
  ],
)
```

All children are **built immediately**. Only use for a few items.

### `ListView.builder` (the default choice)

```dart
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) {
    final item = items[index];
    return ListTile(
      title: Text(item.title),
      subtitle: Text(item.subtitle),
    );
  },
)
```

Items are **built lazily**: only those on (or near) the screen exist. 1,000,000 items = same memory as 20. This is **virtualization**.

### `ListView.separated`

```dart
ListView.separated(
  itemCount: items.length,
  separatorBuilder: (_, __) => const Divider(height: 1),
  itemBuilder: (context, i) => ListTile(title: Text(items[i])),
)
```

### Performance tips

- Prefer `ListView.builder` for anything dynamic.
- Make item widgets `const` / small and cheap.
- Give a fixed `itemExtent` (or `prototypeItem`) if all rows have the same height: faster layout.
- Never use `shrinkWrap: true` with long lists.
- Don't do heavy work (parsing, sorting) inside `itemBuilder`.
- Use `Key`s (`ValueKey(item.id)`) when items can be reordered or removed.

---

## 2. GridView

```dart
GridView.builder(
  padding: const EdgeInsets.all(8),
  gridDelegate: const SliverGridDelegateWithFixedCrossAxisCount(
    crossAxisCount: 3,
    crossAxisSpacing: 8,
    mainAxisSpacing: 8,
    childAspectRatio: 1,
  ),
  itemCount: photos.length,
  itemBuilder: (context, i) => Image.network(photos[i], fit: BoxFit.cover),
)
```

Responsive columns: use `SliverGridDelegateWithMaxCrossAxisExtent(maxCrossAxisExtent: 180)` → as many columns as fit.

---

## 3. Scroll direction & other scroll widgets

```dart
ListView.builder(scrollDirection: Axis.horizontal, ...)   // horizontal list (give it a bounded height!)
SingleChildScrollView(child: Column(...))                 // scroll a small fixed layout
PageView(children: [...])                                 // swipe between pages
ReorderableListView(onReorder: ..., children: [...])      // drag to reorder
```

A **horizontal** list inside a `Column` needs a bounded height:

```dart
SizedBox(
  height: 120,
  child: ListView.builder(scrollDirection: Axis.horizontal, itemBuilder: ...),
)
```

---

## 4. ScrollController

Control and observe scrolling.

```dart
class _FeedState extends State<Feed> {
  final _controller = ScrollController();

  @override
  void initState() {
    super.initState();
    _controller.addListener(() {
      final nearBottom = _controller.position.pixels >
          _controller.position.maxScrollExtent - 200;
      if (nearBottom) _loadMore();
    });
  }

  @override
  void dispose() {
    _controller.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) =>
      ListView.builder(controller: _controller, itemBuilder: ...);
}
```

Jump/animate: `_controller.animateTo(0, duration: ..., curve: Curves.easeOut)`.

### Infinite scroll (pagination) design

1. Keep `items`, `page`, `isLoading`, `hasMore`.
2. When the user is near the bottom and `!isLoading && hasMore` → fetch the next page.
3. Append results, increment `page`.
4. Show a loader item at the end while loading.
5. Handle errors with a "Retry" row.

Guard against duplicate requests with the `isLoading` flag.

---

## 5. Pull to refresh

```dart
RefreshIndicator(
  onRefresh: () async {
    await repository.reload();
  },
  child: ListView.builder(...),
)
```

The `Future` returned controls how long the spinner shows.

---

## 6. Slivers — the engine behind scrolling

A **sliver** is a scrollable area piece. `ListView` and `GridView` are built from slivers.
`CustomScrollView` lets you mix them (collapsing app bars, headers, list + grid).

```dart
CustomScrollView(
  slivers: [
    const SliverAppBar(
      pinned: true,
      expandedHeight: 200,
      flexibleSpace: FlexibleSpaceBar(title: Text('Gallery')),
    ),
    SliverList.builder(
      itemCount: 20,
      itemBuilder: (context, i) => ListTile(title: Text('Row $i')),
    ),
    SliverGrid.count(
      crossAxisCount: 2,
      children: [for (var i = 0; i < 6; i++) Card(child: Center(child: Text('$i')))],
    ),
  ],
)
```

Use slivers when you need multiple scrolling pieces in **one** scroll view. Never nest scrollables in the same direction without `shrinkWrap`/slivers.

---

## 7. Dismiss, empty state, loading state

A real list handles **four** states: loading, error, empty, data.

```dart
Widget build(BuildContext context) {
  if (isLoading) return const Center(child: CircularProgressIndicator());
  if (error != null) return ErrorView(message: error!, onRetry: reload);
  if (items.isEmpty) return const Center(child: Text('Nothing here yet'));
  return ListView.builder(...);
}
```

Swipe to delete:

```dart
Dismissible(
  key: ValueKey(item.id),
  onDismissed: (_) => setState(() => items.remove(item)),
  background: ColoredBox(color: Colors.red),
  child: ListTile(title: Text(item.title)),
)
```

---

## Common mistakes

- `ListView(children: items.map(...).toList())` with thousands of items (builds all).
- Missing `itemCount` (infinite builder → eventually an error/hang).
- `ListView` in `Column` without `Expanded`.
- Forgetting to dispose the `ScrollController`.
- Not handling loading/empty/error states.
- Missing keys on dismissible/reorderable items.

---

## Code challenge

Build a **contacts list**:

1. 200 generated contacts with `ListView.builder`, grouped with a letter header (A, B, C...). (Hint: flatten into a list of "header | contact" items.)
2. A search field filtering the list as you type (debounced).
3. Pull-to-refresh that regenerates the data.
4. A "scroll to top" floating button that appears only after scrolling down 300 px.

<details>
<summary>Hint — grouped list as a sealed class</summary>

```dart
sealed class Row_ {}
class HeaderRow extends Row_ { HeaderRow(this.letter); final String letter; }
class ContactRow extends Row_ { ContactRow(this.name); final String name; }

List<Row_> buildRows(List<String> names) {
  final sorted = [...names]..sort();
  final rows = <Row_>[];
  String? current;
  for (final n in sorted) {
    final letter = n[0].toUpperCase();
    if (letter != current) {
      rows.add(HeaderRow(letter));
      current = letter;
    }
    rows.add(ContactRow(n));
  }
  return rows;
}

// itemBuilder:
// switch (rows[i]) {
//   HeaderRow(:final letter) => Text(letter),
//   ContactRow(:final name) => ListTile(title: Text(name)),
// }
```
</details>

---

## Interview questions

1. Why is `ListView.builder` more efficient than `ListView(children: ...)`?
2. What is virtualization/lazy building?
3. Why does `shrinkWrap: true` hurt performance?
4. How do you implement infinite scrolling? What bugs must you guard against?
5. What are slivers and when do you need `CustomScrollView`?
6. Why are `Key`s important in lists?
7. Which four UI states should every data list handle?

<details>
<summary>Hints</summary>

1. Builds only visible items lazily.
2. Only create widgets/render objects for items near the viewport and recycle them as you scroll.
3. It forces the list to measure every child to compute its size.
4. Scroll listener, `isLoading`/`hasMore` flags, de-duplication, error/retry, append not replace.
5. Scrollable pieces; mix lists, grids, collapsing headers in one scroll area.
6. Keep each element tied to the right data when items are reordered/inserted/removed (state, animations).
7. Loading, error, empty, data.
</details>
