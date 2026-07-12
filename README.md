# 🎬 vocal-edit — AI Video Editing via Conversation

**Edit professional videos entirely through conversation.** Drop footage in a folder and describe what you want in plain English. The AI inventories, transcribes, analyzes, proposes a cut, and renders — all through structured conversation.

---

## 🎯 Core Concept

This project pairs **Claude Code** (the AI coding CLI) with **[video-use](https://github.com/browser-use/video-use)** (a skill that gives it video-editing abilities). Together they form an AI post-production pipeline:

- **Inventory** — ffprobe every source, transcribe every take, pack transcripts
- **Analyze** — read transcripts, identify best takes, verbal slips, key moments
- **Strategize** — propose a cut structure and edit style for user approval
- **Execute** — build an EDL, extract segments, grade, animate, overlay, subtitle
- **Iterate** — user feedback → re-cut → re-render (transcripts are cached)

---

## 🚀 Setup

### 1. AI Gateway: OmniRoute

[OmniRoute](https://omniroute.online) is a free local AI router. It runs on your machine and proxies requests to hundreds of providers, pooling their free tiers and automatically falling back when one hits quota.

**Setup guide:** Follow the detailed walkthrough on [this LinkedIn post](https://www.linkedin.com/feed/update/urn:li:activity:7481586111169892353/) — it covers the full installation and configuration. (Credits to the original author for the setup guide.)

**Current version:** v3.8.46

### 2. Claude Code config (`~/.claude/settings.json`)

```json
{
  "env": {
    "ANTHROPIC_BASE_URL": "http://localhost:20128",
    "ANTHROPIC_MODEL": "oc/deepseek-v4-flash-free",
    "ANTHROPIC_SMALL_FAST_MODEL": "opencode-go/deepseek-v4-pro-max",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "opencode-go/glm-5.2",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "opencode-go/minimax-m3",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "opencode-go/hy3-preview",
    "ANTHROPIC_AUTH_TOKEN": "sk-..."
  },
  "model": "oc/deepseek-v4-flash-free"
}
```

`ANTHROPIC_BASE_URL` points to the OmniRoute gateway. Calls to "Anthropic" models are intercepted and routed to whatever model you specify. **`oc/deepseek-v4-flash-free`** is the main model — completely free via OmniRoute's provider pool.

### 3. Install video-use Skill

This is what gives Claude Code video-editing abilities:

```bash
git clone https://github.com/browser-use/video-use ~/Developer/video-use
cd ~/Developer/video-use && uv sync
mkdir -p ~/.claude/skills
ln -sfn ~/Developer/video-use ~/.claude/skills/video-use
brew install ffmpeg
```

**Current toolchain:** Claude Code v2.1.207 · ffmpeg v8.0.1 · yt-dlp 2026.02.04 · Python 3.14.4 · uv 0.9.18

### 4. Transcription: ElevenLabs Scribe

Transcription requires an ElevenLabs API key with **Speech-to-Text** permission:

```bash
printf 'ELEVENLABS_API_KEY=%s\n' "sk-your-key" > ~/Developer/video-use/.env
chmod 600 ~/Developer/video-use/.env
```

---

## 🧰 The video-use Helper Scripts

Located in `~/Developer/video-use/helpers/`, these Python modules are the engine:

| Script | Purpose |
|---|---|
| `transcribe.py` | Single-file ElevenLabs Scribe transcription with word-level timestamps |
| `transcribe_batch.py` | Multi-worker parallel transcription for bulk source files |
| `pack_transcripts.py` | Raw Scribe JSON → readable markdown transcript with speaker labels |
| `timeline_view.py` | Filmstrip + waveform visualization for timeline review |
| `render.py` | **Core renderer** — EDL ingestion, segment extraction, lossless concat, overlays, subtitles |
| `grade.py` | Color grading filter chains (exposure, contrast, saturation, LUTs) |

### render.py — The Rendering Engine

The most critical script. It reads an **EDL** (Edit Decision List — a JSON file describing every cut and effect), then:

1. **Extracts** segments from source files with ffmpeg (`-c copy` where possible)
2. **Concats** segments losslessly using the concat demuxer
3. **Applies per-segment color grading** via `grade.py` filter chains
4. **Composes overlays** (lower thirds, logos, animated elements) at precise time offsets
5. **Burns subtitles** last (critical ordering — see Hard Rules)

---

## 🎥 The Editing Pipeline

### Workflow

```
┌─────────────────────────────────────────────────────┐
│  INVENTORY                                          │
│  ├─ ffprobe every source file for resolution,       │
│  │   codec, duration, frame count, audio channels   │
│  └─ Transcribe all takes + pack to readable format  │
├─────────────────────────────────────────────────────┤
│  ANALYZE                                            │
│  ├─ Read packed transcripts to understand content   │
│  ├─ Note: verbal slips, filler density, best takes  │
│  └─ Map time-ranges to content segments             │
├─────────────────────────────────────────────────────┤
│  STRATEGIZE (user reviews before execution)         │
│  ├─ AI proposes: structure, take choices, cuts,     │
│  │   animations, transitions, color grade           │
│  └─ User confirms or adjusts                        │
├─────────────────────────────────────────────────────┤
│  EXECUTE                                            │
│  ├─ Build EDL as JSON → render.py                   │
│  ├─ Extract segments with ffmpeg                    │
│  ├─ Apply color grade per segment                   │
│  ├─ Build animations in parallel sub-agents         │
│  ├─ Compose overlays + transitions                  │
│  └─ Burn subtitles LAST                             │
├─────────────────────────────────────────────────────┤
│  PREVIEW & ITERATE                                  │
│  ├─ Quick 720p render for review                    │
│  ├─ Self-evaluate: cut boundaries, audio pops,      │
│  │   overlay alignment, subtitle timing             │
│  └─ User feedback → re-cut → re-render              │
└─────────────────────────────────────────────────────┘
```

### Edit Decision List (EDL) Format

Every cut is represented as a JSON segment:

```json
{
  "segments": [
    {
      "source": "take_01.mp4",
      "trim_start": 12.5,
      "trim_end": 45.2,
      "start": 0.0,
      "end": 32.7,
      "grade": {
        "exposure": 0.85,
        "contrast": 1.1,
        "saturation": 1.05,
        "temp": -500,
        "lut": "warm_cinematic.cube"
      },
      "overlays": [
        {"type": "text", "content": "Chapter 1", "start": 2.0, "end": 6.0, "style": "lower_third"}
      ]
    }
  ]
}
```

### Output Structure

All session artifacts live in `edit/` alongside your source footage:

```
<project_folder>/
├── take_01.mp4              ← original footage (untouched)
├── take_02.mp4
└── edit/
    ├── final.mp4             ← rendered output
    ├── preview.mp4           ← low-res review copy
    ├── subtitles.ass         ← ASS subtitle styles
    ├── takes_packed.md       ← readable transcript with timestamps
    ├── project.md            ← session context (appended each iteration)
    ├── edl.json              ← the cut decisions
    ├── segments/             ← extracted segment files
    └── animations/           ← generated overlay assets
```

### Example: "Haaland for the Win"

Real project: Clash of Clans lyrics video → 30-second YouTube Short.

| Before | After |
|---|---|
| 1280×720 horizontal | **1080×1920 vertical** |
| 1m 51s | **30s** |
| Raw gameplay with lyrics | **Karaoke word-by-word popping subtitles** |
| — | Blurred background fill (padding 16:9 → 9:16) |
| — | Fade in/out transitions |
| — | Orange CAPS 72pt subtitles, popping on beat |

---

## 🛠️ Hard Rules

These are enforced by the video-use skill every render:

| # | Rule | Why |
|---|---|---|
| 1 | **Subtitles applied LAST** | Overlays would hide captions if applied after |
| 2 | **Per-segment extract → lossless concat** | Avoids double-encoding a single filtergraph |
| 3 | **30ms audio fades at segment boundaries** | Prevents audible pops between cuts |
| 4 | **Overlays use `setpts=PTS-STARTPTS+T/TB`** | Shifts overlay frame 0 to its window start |
| 5 | **Output-timeline subtitle offsets** | Captions misalign if timed against source timestamps |
| 6 | **Never cut inside a word** | Snap cut edges to word boundaries from transcription |
| 7 | **Pad every cut edge (30–200ms)** | Absorbs Scribe timestamp drift |
| 8 | **Word-level verbatim ASR only** | SRT/phrase mode loses sub-second gap data |
| 9 | **Cache transcripts per source file** | Never re-transcribe unless the source changed |
| 10 | **Parallel sub-agents for animations** | Wall time ≈ slowest single animation |
| 11 | **Strategy confirmation before execution** | Never render until the user approves the cut plan |
| 12 | **All output in `edit/`** | Never write inside the video-use repo or outside the project |

### Common ffmpeg Patterns Used

- **Lossless concat:** `ffmpeg -f concat -safe 0 -i segments.txt -c copy output.mp4`
- **Padded background fill:** `ffmpeg -i input.mp4 -vf "scale=w:h:force_original_aspect_ratio=decrease,pad=w:h:x:(ow-iw)/2:color=black@0.5"`
- **Subtitle burn (last step):** `ffmpeg -i composed.mp4 -vf "ass=subtitles.ass" output.mp4`
- **Overlay timing:** `ffmpeg -i base.mp4 -i overlay.png -filter_complex "[1:v]setpts=PTS-STARTPTS+T/TB[ovr];[0:v][ovr]overlay=x:y:enable='between(t,T,T+D)'"`

---

## 💡 Practical Tips

- **Transcription costs money** (ElevenLabs Scribe pay-per-minute) but is cached. Once a file is transcribed, you can iterate the edit for free.
- **Keep OmniRoute running** in a separate terminal (`omniroute`) before launching `claude`.
- **Switch models mid-session** by editing `~/.claude/settings.json` — try `auto/free` for zero cost or `auto` for smart routing.
- **Animation engines** are installed lazily per project: PIL (simplest), HyperFrames (HTML/CSS/GSAP), Remotion (React), Manim (math).
- **Ideal for:** talking heads, tutorials, launch videos, montages, interviews, travel vlogs, music videos, gaming clips.

---

## 📁 Quick Reference

| What | Path |
|---|---|
| Project root | `/Users/shalinshah/Developer-Shalin/video` |
| Edit outputs | `/Users/shalinshah/Developer-Shalin/video/edit/final.mp4` |
| video-use repo | `~/Developer/video-use` |
| Claude config | `~/.claude/settings.json` |
| Fire it up | `cd /Users/shalinshah/Developer-Shalin/video && claude` |

---

---

## 📢 Attribution

This repository is based on an original workflow and project concept created by **Shalin Shah** ([Shalin-Shah-2002](https://github.com/Shalin-Shah-2002)).

The underlying tools — **Claude Code**, **video-use**, **OmniRoute**, **FFmpeg**, **ElevenLabs**, and others — belong to their respective creators and are credited in the Resources section below.

If you use this repository as the basis for a video, tutorial, article, or derivative project, please provide proper credit by linking back to:  
**https://github.com/Shalin-Shah-2002/vocal-edit**

---

## 📚 Resources & Credits

| Project | Author | Link |
|---|---|---|
| **video-use** | [browser-use](https://github.com/browser-use) org — [gregpr07](https://github.com/gregpr07), [ShawnPana](https://github.com/ShawnPana) | https://github.com/browser-use/video-use |
| **OmniRoute** | [diegosouzapw](https://github.com/diegosouzapw) (Diego Rodrigues) | https://github.com/diegosouzapw/OmniRoute |
| **ElevenLabs** | ElevenLabs | https://elevenlabs.io |
| **Claude Code** | Anthropic | https://claude.ai/code |
| **ffmpeg** | FFmpeg team | https://ffmpeg.org |
