# 01 — Philosophy

Eight tenets. Every concrete decision later in this skill reduces to one of these. When advising, name the tenet, then cite the concrete consequence.

---

## 1. Form vs Essence (Aristotle)

> *"We're not a web app with some native hooks sprinkled on top. We're a native app that uses web for its UI."* — Raycast

Aristotle distinguished the **matter** (what a thing is made of) from the **form** (what it *is*). A bronze statue is bronze, but a statue. A native-feel desktop app is HTML+JS, but a native app.

The category mistake to avoid: assuming that **what the UI is built from** determines **what the app is**. It doesn't. The app's essence is its *behavior* — how it launches, how it handles focus, how it composites with the OS. The form (DOM and CSS) is incidental.

**Consequence:** Stop measuring success by "is this written in Swift/Rust/whatever native." Measure by "would a skeptical user notice this is a WebView." If the answer is no, the essence is native, regardless of the matter.

---

## 2. 正名 — Rectification of Names (Confucius)

> *"If names be not correct, language is not in accordance with the truth of things."* — Analects XIII.3

A `WKWebView` inside your `NSWindow` is **not a browser tab**. The moment you call it a "browser embed," you import every browser assumption — back/forward buttons, address bar metaphors, autosuspend-when-hidden, scroll inertia tuned for trackpad scrolling of documents.

Name it correctly: it is a **rendering surface** owned by a native app. With the right name, decisions follow:
- It doesn't need a User-Agent. (It's not browsing.)
- It doesn't need to throttle when "hidden." (The native shell knows what hidden means; the WebView doesn't.)
- It doesn't need its own context menu. (The native shell provides one.)

**Consequence:** Audit every default WebView behavior. If it makes sense only because WebViews evolved from web browsers, turn it off.

---

## 3. Wu Wei — Effortless Action (Taoism)

> *"道常无为而无不为"* — Dao De Jing 37 ("The Way takes no action, yet leaves nothing undone.")

The exhausting way to build native feel is to override the platform: re-implement scrolling, re-skin controls, polyfill window chrome. The Taoist way is to *not fight*.

- macOS draws sheets a certain way. Don't ship a modal that floats centered with a backdrop blur unless macOS would.
- Windows uses acrylic title bars. Don't paint your own.
- The OS knows what color the menu bar should be in dark mode. Don't compute it.

**Consequence:** When a design ask conflicts with platform convention, the *first* question is "what does the OS do here?" and the second is "can we just do that?" Custom is the last resort.

---

## 4. The Map is Not the Territory (Korzybski)

> Activity Monitor says 420 MB. Real memory cost: maybe 180 MB.

The numbers your OS shows you are a **map** — a model of reality optimized for system-monitor UIs. The territory is the actual page state in physical RAM:
- **Compressed pages**: counted as resident but cost a fraction in real RAM.
- **Shared frameworks**: counted against every process that links them.
- **Clean pages**: instantly evictable; cost approaches zero under pressure.
- **Dirty pages**: the only thing that actually constrains the system.

**Consequence:** Before optimizing memory, learn to read the map honestly. Most "memory problems" in WebView+Node apps are mapping artifacts. See `references/05-memory-truths.md`.

---

## 5. Ship of Theseus — Identity Through Change

If you rewrite the AppKit shell into Swift, replace the search index with Rust, swap the UI from native to React, and add a Windows port — is it still the same app?

The Raycast answer: **yes, because identity lives in the user's muscle memory.** ⌘-Space still opens the launcher. ⌘-K still triggers actions. The fuzzy match still ranks the same way. The keystrokes-to-result count is the same. *That* is the ship; the planks are replaceable.

**Consequence:** When tempted to "modernize" a UX detail during a rewrite, ask whether you're replacing a plank (allowed) or repainting the ship a different color (a betrayal of identity). Plank swaps preserve identity. Behavior changes destroy it.

---

## 6. Dichotomy of Control (Epictetus / Stoicism)

> *"Of things some are in our power, and others are not."*

You **cannot** make WKWebView cost less than ~50 MB at rest. You **cannot** make Node use less than ~12 MB cold. You **cannot** prevent Chromium from using a GPU helper process.

You **can** control: how many WebViews you instantiate, when you tear them down, how aggressively you reuse them, whether dirty pages get flushed before window-hide, what's in your bundle.

**Consequence:** Don't pick fights with baselines you didn't author. Spend optimization budget on the dirty pages your code creates, not the platform constants your runtime imports.

---

## 7. Plato's Cave (Memory edition)

Activity Monitor's "Memory" column is a shadow on the cave wall. The shadow looks alarming. The real form — the system memory pressure graph — is what tells you whether you have a problem.

Engineers who optimize against the shadow chase phantoms: they make the Activity Monitor number go down by techniques that may *increase* real cost (e.g., forcing decompression to "release" memory, which immediately gets re-allocated and dirtied).

**Consequence:** When the user says "our app uses too much memory," ask: "compared to what, measured how, under what workload, with what pressure?" Most of the time there is no answer, only the shadow.

---

## 8. Skandhas — The App is an Aggregate (Buddhism)

Buddhism teaches the self is not a single entity but a temporary aggregation of five components (skandhas). A modern desktop app is similar:

- A **native shell** process (Swift / C#)
- A **WebContent** process (the WebView's renderer)
- A **GPU helper** process (WebKit / Chromium)
- A **Networking** process
- A **Node backend** process
- A **Rust subprocess** (file indexer, etc.)
- An **Updater** process

These are stitched together by IPC and a single window in the user's mind. None is "the app." The cohesion is in the protocol between them.

**Consequence:** Architectural quality is the quality of the contracts between these processes. See `references/04-ipc-contract.md` — IPC is not plumbing, it is the spine.

---

## Cross-tenet summary

| Tenet | What it forbids | What it requires |
|---|---|---|
| Form vs Essence | Caring about implementation purity | Caring about user-perceived behavior |
| Rectification of Names | Calling the WebView "an embedded browser" | Calling it a rendering surface and acting accordingly |
| Wu Wei | Re-implementing platform behavior | Adopting platform behavior verbatim where possible |
| Map is Not Territory | Trusting Activity Monitor | Reading dirty pages, pressure, compressed state |
| Ship of Theseus | Repainting UX during a rewrite | Preserving muscle-memory invariants |
| Dichotomy of Control | Fighting WebView/Node baselines | Optimizing your own dirty pages |
| Plato's Cave | Chasing shadow metrics | Measuring system pressure |
| Skandhas | Treating "the app" as monolithic | Investing in the IPC contract |

When a recommendation in a later file lands, point back to the tenet it serves. The reader should be able to predict the recommendation from the tenet alone.
