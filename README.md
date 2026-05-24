# Product Video Maker

A Claude Code skill for producing product promo videos entirely from the command line — using [Remotion](https://www.remotion.dev/) (React-based video) + ffmpeg.

No After Effects / Premiere needed. Describe your product, and Claude handles content planning, audio-visual sync, animation, and rendering.

## What It Does

Turn a product idea into a polished 45-65s promo video (9:16 vertical / 16:9 horizontal) through a 5-phase workflow:

1. **Content Planning** — scene outline, voiceover script, design style
2. **Audio Production** — TTS/recorded VO, ffmpeg silence detection for precise speech boundaries
3. **Timeline Engineering** — frame-level VO/scene/SFX alignment (no guesswork)
4. **Code Implementation** — Remotion React components with Apple-style animations
5. **Render & Iterate** — single-variable iteration until final MP4

## Core Methodology: Audio Drives Everything

The key insight from 13 rounds of iteration: **video timing is derived from audio, not the other way around**.

```
VO recording → ffmpeg silence detection → frame calculation → scene duration → animation delay → SFX alignment → BGM ducking
```

Every animation delay is mathematically derived from speech boundaries — not estimated.

## What's Inside

| Topic | Coverage |
|-------|----------|
| Audio-visual sync | ffmpeg silence detection → frame-accurate animation delays |
| Scene pacing | VO-driven vs. pure-visual scenes, overlap transitions, CTA hold time |
| Volume architecture | 3-layer audio (BGM ducking + VO + SFX), volume > 1.0 amplification |
| Animation patterns | Apple-style fade-in, callout cards, scene envelopes |
| Phone mockup | Scale/readability balance, subtitle clearance |
| Iteration workflow | Single-variable changes, sync checklists, feedback-to-fix mapping |
| Anti-patterns | 7 common mistakes and their fixes |
| Cover production | Social media cover images (rembg cutout, Playwright rendering) |

## Install

### As a Claude Code Skill

```bash
# Copy to your Claude skills directory
cp -r . ~/.claude/skills/product-video-maker/
```

Then invoke in Claude Code:

```
/product-video-maker
```

### Prerequisites

- [Node.js](https://nodejs.org/) 18+
- [Remotion](https://www.remotion.dev/) (`npm create video@latest`)
- [ffmpeg](https://ffmpeg.org/) (for audio analysis)
- TTS service (Fish Audio / ElevenLabs) or recorded voiceover

## Quick Example

```bash
# Analyze voiceover speech boundaries
ffmpeg -i vo/02-pain.mp3 -af silencedetect=noise=-30dB:d=0.3 -f null - 2>&1 | grep silence

# Test render (first 5 frames)
npx remotion render PromoVideo --frames=0-5

# Full render
npx remotion render PromoVideo --gl=angle
```

## Project Structure

```
promo-video/
  src/
    Root.tsx              — Composition config (size, FPS, duration)
    PromoVideo.tsx        — Main orchestration (VO + SFX + scenes)
    styles.ts             — Design tokens
    components/
      PhoneMockup.tsx     — iPhone mockup component
      DashboardScreens.tsx — Product UI simulation
  public/
    vo/                   — Voiceover audio files
    *.mp3                 — SFX (tap, pop, whoosh, chime)
    bgm.mp3              — Background music
  out/
    PromoVideo.mp4        — Rendered output
```

## Background

Distilled from 13 iterations (V1 → V13) of a real product promo video. Every rule in this skill exists because we hit the problem it prevents.

## License

MIT
