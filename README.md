# native-feel-skill

A Claude Code Skill for designing **cross-platform desktop apps that feel native** — distilled from Raycast's public 2.0 technical deep-dive and grounded in reverse-engineering of the shipping `Raycast Beta.app` binary.

> *We're not a web app with some native hooks sprinkled on top. We're a native app that uses web for its UI.* — Raycast

## What this is

A reference for architects, tech leads, and engineers who must build a desktop app that:

- runs on **macOS + Windows** (optionally Linux) from a single UI codebase,
- launches in under 500 ms and stays under 500 MB resident,
- is **indistinguishable from a native app** to its users (no `cursor: pointer` tell-tales, no white-flash on launch, no WebKit context menu, no smooth-scroll JS),
- supports a **plugin/extension ecosystem** in TypeScript,
- can share performance-critical code with iOS and a server backend.

This is the four-layer architecture: **native shell → system WebView (WKWebView/WebView2) → Node backend → Rust core**, wired together with a single typed IPC schema that generates clients for every runtime.

## What this is not

- Not for single-OS apps (just build native).
- Not for Electron-style "good enough" apps (the polish budget here is 5–10× higher).
- Not for apps with strict <150 MB or <100 ms cold-start budgets (the floor is real).
- Not for games, document editors, or media players.

Run [`checklists/decision-tree.md`](checklists/decision-tree.md) to find out if this architecture is even right for your project. It rules itself out for several common cases — saying so directly is more useful than over-fitting advice.

## Layout

```
native-feel-skill/
├── SKILL.md                                # entry point for Claude Code
├── references/
│   ├── 01-philosophy.md                    # 8 tenets that drive every decision
│   ├── 02-architecture.md                  # the four-layer architecture
│   ├── 03-webview-survival.md              # WebKit/WebView2 quirks + fixes (the goldmine)
│   ├── 04-ipc-contract.md                  # typed IPC across Rust/Swift/C#/TS
│   ├── 05-memory-truths.md                 # why Activity Monitor lies
│   ├── 06-native-conventions.md            # 70+ items the native-feel audit checks
│   └── 07-evidence-raycast.md              # what a reverse-eng. of Raycast Beta shows
└── checklists/
    ├── decision-tree.md                    # should you use this architecture?
    └── ship-readiness.md                   # 75-item launch audit
```

## Philosophy

The central tension this architecture resolves: **how can a desktop app deliver convenient cross-platform development AND near-native performance, when those goals usually pull against each other?** Eight tenets name the structural moves:

1. **Place the seam at the rendering surface** — share above the WebView, diverge below it; this is the only altitude where both DX and native feel survive.
2. **One schema, many languages** — pay the polyglot tax once at the declaration, never at the call site.
3. **Adopt the platform; don't compete with it** — the OS draws blur, scrolling, materials, and dark mode better than you can.
4. **Performance is a property of perception** — what the user feels, not what Activity Monitor reports.
5. **The short iteration loop is the product** — 200 ms hot reload vs 30 s native rebuild is a 150× compounding advantage.
6. **Cross boundaries intentionally** — IPC has a cost; design every crossing as async, batched, schema-typed.
7. **Identity is muscle memory** — the hotkey, the rank order, the verbs are the app; everything else is implementation.
8. **Separate baseline from margin** — the WebView+Node floor is rented; only your dirty pages are yours to optimize.

Read [`references/01-philosophy.md`](references/01-philosophy.md) first. Everything else is consequence.

## Installation as a Claude Code Skill

```bash
# Clone into your Claude Code skills directory
git clone https://github.com/yetone/native-feel-skill ~/.claude/skills/native-feel-cross-platform-desktop
```

Then in Claude Code, the skill activates automatically when you discuss cross-platform desktop architecture, WebView quirks, or Raycast-style apps.

## Sources

- Raycast's public technical post: [A Technical Deep Dive into the New Raycast](https://www.raycast.com/blog/a-technical-deep-dive-into-the-new-raycast)
- Reverse engineering of `Raycast Beta.app` v0.60.0 (macOS 26+ build, Xcode 17, arm64) — see [`references/07-evidence-raycast.md`](references/07-evidence-raycast.md) for what was found and how.

## License

MIT — see [`LICENSE`](LICENSE).

## Credits

Authored as a Claude Code Skill. The architecture this skill describes is Raycast's; the philosophy is the author's synthesis; the evidence is from the shipping app.
