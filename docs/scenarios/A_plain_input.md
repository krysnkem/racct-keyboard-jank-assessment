# Scenario A — Plain Isolated Input

## Purpose

Test whether keyboard show/hide is inherently janky with a plain input outside the production app shell.

## Route

```text
/debug/keyboard-lab/plain
```

## Code added for this scenario

- `KeyboardJankLabPlainScreen`
- One plain `TextField`
- No `MainScreen`
- No `IndexedStack`
- No production bottom nav
- No Marketplace grid or retained production tabs

## Recording

<video controls src="recordings/A_plain_input.mp4" style="max-height:528px;width:auto;"></video>

(Raw file also committed in [`recordings/`](../recordings/A_plain_input.mp4) for reference.)

## Metrics

Captured on the physical 23129RAA4G device (Android, debug build), three independent ways during a real repeated focus → keyboard-open → dismiss loop:

**Frame jank (`diagnose_jank`, Flutter-side frame timing, 10s window):**

| Metric | Value | What this means |
|---|---:|---|
| Frames sampled | 660 | 10-second window of real keyboard toggles |
| Janky frames (> 16.6ms) | 0 (0.0%) | Flutter's own frame timing reports zero jank here |

**Native IME animation timing (timestamped `adb logcat`, independent of any Flutter/MCP tooling):**

| Cycle | Duration | Frame updates | Avg inter-frame gap | Max inter-frame gap | What this means |
|---|---:|---:|---:|---:|---|
| 1 | 484ms | 54 | 9.1ms | 24.0ms | Typical cycle |
| 2 | 454ms | 53 | 8.7ms | 17.0ms | Tightest gaps of the four |
| 3 | 681ms | 54 | 12.8ms | 23.0ms | Longest cycle, gaps still nowhere near dropped-frame scale |
| 4 | 480ms | 53 | 9.2ms | 19.0ms | Typical cycle |

**CPU profile (DevTools CPU Profiler export, 4,183 samples / 1.47s window):**

| Metric | Value | What this means |
|---|---:|---|
| Marketplace/Viewport/Sliver/ItemCard inclusive touches | 0 | No production code runs in this scenario at all |
| App-code (non-Flutter-framework) attribution | 15/4,183 (0.36%) | Nearly all CPU time is framework/engine, not app code |
| `AccountAccessGate`/`MediaQuery`/`Selector` inclusive touches | 84 / 197 / 93 | The root cascade (below) is present but small next to the whole capture |

### Rebuild flow: trigger to leaf widgets

An 8-second live rebuild-count sample (the Lab-home entries reflect route-transition overlap, not steady-state keyboard activity):

| Widget/group | Rebuilds / 8s | Rate | What this means |
|---|---:|---:|---|
| `StreamBuilder` in `account_access_gate.dart:91` | 139 | 17.4/sec | Rebuilds most — the gate's own Firestore stream adds rebuilds beyond the root cascade below |
| Root wrapper chain (`Selector`, `Localizations`, textured background, `MediaQuery`, `NotificationListener`, `TooltipTheme`, `AccountAccessGate`) | 127 each | 15.9/sec each | The keyboard-driven root cascade — reconstructs together on every inset change |
| Scenario A `Scaffold` | 115 | 14.4/sec | Rebuilds in near-lockstep with the root cascade above it |
| Scenario A `SafeArea` | 29 | 3.6/sec | Only rebuilds on layout-relevant inset changes |
| Previous Lab-home `Scaffold` | 24 | 3.0/sec | Route-transition overlap, not steady-state keyboard activity |
| Action-button `Text` widgets | 3 each | 0.4/sec each | Static content, essentially unaffected |

```text
Android IME show/hide animation
  └─ FlutterView metrics change (`viewInsets`)
      └─ root MediaQuery published by MaterialApp updates
          └─ MaterialApp.router `builder` runs
              └─ Selector<HelperLanguageController, String>
                   127 rebuilds / 8s (15.9/sec)
                   └─ Localizations.override
                        127 rebuilds
                        └─ AnnotatedRegion<SystemUiOverlayStyle>
                             └─ RacctThemeTexturedBackground
                                  127 rebuilds
                                  └─ MediaQuery.of(context).copyWith(textScaler: 1.0)
                                       127 rebuilds
                                       └─ NotificationListener<ScrollNotification>
                                            127 rebuilds
                                            └─ TooltipTheme
                                                 127 rebuilds
                                                 └─ Stack
                                                      └─ AccountAccessGate
                                                           127 rebuilds
                                                           └─ device StreamBuilder
                                                                139 rebuilds
                                                                └─ router child: Scenario A
                                                                     └─ Scaffold
                                                                          115 rebuilds
                                                                          └─ SafeArea
                                                                               29 rebuilds
                                                                               └─ Padding
                                                                                    └─ Column
                                                                                         ├─ scenario heading/instructions
                                                                                         ├─ TextField
                                                                                         ├─ Spacer
                                                                                         └─ navigation buttons
```

A second, independent invalidation source sits inside the gate itself:

```text
Firestore device-access snapshot
  └─ AccountAccessGate StreamBuilder
       └─ rebuilds the routed-child boundary
```

139 vs. 127: the gate rebuilds 12 more times than the root cascade, so ~12 of its rebuilds come from this Firestore stream, not the keyboard.

## Interpretation

Smooth by every measure: 0% janky frames, sub-25ms IME gaps, zero CPU signal from any production widget. Keyboard/input alone does not explain the production stutter.

Caveat: this branch predates the investigation's own fix for the `AccountAccessGate`/root `MediaQuery` cascade, so it's still measurably present here (84/197/93 inclusive CPU touches) — just not expensive enough on its own to drop frames. Consistent with the thesis: modest alone, severe once heavy production subtrees sit underneath it (compare [A3](A3_plain_indexed_stack.md), [B](B_real_shell.md)).

## See more

- [Evidence index](../evidence/index.md)
- [Metrics methodology](../deep_dive/metrics_methodology.md)
