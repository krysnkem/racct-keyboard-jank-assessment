# Scenario B — Real App Shell Guided Test

## Purpose

Test the keyboard interaction with the real app shell, retained tabs, shared wrappers, and production subtrees present.

## Route

```text
/
```

or launch from the lab menu card for Scenario B.

## Guided steps

1. Open Marketplace once so it mounts.
2. Switch to Account or another screen with a keyboard field.
3. Focus the input.
4. Open and dismiss the keyboard 3 times.
5. Capture recording and profiler metrics.

**What was actually captured**: keyboard activity wasn't confined to Account — it also hit Marketplace's own search bar (confirmed in the rebuild data below). This matters: the severe pattern below isn't specific to Account or the country-picker bug — it recurs independently on Marketplace's own keyboard interaction too.

## Code added for this scenario

No synthetic replacement UI is added for the shell itself. This scenario intentionally uses the real app shell.

## Recording

https://github.com/user-attachments/assets/2a26e9b0-95ea-4ddc-8ea9-2f48221a76e9

(Raw file also committed in [`recordings/`](../recordings/B_real_shell.mp4) for reference.)

## A different kind of capture than A/A2/A3

Deliberately not one isolated toggle: a real ~279-second guided session (Marketplace mounted → Account/phone-input focused, keyboard toggled, country picker opened) — testing the real shell with its retained tabs, not one transition in isolation.

## Metrics

Captured on the physical 23129RAA4G device (Android, debug build). The complete DevTools exports are:

- Performance: `B_real_shell_performance.json` (1,961 Flutter frames, 120Hz capture, ~279s span)
- CPU Profiler: `B_real_shell_cpu_profile.json` (16,397 samples / 5.46s sampled CPU time)

**DevTools Performance export:** p50/p90/p95/p99 are percentiles across all sampled frames — p50 is the typical (median) frame, p90/p95/p99 mean only 10%/5%/1% of frames were slower than this. Max is the single worst frame in the capture.

| Metric | p50 | p90 | p95 | p99 | Max | What this measures |
|---|---:|---:|---:|---:|---:|---|
| Build | 11.30ms | 259.56ms | 360.11ms | 500.34ms | **2426.11ms** | UI-thread time: constructing/laying out widgets |
| Raster | 13.57ms | 219.65ms | 260.53ms | 304.53ms | 639.61ms | GPU-thread time: painting the built frame |
| Total elapsed | 39.98ms | 486.64ms | 567.72ms | 771.80ms | **2444.94ms** | Full frame time — what actually reaches the screen |

At the capture's 120Hz refresh rate (8.33ms frame budget):

| Threshold | Frames | Percentage | What this means |
|---|---:|---:|---|
| Total elapsed > 8.33ms | 1,951 / 1,961 | **99.49%** | Virtually every frame misses budget |
| Total elapsed > 16.67ms (2x budget) | 1,743 / 1,961 | **88.88%** | An order of magnitude beyond any synthetic scenario (A3: 56.99%) |
| Total elapsed > 33.33ms | 1,210 / 1,961 | 61.70% | Most frames are 4x+ over budget |
| Total elapsed > 50ms | 868 / 1,961 | 44.26% | Nearly half the session |
| Total elapsed > 100ms | 812 / 1,961 | **41.41%** | 4 in 10 frames took over 100ms — vs. A3's 2.32% |

Worst single frame: **2.44s** end-to-end (2.43s in build). Bucketed into 5-second windows across the whole ~279s session, nearly every bucket shows 90-100% janky frames — chronic, not a spike.

**Confirmed: the severe pattern reproduces on Marketplace's own keyboard interaction, not just Account.** Marketplace's top-bar search field (`marketplace_screen_top_bar_builders.dart:18028`/`:18120`) shows 3 separate rebuild bursts — independent keyboard episodes triggered from Marketplace, not Account:

| Episode | Frames | Janky | Severe (>2x budget) | Max build | What this means |
|---|---:|---:|---:|---:|---|
| Marketplace search, episode 1 | 66 | 100.0% | 100.0% | 1,083.5ms | Frame 22,943 (traced below) falls in this episode |
| Marketplace search, episode 2 | 78 | 100.0% | 100.0% | 743.8ms | Same severity as episode 1 |
| Marketplace search, episode 3 | 67 | 100.0% | 100.0% | 596.6ms | Same severity again — not a one-off |

All three match session-wide severity exactly. **This means the trigger location doesn't matter**: frame 22,943's heavy rebuild (Account's country-picker and category dialog, both just retained in the background) was actually caused by a keyboard toggle on Marketplace's search bar — same mechanism, regardless of which screen asks for the keyboard.

**Rebuild evidence:** 1,127 sampled frames over 206.49s, spanning **32 distinct production files** (vs. A/A2/A3's 1-5 debug-lab files each):

| Widget/group | Rebuilds (session) | What this means |
|---|---:|---|
| `Selector`/`Text` (`helper_language.dart:982/993`) | **44,170 each** | Root translation cascade — magnitude reflects this session's length, not a new mechanism |
| `ListTile`/`HelperText` (`phone_number_input.dart:608/609`) | **5,493 each** | **Country-code picker storm** (§6.3/§28) — missing `itemExtent`, confirmed still unfixed on this branch |
| `ItemCard`/`MarketplaceItemCard` internals | **4,410 each** | First scenario where real production widgets appear in rebuild data at all |
| `wish_list_screen.dart` feed-tile widgets | ~3,080 each | Feed/Offers tab, retained via `IndexedStack`, rebuilding in the background |
| `ListTile`/`CircleAvatar` (`marketplace_screen.dart:17404-17433`) | **2,898 each** | Category-management dialog rows — a foreground-dialog source, not a background leak |
| `TickerMode`/nav bar widgets (`main.dart:18947`) | ~2,682-3,071 each | The shell's own nav/`_wrap()` machinery, entirely unmitigated on this branch |
| `KeyboardActions` (`keyboard_actions.dart:62`) | **1,421** | The global `WidgetsBindingObserver` mechanism (§15.3) at its one true source; +1,463 across 4 call sites, 2,884 total |

### Two concrete heavy frames, traced exactly

**Frame 22,943** (1,570.41ms elapsed, 417 distinct widget locations) — falls inside Marketplace search episode 1, so this keyboard event came from Marketplace's search bar, not Account:

```text
122x Selector/Text (helper_language.dart) — the root translation cascade
 21x ListTile/CircleAvatar/Icon/HelperText/IconButton (marketplace_screen.dart:17404-17433) — category-management dialog rows
 12x ListTile/HelperText (phone_number_input.dart:608-609) — country-picker rows
 10x MarketplaceItemCard/ItemCard and internals (marketplace_screen_components.dart, item_card.dart) — real Marketplace cards
  8x GestureDetector/Container/Icon/Text (marketplace_screen_components.dart:1247-1479) — marketplace controls
  7x TickerMode (main.dart:18947) — retained-tab ticker wrapper
  7x wish_list_screen.dart feed/profile widgets — the retained Feed/Offers tab
```

Key detail: the country-picker and category-dialog rows here are **not** from the user touching Account — the keyboard was raised on Marketplace. They rebuilt purely because Account sits retained in the background. One frame, all real production code, triggered by a keyboard event on a different screen entirely.

**Frame 22,462** (build 2,386.66ms, second-worst in the capture) — a later, Account-focused moment, almost entirely `PhoneNumberInput`/country-picker:

```text
242x Selector/Text (helper_language.dart) — root translation cascade
237x ListTile/HelperText (phone_number_input.dart:608-609) — every visible country-picker row, rebuilding
  7x GestureDetector/Image (account_legal_footer.dart) — the account footer
  1x each: UserAccountScreen, PhoneNumberInput, and their direct scaffolding (confirming the route/screen context)
```

2.39 *seconds* rebuilding 237 country-picker rows — the §6.3/§28 bug, at full severity, unfixed on this branch.

**Known bugs confirmed present** (all unfixed here, since this branch predates the forensic branch's fixes):

| Bug | Samples | Section |
|---|---:|---|
| `_updateMobileActionButtonsOffset` — unthrottled `didChangeMetrics` | 96 / 85 (0.59% / 0.52%) | §6.1 |
| `_CustomCategoryTabsState._measureTextHeight` — `TextPainter` rebuilt every call | 62 (0.38%) | §27.4 |
| `keyboard_actions` — global `WidgetsBindingObserver` on every instance | 52 (0.32%) | §15.3 |
| `_RacctGlassNoisePainter` — per-point `canvas.drawPoints` | 68 (0.41%) | §10 P4 |
| `support_chat_screen` activity | 38 (0.23%) | incidental to this session |

**CPU profile — the clearest contrast with A/A2/A3** (16,397 samples):

| Signal | Share | What this means |
|---|---:|---|
| App-code, nearest-leaf | **17.03%** | 4.6x A2, 7.8x A3, 47x A — real production code doing real work, not synthetic stand-ins |
| `Viewport` present in stack | 42.76% | ~2x A3's synthetic 21.09% — same signature, real shell |
| `SliverMultiBoxAdaptorElement` present in stack | 28.43% | ~2x A3's synthetic 15.82% |
| `Marketplace` / `ItemCard` / `_ItemCardState` substrings | 4.90% / 3.84% / 3.35% | Real Marketplace code, absent from every synthetic scenario |
| `MarketplaceScreen` / `_MarketplaceFeedEntryTile` | 1.66% / 1.35% | Same |
| `ChatList`/`_ChatListState` | 0.15% each | Present but small — Chat wasn't the focus of this session |

Top app-code hotspots agree by both nearest-leaf (`_ItemCardState.build`, `precacheSignupSelfiePromptImage`, `HelperLanguageController.maybeOf`) and whole-stack presence (`_ItemCardState.build` in 2.02% of samples) — all genuine production code absent from A/A2/A3.

## Interpretation

A different class of problem, not a difference in degree: 99.49% janky (vs. A3's 90.85%), 41.41% of frames over 100ms (vs. 2.32%), worst frame 2.44s (vs. 607ms).

No new, undiscovered cause — every contributor here was already found elsewhere in this investigation:

| # | Contributor | Status here |
|---|---|---|
| 1 | Country-code picker storm (5,493 rebuilds) | Already diagnosed and fixed on the forensic branch — confirmed absent its fix here |
| 2 | Real Marketplace grid rebuilding (~4,400/each) | The production content A/A2/A3 excluded by design, now present and costly |
| 3 | Several smaller known bugs (`_measureTextHeight`, `didChangeMetrics`, `keyboard_actions`, glass painter) | Present because this branch predates their fixes |
| 4 | Same `Viewport`/`Sliver` signature found synthetically in A3 | Confirmed at similar magnitude (42.76%/28.43% vs. 21.09%/15.82%) with real tabs standing in for synthetic ones |
| 5 | Trigger location doesn't matter | All 3 Marketplace-triggered episodes: 100% janky, 100% severe — identical to Account-triggered ones |

This is exactly the evidence ladder's purpose: A/A2/A3 showed the mechanism (root cascade + retained tabs) building up with zero production code; B shows that adding the real shell and its actual tabs back in compounds severity by an order of magnitude — without introducing a single new mechanism.

## See more

- [Architecture findings](../deep_dive/architecture_findings.md)
- [CPU profile notes](../deep_dive/cpu_profile_notes.md)
