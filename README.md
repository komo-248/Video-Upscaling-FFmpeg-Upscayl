# Video Upscaling – FFmpeg + Upscayl (realesr-animevideov3-x4)

Frame-by-frame 4x video upscaling pipeline using yt-dlp for source download, FFmpeg for frame extraction and reassembly, and Upscayl with the `realesr-animevideov3-x4` model for AI upscaling. Handles both CFR and VFR source video correctly, and reassembles output with high-quality x265 encoding.

---

## Table of Contents

- [Overview](#overview)
- [Requirements](#requirements)
- [Model Selection](#model-selection)
- [Pipeline](#pipeline)
  - [Step 1 — Download Source](#step-1--download-source)
  - [Step 2 — Pre-Check: CFR vs VFR](#step-2--pre-check-cfr-vs-vfr)
  - [Step 3 — Frame Extraction](#step-3--frame-extraction)
  - [Step 4 — Upscale](#step-4--upscale)
  - [Step 5 — Reassemble](#step-5--reassemble)
- [x265 Encoding Parameters](#x265-encoding-parameters)
- [CFR vs VFR Reference](#cfr-vs-vfr-reference)

---

## Overview

Standard video upscaling tools apply interpolation across the whole video, which blurs fine detail and introduces artifacts at scene cuts. This pipeline instead:

1. Extracts every frame as a lossless PNG
2. Upscales each frame independently at 4x using a trained super-resolution model
3. Reassembles the upscaled frames with the original audio into a high-quality x265 encode

The result preserves sharp linework and fine detail that interpolation-based upscalers destroy — particularly effective on anime and animated content.

---

## Requirements

| Tool | Purpose | Install |
|------|---------|---------|
| [yt-dlp](https://github.com/yt-dlp/yt-dlp) | Download video/audio | `pip install yt-dlp` |
| [FFmpeg](https://ffmpeg.org/download.html) | Frame extract + reassemble | System install |
| [Upscayl](https://upscayl.org/) | AI frame upscaling | Desktop app |
| GPU | Required for Upscayl | dGPU recommended (see Step 4) |

---

## Model Selection

Three models were evaluated for this pipeline:

| Model | Best For | Notes |
|-------|---------|-------|
| **`realesr-animevideov3-x4`** ✓ | Anime / animated video | Best temporal consistency, sharpest linework, designed specifically for video frames |
| `Upscayl Digital Art` | Static illustrations | Oversaturates and haloes on video frames |
| `waifu2x` | Clean line art | Good on stills, flickering artifacts on motion frames |

**`realesr-animevideov3-x4` was used for this pipeline.** It is trained on video frames rather than static images, which significantly reduces inter-frame flickering.

---

## Pipeline

### Step 1 — Download Source

Download the best available audio and video streams separately using yt-dlp. Keeping them separate avoids re-encoding the audio track.

```bash
# Download best audio
yt-dlp -f bestaudio -o audio.m4a <YouTube-URL>

# Download best video (no audio)
yt-dlp -f bestvideo -o video.mkv <YouTube-URL>
```

> Replace `<YouTube-URL>` with the actual URL. Output filenames can be anything — just keep them consistent through the pipeline.

---

### Step 2 — Pre-Check: CFR vs VFR

Before extracting frames you need to know whether the video uses a **Constant Frame Rate (CFR)** or **Variable Frame Rate (VFR)**. This determines which FFmpeg flags to use — using the wrong mode causes frame timing errors in the reassembled output.

```bash
ffprobe -v error -select_streams v:0 \
  -show_entries stream=r_frame_rate,avg_frame_rate \
  -of default=noprint_wrappers=1 video.mkv
```

**Reading the output:**

| Result | Meaning |
|--------|---------|
| `r_frame_rate == avg_frame_rate` | CFR → use the CFR path below |
| `r_frame_rate != avg_frame_rate` | VFR → use the VFR path below |

Example output:
```
r_frame_rate=24000/1001
avg_frame_rate=24000/1001
```
These match → **CFR**.

```
r_frame_rate=60/1
avg_frame_rate=2997/100
```
These differ → **VFR**.

> Note the exact `r_frame_rate` value — you will need it in Steps 3 and 5.

---

### Step 3 — Frame Extraction

Create an output folder for the frames first:
```bash
mkdir frames
```

**CFR:**
```bash
ffmpeg -i video.mkv -c:v png -fps_mode cfr -r <r_frame_rate> frames/%08d.png
```

**VFR:**
```bash
ffmpeg -i video.mkv -c:v png -fps_mode vfr frames/%08d.png
```

Replace `<r_frame_rate>` with the value from Step 2 (e.g. `24000/1001` or `24`).

The `%08d` pattern zero-pads filenames to 8 digits (`00000001.png`, `00000002.png`, ...), ensuring correct sort order on reassembly.

> For a 1-hour video at 24fps this produces ~86,400 PNG files. Make sure you have sufficient disk space — each frame at 1080p is roughly 2–4MB uncompressed.

---

### Step 4 — Upscale

Open Upscayl and configure:

| Setting | Value |
|---------|-------|
| Input folder | `frames/` |
| Output folder | `frames_upscaled/` (create this first) |
| Model | `realesr-animevideov3-x4` |
| Scale | 4x |
| Mode | Batch |
| Output format | PNG |
| GPU | Select GPU **1** (dGPU, not integrated graphics) |

Click **Upscayl** and wait. Progress is shown per-frame. On a mid-range dGPU expect ~2–5 seconds per frame.

> Make sure GPU **1** is selected if you have both integrated and discrete graphics. Using the iGPU (GPU 0) will be significantly slower and may cause out-of-memory errors on large frames.

---

### Step 5 — Reassemble

Combine the upscaled frames with the original audio into a final x265 encode.

**CFR:**
```bash
ffmpeg \
  -framerate <r_frame_rate> \
  -i frames_upscaled/%08d.png \
  -i audio.m4a \
  -c:v libx265 \
  -pix_fmt yuv420p10le \
  -preset medium \
  -x265-params crf=8:aq-mode=3:aq-strength=0.9:psy-rd=1.5:psy-rdoq=1.0:qcomp=0.65:bframes=16:b-adapt=2:rc-lookahead=80:deblock=3,3:strong-intra-smoothing=1:ref=8:temporal-mvp=1:weightb=1:no-sao=1:limit-sao=0 \
  -c:a aac \
  -b:a 320k \
  -shortest \
  output.mkv
```

**VFR:**
```bash
ffmpeg \
  -framerate <r_frame_rate> \
  -pattern_type sequence \
  -i frames_upscaled/%08d.png \
  -i audio.m4a \
  -fps_mode passthrough \
  -c:v libx265 \
  -pix_fmt yuv420p10le \
  -preset medium \
  -x265-params crf=8:aq-mode=3:aq-strength=0.9:psy-rd=1.5:psy-rdoq=1.0:qcomp=0.65:bframes=16:b-adapt=2:rc-lookahead=80:deblock=3,3:strong-intra-smoothing=1:ref=8:temporal-mvp=1:weightb=1:no-sao=1:limit-sao=0 \
  -c:a aac \
  -b:a 320k \
  -shortest \
  output.mkv
```

The key difference between CFR and VFR reassembly:
- **CFR** uses `-fps_mode cfr` (implicit) — frames are stamped at fixed intervals
- **VFR** uses `-pattern_type sequence` and `-fps_mode passthrough` — original frame timestamps are preserved

---

## x265 Encoding Parameters

The x265 parameter string is tuned for maximum quality at low file size, suitable for archiving upscaled animation:

| Parameter | Value | Effect |
|-----------|-------|--------|
| `crf=8` | Near-lossless | Very high quality; use `crf=16–18` for smaller files |
| `aq-mode=3` | HEVC AQ | Adaptive quantization — protects detail in flat areas |
| `aq-strength=0.9` | Moderate AQ | Balanced detail/compression in uniform regions |
| `psy-rd=1.5` | High psychovisual | Preserves texture and perceived sharpness |
| `psy-rdoq=1.0` | Psychovisual RDO | Reduces DCT ringing near edges |
| `qcomp=0.65` | Rate control curve | Slight bias toward quality in complex scenes |
| `bframes=16` | Max B-frames | Maximizes compression efficiency |
| `b-adapt=2` | Adaptive B-frames | Intelligent placement at scene changes |
| `rc-lookahead=80` | Lookahead buffer | More accurate rate control over long sequences |
| `deblock=3,3` | Strong deblock | Smooths block artifacts without blurring linework |
| `strong-intra-smoothing=1` | Intra smoothing | Reduces blocking in flat regions |
| `ref=8` | 8 reference frames | Better motion compensation |
| `temporal-mvp=1` | Temporal MVP | Improves motion vector prediction |
| `weightb=1` | Weighted B-frames | Better handling of fades and blends |
| `no-sao=1` / `limit-sao=0` | SAO disabled | Prevents SAO from softening upscaled detail |
| `pix_fmt yuv420p10le` | 10-bit color | Reduces banding; compatible with most players |

> To reduce file size at the cost of some quality, increase `crf` to `14–18`. Values below `8` produce diminishing returns.

---

## CFR vs VFR Reference

| | CFR | VFR |
|--|-----|-----|
| Frame interval | Fixed | Variable |
| ffprobe check | `r_frame_rate == avg_frame_rate` | `r_frame_rate != avg_frame_rate` |
| Extract flag | `-fps_mode cfr -r <rate>` | `-fps_mode vfr` |
| Reassemble flag | *(default)* | `-pattern_type sequence -fps_mode passthrough` |
| Common sources | Most YouTube uploads | Screen recordings, mixed-rate encodes |
