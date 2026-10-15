# 08 — Navigation

## Goal

Move between screens, pass data, and get results back. Know both the basic `Navigator` and modern routing with `go_router`.

---

## 1. The Navigator is a stack

Screens ("routes") are stacked. `push` adds one on top. `pop` removes the top one.

```
[ Home ]                 push(Details)         [ Home, Details ]      pop → [ Home ]
```

This is the **stack** data structure from the algorithms folder, in real life.

---

## 2. Basic push / pop

```dart
// go to a new screen
Navigator.of(context).push(
  MaterialPageRoute(builder: (_) => const DetailsPage()),
);

// go back
Navigator.of(context).pop();
```

### Passing data forward — constructor arguments

```dart
Navigator.of(context).push(
  MaterialPageRoute(builder: (_) => DetailsPage(productId: 42)),
);

class DetailsPage extends StatelessWidget {
  const DetailsPage({super.key, required this.productId});
  final int productId;
  ...
}
```

### Returning data back

```dart
// Details page:
Navigator.of(context).pop('saved');

// Caller:
final result = await Navigator.of(context).push<String>(
  MaterialPageRoute(builder: (_) => const EditPage()),
);
if (!context.mounted) return;
if (result == 'saved') { ... }
```

### Other operations

```dart
Navigator.of(context).pushReplacement(...)        // replace current screen (e.g. splash → home)
Navigator.of(context).pushAndRemoveUntil(
  MaterialPageRoute(builder: (_) => const HomePage()),
  (route) => false,                               // clear everything (e.g. after logout)
);
Navigator.of(context).popUntil((r) => r.isFirst);
Navigator.of(context).maybePop();                 // respects WillPop / PopScope
```

---

## 3. Named routes (simple apps)

```dart
MaterialApp(
  initialRoute: '/',
  routes: {
    '/': (_) => const HomePage(),
    '/settings': (_) => const SettingsPage(),
  },
)

Navigator.of(context).pushNamed('/settings');
```

Limitations: weak typing for arguments, no deep-link parameters, no nested navigation help. For real apps use a router package.

---

## 4. `go_router` (the recommended package for most apps)

Add: `flutter pub add go_router`

```dart
final router = GoRouter(
  initialLocation: '/',
  routes: [
    GoRoute(
      path: '/',
      builder: (context, state) => const HomePage(),
      routes: [
        GoRoute(
          path: 'product/:id',                    // /product/42
          builder: (context, state) {
            final id = int.parse(state.pathParameters['id']!);
            return ProductPage(id: id);
          },
        ),
      ],
    ),
    GoRoute(path: '/login', builder: (_, __) => const LoginPage()),
  ],
);

void main() => runApp(MaterialApp.router(routerConfig: router));
```

Navigate:

```dart
context.go('/product/42');       // replace the stack with this location (declarative)
context.push('/product/42');     // add on top of the stack
context.pop();
context.pushNamed('product', pathParameters: {'id': '42'});
```

Query params: `/search?q=flutter` → `state.uri.queryParameters['q']`.

### Redirects (auth guards)

```dart
GoRouter(
  redirect: (context, state) {
    final loggedIn = authState.isLoggedIn;
    final goingToLogin = state.matchedLocation == '/login';
    if (!loggedIn && !goingToLogin) return '/login';
    if (loggedIn && goingToLogin) return '/';
    return null;                                   // no redirect
  },
  refreshListenable: authState,                    // re-run when auth changes
  routes: [...],
)
```

### Bottom navigation with state preserved

`StatefulShellRoute.indexedStack` keeps each tab's navigation stack and scroll position.

```dart
StatefulShellRoute.indexedStack(
  builder: (context, state, navigationShell) => ScaffoldWithNavBar(shell: navigationShell),
  branches: [
    StatefulShellBranch(routes: [GoRoute(path: '/home', builder: ...)]),
    StatefulShellBranch(routes: [GoRoute(path: '/profile', builder: ...)]),
  ],
)
```

---

## 5. Deep links

A deep link opens a **specific screen** from outside the app: `myapp://product/42` or `https://example.com/product/42`. Router-based navigation + platform configuration (Android intent filters/App Links, iOS Associated Domains/Universal Links) make them work. This is why URL-like route design matters even in mobile apps.

---

## 6. Dialogs, bottom sheets, and results

They are also routes:

```dart
final ok = await showModalBottomSheet<bool>(
  context: context,
  builder: (_) => const ConfirmSheet(),
);
```

---

## 7. Preventing back / confirming exit

```dart
PopScope(
  canPop: false,
  onPopInvokedWithResult: (didPop, result) async {
    if (didPop) return;
    final leave = await confirmDiscardChanges(context);
    if (leave && context.mounted) Navigator.of(context).pop();
  },
  child: const EditPage(),
)
```

---

## 8. Page transitions and Hero

```dart
// Shared element animation
Hero(tag: 'photo-${item.id}', child: Image.network(item.url))   // same tag on both pages
```

Custom transition: `PageRouteBuilder` or `CustomTransitionPage` in `go_router`.

---

## 9. Design guidance

- **Don't** put business data in the route name. Pass **IDs**, load data in the destination from a repository/cache.
- Keep route paths in one place (constants or typed routes with `go_router_builder`).
- Navigation is **UI logic**; keep it out of low-level data code. Trigger it from the UI in response to state changes.
- Never call `Navigator` with a stale `context` after `await` — check `context.mounted`.

---

## Common mistakes

- Using `context.go` when you wanted `context.push` (back button disappears).
- Passing large objects through routes instead of IDs.
- Using `context` after `await` without `mounted` check.
- Building navigation in `build()` (calls `push` during build → error). Use callbacks or `WidgetsBinding.instance.addPostFrameCallback`.
- Not handling unknown routes (`errorBuilder`).

---

## Code challenge

Build a mini shop:

1. Home: product list. Tap → Details (`/product/:id`) with a `Hero` image.
2. Details: "Add to cart" returns `true` to the previous page; Home shows a snackbar.
3. Cart tab and Home tab in a bottom nav bar, state preserved between tabs.
4. Login redirect: `/cart` requires login; after login return to `/cart`.

<details>
<summary>Hint — remember where the user wanted to go</summary>

```dart
redirect: (context, state) {
  if (!auth.isLoggedIn && state.matchedLocation == '/cart') {
    return '/login?from=${Uri.encodeComponent(state.matchedLocation)}';
  }
  return null;
}
// After login: context.go(state.uri.queryParameters['from'] ?? '/');
```
</details>

---

## Interview questions

1. How does the `Navigator` work conceptually? What data structure is it?
2. `push` vs `pushReplacement` vs `pushAndRemoveUntil`: when do you use each?
3. `go` vs `push` in `go_router`?
4. How do you pass data to a screen and receive a result from it?
5. How would you implement an authentication guard?
6. What are deep links and what must you configure?
7. Why prefer passing IDs instead of full objects in routes?
8. How do you intercept the back button?

<details>
<summary>Hints</summary>

1. A stack of routes (LIFO).
2. Add / replace current (login → home) / clear history (logout).
3. `go` replaces the stack declaratively to a location; `push` adds on top.
4. Constructor/path parameters forward; `pop(result)` and `await push<T>` back.
5. Router `redirect` using auth state, with `refreshListenable`.
6. URLs that open a specific screen; platform config and route patterns.
7. Deep links and process restoration can only carry strings; avoids stale data copies.
8. `PopScope`.
</details>
