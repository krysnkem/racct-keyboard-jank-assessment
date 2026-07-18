# Keyboard Jank Recording Checklist

Use the same device, build mode, and keyboard for every scenario.

## Setup

1. Follow the root `README.md` setup.
2. Run `racct use dev` so `.env.racct-dev` and ignored native dev config files exist.
3. Use VS Code `Flutter Mobile Debug` or `Flutter Mobile Profile`, or an equivalent `./bin/racct run` flow.
4. For lab-first launch, add:

```text
--dart-define=RACCT_KEYBOARD_LAB_HOME=true
```

## Per-scenario recording steps

For each scenario:

1. Show the scenario label on screen.
2. Focus the text field.
3. Wait for the keyboard to fully open.
4. Dismiss the keyboard.
5. Repeat 3 times.
6. Keep the clip short, ideally 10-20 seconds.
7. Capture DevTools Performance JSON and CPU Profiler JSON for the same interaction.
8. Save console `[KeyboardPerf]` logs when available.

## MP4 handling

Use MP4, not GIF.

**Actual approach used for this evidence set**: committed directly under `recordings/` in the report folder, rather than GitHub PR/issue user-attachments — all four scenario recordings combined re-encode to well under GitHub's size limits (7.7MB total), so there's no need for the attachment-upload workflow or Git LFS. This was a deliberate change from the original plan below once actual capture sizes were known; keep committing directly for any future scenario in this same range. If a future recording is large enough that committing directly would meaningfully bloat the repo, fall back to the GitHub user-attachments approach instead (drag into any PR/issue comment box, paste the resulting `https://github.com/user-attachments/assets/<id>` URL into the doc).

Recording names used:

```text
recordings/A_plain_input.mp4
recordings/A2_plain_layered.mp4
recordings/A3_plain_indexed_stack.mp4
recordings/B_real_shell.mp4
```

Compression used (re-encode from the original HEVC capture to H.264 for both size and cross-browser playback compatibility — HEVC decode support is inconsistent outside Safari):

```bash
ffmpeg -i input.mp4 \
  -c:v libx264 \
  -preset slow \
  -crf 28 \
  -pix_fmt yuv420p \
  -movflags +faststart \
  -c:a aac \
  -b:a 96k \
  output_compressed.mp4
```
