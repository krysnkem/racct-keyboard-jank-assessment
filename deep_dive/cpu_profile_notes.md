# CPU Profile Notes

Consolidated summary of the CPU-profile evidence across all four captured scenarios. Full breakdowns (nearest-leaf vs. whole-stack-inclusive attribution, individual frame traces) live in each scenario's own doc — this page is the cross-scenario comparison, not a duplicate of the raw data.

| Scenario | App-code attribution (nearest-leaf) | `Viewport` inclusive | `SliverMultiBoxAdaptorElement` inclusive | Marketplace/`ItemCard` present? | Interpretation |
|---|---:|---:|---:|---|---|
| A — Plain isolated input | 0.36% (15/4,183) | 0% | 0% | No | Clean baseline — no production-shaped CPU cost at all |
| A2 — Plain layered | 3.07% (808/26,286) | 6.38% | 0.87% | No | A small synthetic list adds a small, contained `Viewport` touch; still zero production code |
| A3 — Plain IndexedStack | 2.19% (457/20,910) | 21.09% | 15.82% | No | A synthetic `IndexedStack` + three lightweight lists reproduces roughly half of B's `Viewport`/`Sliver` signature — with zero production widgets involved |
| B — Real app shell | 17.03% (2,793/16,397) | 42.76% | 28.43% | Yes — `Marketplace` 4.90%, `ItemCard` 3.84%, `_ItemCardState` 3.35% | Real production code now dominates; the same `Viewport`/`Sliver` signature appears at roughly 2x A3's synthetic figure |

## What this shows

`Viewport`/`Sliver` climbs in a straight line as retained list content increases — **0% → 6%/1% → 21%/16% → 43%/28%** (A → A2 → A3 → B) — regardless of whether that content is synthetic or real. This is the clearest single signal that the cost is structural, not screen-specific.

App-code attribution tells the complementary story: under 3.1% through A/A2/A3 (all synthetic), 17.03% only in B once real Marketplace code is present — confirming the ladder isolates shell-structure cost from production-code cost cleanly.

Methodology note: figures use the corrected "count each sample once if any matching frame appears in its stack" method, not summed per-function-name counts (which double-count). See [A3](../scenarios/A3_plain_indexed_stack.md) for the before/after correction.
