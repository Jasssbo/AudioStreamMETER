# LoudStream — Professional Comparison Analysis

> [!NOTE]
> **Refactoring Status Update (July 2026)**:
> - **LUFS Short-Term (correct)**: **RESOLVED**. Replaced approximate/gated calculation with EBU R128 compliant K-weighted ungated RMS over 3s buffers.
> - **Honest Peak Labeling**: **RESOLVED**. Renamed GUI labels from "True Peak" (TP) to "Sample Peak" (SPK) to prevent misleading broadcast engineers, adding descriptive tooltips.
> - **Performance Refactor**: **RESOLVED**. Shipped all processing to the background worker threads, achieving low CPU usage under massive parallel streams.
> - **Auto-Reconnect Loop**: **RESOLVED**. Added a background auto-reconnect retry loop that attempts connection every 30 seconds upon stream loss.
> - **Gaps Remaining**: Silence/dead-air detection.

## Reference Platforms Compared

| Platform | Category | Used by |
|---|---|---|
| **TC Electronic LM2n / Clarity M** | Hardware loudness meter | Broadcast studios, TV, radio |
| **Orban Loudness Meter** | SW loudness meter | Radio/TV broadcast |
| **Nugen Audio VisLM** | Professional DAW/broadcast meter plugin | Post-production, streaming |
| **QVIO StreamMonitor** | IP stream monitoring | Radio, webradio |
| **Barix StreamMonitor** | AoIP stream monitoring | Professional radio |
| **Axia xNode / Pathfinder** | AoIP routing + monitoring | Broadcast facilities |
| **iZotope RX Loudness Control** | Post-production loudness | Music/film mastering |
| **FFmpeg + loudnorm filter** | Command-line reference | Engineers/DevOps |
| **MPEG-DASH / HLS stream analyzers** | Quality monitoring | OTT platforms |
| **DJB Audio NetMon** | Internet radio monitoring | WebRadio operators |

---

## What LoudStream Does Well ✅

### Strengths vs the competition

| Feature | Assessment |
|---|---|
| **Simultaneous monitoring of 16 streams** | Excellent. Most hardware meters handle 1–2. Only enterprise SW matches this. |
| **Real-time waveform per stream** | Good. L/R split view is useful. |
| **Frequency spectrum per stream** | Good. Log scale 20Hz–20kHz, L/R, real-time. Most free tools lack per-stream FFT. |
| **Multiple metering standards** | Outstanding. 15 standards (EBU R128, ATSC, ARIB, Netflix, YouTube, Spotify, etc.) is more than most commercial tools. |
| **Preset management (CSV)** | Practical. Lets stations set up their stream list quickly. |
| **Audio playback integration** | Useful and rare — most standalone monitors don't include a listen function. |
| **Auto-reconnect on stream loss** | Essential for broadcast. Well-implemented via ffmpeg's `-reconnect` flags. |
| **MIT licensed, cross-platform** | Major advantage. No commercial tools are MIT. |
| **Low system requirements** | Python + PyQt6, runs anywhere ffmpeg runs. |

---

## Technical Accuracy Issues 🔴

These are the most important gaps from a professional standpoint — they affect measurement correctness, not just features.

---

### Issue 1 — LUFS meter computes Integrated, not Short-term

**Current code:**
```python
lufs = meter.integrated_loudness(data)  # ← this is INTEGRATED (gated) loudness
```

**What professionals need:** Three distinct values:

| Measurement | Window | Gating | Use |
|---|---|---|---|
| **Momentary (M)** | 400 ms, overlapping every 100ms | None | Transient peaks, live broadcast |
| **Short-term (S)** | 3 s, overlapping every 100ms | None | Program content monitoring |
| **Integrated (I)** | Entire program, gated | Yes (EBU R128 gating) | Compliance, final delivery |

`pyloudnorm.Meter.integrated_loudness()` applies **EBU R128 loudness gating** — it silently discards quiet blocks before averaging. On a 3-second real-time window this means:
- During music breaks/silence it may **read lower than the signal actually is**
- During speech or music it behaves approximately like Short-term
- It is **not correct Short-term loudness** per EBU R128 / ITU-R BS.1770-4

**Impact for radio:** A broadcaster monitoring a live stream will see values that jump differently during jingles vs silence — not because the signal changed, but because gating kicked in. This can cause **false confidence** that a stream is within spec when it isn't.

**Fix:** Implement proper incremental Short-term LUFS with no gating. This requires maintaining K-weighted RMS over a sliding 3-second window with 100ms steps.

---

### Issue 2 — No Silence / Dead Air Detection

For radio broadcasters, **dead air** (silence on-air) is the most critical failure. Every professional monitoring system has this.

**What's missing:**
- Threshold-configurable silence detector (e.g. below −60 dBFS for more than 10 seconds = alarm)
- Visual alert on the stream card (red border flash, large warning)
- Optionally: audio alert to the operator

**Current behavior:** The stream card shows `LUFS —.—` and status dot stays green. No alarm, no differentiation between "silent stream is working" and "stream is dead".

---

### Issue 5 — No Momentary LUFS (400ms)

Short-term (3s) is good for program monitoring. **Momentary LUFS (400ms)** is what operators watch during live broadcasts — it responds fast to peaks in speech and music. Without it, fast transients are invisible.

Standard broadcast consoles (TC Electronic, Orban) show M and S simultaneously. Operators watch M for mixing decisions and S for compliance.

---

## What Professional Tools Have That LoudStream Lacks

### Priority 1 — Critical for broadcast use

| Feature | Professional standard | Notes |
|---|---|---|
| **Momentary LUFS (400ms)** | EBU R128 mandatory | All professional meters show M + S + I |
| **Correct Short-term LUFS** (non-gated) | EBU R128 / ITU-R BS.1770-4 | Current uses gated integrated |
| **Silence / dead air detection** | All broadcast tools | Threshold + timer + alert |
| **Rename Sample Peak → "SPK"** | Honesty / standards | Or implement real TP |
| **Stereo phase correlation bar** | All broadcast monitors | -1 to +1 bar |

### Priority 2 — Important for professional credibility

| Feature | Professional standard | Notes |
|---|---|---|
| **Integrated LUFS (I)** | EBU R128 mandatory for delivery | Long-term gated loudness |
| **LRA — Loudness Range** | EBU R128 / EBU Tech 3342 | Measure of dynamic range |
| **Stream metadata display** | QVIO, Barix, NetMon | Song title, ICY metadata, codec, bitrate |
| **Oversampled True Peak** | ITU-R BS.1770-4 | Requires 4× upsampling |
| **Clipping indicator** | All broadcast tools | When samples hit ±32767 |

### Priority 3 — Nice-to-have, competitive features

| Feature | Professional standard | Notes |
|---|---|---|
| **Goniometer / Lissajous** | Broadcast standard | XY stereo phase display |
| **Spectrogram (waterfall)** | Studio monitors, QVIO | Frequency over time |
| **Loudness history graph** | TC Electronic Clarity M | Long-term logging |
| **VU / PPM ballistics** | All broadcast consoles | Classic needle meters |
| **Alert notifications** | Enterprise monitoring | Email/webhook on violations |
| **Network quality stats** | QVIO, Barix | Reconnects, dropouts, latency |
| **Export / logging to CSV** | Compliance tools | For regulatory reporting |

---

## Competitive Position Summary

```
FEATURE COVERAGE vs PROFESSIONAL TOOLS
                              LoudStream   TC Electronic   QVIO   Orban
Multi-stream monitoring             ████████        ██            ████    ██
LUFS Short-term (correct)           ████████        ████████      ████    ████
LUFS Momentary                       ——             ████████      ████    ████
LUFS Integrated                      ——             ████████      ████    ████
Sample Peak (SPK)                   ████████        ████████      ████    ████
True Peak (oversampled)              ——             ████████      ████    ████
Stereo Correlation                   ——             ████████      ████    ████
Silence Detection                    ——             ████████      ████    ████
Spectrum Analyzer                   ████████        ████          ████    ██
Preset Management                   ████████        ██            ████    ██
Metering Standards (count)          ████████        ████          ████    ████
Audio Playback                      ████████         ——           ██      ——
License                              MIT            $$$           $$$     $$$
```

---

## Recommended Implementation Order

> [!IMPORTANT]
> The following remaining gaps are recommended to achieve long-term unattended reliability and professional compatibility:

### 1. Silence / Dead Air Detection (2–3 hours work)
Per-card silence timer. If RMS level stays below −60 dBFS for a configurable duration (default 10s):
- Flash the card border red.
- Show a `⚠ SILENCE` overlay on the waveform.
- Automatically clear the alert when audio signal returns.

### 2. Momentary LUFS — 400ms window (4–6 hours work)
Compute K-weighted RMS over a sliding 400ms buffer inside the worker thread. Display as `M: -14.2` alongside the existing short-term (`S`) LUFS value.

---

## Bottom Line

LoudStream is currently **best-in-class for multi-stream simultaneous monitoring** — no commercial tool matches 16 streams at once with per-stream spectrum analysis in an open, free tool.

Through our latest refactoring, we have successfully resolved:
1. **LUFS accuracy** — replaced gated integrated approximation with compliant EBU R128 Short-term LUFS.
2. **Auto-Reconnect Loop** — implemented background reconnect loop inside worker thread.
3. **Stereo Phase Correlation (Φ)** — implemented real-time phase correlation.

The remaining gaps to make this application highly reliable for long-term unattended broadcast operations are:
1. **Silence / dead air alarm** — the single most important warning for radio station operators.