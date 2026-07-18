# Evidence Index

MP4 recordings are committed directly in [`recordings/`](../recordings/) — all four are re-encoded H.264, 7.7MB combined, well under GitHub's size limits. Click a link and GitHub's own blob viewer plays it inline.

Raw DevTools/CPU Profiler JSON and console/logcat captures are **not** committed — large, machine-specific files. Every number is quoted directly in each scenario's Metrics section (the source of truth); filenames below are for traceability only.

| Scenario | MP4 | Performance JSON | CPU JSON | Console logs |
|---|---|---|---|---|
| A — Plain isolated input | [Watch](../recordings/A_plain_input.mp4) | Not exported — used a live Flutter frame-timing check instead | 4,183 samples — summarized in [A_plain_input.md](../scenarios/A_plain_input.md) | Timestamped `adb logcat`, 4 IME cycles — summarized in scenario doc |
| A2 — Plain layered input | [Watch](../recordings/A2_plain_layered.mp4) | `scenerio a2.json` (2,308 frames) | `scenrio a2 2.json` (26,286 samples) | Timestamped `adb logcat`, 5 IME cycles — summarized in scenario doc |
| A3 — Plain IndexedStack + bottom nav | [Watch](../recordings/A3_plain_indexed_stack.mp4) | `A3_indexed_stack_performance.json` (2,197 frames) | `A3_indexed_stack_cpu_profile.json` (20,910 samples) | Live frame-timing check only — see scenario doc |
| B — Real app shell | [Watch](../recordings/B_real_shell.mp4) | `B_real_shell_performance.json` (1,961 frames, ~279s) | `B_real_shell_cpu_profile.json` (16,397 samples) | — |
