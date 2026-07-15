# AudioStreamMETER — Professional Comparison Analysis

> [!NOTE]
> **Refactoring Status Update (July 2026)**:
> - **LUFS Short-Term (correct)**: **RESOLVED**. Replaced approximate/gated calculation with EBU R128 compliant K-weighted ungated RMS over 3s buffers.
> - **Honest Peak Labeling**: **RESOLVED**. Renamed GUI labels from "True Peak" (TP) to "Sample Peak" (SPK) to prevent misleading broadcast engineers, adding descriptive tooltips.
> - **Performance Refactor**: **RESOLVED**. Shipped all processing to the background worker threads, achieving low CPU usage under massive parallel streams.
> - **Auto-Reconnect Loop**: **RESOLVED**. Added a background auto-reconnect retry loop that attempts connection every 30 seconds upon stream loss.
> - **Gaps Remaining**: Silence/dead-air detection and stereo correlation bar.

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

## What AudioStreamMETER Does Well ✅

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

### Issue 2 — True Peak is Sample Peak, not Inter-Sample Peak

**Current code:**
```python
peak_l = int(np.max(np.abs(samples_stereo[:, 0])))  # ← sample-domain peak
tp_l = 20.0 * math.log10(peak_l / 32768.0)
```

**What the label says:** `TP L:-1` (True Peak)
**What it actually measures:** Sample Peak

These are fundamentally different:

| Metric | What it measures | Standards compliance |
|---|---|---|
| **Sample Peak** | Max amplitude in the digital sample grid | Not TP — can under-read by up to **3.5 dB** |
| **True Peak (inter-sample)** | Reconstructed analog waveform peak via 4× oversampling | Required by ITU-R BS.1770-4, EBU R128, ATSC A/85 |

When an audio waveform is encoded at 192 kbps MP3, inter-sample peaks (peaks that fall *between* two sample points) can exceed 0 dBFS in the reconstructed analog signal even when every sample is below 0 dBFS. This is exactly why True Peak limiting at -1 dBTP is mandated — to leave headroom for codec reconstruction.

Calling sample peak "True Peak" is a **labeling inaccuracy** that could mislead broadcast engineers about compliance.

**Fix:** Either:
- Rename to "Sample Peak" (honest, quick) — and add a note that TP requires oversampling
- Implement proper 4× upsampled True Peak using scipy or a pre-computed interpolation filter (correct, but adds scipy dependency)

---

### Issue 3 — No Stereo Phase Correlation Meter

Every professional broadcast monitor includes a **phase correlation / goniometer** display. It is arguably as important as loudness for radio.

| Why it matters |
|---|
| Radio stations transmit in stereo but must be **mono-compatible** |
| Out-of-phase signals cancel in mono, causing drops of up to −∞ dB |
| Many streams have phase errors from poor compression or encoding settings |
| Broadcasters routinely check correlation before approving a source |

**What tools show:**
- **Correlation bar**: −1 (fully out of phase) to +1 (perfectly in phase). Green zone is typically +0.5 to +1.0.
- **Goniometer (Lissajous)**: XY plot of L vs R — circle = mono, diagonal = pure stereo, horizontal = 100% out of phase.

This is a **significant professional gap**.

---

### Issue 4 — No Silence / Dead Air Detection

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

## What Professional Tools Have That AudioStreamMETER Lacks

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
                              AudioStreamMETER   TC Electronic   QVIO   Orban
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

### 1. Auto-Reconnect / Stream Auto-Restart (1–2 hours work)
Add a retry loop with exponential backoff in the background worker thread. When a stream drop causes the FFmpeg process to die, the worker should automatically attempt to reconnect (e.g., after 2s, 4s, 8s, capping at 60s) rather than leaving the card permanently dead (red).

### 2. Silence / Dead Air Detection (2–3 hours work)
Per-card silence timer. If RMS level stays below −60 dBFS for a configurable duration (default 10s):
- Flash the card border red.
- Show a `⚠ SILENCE` overlay on the waveform.
- Automatically clear the alert when audio signal returns.

### 3. Stereo Phase Correlation Bar (3–4 hours work)
Compute phase correlation index `corr = mean(L * R) / sqrt(mean(L^2) * mean(R^2))`. Display as a thin horizontal level bar in each card from −1 (fully out of phase) to +1 (perfectly in-phase), with color bands (green for positive, red for negative).

### 4. Momentary LUFS — 400ms window (4–6 hours work)
Compute K-weighted RMS over a sliding 400ms buffer inside the worker thread. Display as `M: -14.2` alongside the existing short-term (`S`) LUFS value.

---

## Bottom Line

AudioStreamMETER is currently **best-in-class for multi-stream simultaneous monitoring** — no commercial tool matches 16 streams at once with per-stream spectrum analysis in an open, free tool.

Through our latest refactoring, we have successfully resolved:
1. **LUFS accuracy** — replaced gated integrated approximation with compliant EBU R128 Short-term LUFS.
2. **Technical Peak Labeling** — changed the label from `TP` (True Peak) to `SPK` (Sample Peak) and added explanatory tooltips.

The remaining gaps to make this application highly reliable for long-term unattended broadcast operations are:
1. **Stream Auto-Restart retry loop** — to automatically recover from network drops.
2. **Silence / dead air alarm** — the single most important warning for radio station operators.
3. **Stereo phase correlation** — to monitor mono-compatibility on-air.
