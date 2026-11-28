# Video Pipeline Quality & Retention Upgrade

This document outlines the architectural enhancements, visual retention mechanics, audio mastering pipeline, and quality gates introduced in the pipeline upgrade (`premium-quality` branch).

---

## 1. Upgrade Changelog

### Phase 1: Sync, Audio & Render Correctness
- **Audio-Driven Timing (`orchestrator/align.py`)**: Replaced crude proportional word-count estimates with AI forced alignment using `faster-whisper` (CPU int8, `base.en` / multilingual) following silence trimming (`ffmpeg silenceremove`).
- **Exact Script Spelling**: Matched recognized audio words to original script text via `difflib.SequenceMatcher`, preserving human-written casing, acronyms, and punctuation while acquiring exact start/end millisecond timestamps.
- **Dynamic Compositions (`Root.tsx`)**: Replaced hardcoded frame durations with dynamic `calculateMetadata` derived directly from voiceover duration and props. Works seamlessly in Remotion Studio and headless CLI.
- **Kinetic Subtitles**: Subtitles are always active by default (`captions: true`), rendered via word-level timestamps with karaoke highlighting and natural punctuation-based chunking.
- **Mobile Safe Zones**: Calibrated for 1080x1920 vertical shorts. Captions positioned at ~560px from the bottom, strictly clearing the bottom 380px and right 150px UI chrome (like/comment buttons).
- **Audio Mastering (`orchestrator/audio_master.py`)**: Two-pass EBU R128 `loudnorm` mastering targeting **-14 LUFS**, True Peak $\le -1.5\text{ dBFS}$, LRA 11, linear normalization, and 48 kHz AAC 192k.
- **Render Fidelity**: Configured `--pixel-format yuv420p`, `--color-space bt709`, and `--crf 16` for crisp typography and broad mobile compatibility. Google Fonts loaded via `@remotion/google-fonts` (`Montserrat`, `JetBrains Mono`, `Noto Sans Telugu`).
- **Determinism**: Replaced CSS transitions in `CodeBlock` with frame-driven springs; replaced non-deterministic `Math.random` with seeded `random(seed)`.

### Phase 2: Retention & Premium Visual Design
- **Micro-Pacing System (T7)**: Short-form video standardized to 10 scenes (35–45s total, 8–14 words/scene, Edge TTS `+15%` ~160 WPM). Words and diagram elements reveal synchronously when spoken.
- **Dynamic Motion Rule**: No foreground visual stays static for more than 1.8s. Added slow camera drift (1.00 $\to$ 1.04 scale), keyword zoom punches (1.00 $\to$ 1.06 scale), and animated highlight sweeps.
- **J-Cut Transitions (T8)**: Eliminated fade-to-empty backgrounds. Visuals cut 6 frames before speech starts for high-energy momentum.
- **Hook & Infinite Loop (T9)**: Hook text visible on frame 0; first spoken word starts within 150ms; opening flash clamped to $\le 0.28$ opacity. Final scene echoes the hook premise with seamless loop transition.
- **Design System (`remotion-project/src/theme.ts`)**: Unified typography scales, spacing, color palettes, and motion physics (`damping: 12`, `stiffness: 140`, `mass: 0.7`, `exitEasing: cubic-bezier(0.7, 0, 0.3, 1)`).
- **Film Grain & Visual Accents**: Subtle organic film grain texture at 0.045 opacity with per-frame offset; branded animated backgrounds (cyber grid, gradient meshes, particles) for technical scenes; stock photos reserved for hook and CTA scenes with deduplication tracked in `used_images.json`.

### Phase 3: Audio Assets, Delivery & Hardening
- **Audio Assets & Upgraded Fallbacks (T11)**:
  - Supported real music files in `remotion-project/public/audio/bgm/` and sound effects in `remotion-project/public/sfx/`.
  - Upgraded procedural fallback to **48 kHz, layered, filtered** audio: multi-oscillator detuned ambient pads with sub-bass for BGM, and resonant IIR-filtered `whoosh`, `pop`, and `riser` effects.
  - Sound effects trigger dynamically on keyword timestamps (accenting visual zoom punches) as well as scene boundaries.
  - Replaced linear ducking with a smooth cosine S-curve volume swell.
- **Automated QA Gate (`orchestrator/qa_check.py`)**:
  - Enforces broadcast loudness window: $[-15.0, -13.0]\text{ LUFS}$ and $\text{True Peak} \le -1.0\text{ dBFS}$.
  - Enforces video vs audio duration matching within 0.5s.
  - Detects solid black screens (`blackdetect`) and frozen/blank near-background frames.
  - Validates safe zones for vertical mobile video.
  - Integrated into `validate_render.py` and GitHub Actions workflow prior to artifact upload or publishing.
- **Hardening & Resiliency (T13)**:
  - Structured output (`generationConfig: {"responseMimeType": "application/json"}`) with fallback fence-stripping for Gemini script generation.
  - Sanitized Scene 5 / visual scene `imageQuery` to prevent copying full narration sentences into image search APIs.
  - Added `thumb_offset` (default 1500ms) to Instagram Reels upload for attractive cover frames.
  - Guarded YouTube custom thumbnails for long-form mode so upload never aborts if thumbnail permissions error.
  - Added `skip_cheat_sheet_deploy` workflow input (default `false`) for safe dry-run testing.

---

## 2. How to Preview Locally

### Step 1: Environment Setup
Ensure your minimal `.env` file in `my-video-pipeline/` contains your API keys:
```env
GEMINI_API_KEY="your-gemini-key"
PEXELS_API_KEY="your-pexels-key"
```

### Step 2: Generate Content & Forced Alignment
Generate the 10-scene short video script, synthesize narration, and run forced alignment:
```powershell
Set-Location my-video-pipeline
$env:PYTHONPATH=".;orchestrator"
python -m orchestrator.generate_content "Apache Kafka vs RabbitMQ" `
  --project-dir remotion-project `
  --aspect-ratio 9:16 `
  --video-mode short
```

### Step 3: Interactive Remotion Studio Preview
Preview all scenes, animations, captions, and transitions interactively:
```powershell
Set-Location remotion-project
npm run dev
```
Open `http://localhost:3000` in Chrome to inspect `GeneratedVideo9x16`.

### Step 4: Render Video
Render vertical short with production encoding settings:
```powershell
Set-Location remotion-project
npx remotion render src/index.ts GeneratedVideo9x16 out/video_raw.mp4 `
  --pixel-format yuv420p `
  --color-space bt709 `
  --crf 16
```

### Step 5: Audio Mastering & QA Check
Run two-pass loudnorm mastering and the automated QA gate:
```powershell
Set-Location my-video-pipeline
$env:PYTHONPATH="."

# Master audio to -14 LUFS broadcast standard
python -m orchestrator.audio_master remotion-project/out/video_raw.mp4 `
  --output remotion-project/out/video.mp4 `
  --project-dir remotion-project

# Run the QA Check Gate
python -m orchestrator.qa_check remotion-project/out/video.mp4 `
  --project-dir remotion-project `
  --props remotion-project/generated/props.json
```

---

## 3. Rollback & Fallback Notes

Every new capability was engineered with strict fallback safety:
1. **Audio Alignment Fallback**: If `faster-whisper` is unavailable, missing, or encounters an unexpected format, `align.py` falls back automatically to proportional word-count estimation with a logged warning. The render pipeline will not abort.
2. **Audio Assets Fallback**: If no user-provided music tracks are placed in `public/audio/bgm/`, the pipeline synthesizes procedural 48 kHz ambient tech music. If no licensed SFX are provided in `public/sfx/`, procedural 48 kHz whoosh, pop, and riser effects are generated.
3. **Structured Output Fallback**: If Gemini does not return clean JSON directly, markdown code fences (````json ... ````) are stripped cleanly. If API errors persist across configured keys, the pipeline falls back to `FALLBACK_SCRIPT`.
4. **Thumbnail Guard**: If YouTube rejects a custom thumbnail upload (e.g. unverified channel), the exception is logged as a notice and swallowed—the video upload itself remains successful.
5. **Git Rollback**: If you ever need to revert to production `main`:
   ```bash
   git checkout main
   ```
   The `premium-quality` branch is completely isolated and touches zero production branches until you explicitly merge it.
