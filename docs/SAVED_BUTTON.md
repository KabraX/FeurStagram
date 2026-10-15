# Saved button

An optional bottom-bar button that opens the saved-posts screen. Instagram has no
Saved tab, so the button borrows the **Reels slot** (`clips_tab`), which is hidden by
default: it is shown, re-skinned with a bookmark drawn in code, and its taps are taken
over. Toggle: *Settings → Navigation bar → Saved* (`saved_button`, off by default).

Code: `SavedButton.java` (icon, touch/click/accessibility, deep link),
`Hiders.SavedTabHook` (re-applies it every layout pass), plus small guards in
`Hiders.VisibilityHider`, `HiddenTabSwipeSkipper` and `Settings`.

## Design notes

- **Not a `nav_show_*` key.** Such keys are lockable; one that starts off would be
  frozen off by the permanent lock. Saved reveals the user's own bookmarks, not a feed.
- **Reels icon toggle stands idle** while Saved is on (the slot is Saved's). The
  `nav_show_reels` hider skips `clips_tab`, otherwise the two listeners would flip
  its visibility on every pass.
- **The Reels page stays unreachable.** The touch listener swallows every event on the
  slot, and `HiddenTabSwipeSkipper` treats `clips_tab` as unreachable while Saved is
  on, so neither a tap nor a swipe can land on Reels — including under the lock.
- **Degrades to nothing.** If `clips_tab` is missing in a build, no button appears.
  If the deep link has no handler, the tap falls back to the Profile tab.

## Verifying the destination (required)

`Config.DEFAULT_SAVED_URI` (`instagram://saved`) is **unverified**. Confirm it before
relying on it.

1. Quick runtime probe on a device with stock Instagram signed in:
   ```sh
   adb shell am start -a android.intent.action.VIEW -d "instagram://saved" com.instagram.android
   ```
   Saved posts on screen → confirmed. Home/nothing → wrong route.
2. Static: decompile the APK with jadx and search the deep-link router.
   - `AndroidManifest.xml`: the activity with `<data android:scheme="instagram"/>`
     (the URL handler) and any `https` host filters for `instagram.com`.
   - Strings near other known routes in the router (`"direct-inbox"`, `"camera"`,
     `"user"`, `"media"`): look for `saved`, `collection`, `saved_collection`.
   - Web form as a candidate: `https://www.instagram.com/<username>/saved/`
     (needs the username, so prefer an `instagram://` route).
3. Try a candidate on the patched build without rebuilding (`--debug` build):
   ```sh
   adb shell am broadcast -p <package> -a com.feurstagram.debug.SET \
       --es key saved_uri --es value "instagram://<candidate>"
   ```
   Then put the confirmed value in `Config.DEFAULT_SAVED_URI`.

Note: `am monitor` will not reveal the route — Saved is a fragment inside the main
activity, not a separate activity.

## Inspecting the slot

```sh
adb shell uiautomator dump /sdcard/ui.xml && adb pull /sdcard/ui.xml
```
Look for `resource-id="…:id/clips_tab"` and its children. `SavedButton.findIcon` takes
the largest visible `ImageView` in the slot; if the icon is drawn by a non-`ImageView`
(custom view, Lottie wrapper), the button still works but keeps the Reels glyph —
adapt `findIcon` then. Android Studio's Layout Inspector shows the same tree live.

## Test checklist

- [ ] `./build.sh instagram.apk --clone --install`: clone opens *its own* Saved screen.
- [ ] Toggle on → Done → bookmark icon in the Reels slot; tap opens Saved.
- [ ] Toggle off → Done → slot follows the Reels icon toggle again.
- [ ] Reels icon row greys out while Saved is on, and back when off.
- [ ] Survives a cold restart and a return from background (listeners re-applied).
- [ ] Permanent lock on with Reels hidden: Saved can still be toggled; neither tap nor
      swipe ever lands on Reels.
- [ ] Landing page (search/direct/profile) still redirects; Saved is not a landing option.
- [ ] Swiping across the bar skips the Saved slot.
- [ ] TalkBack announces "Saved, button"; double-tap opens Saved.
- [ ] Rotation; Instagram light and dark themes (icon colour follows the other icons).
