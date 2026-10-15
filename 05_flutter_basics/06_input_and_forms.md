# 06 — Input and Forms

## Goal

Collect and validate user input safely.

---

## 1. TextField and TextEditingController

```dart
class _SearchState extends State<Search> {
  final _controller = TextEditingController();

  @override
  void dispose() {
    _controller.dispose();            // always!
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return TextField(
      controller: _controller,
      decoration: const InputDecoration(
        labelText: 'Search',
        hintText: 'Type something...',
        prefixIcon: Icon(Icons.search),
        border: OutlineInputBorder(),
      ),
      onChanged: (text) => print('Changed: $text'),
      onSubmitted: (text) => print('Submitted: $text'),
    );
  }
}
```

Read the value: `_controller.text`. Set it: `_controller.text = 'abc'`. Clear: `_controller.clear()`.

Useful properties:

| Property | Use |
|----------|-----|
| `keyboardType` | `TextInputType.emailAddress`, `.number`, `.phone` |
| `textInputAction` | `TextInputAction.next` / `.done` / `.search` |
| `obscureText` | Password dots |
| `maxLines`, `maxLength` | Limits |
| `autofocus` | Focus on show |
| `inputFormatters` | Restrict input (`FilteringTextInputFormatter.digitsOnly`) |
| `autofillHints` | Password manager integration |
| `enabled`, `readOnly` | Disable / read-only |
| `focusNode` | Control focus |

---

## 2. Form and TextFormField (validation)

```dart
class SignUpForm extends StatefulWidget {
  const SignUpForm({super.key});
  @override
  State<SignUpForm> createState() => _SignUpFormState();
}

class _SignUpFormState extends State<SignUpForm> {
  final _formKey = GlobalKey<FormState>();
  final _email = TextEditingController();
  final _password = TextEditingController();
  bool _hidePassword = true;

  @override
  void dispose() {
    _email.dispose();
    _password.dispose();
    super.dispose();
  }

  String? _validateEmail(String? v) {
    if (v == null || v.trim().isEmpty) return 'Email is required';
    final ok = RegExp(r'^[^@\s]+@[^@\s]+\.[^@\s]+$').hasMatch(v.trim());
    return ok ? null : 'Enter a valid email';
  }

  String? _validatePassword(String? v) {
    if (v == null || v.length < 8) return 'At least 8 characters';
    return null;
  }

  void _submit() {
    if (_formKey.currentState!.validate()) {
      // all fields valid
      print('Sign up ${_email.text}');
    }
  }

  @override
  Widget build(BuildContext context) {
    return Form(
      key: _formKey,
      autovalidateMode: AutovalidateMode.onUserInteraction,
      child: Column(
        children: [
          TextFormField(
            controller: _email,
            keyboardType: TextInputType.emailAddress,
            textInputAction: TextInputAction.next,
            decoration: const InputDecoration(labelText: 'Email'),
            validator: _validateEmail,
          ),
          const SizedBox(height: 16),
          TextFormField(
            controller: _password,
            obscureText: _hidePassword,
            textInputAction: TextInputAction.done,
            onFieldSubmitted: (_) => _submit(),
            decoration: InputDecoration(
              labelText: 'Password',
              suffixIcon: IconButton(
                icon: Icon(_hidePassword ? Icons.visibility : Icons.visibility_off),
                onPressed: () => setState(() => _hidePassword = !_hidePassword),
              ),
            ),
            validator: _validatePassword,
          ),
          const SizedBox(height: 24),
          FilledButton(onPressed: _submit, child: const Text('Sign up')),
        ],
      ),
    );
  }
}
```

How it works:

- `Form` groups fields. `GlobalKey<FormState>` gives access to `validate()`, `save()`, `reset()`.
- Each `validator` returns an **error string** or `null` if valid.
- `autovalidateMode`: when to run validation.

---

## 3. Other input widgets

```dart
Checkbox(value: agree, onChanged: (v) => setState(() => agree = v ?? false))
Switch(value: dark, onChanged: (v) => setState(() => dark = v))
Slider(value: volume, min: 0, max: 100, divisions: 10, label: '$volume', onChanged: (v) => setState(() => volume = v))
DropdownButton<String>(
  value: country,
  items: const [
    DropdownMenuItem(value: 'tr', child: Text('Turkey')),
    DropdownMenuItem(value: 'us', child: Text('USA')),
  ],
  onChanged: (v) => setState(() => country = v),
)
RadioListTile<int>(value: 1, groupValue: choice, onChanged: ..., title: const Text('Option'))
```

Pickers:

```dart
final date = await showDatePicker(
  context: context,
  initialDate: DateTime.now(),
  firstDate: DateTime(2000),
  lastDate: DateTime(2100),
);
if (!context.mounted) return;     // after await!
```

Dialogs, bottom sheets, snackbars:

```dart
showDialog<bool>(
  context: context,
  builder: (context) => AlertDialog(
    title: const Text('Delete?'),
    actions: [
      TextButton(onPressed: () => Navigator.pop(context, false), child: const Text('Cancel')),
      FilledButton(onPressed: () => Navigator.pop(context, true), child: const Text('Delete')),
    ],
  ),
);

ScaffoldMessenger.of(context).showSnackBar(const SnackBar(content: Text('Saved')));
```

---

## 4. Focus

```dart
final _emailFocus = FocusNode();
final _passFocus = FocusNode();

TextField(focusNode: _emailFocus, onSubmitted: (_) => _passFocus.requestFocus());
// dispose both focus nodes!

FocusScope.of(context).unfocus();   // hide keyboard (e.g. tap outside)
```

---

## 5. Keyboard overlap

The keyboard covers fields. Solutions:

- `Scaffold` already resizes the body (`resizeToAvoidBottomInset: true` default).
- Wrap forms in `SingleChildScrollView` so they can scroll when the keyboard shows.
- Read keyboard height: `MediaQuery.viewInsetsOf(context).bottom`.

---

## 6. Design principles for forms

1. Validate **on user interaction**, not before they type anything.
2. Messages must say **how to fix**: "Password needs 8+ characters", not "Invalid".
3. Right keyboard type per field.
4. Disable/ show progress on the submit button while submitting (prevent double submit).
5. **Validate again on the server.** Client validation is a UX nicety, not security.
6. Support autofill and accessibility labels.

---

## 7. Debounce typing

```dart
Timer? _debounce;

void _onChanged(String text) {
  _debounce?.cancel();
  _debounce = Timer(const Duration(milliseconds: 400), () => _search(text));
}

@override
void dispose() {
  _debounce?.cancel();
  super.dispose();
}
```

---

## Common mistakes

- Not disposing controllers/focus nodes.
- Creating the controller inside `build` (text resets every rebuild).
- Using `controller.text` for every keystroke when `onChanged` is enough.
- Trusting client-side validation for security.
- Forgetting `context.mounted` after `await showDatePicker(...)`.
- No loading state on submit → duplicate submissions.

---

## Code challenge

Build a **registration form** with:

- Name (required), email (format), password + confirm password (must match), age (number 13–120), terms checkbox (must be checked).
- Show/hide password.
- "Next" key moves focus; "Done" submits.
- Submit button shows a spinner for 2 seconds (fake request), is disabled while loading, then shows a snackbar.
- Extra: write the validators as **pure functions** in their own file and unit test them (see `07_flutter_advanced/02_testing.md`).

<details>
<summary>Hint — confirm password</summary>

```dart
validator: (v) => v == _password.text ? null : 'Passwords do not match',
```

For the checkbox you can wrap it in a `FormField<bool>` to include it in `validate()`.
</details>

---

## Interview questions

1. `TextField` vs `TextFormField`: difference?
2. What does `GlobalKey<FormState>` give you?
3. Why must you dispose a `TextEditingController`?
4. How do you prevent double submission of a form?
5. Why validate on the server as well?
6. What is debouncing and why use it for search?
7. How do you handle the keyboard covering a field?

<details>
<summary>Hints</summary>

1. `TextFormField` integrates with `Form` (validation, save, reset).
2. Access to `validate()`, `save()`, `reset()` of the form.
3. It holds listeners/resources; not disposing leaks memory.
4. Disable the button / guard with an `isSubmitting` flag.
5. Client code can be bypassed; the server is the source of truth.
6. Wait for the user to pause typing before firing work, reducing needless requests.
7. Scaffold resizing, scroll view, `viewInsets`.
</details>
