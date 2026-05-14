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

Eight tenets. Each section of the skill cites the tenet it serves.

1. **Form vs Essence (Aristotle)** — the matter is HTML; the essence is a native app.
2. **Rectification of Names (Confucius)** — a WebView is *not* a browser tab.
3. **Wu Wei (Taoism)** — don't fight the platform; adopt its conventions verbatim.
4. **Map ≠ Territory (Korzybski)** — Activity Monitor's numbers are a map, not the reality.
5. **Ship of Theseus** — identity lives in user muscle memory, not implementation.
6. **Dichotomy of Control (Stoicism)** — you can't beat the WebView+Node baseline; optimize what you can.
7. **Plato's Cave** — memory shadows ≠ memory pressure.
8. **Skandhas (Buddhism)** — the app is an aggregate of processes; quality = IPC quality.

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
