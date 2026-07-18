# Metrics Methodology

## Capture types

For each scenario, collect:

- MP4 screen recording.
- DevTools Performance JSON.
- DevTools CPU Profiler JSON.
- Console `[KeyboardPerf]` logs when available.
- Optional `get_widget_rebuild_counts` snapshot.

## Metrics to summarize

Use the same metrics for every scenario:

- frame count,
- build p95,
- build p99,
- build max,
- severe jank percentage,
- key CPU markers.

## Severe jank

Use a consistent threshold across scenarios. For 120Hz devices, 2× frame budget is approximately 16.7ms. If a different refresh rate is used, note it clearly.

## Interpretation rules

- Do not compare captures from different devices/build modes as if they are equal.
- Prefer same-session, same-device, same-build comparisons.
- Treat screen recordings as visual evidence and profiler exports as quantitative evidence.
- Keep detailed CPU interpretation out of the main report unless needed.
