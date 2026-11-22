---
name: clip-video
description: Clip a provided YouTube/Twitch VOD URL or local video file end-to-end into verified, titled 1080x1920 vertical clips. Use when the user provides a video link or file and asks to clip it, extract all high-score clips, or turn a full video into vertical shorts.
---

# Clip a provided video

Process a full YouTube/Twitch VOD URL or local video file end-to-end into verified, titled `1080x1920` vertical clips.

## Mode A: Step-by-step high-score batch clipping (no Telegram/queue side effects)

### 1. Score the full video (audio-first)

```bash
.venv/bin/python scripts/moments.py "<url-or-file>" \
  --top 50 --max-minutes 0 --min-score 2.0
```

- For remote URLs, downloads audio-only (`source_audio.m4a`) plus live chat replay and YouTube "most replayed" heatmap when available, saving ~95% bandwidth compared to downloading the full video upfront.
- Transcribes speech with `faster-whisper` (`small`, CUDA `float16` when available), snaps clip boundaries to natural sentence starts/ends (`15–40s`), and writes `jobs/<id>/candidates.json` and `jobs/<id>/candidates.md`.
- Raise `--top 100` (or higher) on multi-hour VODs when the user wants as many high-score clips as possible.
- If system `libcublas.so.12` lives under `/usr/local/lib/ollama/cuda_v12`, prefix with `LD_LIBRARY_PATH=/usr/local/lib/ollama/cuda_v12` so `faster-whisper` runs on GPU instead of falling back to CPU.

### 2. Cut each high-score candidate into a 1080x1920 vertical clip

```bash
.venv/bin/python scripts/cut.py jobs/<id> \
  --pick <n> --layout facecam --encoder auto
```

- For remote URLs, fetches only the padded 1080p clip segment on demand (`jobs/<id>/segments/seg_<n>_*.mp4`).
- Encodes with RTX `h264_nvenc` (automatic CPU `libx264` fallback), normalizes loudness to `-14 LUFS`, burns word-by-word captions and hook box, and writes `jobs/<id>/edits/<channel>_clip_<n>_<title_slug>.mp4` + `.json`.
- Layouts: `facecam` (cam over gameplay, falls back to `blur` if no face is found), `react` (face on top, game below, title box on seam), `blur` (full frame centered over blurred background), `fill` (center crop).

### 3. Pre-post rule & quality check

```bash
.venv/bin/python scripts/check.py \
  "jobs/<id>/edits/<channel>_clip_<n>_<title_slug>.mp4" \
  --campaign vice-city-daily/campaigns.md \
  --caption $'<hook>\n\n🎥 Credit: @<channel>\n#GTA6 #GTAVI #RockstarGames #Gaming #ViceCity'
```

## Mode B: Full automated pipeline with smart queue & Telegram approval

```bash
.venv/bin/python scripts/auto_pipeline.py \
  --url "<video-url>" \
  --max-clips 0 \
  --min-score 2.0 \
  --max-minutes 0 \
  --layout facecam \
  --campaign vice-city-daily/campaigns.md
```

Runs `moments.py` -> `cut.py` (with automatic `blur` fallback) -> `check.py` -> `smart_queue.py` -> Telegram approval dispatch in one command (`--max-clips 0` cuts up to 50 candidates clearing `--min-score`).

## Runtime estimation guide (when the user asks how long a run will take)

1. Run `speedtest-cli --simple` to measure actual download speed.
2. Run `yt-dlp -F --no-playlist "<url>"` and `yt-dlp --print "%(duration)s %(duration_string)s %(title)s" --no-playlist "<url>"` to inspect audio size (`format 140` m4a) and video duration.
3. Estimate total runtime as:
   - Audio download (`audio_MB / download_MBps`) + `~30s` WAV conversion
   - Transcription: `video_duration / ~35` on CUDA GPU (or `/ ~4` on CPU)
   - Rendering: `num_clips × ~25s` for on-demand 1080p segment fetch + NVENC cut
