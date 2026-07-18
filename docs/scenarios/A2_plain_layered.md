# Scenario A2 — Plain Layered Input

## Purpose

Test whether ordinary nested layout and a composer-style UI are enough to reproduce the severe production keyboard stutter.

## Route

```text
/debug/keyboard-lab/layered
```

## Code added for this scenario

- `KeyboardJankLabLayeredScreen`
- `Scaffold`, `SafeArea`, `Stack`, nested layout
- Small synthetic message list
- Bottom input/composer area
- No real production shell
- No retained production tabs

## Recording

<video controls src="../recordings/A2_plain_layered.mp4" style="max-height:528px;width:auto;"></video>

(Raw file also committed in [`recordings/`](../recordings/A2_plain_layered.mp4) for reference.)

## Metrics

Captured on the physical 23129RAA4G device (Android, debug build). The complete DevTools exports are:

- Performance: `scenerio a2.json` (2,308 Flutter frames, 120Hz capture)
- CPU Profiler: `scenrio a2 2.json` (26,286 samples / 8.88s sampled CPU time)

**DevTools Performance export:** p50/p95/p99 are percentiles across all sampled frames — p50 is the typical (median) frame, p95 means only 5% of frames were slower than this, p99 means only 1% were. Max is the single worst frame in the capture.

| Metric | p50 | p95 | p99 | Max | What this measures |
|---|---:|---:|---:|---:|---|
| Build | 12.05ms | 17.94ms | 25.31ms | 199.44ms | UI-thread time: constructing/laying out widgets |
| Raster | 4.34ms | 10.47ms | 15.51ms | 69.09ms | GPU-thread time: painting the built frame |
| Total elapsed | 19.64ms | 31.22ms | 45.45ms | 204.48ms | Full frame time — what actually reaches the screen |

At the capture's 120Hz refresh rate (8.33ms frame budget):

| Threshold | Frames | Percentage | What this means |
|---|---:|---:|---|
| Total elapsed > 8.33ms | 2,178 / 2,308 | 94.37% | Almost every frame misses the 120Hz budget at all |
| Total elapsed > 16.67ms (2x budget) | 1,794 / 2,308 | 77.73% | Most frames miss it by 2x or more — the "severe jank" figure |
| Build > 16.67ms | 191 / 2,308 | 8.28% | Build-phase cost alone rarely exceeds budget on its own |
| Total elapsed > 33.33ms | 89 / 2,308 | 3.86% | A small tail of notably slow frames |
| Total elapsed > 50ms | 20 / 2,308 | 0.87% | Rare but present |
| Total elapsed > 100ms | 5 / 2,308 | 0.22% | The extreme tail |

(Two short live `diagnose_jank` windows reported 0 jank — they simply didn't overlap the heavier parts of this longer capture; the table above is the broader, more conservative dataset.)

**Rebuild evidence (exact 8.29-second DevTools window):**

131 recorded frames (frame 947–1,082, 8.288s), cross-checked against an independent 8-second live snapshot — the two agree closely:

| Widget/group | DevTools rebuilds / 8.29s | Rate | Live rebuilds / 8s | What this means |
|---|---:|---:|---:|---|
| `StreamBuilder` in `account_access_gate.dart:91` | 99 | 11.9/sec | 97 | The app-root access stream rebuilds on almost every keyboard-driven frame in this window. |
| A2 `Scaffold` at `keyboard_jank_lab_layered.dart:40` | 97 | 11.7/sec | 95 | The layered screen's root scaffold rebuilds nearly in lockstep with the app-root wrapper. |
| Root `Selector`, `Localizations`, textured background, `MediaQuery`, `NotificationListener`, `TooltipTheme`, and `AccountAccessGate` | 94 each | 11.3/sec each | 92 each | The entire shared root wrapper group is being reconstructed repeatedly even on this synthetic route. |
| Twelve synthetic message-row `Container`s / `Text`s | 48 each in aggregate | 5.8 row-builds/sec | 36 each | These 48 builds came from only 4 frames (12 rows × 4 rebuild passes), not from every animation update. |
| A2 `SafeArea` | 24 | 2.9/sec | 23 | Layout-sensitive inset handling rebuilds during part of the transition. |
| A2 screen, `ListView`, positioned composer, `Material`, input `Expanded`, `TextField` | 4 each | 0.48/sec each | The actual input/list subtree rebuilt only four times; most repeated work was above it in the shared root/scaffold boundary. |

The pattern: a small set of root widgets rebuilding 11-12×/sec, not many different widgets — the message rows rebuilt in 4 batches, and the `TextField` itself only 4 times.

### Rebuild flow: trigger to leaf widgets

```text
Android IME show/hide animation
  └─ FlutterView metrics change (`viewInsets`) on each native animation update
      └─ root MediaQuery published by MaterialApp updates
          └─ MaterialApp.router `builder` runs again
              ├─ MediaQuery.of(context).copyWith(textScaler: 1.0)
              │    94 rebuilds / 8.29s (11.3/sec)
              │
              └─ recreates the shared wrapper chain:
                   Selector<HelperLanguageController, String>
                     94 rebuilds (11.3/sec)
                     └─ Localizations.override
                          94 rebuilds (11.3/sec)
                          └─ RacctThemeTexturedBackground
                               94 rebuilds (11.3/sec)
                               └─ MediaQuery override
                                    94 rebuilds (11.3/sec)
                                    └─ NotificationListener<ScrollNotification>
                                         94 rebuilds (11.3/sec)
                                         └─ TooltipTheme
                                              94 rebuilds (11.3/sec)
                                              └─ Stack
                                                   └─ AccountAccessGate
                                                        94 rebuilds (11.3/sec)
                                                        └─ device StreamBuilder
                                                             99 rebuilds (11.9/sec)
                                                             └─ router child: Scenario A2
                                                                  └─ Scaffold
                                                                       97 rebuilds (11.7/sec)
                                                                       └─ SafeArea
                                                                            24 rebuilds (2.9/sec)
                                                                            └─ Stack
                                                                                 ├─ Positioned.fill
                                                                                 │    └─ DecoratedBox
                                                                                 │         └─ ListView.separated
                                                                                 │              4 rebuilds
                                                                                 │              └─ 12 rows
                                                                                 │                   ├─ Container: 48 aggregate builds
                                                                                 │                   └─ Text: 48 aggregate builds
                                                                                 └─ Positioned composer
                                                                                      └─ Material
                                                                                           └─ Padding → Column → Row
                                                                                                └─ Expanded → TextField
                                                                                                     4 rebuilds
```

A second invalidation source sits at the gate: 99 vs. 94 means at least 5 of `AccountAccessGate`'s rebuilds come from its own Firestore stream, independent of the keyboard trigger. The two mechanisms can overlap:

```text
Firestore device-access snapshot
  └─ AccountAccessGate StreamBuilder
       └─ rebuilds the routed child boundary

Keyboard viewInsets update
  └─ broad root MediaQuery dependency
       └─ reconstructs the full shared wrapper chain and AccountAccessGate
```

Net effect: the repeated work sits **above** the synthetic input (root/gate/scaffold rebuilt 94–99×) while the `ListView`/`TextField` leaves rebuilt only 4× — Flutter's reconciliation stopped most of it from reaching the leaves, but still paid the wrapper/scaffold cost every time.

**CPU profile** (26,286 samples):

| Signal | Share | What this means |
|---|---:|---|
| App-code nearest-frame | 808 (3.07%) | Small — most cost is framework, not app logic |
| `_AccountAccessGateState.build` | 236 (0.90%) / 358 inclusive (1.36%) | Top app-code attribution — the root gate itself |
| Root `MediaQuery`/`AccountAccessGate`/`Selector`/textured-bg | 2.68% / 1.46% / 1.19% / 0.43% | The known root cascade, all present |
| `Viewport` / `SliverMultiBoxAdaptorElement` | 6.38% / 0.87% | This scenario's own small `ListView` — no `ItemCard`/Marketplace code anywhere |

**Timestamped native IME log:** 10 completed keyboard animations, 818–1,303ms each, most gaps 26-50ms (one reached 134ms) — measurably heavier than Scenario A's 454–681ms/17-24ms even without any production shell.

**System-compositor confirmation (`dumpsys gfxinfo`):** an initial 2,836-frame/61.42% capture was rejected — it included a 312-frame cold-boot Choreographer skip, contaminating the number with startup cost. Clean recapture, reset immediately before an isolated round of keyboard toggles on an already-warm screen:

| Metric | Value | What this means |
|---|---:|---|
| Total frames rendered | 327 | Isolated keyboard-toggle window only |
| Janky frames | 153 (46.79%) | Confirms real jank at the system-compositor level, independent of Flutter's own timing |
| Janky frames (legacy) | 288 (88.07%) | Stricter legacy metric — most frames miss it |
| 50th / 90th / 95th / 99th percentile | 25 / 32 / 34 / 42ms | Consistent, moderately elevated — not a few extreme outliers |

## Interpretation

Real but modest cost: build p95 17.94ms, `gfxinfo` confirms 46.79% janky (88.07% legacy) independently of Flutter's own timing. Still zero Marketplace/`ItemCard` signal — the app-code cost here is just the root `AccountAccessGate`/`MediaQuery` cascade plus this scenario's own small `ListView`.

Useful evidence, not a failed control: ordinary layering adds cost but doesn't reproduce the production-specific signature. [A3](A3_plain_indexed_stack.md) tests whether retaining multiple tabs is the next material step.

## See more

- [Evidence index](../evidence/index.md)
- [Metrics methodology](../deep_dive/metrics_methodology.md)
