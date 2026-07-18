# Keyboard Jank Assessment Report

## Executive Summary

A plain, isolated input screen shows zero jank. The same keyboard interaction inside the real app shell is severely janky. The cause is architectural, not a single widget: retained tabs, shared wrappers, and heavy production subtrees all react to every keyboard toggle, anywhere in the app.

This branch adds a small evidence lab so that's verifiable directly, without reading the full investigation.

## Evidence at a Glance

Each scenario's recording is committed directly in [`recordings/`](recordings/) and embedded below — GitHub renders these inline when viewing this page on github.com.

### A — Plain isolated input

`/debug/keyboard-lab/plain` · no shell · no `IndexedStack` · no production widgets

<video src="recordings/A_plain_input.mp4" controls playsinline width="320"></video>

**0.0% severe jank** (0/660 frames). Clean baseline — input/keyboard alone is not the problem.

### A2 — Plain layered input

`/debug/keyboard-lab/layered` · no shell · no `IndexedStack` · no production widgets

<video src="recordings/A2_plain_layered.mp4" controls playsinline width="320"></video>

**Build p95 17.94ms · 46.79% janky** (`gfxinfo`). Ordinary layering adds real, modest cost — still no production signature.

### A3 — Plain IndexedStack + bottom nav

`/debug/keyboard-lab/indexed-stack` · synthetic shell · `IndexedStack`: yes · no production widgets

<video src="recordings/A3_plain_indexed_stack.mp4" controls playsinline width="320"></video>

**Build p95 43.87ms · 67.63% janky** (`gfxinfo`, worst of the synthetics). A synthetic retained-tab shell alone reproduces **21%** of the real shell's Viewport/Sliver CPU signature — with zero production code.

### B — Real app shell

`/` guided path · real shell · `IndexedStack` · production widgets

<video src="recordings/B_real_shell.mp4" controls playsinline width="320"></video>

**Build p95 360ms · 88.88% severe jank · worst frame 2.44s** — an order of magnitude worse than any synthetic scenario. Same Viewport/Sliver signature, now at 43%, plus real Marketplace code. Trigger location doesn't matter: Marketplace's own search bar is just as janky as Account's phone field. This session was organic, multi-screen production use — no debug-lab UI, no script beyond the guided steps above — so it stands as the production-flow evidence too.

## How to Read This Report

Start with this page. It is intentionally short.

For more detail, use the per-scenario pages:

- [Scenario A — Plain isolated input](scenarios/A_plain_input.md)
- [Scenario A2 — Plain layered input](scenarios/A2_plain_layered.md)
- [Scenario A3 — Plain IndexedStack + bottom nav](scenarios/A3_plain_indexed_stack.md)
- [Scenario B — Real app shell](scenarios/B_real_shell.md)

Technical details are separated into:

- [Architecture findings](deep_dive/architecture_findings.md)
- [Metrics methodology](deep_dive/metrics_methodology.md)
- [CPU profile notes](deep_dive/cpu_profile_notes.md)

## Running the Evidence Lab

Follow the repo setup in the root `README.md`.

Important setup source of truth:

- `.vscode/launch.json` uses `--dart-define-from-file=.env.racct-dev` for mobile debug/profile/release.
- `racct use dev` generates `.env.racct-dev` and the ignored native dev config files.
- Generated ignored files and secrets should not be committed.

Launch normally for production:

```text
/
```

Launch directly into the lab menu with:

```text
--dart-define=RACCT_KEYBOARD_LAB_HOME=true
```

or navigate to:

```text
/debug/keyboard-lab
```

## Proposed Solution Direction

Three tiers, lowest effort first, each validated against this harness before the next. Full rationale: [Architecture findings](deep_dive/architecture_findings.md).

1. **Quick fix** — the specific bugs already identified: country-picker missing `itemExtent`, `_measureTextHeight` caching, `keyboard_actions` visibility gating, `IndexedStack`'s non-`const` children list. Low risk, no architecture change.
2. **MediaQuery** — scope every `MediaQuery.of`/`.maybeOf` read to what's actually needed (down to Flutter's own `Scaffold`), isolate any genuinely-broad read behind a small wrapper, and apply the same `const`/stable-identity discipline to widget construction generally. What A3 and B most directly point at.
3. **Rearchitecting** — not conditional on 1-2 falling short; justified on its own by (a) the retain-everything shell being the structural cost driver, and (b) several files here (`main.dart`, `chat_view.dart`, `marketplace_screen.dart`, `manage_listings_workspace_screen.dart` — all 20,000+ lines) being hard to maintain regardless of the jank numbers. Needs care: tab retention is currently the *only* state-preservation mechanism (no separate store for scroll position/form state), so releasing a tab requires building that save/restore layer first.

## Recommendation

Treat the full keyboard-jank resolution as a scoped performance/audit task, staged through the three tiers above, rather than a one-widget assessment fix.
