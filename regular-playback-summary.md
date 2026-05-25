# Vega HW Decoder Freeze on DRM Key Rotation

**Date:** 2026-05-18
**Device:** Fire TV Stick 4K Select (`callie`, ARMv7), Vega SDK 0.22
**Source log:** `metro.log` (~30-minute live session)

All recovery paths disabled for this capture — the natural decoder behavior on DRM key rotation is recorded unmodified.

---

## Summary

3 DRM license rotations occurred during the session. **Every rotation was followed within seconds by a playback freeze.** Playback never recovered without external `VideoPlayer.deinitialize()` + recreate.

| Rotation | License `t` since load | Time to freeze after rotation | Frozen `currentTime` | Outcome |
|---|---:|---:|---:|---|
| #1 | 619.9 s | +17.9 s | 24023304.011 | 28 s of stuck `currentTime`, then 5 s `liveSync` jump, stalled again |
| #2 | 673.7 s | < 1 s | 24023314.453 | freeze chained directly onto #1's stall recovery |
| #3 | 1529.7 s | (already stalled) | 24024171.122 | 86 cumulative `stalldetected`, still no recovery |

---

## Freeze signature

After the license response is received, the W3C VideoPlayer enters a state where the playback surface stops updating while every other diagnostic reports as healthy.

`metro.log` lines 1670–1774, around rotation #1:

| Δ from rotation | `currentTime` | `totalFrames` | `buffered.end` | `liveLatency` | `readyState` | `isBuffering` |
|---:|---:|---:|---:|---:|---:|---:|
| −2.1 s | 24023287.495 | 18357 | 24023299.87 | 35.57 | 3 | true |
| **0** | **24023289.577** | **18430** | **24023305.89** | **35.57** | **3** | **true** ← rotation-detected |
| +3.0 s | 24023292.599 | 18495 | 24023311.88 | 35.57 | 3 | true |
| +5.4 s | 24023295.010 | 18562 | 24023317.90 | 35.57 | 3 | true |
| +8.0 s | 24023297.624 | 18632 | 24023317.90 | 35.57 | 3 | true |
| +10.5 s | 24023300.083 | 18767 | 24023317.90 | 35.57 | 3 | true |
| +13.0 s | 24023302.542 | 18888 | 24023317.90 | 35.57 | 3 | true |
| +15.6 s | 24023303.911 | 18943 | 24023323.97 | 36.87 | 3 | true |
| **+17.9 s** | **24023304.011** | **19119** | **24023329.91** | **39.00** | **3** | **true** ← **freeze begins** |
| +19.9 s | 24023304.011 | 19119 | 24023329.91 | 41.00 | 3 | true |
| +22.3 s | 24023304.011 | 19125 | 24023329.91 | 43.47 | 3 | true |
| +24.7 s | 24023304.011 | 19125 | 24023329.91 | 45.82 | 3 | true |
| +27.5 s | 24023304.011 | 19125 | 24023329.91 | 48.61 | 3 | true |
| +29.9 s | 24023304.011 | 19125 | 24023335.90 | 51.00 | 3 | false |
| +32.4 s | 24023304.011 | 19125 | 24023335.90 | 53.52 | 3 | false |
| +34.8 s | 24023304.011 | 19125 | 24023335.90 | 55.89 | 3 | false |
| +36.8 s | 24023304.011 | 19125 | 24023335.90 | 57.93 | 3 | false |
| +38.8 s | 24023304.011 | 19125 | 24023335.90 | 59.94 | 3 | false |
| +40.8 s | 24023304.011 | 19125 | 24023335.90 | 61.94 | 3 | false |
| +42.8 s | 24023304.011 | 19125 | 24023335.90 | 63.95 | 3 | false |
| +45.9 s | 24023309.281 | 19313 | 24023335.90 | 61.74 | 1 | false |

### Observations

- **`currentTime` frozen at `24023304.011` for ~28 s** straight (+17.9 s through +42.8 s).
- **`liveLatency` rose linearly +25.0 s in 25.0 s of wall clock** — exact 1:1 ratio. This is the cleanest possible proof that the playhead is stuck while real time keeps moving.
- **`totalFrames` advanced only 6 frames in 28 s** (19119 → 19125) — the decoder essentially stopped producing output.
- **`droppedFrames` stayed at 1** for the entire freeze — no dropped frames, the decoder just stops emitting.
- **`buffered.end` kept growing** (24023329.91 → 24023335.90) — Shaka's `MediaSource` append loop is fully healthy. Nothing wrong with the manifest, segments, or network.
- **`readyState` stayed at 3 (HAVE_FUTURE_DATA)** the entire time. From the W3C API's perspective the player has data and is ready.
- **`isBuffering` reported as `false`** from +29.9 s onward — Shaka itself does not consider the player to be buffering.

The "recovery" at +45.9 s is a `liveSync` jump (`liveSyncPlaybackRate: 1.3`, `liveSyncMaxLatency: 5`) — `currentTime` snapped forward by 5.27 s and the freeze immediately resumed at a new value.

---

## Where the problem is

The freeze is below Shaka and below the W3C MediaSource API surface. Everything upstream of the decoder reports healthy:

| Layer | State during freeze | Verdict |
|---|---|---|
| Network (Shaka segment fetch) | `buffered.end` keeps advancing | ✅ OK |
| Manifest parsing | no errors | ✅ OK |
| DRM license fetch | response received in 618 ms (rotation #1) | ✅ OK |
| MSE `SourceBuffer.appendBuffer` | buffered range grows | ✅ OK |
| W3C `HTMLMediaElement` | `readyState=3`, `error=null`, `paused=false` | ✅ reports OK |
| Shaka stall detector | fires `stalldetected` 86 times across session | ⚠ detects, can't fix |
| Shaka `stallSkip: 0.5` | seeks fire, surface does not unstick | ❌ ineffective |
| **HW decoder output / compositor** | **`currentTime` stuck, frames not rendering** | ❌ **frozen** |

**The problem sits between "license response received" and "frame rendered to surface".** All conventional recovery levers (Shaka stall skip, MSE recovery, W3C error) are inert because none of those layers see anything wrong.

The only intervention that recovers playback is a full `VideoPlayer.deinitialize()` followed by recreating the `VideoPlayer` and re-loading via Shaka. A Shaka-only reload (skipping `videoPlayer.deinitialize`) consistently SIGABRTs with `app-demux-sourc`, indicating the native pipeline state is also corrupted.

---

## Cumulative damage in the session

The freeze does not self-clear, so subsequent rotations layer on top of an already broken state:

- **Rotation #2** (54 s after #1, while still stalled): freeze starts immediately, `currentTime` pinned at 24023314.453.
- **Rotation #3** (~15 min later): `stallsDetected = 86`, `liveLatency = 64.6 s`. Shaka has been firing stall events the entire time with no effect.

---

## Log artifacts

| Tag | Where | Use |
|---|---|---|
| `[KEY-EXCHANGE] request/response #N ... rotation=true/false` | `src2/shakaplayer/ShakaPlayer.ts` | every DRM license event with timing and size |
| `[FREEZE-EVIDENCE] rotation-detected {...}` | `src2/hooks/videoPlayer/useLiveDecoderWatchdog.ts` | full player snapshot at rotation moment |
| `[FREEZE-EVIDENCE] tick stalls30s=N {...}` | same | full player snapshot every 2 s |

`[FREEZE-EVIDENCE]` JSON fields:

- `currentTime` — `VideoPlayer.currentTime` (W3C)
- `totalFrames` / `droppedFrames` — `getVideoPlaybackQuality()`
- `buffered` — `VideoPlayer.buffered[0]` start-end
- `liveLatency` — Shaka `getStats().liveLatency`
- `readyState`, `networkState`, `paused`, `ended`, `seeking`, `playbackRate` — `VideoPlayer` direct
- `isBuffering`, `stallsDetected` — Shaka diagnostics
