---
name: zunexa-ffmpeg-montage
description: "CLI-only (ffmpeg) short-form video montage, subtitle generation, color grading and platform export for Zunexa AI X Aqlix content, tuned for low-spec hardware (i3-4005U / 12GB RAM / Debian Trixie / Xfce, no GPU acceleration). Fast staged pipeline: per-cue scene intermediates, xfade transitions, split-screen gameplay, sidechain-ducked music, SFX, burned captions. Use for Instagram Reels / YouTube Shorts / TikTok vertical video production."
---

# Zunexa — ffmpeg Montage & Subtitle Skill (fast staged pipeline)

## 0. Constraints that shape every decision
- **Hardware**: Intel i3-4005U (dual-core, 2014, no AVX2), 12GB RAM, Haswell-ULT iGPU (no usable hardware encode), 1366x768 screen, Debian 13 (Trixie), Xfce 4.20.
- **Network**: often slow/shared wifi (~2mb/s, café). No multi-GB downloads (Docker images, ASR weights, GPU toolkits).
- **Rule of thumb**: ffmpeg + python3 (both native, zero downloads) cover everything. Container/model/GPU-stack = wrong tool here.
- **The PC freezes if we are careless**: never decode more than needed, never touch 4K unless unavoidable, always run renders at low priority (§3.0).

## 1. Preferences (apply by default unless the user overrides)
- Language of subtitles/scripts/replies: **Darija (Arabic script)**.
- Branding watermark: **`Zunexa AI X Aqlix`** (Aqlix = his YouTube + Instagram), bottom-right, `x=w-tw-18:y=h-th-30:fontsize=18`, white@0.9 on black@0.35 box.
- Captions: **white text, keywords highlighted yellow** (`<font color="#FFFF00">` in the SRT), bold, black outline, **centered in the middle of the screen** (`{\an5}` + `Alignment=5`). Always pass `PlayResX/PlayResY` (§4.2).
- Default export: **720x1280 (9:16) @ 60fps**, H.264. Go 1080x1920 only if the user asks. **Never exceed 60fps** (no `minterpolate`; platforms cap at 60).
- Encoders: intermediates `libx264 -preset ultrafast -crf 17`; final `-preset superfast -crf 21`; always `-threads 2`.
- **Delivery format**: the user wants a **ready-to-run copy-paste terminal command with his real paths** (never a generic template): install missing tools, write the script via heredoc, run it, write a **log file** into the output folder. Test it in the sandbox on synthetic assets before handing it over.
- Output folder convention: `~/Workspaces/file_project/reel/<project>/`; stock in `~/Workspaces/file_project/tools_for_edite/{stock videos,sounds effect,music background}`.

## 2. Subtitle generation — 10-parameter SRT spec
When no ASR is available (usual case), use **silence-aware, character-weighted alignment**:
1. `ffmpeg -i audio.mp3 -af silencedetect=noise=-35dB:d=0.25 -f null -` → real pauses.
2. Speech intervals = complement of silences; sanity-check count vs. clauses.
3. Split script into clauses at `،` `.` and natural phrase breaks.
4. Allocate clause time proportional to characters over total *speech* time, walking the real speech intervals (word-level ASR not needed).

| # | Parameter | Target |
|---|---|---|
| 1 | Max chars/line | 41 (safe 38–42) |
| 2 | Max lines/cue | 2 |
| 3 | CPS | 12–17 (audio pace wins; lengthen into the following pause, never beyond next cue − gap) |
| 4 | Min duration | 1.0s |
| 5 | Max duration | 5.58s |
| 6 | Gap between cues | 80–120ms |
| 7 | Frame lock | snap timestamps to 1/60s |
| 8 | Segmentation | by pause/meaning, never raw word count |
| 9 | RTL | UTF-8; prefix each line with U+200F (RLM) so `،` `.` don't flip; Latin words stay inline |
| 10 | Lead-in | −80ms before speech; never truncate mid-word |

Styling inside the SRT: `{\an5}` at the start of each cue's text + `<font color="#FFFF00">keyword</font>` (AI Agent, brand names, numbers). Don't highlight a fragment of an Arabic word (e.g. "ال**هدف**" breaks shaping) — highlight whole words/Latin terms only.
Validate: no overlaps, no line > 41 chars, monotonic times.

## 3. Fast staged pipeline (default for any montage)
Scene length = one SRT cue ⇒ a visual change every ~2s (retention). Layout 720x1280: **top 720x640 b-roll/images, bottom 720x640 gameplay**, captions on the seam (middle of screen), neon seam line, hook text, progress bar, watermark.

### 3.0 Rules that make it fast
1. **Never scan/probe/decode-sample the whole stock library.** A 143-file ffprobe + frame-extract scan froze his PC. Pick from **filename metadata only**: regex `(\d{3,4})_(\d{3,4})_(\d{2})fps` → keep max side ≥1080 and ≤2000 and ≤2.3MP; skip `uhd`/4K, ultra-wide (w/h>2), names containing `(`, news/channel clips (`Video by…`, `Former OpenAI…`, `Gemini Robotics…`, subscribe-button animations, LG OLED demos) and the gameplay files. `ffprobe -show_entries format=duration` **only on the ~20 chosen clips**.
2. Seek before decode: `-ss S -t L -i clip` (input seeking), scale down immediately, `-an` for video-only stages.
3. Heavy work in **small intermediates** (720x640 ultrafast), one final pass at superfast.
4. Run everything niced: `nice -n 19 ionice -c3 python3 make_reel.py` (PC stays usable).
5. Cache anything reusable between runs; expose `SEED=n` (re-roll clips/transitions) and `KEEP=1` (keep intermediates).
6. Relevance: Pexels filenames are bare numeric IDs — there is no cheap way to know content. Use semantic hooks that ARE cheap: SRT keywords → assets (`هيغن`/Hugging Face → `huggingface.png`, `Hugging-Face-meaning-1.jpg`; `شات جي بي تي`/`OpenAI`/`أوبن` → ChatGPT images + camera-shutter SFX). Do **not** score frames by decoding files.

### 3.1 Stage A — one intermediate per cue
Boundaries = cue start times (first = 0, last ends at narration length). `d_i` = boundary gap; intermediate length `L_i = d_i + T` (T = 0.2s transition) except the last (`L = d_last`).
```bash
# video clip: pan inside a 1.2x crop, direction alternates per scene
ffmpeg -y -ss $SS -t $L -i "$CLIP" -an \
  -vf "scale=864:768:force_original_aspect_ratio=increase,crop=720:640:x='(iw-720)*min(t/$L,1)':y='(ih-640)/2',fps=60,eq=saturation=1.15:contrast=1.06,format=yuv420p" \
  -c:v libx264 -preset ultrafast -crf 17 -pix_fmt yuv420p -r 60 -threads 2 seg_NN.mp4
```
Images/logos: `-loop 1 -framerate 60 -t $L -i img` + filter_complex: blurred cover background (`format=rgb24,scale=720:640:force_original_aspect_ratio=increase,crop=720:640,gblur=sigma=30,eq=brightness=-0.12`) with the contained foreground (`format=rgba,scale=600:520:force_original_aspect_ratio=decrease,scale=w='trunc(iw*(0.92+0.1*min(t/$L,1))/2)*2':h=-2:eval=frame`) overlaid centered (`overlay=x=(W-w)/2:y=(H-h)/2:format=auto`).

### 3.2 Stage B — smooth transitions (xfade chain)
```
[0:v][1:v]xfade=transition=fade:duration=0.2:offset=O1[x1];[x1][2:v]xfade=...:offset=O2[x2]; ...
```
**`O_k` = cumulative sum of `d_0..d_{k-1}` (NOT of L)**, so total length = narration. All inputs must share size/fps/timebase (they do if made by Stage A). Rotate: `fade smoothleft circleopen smoothright fadegrays smoothup zoomin smoothdown`. Output → `top.mp4` (ultrafast).

### 3.3 Stage C — gameplay layer (bottom)
Prefer a vertical 9x16 file (e.g. Subway Surfers 2024 9x16), random start ≥20s in: `-ss R -t DUR+0.5 -i game -vf "scale=720:640:force_original_aspect_ratio=increase,crop=720:640,fps=60,eq=saturation=1.1,format=yuv420p"` → `game.mp4`. **This is the slowest stage** (4K VP9/AV1 software decode). Idea if it hurts: shrink the gameplay once to 720p and cache it. No gameplay found → fall back to full-height b-roll.

### 3.4 Stage D — final composite (single pass)
Video chain: `vstack=inputs=2` → `drawbox` seam (y=638,h=4, 0x00E5FF@0.9) → `subtitles=subtitles.srt:force_style='FontName=DejaVu Sans,FontSize=32,PrimaryColour=&H00FFFFFF,SecondaryColour=&H0000FFFF,OutlineColour=&H00000000,BorderStyle=1,Outline=3,Shadow=0,Bold=1,Alignment=5,MarginV=0,PlayResX=720,PlayResY=1280'` → hook `drawtext` (first 2.6s, fade in/out via `alpha=`, **Latin text, y≥195 to clear Instagram's top ~14% UI zone**) → progress bar (`color=c=0xFFD400:s=720x6` overlaid with `x='-w+w*t/DUR'`) → watermark.

Audio chain:
```
[voice]highpass=f=70,acompressor=threshold=-18dB:ratio=3:attack=5:release=80:makeup=3,asplit=2[vo][vsc]
[music(-stream_loop -1)]atrim=0:DUR,asetpts=PTS-STARTPTS,volume=0.55,afade=t=in:d=0.8,afade=t=out:st=DUR-1.6:d=1.6[mu0]
[mu0][vsc]sidechaincompress=threshold=0.02:ratio=10:attack=15:release=350:makeup=1[mu]   # music ducks under voice
SFX: asplit per file → adelay=ms|ms,volume=v   (shutter 0.9 on image scenes, whoosh 0.5 on every 2nd transition if a whoosh/swoosh file exists)
[vo][mu][sfx…]amix=inputs=N:duration=first:dropout_transition=0:normalize=0,loudnorm=I=-14:TP=-1.5:LRA=11,aresample=48000
```
Encode: `-c:v libx264 -preset superfast -crf 21 -profile:v high -level 4.2 -pix_fmt yuv420p -r 60 -c:a aac -b:a 192k -ar 48000 -threads 2 -movflags +faststart -t DUR`, plus `-metadata title=… artist=Aqlix`.

### 3.5 Stage E — outputs & validation (always)
Write to the output folder: final mp4, `scene_plan.tsv` (scene, start, dur, asset), `contact_sheet.jpg` (`fps=1/4,scale=216:-2,tile=5x3`), timestamped `.log` (every command + timings). Validate with ffprobe: `720x1280`, `60/1`, audio present, duration within 0.3s of narration; print `VALIDATION: OK`.
Preflight at script start: ffmpeg/ffprobe present, filters `subtitles drawtext xfade sidechaincompress loudnorm` exist, font exists, audio/srt found (search `~/Downloads`, output dir, project dir; allow `AUDIO=` / `SRT=` overrides).

## 4. Gotchas (hit and fixed during real use)
### 4.1 `-filter_complex_script /dev/null` is NOT a no-op
Breaks the graph (`Invalid argument`). Omit unless loading a real script.
### 4.2 Subtitle font looks 40% too big
`FontSize` is relative to `PlayResY` (default small). Always put `PlayResX=<w>,PlayResY=<h>` in the same `force_style`.
### 4.3 120fps request
Push back: needs `minterpolate`, 3–5x render time, platforms cap at 60. Only if the user insists after hearing the cost.
### 4.4 No local Whisper/ASR in restricted networks
Check with `curl -sS -o /dev/null -w "%{http_code}" https://huggingface.co`; otherwise use §2's silence-aware method or ask for the script.
### 4.5 Docker/containers are the wrong call here
Use apt-native tools (`ffmpeg`, `shotcut`, `kdenlive`).
### 4.6 Scanning the stock folder freezes the PC
See §3.0.1 — filename metadata only; ffprobe just the chosen clips.
### 4.7 Shell pitfalls when testing in the sandbox
`time` is not available under `/bin/sh`; a plain `nohup … &` gets killed when the tool call ends → use `setsid nohup cmd > log 2>&1 < /dev/null &` and poll the log.
### 4.8 xfade offsets
Use cumulative **scene** durations, not intermediate lengths (§3.2); mismatched fps/size/timebase between inputs makes xfade fail.
### 4.9 Arabic text rendering
DejaVu Sans renders Arabic correctly through libass; keep burned-in `drawtext` overlays in Latin unless shaping was verified. RLM prefix per line keeps punctuation in place.
### 4.10 Copyright
Skip news/channel footage and game-publisher clips as B-roll; prefer Pexels/Pixabay/Mixkit/Coverr (check each license). Instagram's own music library avoids audio claims inside IG only.

## 5. GUI editors (only if the user wants a visual timeline)
| Tool | apt package | Notes |
|---|---|---|
| **Shotcut** | `shotcut` | CapCut-like; use proxy editing |
| **Kdenlive** | `kdenlive` | Premiere-like, heavier, stable with proxies |
| **Blender VSE** | `blender` | Motion graphics, steep curve |

No CapCut/After Effects on Linux — don't suggest Wine/VM.

## 6. Platform export specs (verify before big batches)
| Platform | Resolution | Codec | FPS | Audio |
|---|---|---|---|---|
| Instagram Reels | 1080×1920 (min 720×1280) | H.264 High | 30, 60 for fast motion | AAC 44.1/48kHz |
| YouTube Shorts | 1080×1920 | H.264 | 30/60 | AAC 128–320kbps |
| TikTok | 1080×1920 | H.264 | 30 | AAC |

Always `-movflags +faststart`. Keep captions/hook inside the safe zone (top ~14%, bottom ~35% and right edge are covered by app UI) — the middle is always safe.

## 7. Open feature backlog (build only on request)
- Cache a pre-shrunk 720p copy of the gameplay file to remove the 4K-decode cost on every run.
- LUT-based grading (`lut3d`) instead of `eq`.
- Batch export to all platform specs from one render.
- Word-level karaoke highlight (ASS `\k` tags) for the SRT.
