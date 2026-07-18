# Scenario A3 — Plain IndexedStack + Production-like Bottom Nav

## Purpose

Test whether `IndexedStack` and retained bottom-navigation structure alone reproduce the severe keyboard stutter, without real production widgets.

## Route

```text
/debug/keyboard-lab/indexed-stack
```

## Code added for this scenario

- `KeyboardJankLabIndexedStackScreen`
- Synthetic `IndexedStack`
- Multiple lightweight retained tabs
- Production-like bottom nav styling
- One input tab for keyboard testing
- No real Marketplace, Chat, Account, or heavy production widgets

## Recording

<video controls src="../recordings/A3_plain_indexed_stack.mp4" style="max-height:528px;width:auto;"></video>

(Raw file also committed in [`recordings/`](../recordings/A3_plain_indexed_stack.mp4) for reference.)

## Metrics

Captured on the physical 23129RAA4G device (Android, debug build). The complete DevTools exports are:

- Performance: `A3_indexed_stack_performance.json` (2,197 Flutter frames, 120Hz capture)
- CPU Profiler: `A3_indexed_stack_cpu_profile.json` (20,910 samples / 7.42s sampled CPU time, 79.06s wall-clock capture span)

**DevTools Performance export:** p50/p95/p99 are percentiles across all sampled frames — p50 is the typical (median) frame, p95 means only 5% of frames were slower than this, p99 means only 1% were. Max is the single worst frame in the capture.

| Metric | p50 | p95 | p99 | Max | What this measures |
|---|---:|---:|---:|---:|---|
| Build | 9.81ms | 43.87ms | 137.04ms | 588.69ms | UI-thread time: constructing/laying out widgets |
| Raster | 4.34ms | 12.71ms | 16.52ms | 84.22ms | GPU-thread time: painting the built frame |
| Total elapsed | 17.70ms | 60.02ms | 166.28ms | 607.17ms | Full frame time — what actually reaches the screen |

At the capture's 120Hz refresh rate (8.33ms frame budget):

| Threshold | Frames | Percentage | What this means |
|---|---:|---:|---|
| Total elapsed > 8.33ms | 1,996 / 2,197 | 90.85% | Almost every frame misses the 120Hz budget at all |
| Total elapsed > 16.67ms (2x budget) | 1,252 / 2,197 | 56.99% | Lower share than A2 (77.73%) at this specific threshold — but see the rows below |
| Total elapsed > 33.33ms | 303 / 2,197 | 13.79% | ~3.6x A2's share — A3's slow frames are rarer but far more extreme |
| Total elapsed > 50ms | 153 / 2,197 | 6.96% | ~8x A2's share at this threshold |
| Total elapsed > 100ms | 51 / 2,197 | 2.32% | ~10x A2's share — A3 has fewer bad frames overall, but a much longer, worse tail |

(A short live `diagnose_jank` window reported 0 jank — didn't overlap the capture's heavier parts, same limitation as A2.)

**System-compositor confirmation (`dumpsys gfxinfo`):** clean, isolated capture — reset immediately before 3-4 rounds of keyboard toggles on an already-warm Input tab, nothing else happening:

| Metric | Value | What this means |
|---|---:|---|
| Total frames rendered | 414 | Isolated keyboard-toggle window only |
| Janky frames | 280 (67.63%) | Worst of the three synthetic scenarios (A: 0%, A2: 46.79%) |
| Janky frames (legacy) | 410 (99.03%) | Essentially every frame missed its deadline by this stricter metric |
| 50th / 90th / 95th / 99th percentile | 40 / 65 / 85 / 133ms | Consistently elevated, not just a few outliers |

**Rebuild evidence** (129 frames, frame 13,515–13,643, 8.50s):

| Widget/group | Rebuilds / 8.50s | Rate | What this means |
|---|---:|---:|---|
| `StreamBuilder` in `account_access_gate.dart:91` | 111 | 13.1/sec | The app-root access stream rebuilds on almost every keyboard-driven frame in this window. |
| Root `Selector`, `Localizations`, textured background, `MediaQuery`, `NotificationListener`, `TooltipTheme`, and `AccountAccessGate` | 105 each | 12.4/sec each | The entire shared root wrapper group reconstructs repeatedly, matching the pattern already confirmed in A and A2. |
| A3 `Scaffold` at `keyboard_jank_lab_indexed_stack.dart:40` | 72 | 8.5/sec | This app-code `Scaffold` constructor call site rebuilds far less often than the root cascade above it. |
| A3 bottom-nav `SafeArea` at `keyboard_jank_lab_indexed_stack.dart:59` | 72 | 8.5/sec | Rebuilds in lockstep with the Scaffold above it. |
| `IndexedStack` at `keyboard_jank_lab_indexed_stack.dart:50` | 4 | 0.47/sec | The `IndexedStack` widget itself — i.e. a genuinely new `IndexedStack(...)` instance passed down from this app's own State — is reconstructed only 4 times in the whole window. |
| Retained-tab list rows: `ListTile` / `Icon` / primary `Text` at lines 172-174 | 153 each | 18/sec each (in aggregate) | These 153 builds occurred in only 7 of the 129 sampled frames — not spread evenly. |
| Retained-tab secondary `Text` at line 175 | 81 | — | Same 7-frame pattern as the row builds above. |
| `_RetainedSampleTab`, `_InputTab`, `TextField`, `OutlinedButton` (Input tab) | 4 each | 0.47/sec each | These rebuild in lockstep with `IndexedStack` — confirming all 3 offscreen tabs and the visible Input tab are reconstructed together, as one event. |

The gap between `Scaffold` (72) and `IndexedStack` (4) is the key signal in this scenario, and it is explained below.

### Rebuild flow: two distinct mechanisms, not one

Tracing the 7 frames with any retained-tab-row rebuild activity shows two different patterns, not one cascade:

**Mechanism 1 — the established root cascade** (small, frequent, already confirmed in A/A2):

```text
Android IME show/hide animation
  └─ FlutterView metrics change (`viewInsets`)
      └─ root MediaQuery published by MaterialApp updates
          └─ MaterialApp.router `builder` runs
              └─ Selector / Localizations / RacctThemeTexturedBackground / MediaQuery /
                 NotificationListener / TooltipTheme / AccountAccessGate
                   105 rebuilds / 8.50s (12.4/sec)
                   └─ device StreamBuilder
                        111 rebuilds (13.1/sec)
```

3 of the 7 frames (13,603 / 13,608 / 13,614 — 3 `ListTile`s each) co-occur with `StreamBuilder`/`Selector` firing — genuinely this cascade.

**Mechanism 2 — a separate, rare, much more expensive event** (the dominant cost):

The other 4 frames (13,515 / 13,516 / 13,559 / 13,560 — 36 `ListTile`s each, 144 total) show `IndexedStack` itself rebuilding — but only 1 of those 4 co-occurs with Mechanism 1's cascade. The root cascade alone doesn't explain the largest, most expensive rebuild events:

```text
(trigger not fully isolated in this pass — see below)
  └─ A3 Scaffold (keyboard_jank_lab_indexed_stack.dart:40)
       72 rebuilds/8.50s, but only 4 reconstruct a genuinely new IndexedStack instance
       └─ IndexedStack (keyboard_jank_lab_indexed_stack.dart:50)
            4 rebuilds/8.50s — reconstructs ALL FOUR IndexedStack children at once,
            because the `children:` list is built fresh, non-const, on every call
            ├─ _RetainedSampleTab('Browse')  → 24-row ListView.separated
            │     up to 36 ListTile/Icon/Text rebuilds per event
            ├─ _InputTab (visible)           → TextField, OutlinedButton
            ├─ _RetainedSampleTab('Orders')  → 24-row ListView.separated
            │     up to 36 ListTile/Icon/Text rebuilds per event
            └─ _RetainedSampleTab('Account') → 24-row ListView.separated
                  up to 36 ListTile/Icon/Text rebuilds per event
```

Two facts anchor Mechanism 2 to source:

1. **Confirmed in code**: `_KeyboardJankLabIndexedStackScreenState.build()` builds `IndexedStack`'s `children:` as a fresh, non-`const` list on every call (`keyboard_jank_lab_indexed_stack.dart:52-57`). No tab is `const` or keyed, so Flutter can't tell them apart from new — every one of the 4 events rebuilds **all four tabs**, including the three offscreen ones.
2. **A framework-level contributor**: `Scaffold`'s own SDK code (`scaffold.dart:2917`) reads unscoped `MediaQuery.of(context)` to support `resizeToAvoidBottomInset` (on by default, not overridden here). This re-wraps `body` in a fresh `MediaQuery` on all 72 `Scaffold` rebuilds without reconstructing `IndexedStack` itself — plausibly explaining most of the 72-vs-4 gap.

Open question: what triggers the 4 rare full `Scaffold`+`IndexedStack` reconstructions specifically (vs. Scaffold's own non-cascading reaction). Consistent with ~2 toggles × 2 boundary points, but not conclusively isolated.

**CPU profile** (corrected — see note):

| Metric | Value | What this means |
|---|---:|---|
| App-code nearest-frame attribution | 457 / 20,910 (2.19%) | Small, as expected — no production code exists here |
| Top app attribution | `_AccountAccessGateState.build` (125), `_RetainedSampleTab` item builder (67) | The known root gate plus this scenario's own retained-list rows |
| `Viewport`-related frames present in stack | 4,410 / 20,910 (**21.09%**) | The headline finding — see below |
| `SliverMultiBoxAdaptorElement` present in stack | 3,309 / 20,910 (15.82%) | Same signature, second marker |
| `IndexedStack` present in stack | 1,010 / 20,910 (4.83%) | The widget itself is a small share despite driving the above |
| `MediaQuery` present in stack | 478 / 20,910 (2.29%) | The root-cascade / Scaffold mechanisms above |
| `AccountAccessGate` present in stack | 208 / 20,910 (0.99%) | Small on its own, as in A/A2 |
| `ItemCard` / `MarketplaceScreen` attribution | 0 | Confirms zero production code anywhere in this capture |

> Methodology note: summing per-function-name inclusive counts (many distinct `RenderViewport*` names added separately) initially gave an inflated 43.19%/46.22%. Corrected method: count each sample once if *any* Viewport/Sliver frame appears in its stack — the 21.09%/15.82% above.

**Why 21.09% matters**: this synthetic, production-free scenario touches the same `Viewport`/`SliverMultiBoxAdaptorElement` machinery that the real Marketplace-grid shell (Scenario B / forensic §27) drives to 47-54% — with zero `ItemCard`/`MarketplaceScreen` code anywhere here. The signature is not specific to Marketplace's own code; a synthetic `IndexedStack` with three plain `ListView`s reproduces a meaningful share of it alone.

## Interpretation

Worst of the three synthetic scenarios by every measure: build p99 137ms, `gfxinfo` 67.63% janky (99.03% legacy), 21% of CPU samples touching `Viewport`/`Sliver` — with no production content anywhere.

Not one single cause — three mechanisms compound:

| # | Mechanism | Role |
|---|---|---|
| 1 | `IndexedStack`'s non-`const` children list forces all 3 offscreen tabs to rebuild together on each of its 4 events | Direct cause of the worst individual frames (360-500ms builds) |
| 2 | Root `MediaQuery`/`AccountAccessGate` cascade, ~105-111/8.5s (same as A/A2) | Explains the smaller, frequent churn — not the 4 large events |
| 3 | `Scaffold`'s own unscoped internal `MediaQuery` read (`resizeToAvoidBottomInset`) | Explains most of the 72-vs-4 rebuild gap; adds real, currently-unmeasured cost |

Neither `IndexedStack` alone nor the root cascade alone is the full cause — the measured cost is the *combination* of how much retained content must reconcile when the `IndexedStack` host rebuilds, plus a framework-level `MediaQuery` dependency inside `Scaffold` that can't be bypassed without `resizeToAvoidBottomInset: false`.

## See more

- [Evidence index](../evidence/index.md)
- [Architecture findings](../deep_dive/architecture_findings.md)
