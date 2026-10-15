# 06 — Strings

## Goal

Work with text confidently. Strings appear in every app and in many interview problems.

---

## 1. Creating strings

```dart
final a = 'single';
final b = "double";
final c = 'It\'s fine';      // escape
final d = "It's fine";       // easier
final e = r'raw \n text';    // raw: no escapes
final multi = '''
line 1
line 2
''';
```

Adjacent literals join: `'a' 'b'` → `'ab'`.

---

## 2. Strings are immutable

You can never change a string. Methods return **new** strings.

```dart
var s = 'hello';
s.toUpperCase();     // returns 'HELLO' but s is still 'hello'
s = s.toUpperCase(); // now s is 'HELLO'
```

---

## 3. Everyday methods

```dart
const s = '  Hello, Dart!  ';
print(s.trim());                       // 'Hello, Dart!'
print(s.length);                       // 16
print(s.toLowerCase());
print(s.contains('Dart'));             // true
print(s.startsWith('  H'));
print(s.indexOf('Dart'));              // 9
print(s.replaceAll('Dart', 'Flutter'));
print(s.split(','));                   // ['  Hello', ' Dart!  ']
print(s.substring(2, 7));              // 'Hello'
print('ab' * 3);                       // 'ababab'
print('5'.padLeft(3, '0'));            // '005'
print('abc'[0]);                       // 'a'
```

Compare: `'a' == 'a'` is true (value equality). `'a'.compareTo('b')` returns negative.

---

## 4. Building strings efficiently

Because strings are immutable, `s += x` in a loop creates many temporary strings.
Use `StringBuffer`:

```dart
final buffer = StringBuffer();
for (var i = 0; i < 5; i++) {
  buffer.write(i);
  buffer.write(',');
}
print(buffer.toString()); // 0,1,2,3,4,
```

Or `join`:

```dart
print([1, 2, 3].join('-')); // 1-2-3
```

---

## 5. Characters, code units, runes

Dart strings are sequences of **UTF-16 code units**. Emoji can be 2 units!

```dart
const s = '👍';
print(s.length);         // 2  (not 1!)
print(s.runes.length);   // 1  (one Unicode code point)
```

For user-visible characters (grapheme clusters, like emoji with skin tones), use
the [`characters`](https://pub.dev/packages/characters) package (already in Flutter):

```dart
import 'package:characters/characters.dart';
print('👨‍👩‍👧'.characters.length); // 1
```

Interview-grade tip: "reverse a string" with `split('')` breaks emoji. Say that out loud.

---

## 6. Character codes

```dart
print('A'.codeUnitAt(0));          // 65
print(String.fromCharCode(97));    // 'a'
print('c'.codeUnitAt(0) - 'a'.codeUnitAt(0)); // 2  (position in alphabet)
```

Used in frequency counting with an array of 26 slots.

---

## 7. Regular expressions

```dart
final emailRegex = RegExp(r'^[\w.+-]+@[\w-]+\.[\w.-]+$');
print(emailRegex.hasMatch('ada@x.com')); // true

final digits = RegExp(r'\d+');
print(digits.allMatches('a1b22c333').map((m) => m.group(0)).toList());
// [1, 22, 333]
```

Use raw strings `r'...'` for regex so you don't double-escape.

---

## Common mistakes

- Expecting `s.toUpperCase()` to change `s`.
- Concatenating with `+=` in huge loops.
- Using `length` as "number of visible characters".
- Using `==` on `StringBuffer` (compare `toString()`).

---

## Code challenge

1. `bool isPalindrome(String s)` — ignore case and non-letters. `"A man, a plan, a canal: Panama"` → `true`.
2. `Map<String, int> charCount(String s)` — counts letters.
3. `bool isAnagram(String a, String b)` — `"listen"`, `"silent"` → `true`.

<details>
<summary>Solution</summary>

```dart
bool isPalindrome(String s) {
  final cleaned = s.toLowerCase().replaceAll(RegExp(r'[^a-z0-9]'), '');
  var i = 0, j = cleaned.length - 1;       // two pointers
  while (i < j) {
    if (cleaned[i] != cleaned[j]) return false;
    i++;
    j--;
  }
  return true;
}

Map<String, int> charCount(String s) {
  final counts = <String, int>{};
  for (final ch in s.split('')) {
    counts[ch] = (counts[ch] ?? 0) + 1;
  }
  return counts;
}

bool isAnagram(String a, String b) {
  if (a.length != b.length) return false;
  final counts = <String, int>{};
  for (final ch in a.split('')) {
    counts[ch] = (counts[ch] ?? 0) + 1;
  }
  for (final ch in b.split('')) {
    final c = counts[ch];
    if (c == null || c == 0) return false;
    counts[ch] = c - 1;
  }
  return true;
}
```
</details>

---

## Interview questions

1. Are Dart strings mutable? What are the consequences for performance?
2. Why is `'👍'.length` equal to 2?
3. When do you use `StringBuffer`?
4. How do you check if two strings are anagrams in `O(n)`?
5. What is a raw string and when is it useful?

<details>
<summary>Hints</summary>

1. Immutable. Repeated `+=` creates new strings each time → `O(n²)`. Use `StringBuffer`/`join`.
2. UTF-16: emoji outside the basic plane use two code units (surrogate pair).
3. Building a string over many steps, especially in loops.
4. Count characters in a map/array for the first, subtract for the second.
5. `r'...'`: no escape processing. Great for regex and Windows paths.
</details>
