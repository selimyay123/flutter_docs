# Project 2 — Weather App

## Goal

Practice **networking, JSON, async, architecture, error handling, and caching** with a real API.
Estimated time: 1–2 weekends.

---

## API

Use **Open-Meteo** (free, no API key): <https://open-meteo.com/en/docs>

- Geocoding: `https://geocoding-api.open-meteo.com/v1/search?name=Istanbul&count=5`
- Forecast: `https://api.open-meteo.com/v1/forecast?latitude=41.01&longitude=28.95&current=temperature_2m,weather_code,wind_speed_10m&daily=temperature_2m_max,temperature_2m_min,weather_code&timezone=auto`

Read the docs, run the URLs in a browser, **look at the JSON** before writing code. (Real-world habit.)

---

## Requirements

### Must have

1. Search for a city (autocomplete suggestions, **debounced** 400 ms).
2. Show current temperature, condition icon/text, and wind.
3. Show a 7-day forecast list.
4. Loading, error (with retry), and empty states.
5. Remember the last selected city.
6. Pull-to-refresh.

### Should have

7. Favorites: save multiple cities, swipe between them (`PageView`).
8. Units toggle: °C/°F.
9. Cache the last response; show it instantly (**stale-while-revalidate**) and mark "Updated 5 min ago".
10. Offline mode banner.

### Stretch

11. Use device location (`geolocator`) with permission handling.
12. Hourly chart with `CustomPainter`.
13. Animated background by weather.
14. Home-screen widget (native).

---

## Architecture

```
UI (WeatherPage)
   ↓ watches
WeatherViewModel (AsyncNotifier / Cubit)
   ↓ calls
WeatherRepository (interface)           ← caching, mapping, errors → AppException
   ↓ uses
WeatherApi (http)  +  WeatherCache (shared_preferences / DB)
```

DTOs vs domain: `WeatherDto` (mirrors JSON) → mapper → `Weather` (clean).

---

## Step-by-step plan

### Step 1 — Explore the API

Save two sample responses in `test/fixtures/*.json`. You will use them for tests.

### Step 2 — Models and parsing

```dart
class CurrentWeather {
  const CurrentWeather({required this.temperatureC, required this.windKmh, required this.code});
  final double temperatureC;
  final double windKmh;
  final int code;

  factory CurrentWeather.fromJson(Map<String, dynamic> json) {
    if (json case {
      'temperature_2m': num t,
      'wind_speed_10m': num w,
      'weather_code': int c,
    }) {
      return CurrentWeather(temperatureC: t.toDouble(), windKmh: w.toDouble(), code: c);
    }
    throw const FormatException('Bad current weather JSON');
  }
}
```

Map **weather codes** to a description/icon with an `enum` + a `switch` expression (see the WMO code table in the docs).

### Step 3 — API client (with injected `http.Client`)

- Timeouts, status code checks, error mapping.
- Unit tests with `MockClient` and your fixtures: success, 500, bad JSON, timeout.

### Step 4 — Repository

- `Future<Weather> getWeather(City city)`.
- Cache logic: memory → disk → network. Add a TTL (e.g. 10 minutes).
- Convert exceptions into a sealed `AppException`.

### Step 5 — View model with explicit states

`WeatherState = Loading | Data(weather, isStale, updatedAt) | Failure(error, cachedData?)`.

### Step 6 — Search with debounce & cancellation

Ignore **stale** responses (user typed "Ista", then "Istanbul" — "Ista" must not overwrite the result of "Istanbul").

```dart
int _requestId = 0;
Future<void> search(String q) async {
  final id = ++_requestId;
  final results = await repo.searchCities(q);
  if (id != _requestId) return;       // a newer search started; drop this result
  state = SearchResults(results);
}
```

### Step 7 — UI

- Beautiful current-weather card (gradient by condition), forecast `ListView`.
- Skeleton/shimmer loading.
- Error view with retry.

### Step 8 — Tests

Unit (parsing, cache TTL, repository), view model (fake repo), widget (error → retry → data).

---

## Algorithm corner

1. **Min/max across days** for a mini-chart: map temperatures to pixel Y with linear interpolation. Edge case: all equal temps (division by zero!).
2. **City search ranking:** exact match first, then prefix, then contains; ties broken by population. Write the comparator and test it.
3. **TTL + LRU cache** for responses (reuse your `LruCache`, add expiry). Complexity of `get`/`put`?
4. **Debounce** implemented via a `Timer` vs via a `Stream` transformer — compare.
5. **Time zones:** forecast arrives in the city's local time; "today" in that city may be different from "today" on the phone. How do you handle it?

<details>
<summary>Hint — min/max mapping</summary>

```dart
double normalize(double v, double min, double max) =>
    max == min ? 0.5 : (v - min) / (max - min);
```
</details>

---

## Self-review rubric

| Area | Question |
|------|----------|
| Resilience | Airplane mode? Slow network? 500? Garbage JSON? |
| Correctness | Rapid typing → only the latest search result is shown? |
| Architecture | Widgets know nothing about HTTP or JSON? |
| UX | Stale data visible with a clear "updated" label? |
| Tests | Parsing and repository tested with fixtures? |
| Security | No secrets in code (Open-Meteo needs none, but think about it) |

---

## Interview questions

1. How does your app behave offline? Walk through the data flow.
2. How do you prevent race conditions in search-as-you-type?
3. Where do you map DTOs to domain models and why?
4. How did you design your cache invalidation?
5. How do you test the API layer without the internet?
6. What failure modes did you handle, and how are they shown to the user?
7. How would you add push notifications for severe weather?
8. If the API were rate-limited at 10 req/min, what would you change?

<details>
<summary>Hints</summary>

1. Cache first → show stale → refresh → update or show failure with cached data.
2. Request IDs / cancel tokens / `restartable` transformer; debounce.
3. In the repository so the UI and domain are decoupled from API shape.
4. TTL, manual refresh, versioned cache keys, clear on logout/units change.
5. Inject `http.Client`; `MockClient` with fixture responses.
6. Offline, timeout, server, parse errors → typed errors with friendly messages and retry.
7. Needs a backend (FCM/APNs), device tokens, server-side evaluation of alerts.
8. Aggressive caching, debounce, batching, back-off, request deduplication.
</details>
