# Audio Cue Mapper

Browser-based audio analysis tool for motion design sync workflows. Drop in an audio or video file, get back a structured set of timed cues as an After Effects script.

**Live:** `<add GitHub Pages URL>`
**Part of:** [The Dream Team](https://davidpreli.com) — Studio David Preli

---

## What it does

Cue Mapper analyses an audio track and produces a set of timed markers for animating to music in After Effects. It was built for *The Dream Team*, a pixel-art piece synced to a loose, live-recorded track by Fievel Is Glauque, where grid-locked beat-detection plugins (BeatEdit and similar) produced unusable output.

The entire application is one HTML file. All decoding, analysis, rendering, and export run client-side. No dependencies, no build step, no npm, no framework, no backend.

## How it works

A dropped file is read as an `ArrayBuffer` and passed through seven sequential stages. Computation runs on the main thread in yielding chunks via `setTimeout` so the UI stays responsive.

| # | Stage | Detail |
|---|-------|--------|
| 1 | Decoding | `decodeAudioData()` (MP3, WAV, FLAC, OGG, AAC, M4A), mixed to mono |
| 2 | Waveform downsampling | Mono samples reduced to 1,200 min/max pairs for display |
| 3 | Spectral flux onset detection | 2048-sample frames, 512 hop (~11.6 ms), Hanning window, hand-written Cooley–Tukey radix-2 FFT |
| 4 | Peak picking and tiering | Local maxima above the 85th percentile, minimum-distance constraint, sorted into three strength tiers |
| 5 | RMS envelope | Per-frame loudness for background shading and segmentation |
| 6 | Structural segmentation | Deep minima in a heavily smoothed RMS envelope become section boundaries |
| 7 | Tempo estimation | Autocorrelation of the flux signal across 50–240 BPM |

### Onset tiers

| Tier | Threshold | Use |
|------|-----------|-----|
| 1 — Major | ≥ 72% of max | Key poses, cuts, major transitions |
| 2 — Mid | 48–72% | Secondary accents, phrase markers |
| 3 — Minor | < 48% | Texture, micro-timing, detail passes |

## Visualisation

Results render to two `<canvas>` panels via the Canvas 2D API, both using device-pixel-ratio scaling for high-DPI displays and resizing on window resize.

- **Waveform:** the 1,200-point min/max array drawn as a filled, gradient polygon with section boundaries overlaid.
- **Cue map:** three tiers of peaks as vertical bars (tall green Tier 1 with glow, medium periwinkle Tier 2, short amber Tier 3), dashed section lines, a 5-second grid, and a hover tooltip showing the nearest peak's timecode, tier, and strength.

## After Effects export

The primary deliverable is a `.jsx` ExtendScript file. Run via **File › Scripts › Run Script File** in the active composition, it creates five null layers, each carrying markers placed with `MarkerValue` objects and `setValueAtTime()`. Layers are added in reverse order so the beat grid sits at the top of the stack.

| Layer | AE label | Contents |
|-------|----------|----------|
| ♪ Beat grid | Sea Foam | Every beat at estimated BPM. Downbeats M1, M2…; off-beats 1.2, 1.3, 1.4 |
| § Sections | Yellow | One marker per structural boundary, with index and timecode |
| ▲ Major hits | Tangerine | Tier 1 peaks, timecode and strength % |
| ● Mid hits | Aqua | Tier 2 peaks, timecode and strength % |
| · Minor hits | Lavender | Tier 3 onsets, timecode and strength % |

**Suggested workflow:** solo Sections to block structure, solo Major Hits for key poses and cuts, use Beat Grid for secondary animation and transitions, reference Mid and Minor for texture and micro-timing.

## Tech stack

| Layer | Detail |
|-------|--------|
| Runtime | Vanilla JS, Web Audio API, Canvas 2D API |
| Hosting | GitHub Pages (static file, zero infrastructure) |
| Embedding | Cargo 3 iframe with a `postMessage` resize bridge |
| Fonts | IBM Plex Mono + DM Serif Display via Google Fonts |
| Dependencies | None |

After analysis completes, the tool calls `window.parent.postMessage()` with its content height so the host Cargo page can size the iframe (Cargo 3 has no server-side scripting and limited `<head>` access, so the tool is hosted externally and embedded).

## Design

Shares a visual language with the Pokédex viewer and the Dream Team case study for coherence across embedded tools.

| Token | Hex | Role |
|-------|-----|------|
| `--bg` | `#141414` | Page background |
| `--accent` | `#76c17d` | Primary / Tier 1 (Pokédex green) |
| `--accent2` | `#b9b0ff` | Tier 2 (periwinkle) |
| `--accent3` | `#e8b86d` | Tier 3 / beat grid (warm amber) |
| `--text` | `#e1e1e1` | Primary text (WCAG AAA on `--bg`) |
| `--muted` | `#888888` | Secondary text (WCAG AA) |

Typography is IBM Plex Mono throughout, with DM Serif Display italic for the track title and upload prompt. The loading state is a CSS-only quarter note hopping along a five-line staff with trailing ghost notes (no JS, no canvas).

## Known constraints

- **Tempo is approximate.** Irregular pulse, rubato, or polyrhythm will pull the beat grid off the actual pulse. Treat the grid as a starting reference.
- **Segmentation is heuristic.** The RMS-valley approach suits music with clear phrase structure; through-composed material yields fewer meaningful boundaries.
- **The FFT is unoptimised.** Correct but not performance-tuned; files over ~10 minutes may be slow on lower-powered devices.
- **Codec support varies.** `decodeAudioData()` does not cover every container/codec. WAV and FLAC are the most reliable.
- **Markers land on null layers,** not the audio layer. Snapping works identically; the visual association is manual.

---

Studio David Preli — [davidpreli.com](https://davidpreli.com)
Built for *The Dream Team*, 2026.
