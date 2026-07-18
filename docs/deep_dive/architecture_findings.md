# Architecture Findings

## Summary

The controlled scenario ladder confirms the severe keyboard stutter is architectural/performance-structural, not isolated to one `TextField`:

1. **Keyboard/input alone (A)** — clean: 0% janky frames, zero production CPU signal.
2. **Ordinary layered UI (A2)** — real but modest cost (46.79% janky per isolated `gfxinfo`, 88.07% by the legacy metric), still zero production-shell signature.
3. **Synthetic retained-tab shell (A3)** — with no production widgets at all, reproduces 21.09%/15.82% `Viewport`/`Sliver` CPU touches — roughly half of what the real shell shows — purely from `IndexedStack` retaining lightweight tabs together.
4. **Real retained production shell (B)** — an order of magnitude worse than every synthetic scenario (88.88% severe jank vs. A3's 56.99%; worst single frame 2.44s vs. A3's 607ms), with the same `Viewport`/`Sliver` signature now at 42.76%/28.43% and real Marketplace/`ItemCard` code visibly costly (17.03% app-code attribution vs. A3's 2.19%). This session was organic, multi-screen production use on the real route, so it also stands as the production-flow evidence.

Severity rises in a straight line as retained production content increases, with the same `Viewport`/`Sliver` signature present even at zero production code (A3). Not five coincidentally similar bugs — one mechanism (retained heavy subtrees rebuilding together on every keyboard-inset change) that compounds with whatever real screens sit underneath it. See [CPU profile notes](cpu_profile_notes.md) for the full cross-scenario numbers.

## Important language

Use:

- architectural/performance-structure issue
- multiple interacting rebuild/layout paths
- broader than a single local widget fix
- retained production subtrees

Avoid:

- bad architecture
- messy codebase
- needs a full rewrite

## Solution direction

A complete fix has three tiers, lowest effort first. Each stage should be validated against this same evidence harness (A/A2/A3/B) and frame metrics before moving to the next, so a regression is caught immediately rather than discovered later.

### 1. Quick fix

Apply the specific, already-identified bugs directly — narrow in scope, low risk, no architecture change, independently worth doing regardless of what else happens:

- Missing `itemExtent` on the country-code picker's `ListView.builder` — confirmed still present and unfixed on this branch (this evidence set's own §6.3/§28 finding); every row reconstructs on every animation frame instead of only the visible ones.
- `_CustomCategoryTabsState._measureTextHeight` reconstructing a fresh `TextPainter` on every `build()` instead of caching its two fixed possible heights.
- Unthrottled `didChangeMetrics` in the Input Photos screen's action-button remeasurement, firing a real render-tree walk on every keyboard animation frame instead of once after metrics settle.
- The `keyboard_actions` package's global `WidgetsBindingObserver`, which triggers an unconditional `setState` on every mounted instance — including backgrounded tabs — on every keyboard event anywhere in the app.
- `IndexedStack`'s `children:` list built fresh and non-`const` on every call (confirmed directly in both the synthetic A3 lab and the real `MainScreen` tab list) — none of the tab widgets are `const` and none carry a stable key, so Flutter's element diffing can't tell an unchanged offscreen tab from a new one and reconstructs all of them together on every rebuild, regardless of whether `MediaQuery` scoping is fixed.

### 2. MediaQuery

Systematically close the scoping gap this evidence traces through every layer this investigation touched — from the app root down to a single reusable widget, and even inside Flutter's own `Scaffold` implementation:

- Replace unscoped `MediaQuery.of`/`.maybeOf` reads with the aspect-scoped accessor that matches what's actually needed — `viewInsetsOf` only where the keyboard inset is genuinely wanted, `viewPaddingOf`/`sizeOf` for values that should be keyboard-immune. (`padding`/`paddingOf` is *not* keyboard-immune the way it looks — it's derived from `viewInsets`, so it still tracks the keyboard.)
- Isolate any widget that genuinely needs a broad `MediaQuery` read (e.g. a text-scale override) behind a small, dedicated wrapper, so that read doesn't force every widget beneath it to rebuild together.
- Explicitly set `resizeToAvoidBottomInset: false` on shells/screens that already handle the keyboard inset themselves — `Scaffold`'s own internal implementation carries an unscoped `MediaQuery` read that can't otherwise be bypassed (confirmed directly in the Flutter SDK source, see [Scenario A3](../scenarios/A3_plain_indexed_stack.md)).
- Pair this with disciplined `const`/stable widget identity wherever construction allows it. This is the same underlying principle as the scoping fixes above — give Flutter's element diffing a way to tell "unchanged" from "new" — just applied to widget construction instead of `MediaQuery`. A correctly-scoped `MediaQuery` read still won't help if the parent rebuilds for some *other* reason (a `ValueListenableBuilder`, a `setState` elsewhere) and reconstructs a non-`const`, unkeyed child list on every pass — which is exactly the mechanism the `IndexedStack` quick-fix above addresses in one specific place; this tier is the general practice of applying that same discipline everywhere construction allows it.

This is the fix this evidence set most directly supports: Scenario A3 shows a synthetic `IndexedStack` alone already reproduces a large share of the production signature purely through this mechanism, and Scenario B shows the same mechanism at full scale once real screens are attached.

### 3. Rearchitecting (layered and controlled)

Not conditional on tiers 1-2 falling short — justified independently, regardless of how much they improve the numbers:

| Reason | Evidence |
|---|---|
| **Performance** | The retain-everything shell is the structural cost driver (A3's synthetic shell alone reproduces ~half the real shell's `Viewport`/`Sliver` signature). Tiers 1-2 cut per-rebuild cost, but every visited tab stays mounted and reactive for the rest of the session regardless. |
| **Maintainability** | Several files this investigation traced through are god-files by line count: `main.dart` (22,367), `chat_view.dart` (23,364), `marketplace_screen.dart` (24,602), `manage_listings_workspace_screen.dart` (36,125). This size is exactly what made the same `MediaQuery`-scoping mistake slow to spot across 540 call sites in one sweep — no file was small enough to make the pattern obvious. |

**Needs more care than tiers 1-2**: tab retention *is* the app's current state-preservation mechanism — no separate store exists for a tab's scroll position, in-progress form, or camera state; it survives only because that subtree has never been unmounted (`_builtTabIds` is set once, never cleared). Unmounting a tab before giving that state somewhere else to live would silently lose it. So this tier has to build the save/restore layer *first*, not start with "release idle tabs":

- **Stage 1**: an explicit state layer per heavy tab (scroll position, form fields, selection), living outside the widget tree.
- **Stage 2**: lazy-mount on first visit (partially true today), release a backgrounded tab's subtree after a defined idle period, restore from Stage 1's state layer on return.
- **Stage 3**: only if still needed, move the heaviest screens (Marketplace) onto their own route-level lifecycle instead of a permanently-mounted `IndexedStack` tab.

Preserves user-facing behavior while reducing how much retained content pays the keyboard-inset cost on every toggle, anywhere in the app.
