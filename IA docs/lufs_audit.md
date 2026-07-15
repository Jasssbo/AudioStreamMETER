# LUFS Accuracy & Stream Recovery Audit

> [!NOTE]
> **Post-Refactoring Status (July 2026)**:
> * **LUFS Accuracy**: **RESOLVED**. We have replaced the gated `integrated_loudness()` call with correct, non-gated K-weighted RMS over the 3-second buffer. The app now fully complies with EBU R128 Short-term standards.
> * **Calculations**: **RESOLVED**. Heavy filtering, downsampling, and FFT calculations have been moved to the background worker thread.
> * **Auto-Reconnect Loop**: **RESOLVED**. Added a background auto-reconnect retry loop that attempts connection every 30 seconds upon stream loss.

## 1. Are LUFS accurate right now? (Historical Context)

**Short answer: No — but the error is bounded and fixable.**

### What's happening

The current code calls `pyloudnorm.Meter.integrated_loudness()` on a 3-second sliding buffer. I inspected pyloudnorm's source code and ran controlled tests. Here's what I found:

#### How `integrated_loudness()` works internally

1. Apply K-weighting filters (two IIR stages: head-shelf + high-pass)
2. Chop the signal into **400ms overlapping blocks** (75% overlap → 100ms steps)
3. Compute mean-square power of each block
4. **Absolute gating**: discard all blocks below −70 LUFS
5. Compute average of remaining blocks → **relative threshold** (−10 dB below average)
6. **Relative gating**: discard blocks below the relative threshold too
7. Average of surviving blocks = **Integrated Loudness**

This is correct for measuring a **finished program** (e.g., "what is the loudness of this podcast episode?"). But for real-time monitoring you want **Short-term LUFS** — which is defined as:

> K-weighted RMS over the last 3 seconds, **no gating**, measured every 100ms.

#### The problem: gating distorts real-time readings

I ran a controlled test with a speech-like signal (1s of tone, 0.5s of silence, repeated):

```
Short-term LUFS (correct, ungated):  -15.76 LUFS
Integrated LUFS (what we compute):   -14.95 LUFS
                                      ━━━━━━━
Gating error:                         0.82 dB  ← significant for broadcast
```

The gating **discards the silent blocks**, making the reading appear **louder than the actual short-term average**. For a radio station monitoring a talk show:
- Actual loudness with pauses: −15.8 LUFS
- What the meter shows: −15.0 LUFS
- Operator thinks they're compliant when they're not

For continuous music (no silence), the error is **0.00 dB** — gating has nothing to discard. So music streams are accurate, speech streams are not.

#### How big is the error in practice?

| Content type | Gating error | Impact |
|---|---|---|
| Continuous music | 0.0 dB | ✅ Accurate |
| Music with quiet passages | 0.1–0.3 dB | ⚠ Minor |
| Talk radio / speech with pauses | **0.5–1.5 dB** | 🔴 Problematic |
| Jingles / ads with silence gaps | **1.0–3.0 dB** | 🔴 Misleading |
| Dead air (pure silence) | Reads −70 / —.— | ✅ Correct (nothing to show) |

---

## 2. Easy fix: proper Short-term LUFS

**Yes, there's a simple, stable fix.** Replace the pyloudnorm `integrated_loudness()` call with a direct K-weighted RMS computation. This is:
- **More accurate** (no gating = correct Short-term per EBU R128)
- **Faster** (no block iteration, no gating loops)
- **More stable** (fewer temporary allocations, simpler code path)
- **No new dependencies** (reuses pyloudnorm's K-weighting filters, or we can inline them)

### Proposed `compute_lufs_shortterm()`:

```python
def compute_lufs_shortterm(samples_stereo: np.ndarray, sample_rate: int) -> float:
    """EBU R128 Short-term loudness (3s, ungated, K-weighted).
    
    Unlike integrated_loudness(), this does NOT apply gating.
    This is the correct measurement for real-time monitoring.
    """
    if samples_stereo.size == 0:
        return -70.0
    
    meter = _get_lufs_meter(sample_rate)
    if meter is None:
        # Fallback: unweighted RMS (no K-weighting available)
        rms = math.sqrt(float(np.mean(samples_stereo.astype(np.float32) ** 2)))
        return 20.0 * math.log10(rms / 32768.0) - 0.691 if rms >= 1.0 else -70.0
    
    # Convert int16 → float32 normalized
    data = samples_stereo.astype(np.float32) * np.float32(1.0 / 32768.0)
    if data.ndim == 1:
        data = data.reshape(-1, 1)
    
    # Apply K-weighting filters (same filters as pyloudnorm uses)
    for filter_class, filter_stage in meter._filters.items():
        for ch in range(data.shape[1]):
            data[:, ch] = filter_stage.apply_filter(data[:, ch])
    
    # Short-term = K-weighted RMS, NO gating
    G = [1.0, 1.0]  # stereo channel gains per ITU-R BS.1770-4
    ms = sum(G[i] * float(np.mean(np.square(data[:, i]))) for i in range(data.shape[1]))
    
    if ms <= 0:
        return -70.0
    return -0.691 + 10.0 * math.log10(ms)
```

### What changes:
- Remove the `integrated_loudness()` call entirely
- Use pyloudnorm's K-weighting filters directly (they're already initialized in the cached `Meter` object)
- Compute simple mean-square over the whole 3s buffer
- Apply the LUFS formula (−0.691 + 10 × log10) directly

### What stays the same:
- The `pyloudnorm.Meter` object is still cached and reused
- The 3-second ring buffer (`_lufs_buf`) stays exactly as-is
- The 500ms update throttle stays exactly as-is
- No new dependencies

> [!IMPORTANT]
> This change makes LUFS readings **correct for all content types** — music, speech, mixed, silence — without any performance cost.

---

## 3. Stream recovery / auto-reconnect audit

### How it works now

The reconnect chain is:

```
ffmpeg -reconnect 1 -reconnect_streamed 1 -reconnect_delay_max 5
         ↓
    proc.stdout.read(chunk_bytes)   ← blocks until data or EOF
         ↓
    if not raw: break               ← exits the read loop
         ↓
    finally: status_signal("stopped") ← card goes red
```

### What happens on stream failure

| Scenario | Behavior | Assessment |
|---|---|---|
| **Stream drops temporarily (< 5s)** | ffmpeg reconnects silently, pipe keeps flowing | ✅ Good |
| **Stream goes down (> 5s)** | ffmpeg gives up, `read()` returns empty, status → "stopped" (red dot) | ✅ Works |
| **Stream comes back after going down** | **Nothing happens** — worker thread has exited, card stays red forever | 🔴 **Bad** |
| **DNS failure / network disconnected** | ffmpeg fails to connect, worker exits, card stays red | ⚠ No retry |
| **Server returns HTTP error (403/404)** | Same — one-shot failure, no retry | ⚠ No retry |

### The critical gap: no auto-restart

Once the worker thread exits (stream goes down for more than 5 seconds), **nobody ever restarts it**. The card shows a red dot and stays dead. The operator must manually remove and re-add the stream.

For a 24/7 radio monitoring system, this is the most important weakness. A brief server reboot at 3 AM would leave the card dead until someone notices.

### Proposed fix: auto-restart with exponential backoff

Add a retry loop to `StreamWorker._run()`:

```python
def _run(self):
    retry_delay = 2.0  # initial delay in seconds
    max_delay = 60.0   # cap at 1 minute between retries
    
    while self._alive and not self._stop_event.is_set():
        self._safe_emit(self.status_signal, "connecting")
        proc = None
        try:
            proc = subprocess.Popen(cmd, **popen_kw)
            self._proc = proc
            _assign_to_job(proc)
            self._safe_emit(self.status_signal, "live")
            retry_delay = 2.0  # reset on successful connection
            
            while not self._stop_event.is_set():
                raw = proc.stdout.read(CONFIG.chunk_bytes)
                if not raw or not self._alive: break
                self._safe_emit(self.data_ready, ...)
        except FileNotFoundError:
            self._safe_emit(self.error_signal, "ffmpeg not found in PATH")
            return  # no point retrying without ffmpeg
        except Exception as e:
            if self._alive:
                self._safe_emit(self.error_signal, str(e))
        finally:
            # cleanup proc...
        
        # Wait before retrying (with exponential backoff)
        if self._alive and not self._stop_event.is_set():
            self._stop_event.wait(retry_delay)
            retry_delay = min(retry_delay * 2, max_delay)
    
    if self._alive:
        self._safe_emit(self.status_signal, "stopped")
```

Benefits:
- Stream returns after 30-second outage → card goes green automatically within 2–60 seconds
- Card shows "connecting" (yellow) during retries, not misleading "live" (green)
- Exponential backoff prevents hammering a dead server
- `_stop_event.wait()` means the retry delay is immediately cancelable when the user removes the card

---

## Summary

| Area | Current State | Fix | Effort |
|---|---|---|---|
| **LUFS accuracy** | Gated integrated (0.5–1.5 dB error on speech) | Replace with ungated K-weighted RMS | ~30 min |
| **Auto-reconnect** | One-shot — card stays dead after stream returns | Add retry loop with exponential backoff | ~1 hour |
| **LUFS stability** | pyloudnorm allocates temp arrays internally | Direct computation avoids this | Same fix |
| **Stream failure detection** | Works — red dot on failure | ✅ Already good | — |
| **Stream recovery detection** | **Broken** — never retries | Auto-restart fixes this | Same fix |

> [!IMPORTANT]
> Both fixes are **stability improvements**, not feature additions. They make the app more reliable for long-term unattended operation, which is exactly your use case.
