---
name: clip-live
description: Continuously watch a currently-live Twitch or YouTube stream and automatically produce verified, titled 1080x1920 vertical clips on every chat spike. Use when the user gives a live stream URL or asks to clip a currently-live stream.
---

# Clip a currently-live stream

Continuously monitor a live Twitch or YouTube stream, detect chat explosions, and automatically cut verified, titled `1080x1920` vertical clips in near real time.

## 1. Run the live watch-and-cut pipeline

```bash
.venv/bin/python scripts/live_watch_cut.py <live-stream-url> \
  --cam auto \
  --ratio 2.0 \
  --minutes 720 \
  --campaign vice-city-daily/campaigns.md \
  --out-dir vice-city-daily/media/edits
```

- `--watch-only`: Attach to an already-running `scripts/live.py` job for the same stream URL (`jobs/<id>/live/`) without spawning a second recorder.
- `--cam x:y:w:h`: Pass fixed fractional webcam coordinates if known, or `--cam auto` to auto-detect the facecam.
- `--channel <name>`: Override the inferred streamer handle.
- GPU acceleration: If system `libcublas.so.12` is installed under `/usr/local/lib/ollama/cuda_v12`, prefix the command with `LD_LIBRARY_PATH=/usr/local/lib/ollama/cuda_v12` for CUDA-accelerated Whisper transcription.

## 2. What happens automatically on each chat spike

1. `live.py` records the stream in 30s chunks and cuts raw `jobs/<id>/live/spike_<n>.mp4` segments around chat spikes (`[spike-15s, spike+20s]`).
2. `live_watch_cut.py` waits for each spike file to stabilize and skips Twitch commercial/intermission slates (`is_ad_screen`).
3. Normalizes the clip to 720p30 CFR, transcribes speech with `faster-whisper`, and generates a clean reaction title/hook via `moments.build_hooks`.
4. Renders a `1080x1920` vertical clip using `cut.py --layout react` (automatically falling back to `--layout blur` if no webcam/gameplay split is available).
5. Verifies resolution, fps, audio, duration, and banned terms with `check.py`, saves `<channel>_clip_<n>_<title_slug>.mp4` and `.json` to `jobs/<id>/edits/`, copies the `.mp4` to `--out-dir`, and appends ready-to-post details + `⚠️ POSSIBLE SPOILER` flags to `jobs/<id>/ready_clips.md`.

## 3. Safety & review rules

- Before starting, confirm the stream is public and not a leak, pirated copy, or "play early" broadcast.
- Review `jobs/<id>/ready_clips.md` and inspect any clip flagged `⚠️ POSSIBLE SPOILER` before posting.
- Always credit `@<channel>` in the post caption (never burned onto the video).
