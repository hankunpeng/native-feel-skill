# 03 — WebView Survival Guide

This is the densest file in the skill. Every section is a real, reproducible bug in `WKWebView` or `WebView2` when you try to use it as a native UI rendering surface instead of as a browser embed. For each, the symptom, the cause, and the exact fix.

If you take only one tenet from `references/01-philosophy.md` into this file, take *Rectification of Names*: the WebView is not a browser tab, and most of these bugs exist because its defaults assume it is one.

---

## Section A — macOS / WKWebView

### A.1 Hidden window throttling

**Symptom:** Open the launcher with a hotkey. The first animation hitches. Counters/timers seem frozen for the first frame. `requestAnimationFrame` fires at 1 Hz instead of 60 Hz when the window was offscreen.

**Cause:** WebKit aggressively throttles `requestAnimationFrame`, CSS animations, and timers when it judges the view to be occluded or off-screen. For a hidden launcher that's prewarmed but invisible, *every* WebKit heuristic concludes "this is not visible, slow it down." Browser-correct, native-app-wrong.

**Fix (two parts):**

```swift
// 1. Disable WebKit's occlusion detection on the host window.
//    Without this, WK will throttle even when the window is technically present.
window.setValue(false, forKey: "windowOcclusionDetectionEnabled")

// 2. To prewarm: order the window front but keep it visually invisible.
//    alphaValue = 0 keeps it in the front-layer for animation budgeting but invisible.
window.alphaValue = 0
window.orderFront(nil)
// …later, when actually showing:
window.alphaValue = 1
```

Combined, the WebView believes it is visible (so it doesn't throttle), but the user sees nothing until you flip alpha.

**Why this is safe:** You're not deceiving WebKit about anything that affects correctness — only about animation budgeting. Throttling is an opportunistic optimization, not a security boundary.

---

### A.2 Startup flicker — empty frame before first paint

**Symptom:** Window appears as a white/black rectangle for one frame, then content fades in. Especially visible on hotkey-triggered launches.

**Cause:** `NSWindow.orderFront()` makes the window visible *before* the WebView has rendered its first frame. The window shows whatever the backing surface contains — which is empty.

**Fix:** Use the private but stable `_doAfterNextPresentationUpdate` API to synchronize window visibility with WebView paint completion.

```swift
webView.perform(Selector(("_doAfterNextPresentationUpdate:")), with: { [weak window] in
    window?.orderFront(nil)
} as @convention(block) () -> Void)
```

This call schedules a callback for after the next time WebKit hands a rendered frame to the system. Order-front *inside* the callback. The window now appears with content already drawn.

**Caveat:** This API is private. It has been stable for ~10 years and Raycast Beta is shipping with it as of 2026. If Apple removes it, fall back to a fixed 50ms delay after `loadHTMLString` returns.

---

### A.3 Expand-from-compact rendering gap

**Symptom:** Compact launcher window grows vertically when results appear. The newly revealed area is blank for 1–2 frames, then fills in.

**Cause:** WebKit measures the visible viewport of `WKWebView`. When the WebView's *frame* changes size, WebKit has to re-layout and re-paint the newly exposed region. There's a one-frame gap between "frame became taller" and "newly tall region painted."

**Fix:** Keep the `WKWebView`'s frame at the **maximum** size always. Resize only the host window. The WebView always has the full content rendered; the window simply unmasks more of it.

```swift
// Don't do this:
webView.frame = NSRect(x: 0, y: 0, width: windowWidth, height: currentContentHeight)

// Do this:
webView.frame = NSRect(x: 0, y: 0, width: windowWidth, height: MAX_HEIGHT)
window.setFrame(NSRect(..., height: currentContentHeight), display: true, animate: true)
```

The WebView is bigger than its container; the window clips it. Frame-perfect resize.

---

### A.4 Animated window resize stutters

**Symptom:** When the window animates (e.g., growing on result expansion), the WebView contents stutter or freeze mid-animation.

**Cause:** `NSWindow.setFrame(_:display:animate:)` with `animate: true` uses Cocoa's animation primitive that *suspends drawing on subviews* during the animation. WebKit specifically opts out of drawing during this suspension.

**Fix:** Replace `animate: true` with an explicit Core Animation transaction:

```swift
NSAnimationContext.runAnimationGroup({ ctx in
    ctx.duration = 0.18
    ctx.timingFunction = CAMediaTimingFunction(name: .easeOut)
    ctx.allowsImplicitAnimation = true
    window.setFrame(newFrame, display: true)
}, completionHandler: nil)
```

Now the WebView keeps drawing every frame because it's a normal layer-backed view participating in implicit Core Animation, not subject to the legacy Cocoa animation suspension.

---

### A.5 Translucency / vibrancy / Liquid Glass

**Symptom:** You want the WebView to blend into the window's vibrancy material (frosted/blurred background). Default WebView paints opaque white.

**Fix:**

```swift
webView.setValue(false, forKey: "drawsBackground")
// And on the underlying scrollview/layer, ensure transparency:
webView.layer?.backgroundColor = NSColor.clear.cgColor
```

On the CSS side: do not set `body { background: white }`. Use `background: transparent` and let the window's `NSVisualEffectView` (or `Liquid Glass` material on macOS Tahoe/26+) show through.

**Liquid Glass specifically (macOS 26+):** Use `NSGlassEffectView` (the new material introduced for the macOS Tahoe-era design language) as the window's content background, and ensure the WebView is fully transparent above it.

---

### A.6 Context menu, link previews, magnifying glass

**Symptom:** Right-clicking opens WebKit's web context menu ("Reload", "Inspect Element"). Long-pressing on a word opens the Dictionary lookup. Force-touch reveals link previews.

**Cause:** Browser defaults. None of these belong in a native app.

**Fix:**

```swift
class CustomWebView: WKWebView {
    override func willOpenMenu(_ menu: NSMenu, with event: NSEvent) {
        menu.removeAllItems()  // or repopulate with native items
    }
    override func mouseDown(with event: NSEvent) {
        // suppress double-click word selection's auto-lookup if needed
        super.mouseDown(with: event)
    }
}
```

For Force Touch link previews, set the link's CSS `-webkit-touch-callout: none` or intercept at the navigation delegate.

---

### A.7 Popovers and tooltips inside the WebView

**Symptom:** Tooltips and popovers built as DOM elements inside the WebView don't match native styling, can be clipped by the window, can't extend beyond the window edge.

**Fix:** **Render popovers as native windows, not DOM elements.** When the React UI wants to show a tooltip, send an IPC message to the native shell:

```
{ kind: "show-popover", anchor: {x, y, w, h}, content: "…", direction: "below" }
```

The shell opens an `NSPopover` (mac) / a custom borderless `Window` (Windows). The native popover handles edge clipping, shadow, animation, focus. The WebView just *requests* it.

This is one of the highest-leverage native-feel changes you can make.

---

### A.8 Scroll inertia and rubber-banding

**Symptom:** Scrolling lists feels like scrolling a web page (slightly delayed, document-style inertia) rather than like scrolling a native list (snappier, contact-tracking).

**Fix:** This is hard to fully fix because WebKit owns the scroll physics. Mitigations:
- Use `overscroll-behavior: contain` in CSS to kill the rubber band on inner scrollers.
- For your hottest list (the search results), consider rendering it natively (above the WebView as a native overlay) and only using WebView for the parts that don't scroll. Most apps don't do this; it's a hard-mode option.

---

## Section B — Windows / WebView2

### B.1 Initialization white-rectangle flash

**Symptom:** When WebView2 initializes inside your window, there's a 100–300 ms white flash before content paints.

**Cause:** `CoreWebView2` initializes asynchronously. Your window is visible before the WebView has loaded its environment, navigated, and painted.

**Fix:**

1. Start the WebView2 environment *before* showing the window:
   ```csharp
   var env = await CoreWebView2Environment.CreateAsync(
       userDataFolder: appDataPath,
       options: new CoreWebView2EnvironmentOptions {
           AdditionalBrowserArguments = "--disable-features=msSmartScreenProtection",
       });
   await webView.EnsureCoreWebView2Async(env);
   ```
2. Set `webView.DefaultBackgroundColor = Color.Transparent` (or your window background color) *before* `EnsureCoreWebView2Async` completes — the default is white.
3. Don't show the window until `NavigationCompleted` fires for the first time.

---

### B.2 Acrylic blur with custom title bar

**Symptom:** You want acrylic blur behind the WebView with a custom (chromeless) title bar. The acrylic doesn't render, or the WebView paints opaque on top of it.

**Fix (Windows 11+):**

1. Enable acrylic on the window via `DwmSetWindowAttribute` with `DWMWA_SYSTEMBACKDROP_TYPE = DWMSBT_TRANSIENTWINDOW` (acrylic) or `DWMSBT_MAINWINDOW` (mica).
2. Extend client area to entire window: `DwmExtendFrameIntoClientArea(hwnd, new MARGINS { cyTopHeight = -1 })`.
3. Set WebView2 background to transparent: `webView.DefaultBackgroundColor = Color.FromArgb(0, 0, 0, 0)`.
4. In CSS: `body { background: transparent; }`.
5. Handle hit-testing manually for drag regions (CSS `-webkit-app-region: drag` works in WebView2 only with `IsNonClientRegionSupportEnabled = true`).

This is finicky. Test on Windows 10, 11, 11 22H2, and 11 24H2 — each had subtle changes.

---

### B.3 Background window throttling (Chromium)

**Symptom:** Your app needs to update its tray icon, refresh data, or render notifications while the main window is hidden. Chromium throttles aggressively.

**Fix:**

```csharp
webView.CoreWebView2.Settings.IsBackgroundResourceLoadingEnabled = true;
// And via additional browser arguments at env creation:
AdditionalBrowserArguments = "--disable-renderer-backgrounding --disable-background-timer-throttling --disable-backgrounding-occluded-windows"
```

Chromium will still suspend the *frame* (no painting), but timers and JS execution continue. If you need actual rendering while hidden (rare), keep the window at `alphaValue`-equivalent (Win32: `SetLayeredWindowAttributes` with alpha 0) instead of `Hide()`.

---

### B.4 Per-window WebView2 environment

**Symptom:** Multiple windows share one WebView2 environment. One window's crash takes down all of them.

**Fix:** Create one `CoreWebView2Environment` per *logical window kind*, with appropriate user data folder isolation:

```csharp
var envForLauncher = await CoreWebView2Environment.CreateAsync(
    userDataFolder: Path.Combine(appData, "launcher"));
var envForSettings = await CoreWebView2Environment.CreateAsync(
    userDataFolder: Path.Combine(appData, "settings"));
```

This is more memory but isolates failure domains. Settings window crashing should never kill the launcher.

---

### B.5 IME / composition input

**Symptom:** CJK input methods (Pinyin, Japanese kana, Korean) misbehave inside the WebView — wrong composition box position, ghost characters, or events fire twice.

**Cause:** WebView2 forwards IME events but the focus model between WPF/WinUI and WebView2 doesn't always agree on which control has focus.

**Fix:**

```csharp
webView.GotFocus += (_, _) => {
    // Ensure WebView2 receives IME events
    webView.CoreWebView2.Settings.IsBuiltInErrorPageEnabled = true;
};
// In your custom title bar, make sure it doesn't steal focus
titleBarPanel.Focusable = false;
```

Test with Pinyin specifically — it's the most demanding combination of fast typing + composition window + candidate selection.

---

## Section C — Cross-platform behaviors

### C.1 Do not show `cursor: pointer` on rows

Native apps do not change the cursor on hoverable list items. Web apps do. The single most "this feels like a web app" tell.

```css
.row, button, a {
  cursor: default;
}
```

Override only for true draggable handles or text-selection-disabled areas where a different cursor is platform-conventional.

---

### C.2 Disable text selection by default

Native widgets do not allow selecting the text in their labels. Web pages do.

```css
* { user-select: none; -webkit-user-select: none; }
.user-content, .editable { user-select: text; }  /* opt back in */
```

---

### C.3 Match platform scrollbar conventions

macOS: scrollbars only appear while scrolling (overlay style). Windows 11: thin scrollbars by default; classic scrollbars otherwise.

```css
/* macOS */
@media (prefers-color-scheme: dark) {
  ::-webkit-scrollbar { width: 8px; }
  ::-webkit-scrollbar-thumb { background: rgba(255,255,255,0.2); border-radius: 4px; }
}
```

Better: send a CSS variable from the host shell encoding the platform's scrollbar policy and conditionally render.

---

### C.4 Disable smooth-scroll JS polyfills

Browsers added smooth-scrolling JS APIs that *feel* premium on the web. They feel laggy in a native app.

```js
window.scrollTo({ top: y })   // default 'auto' — instant, native
// Avoid:
window.scrollTo({ top: y, behavior: 'smooth' })
```

---

### C.5 No page transitions

Native windows don't fade between pages. They cut. Make sure your router (React Router etc.) is not animating transitions.

---

### C.6 Honor system reduced-motion preference

Cheap and high-impact:

```js
const reduced = window.matchMedia('(prefers-reduced-motion: reduce)').matches;
```

Disable your show/hide animations when this is true. Native apps already honor this; users will notice if you don't.

---

## How to use this file

When the user reports a specific symptom ("the window flashes white on launch"), grep this file for the symptom verbatim and apply the fix. Don't reinvent.

When the user is *designing* their WebView host, read sections A and B end-to-end and apply preventively. Most of these bugs are easier to design around than to fix after the fact.
