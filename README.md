# Flutter web: TextField loses focus when the window briefly loses focus

Minimal reproduction for a Flutter web bug: when another window briefly takes
window focus (for example a password manager's autofill UI on Android, or a
system panel), a focused `TextField` is unfocused for good. A plain HTML
`<input>` keeps focus in the same situation.

- `lib/main.dart`: a single `TextField`.
- `web/plain.html`: a plain HTML `<input>` to compare against.

## Steps (Android Chrome)

1. `flutter run -d web-server --web-hostname 0.0.0.0 --web-port 8080 --release`
   (or `flutter build web` and serve `build/web`), then open
   `http://<host>:8080/` in Chrome on an Android device.
2. Tap the text field. The on-screen keyboard opens.
3. Give another window focus briefly, then return:
   - `adb shell am start -a android.settings.panel.action.INTERNET_CONNECTIVITY`,
     then dismiss the panel; or
   - with a password manager such as 1Password as the Android autofill
     service, let its autofill UI appear over the field.
4. Repeat with `http://<host>:8080/plain.html`.

**Expected:** the field still has focus and the keyboard is still open (as on
`plain.html`, where `document.activeElement` stays the `INPUT`).

**Actual:** the Flutter field is unfocused and the keyboard is gone;
`document.activeElement` is `FLUTTER-VIEW`.

## Desktop simulation

On Android the browser dispatches the input `blur` and the window `blur` as
separate tasks. That ordering can be simulated on desktop Chrome: paste this
into the DevTools console, then click the text field within 3 seconds.

```js
setTimeout(() => {
  const input = document.activeElement;
  document.hasFocus = () => false;
  input.blur();
  setTimeout(() => window.dispatchEvent(new FocusEvent('blur')), 50);
  setTimeout(() => {
    delete document.hasFocus;
    window.dispatchEvent(new FocusEvent('focus'));
    setTimeout(() => console.log(document.activeElement.tagName), 500);
  }, 300);
}, 3000);
```

It logs `BODY`: the field stayed unfocused. If the window `blur` is
dispatched in the same task as `input.blur()`, focus is restored and it logs
`INPUT`.
