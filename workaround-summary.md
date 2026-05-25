# Vega HW Decoder Freeze on DRM Key Rotation — Evidence Report

**Date captured:** 2026-05-18
**Device:** Fire TV Stick 4K Select (`callie`, ARMv7)
**Vega SDK:** 0.22
**Source log:** `metro.log` (10705 lines, ~30-minute live session)

---

## Environment

| Component | Value |
|---|---|
| Device | Fire TV Stick 4K Select (`callie`) |
| Vega SDK | 0.22 |
| Runtime | Hermes |
| W3C Media | `@amazon-devices/react-native-w3cmedia ~2.1.80` |
| Player | Shaka Player 4.8.5-r1.5 (Amazon fork, bundled) |
| Backend | Inline (not headless) |
| Manifest | DASH |
| Video codec | H.264 |
| Audio codec | EC-3 |
| DRM | Widevine, `HW_SECURE_ALL` video / `SW_SECURE_CRYPTO` audio |
| Live profile | `bufferingGoal: 15`, `bufferBehind: 2`, `liveSync: true`, no ABR cap |

---

## Log conventions

Diagnostic logs are tagged with prefixes for easy grep:

| Prefix | Source | Meaning |
|---|---|---|
| `[KEY-EXCHANGE]` | `ShakaPlayer.ts` license filters | Every DRM license request/response with sequence number, timing, payload size, rotation flag (response received >60s post-load) |
| `[FREEZE-EVIDENCE]` | `useLiveDecoderWatchdog.ts` | Full snapshot of W3C VideoPlayer state every 2s + dedicated snapshots at rotation moment and pre-reload moment |
| `[LiveDecoderWatchdog]` | `useLiveDecoderWatchdog.ts` | Rotation detection + soft-reload trigger |
| `[SOFT-RELOAD]` | `useInitPlayer.ts` `softReload` | Phase-by-phase reload timing |

`[FREEZE-EVIDENCE]` snapshot shape:

```json
{
  "ts": 1779099434181,           // wall-clock ms
  "currentTime": "24019674.364", // W3C VideoPlayer.currentTime (live, seconds)
  "paused": false,
  "ended": false,
  "seeking": false,
  "playbackRate": 1,
  "readyState": 3,               // HAVE_FUTURE_DATA
  "networkState": 0,             // NETWORK_EMPTY
  "totalFrames": 23845,          // VideoPlaybackQuality.totalVideoFrames
  "droppedFrames": 0,            // VideoPlaybackQuality.droppedVideoFrames
  "buffered": "24019642.22-24019696.31",
  "liveLatency": "39.16",
  "isBuffering": false,
  "stallsDetected": 0
}
```

---

## Session timeline

Four playback sessions, separated by three soft reloads:

```
Session 1 ─── rotation #1 (t=804.4s) → SOFT-RELOAD (4023 ms)
Session 2 ─── rotation #2 (t=896.3s) → SOFT-RELOAD (3916 ms)
Session 3 ─── rotation #3 (t=897.8s) → SOFT-RELOAD (3798 ms)
Session 4 ─── (no rotation reached; capture ended mid-session)
```

Wall-clock cadence between consecutive rotations:

| Boundary | Wall clock (ms) | Δ from previous |
|---|---|---|
| Rotation #1 detected | 1779099434181 | — |
| Rotation #2 detected | 1779100338370 | 15 min 4 s |
| Rotation #3 detected | 1779101244100 | 15 min 6 s |

Cadence: **~15.1 minutes**, stable.

---

## License exchange — all events

All `[KEY-EXCHANGE]` lines from the log, grouped by session.

### Session 1 (initial)

| # | Direction | t (s) | Duration | Size | Rotation |
|---|---|---|---|---|---|
| 1 | request | 6.8 | — | 2206 b | false |
| 1 | response | 8.5 | 1635 ms | 955 b | false |
| 2 | request | 15.7 | — | 2206 b | false |
| 2 | response | 16.3 | 575 ms | 955 b | false |
| **3** | **request** | **804.4** | — | **2206 b** | **true** |
| **3** | **response** | **805.3** | **914 ms** | **955 b** | **true** |

### Session 2 (after first soft reload)

| # | Direction | t (s) | Duration | Size | Rotation |
|---|---|---|---|---|---|
| 1 | request | 4.0 | — | 2206 b | false |
| 2 | request | 4.3 | — | 2206 b | false |
| 2 | response | 5.6 | 1298 ms | 955 b | false |
| 2 | response | 5.6 | 1299 ms | 955 b | false |
| 3 | request | 9.8 | — | 2206 b | false |
| 3 | response | 10.9 | 1023 ms | 955 b | false |
| 4 | request | 12.7 | — | 2206 b | false |
| 4 | response | 13.3 | 640 ms | 955 b | false |
| 5 | request | 17.7 | — | 2206 b | false |
| 5 | response | 18.3 | 540 ms | 955 b | false |
| **6** | **request** | **896.3** | — | **2206 b** | **true** |
| **6** | **response** | **897.1** | **788 ms** | **955 b** | **true** |

### Session 3 (after second soft reload)

| # | Direction | t (s) | Duration | Size | Rotation |
|---|---|---|---|---|---|
| 1 | request | 7.0 | — | 2206 b | false |
| 2 | request | 7.2 | — | 2206 b | false |
| 2 | response | 7.6 | 402 ms | 955 b | false |
| 2 | response | 7.7 | 528 ms | 955 b | false |
| 3 | request | 11.8 | — | 2206 b | false |
| 3 | response | 12.3 | 546 ms | 955 b | false |
| 4 | request | 13.2 | — | 2206 b | false |
| 4 | response | 13.9 | 616 ms | 955 b | false |
| **5** | **request** | **897.8** | — | **2206 b** | **true** |
| **5** | **response** | **898.6** | **793 ms** | **955 b** | **true** |

### Session 4 (after third soft reload — startup only, session still running at end of capture)

| # | Direction | t (s) | Duration | Size | Rotation |
|---|---|---|---|---|---|
| 1 | request | 9.4 | — | 2206 b | false |
| 1 | response | 12.8 | 3409 ms | 955 b | false |
| 2 | request | 24.4 | — | 2205 b | false |
| 2 | response | 26.8 | 2452 ms | 955 b | false |
| 3 | request | 32.3 | — | 2206 b | false |
| 3 | response | 36.1 | 3847 ms | 955 b | false |

License response durations in session 4 are 3–4 seconds, vs. ~600 ms in sessions 2/3 — unrelated to the rotation freeze but flagged for completeness.

---

## Per-rotation detail

### Rotation #1 — t = 804.4 s

Log lines:

```
[KEY-EXCHANGE] request #3 sent at t=804.4s rotation=true bodySize=2206b
[KEY-EXCHANGE] response #3 received at t=805.3s duration=914ms size=955b rotation=true
```

State at rotation detection (ts=1779099434181):

```json
{
  "currentTime": "24019674.364",
  "totalFrames": 23845,
  "droppedFrames": 0,
  "buffered": "24019642.22-24019696.31",
  "liveLatency": "39.16",
  "isBuffering": false,
  "readyState": 3
}
```

State 5 s later, immediately before soft reload (ts=1779099439268, Δ=5087 ms):

```json
{
  "currentTime": "24019679.452",  // +5.088 s — advancing normally
  "totalFrames": 23978,            // +133 frames in ~5 s (~26 fps)
  "droppedFrames": 0,
  "buffered": "24019642.22-24019702.28",  // buffer end grew +6 s
  "liveLatency": "39.16",
  "isBuffering": false,
  "readyState": 3
}
```

Soft reload timing:

```
[SOFT-RELOAD] start
[SOFT-RELOAD] teardown done in 1236ms
[SOFT-RELOAD] reinit done in 79ms
[SOFT-RELOAD] load done in 2708ms — total 4023ms
```

### Rotation #2 — t = 896.3 s

State at rotation detection (ts=1779100338370):

```json
{
  "currentTime": "24020582.051",
  "totalFrames": 26653,
  "droppedFrames": 40,             // 40 frames dropped earlier in session
  "buffered": "24020551.13-24020603.17",
  "liveLatency": "34.43",
  "isBuffering": true,             // buffering at rotation moment
  "readyState": 3
}
```

State 5 s later (ts=1779100343684, Δ=5314 ms):

```json
{
  "currentTime": "24020587.365",  // +5.314 s — advancing normally
  "totalFrames": 26854,            // +201 frames in ~5.3 s (~38 fps)
  "droppedFrames": 40,             // no new drops in the window
  "buffered": "24020551.13-24020603.17",  // buffer end DID NOT advance
  "liveLatency": "34.43",
  "isBuffering": true,
  "readyState": 3
}
```

> `buffered.end` did not advance during the 5-second window — Shaka stopped appending segments. Most suspicious signal in the log; suggests the new key was not bound to the active session before our preventive reload kicked in.

Soft reload timing:

```
[SOFT-RELOAD] start
[SOFT-RELOAD] teardown done in 1034ms
[SOFT-RELOAD] reinit done in 89ms
[SOFT-RELOAD] load done in 2793ms — total 3916ms
```

### Rotation #3 — t = 897.8 s

State at rotation detection (ts=1779101244100):

```json
{
  "currentTime": "24021489.369",
  "totalFrames": 26834,
  "droppedFrames": 6,
  "buffered": "24021456.07-24021504.07",
  "liveLatency": "35.54",
  "isBuffering": true,
  "readyState": 3
}
```

State 5 s later (ts=1779101249147, Δ=5047 ms):

```json
{
  "currentTime": "24021494.416",  // +5.047 s — advancing normally
  "totalFrames": 26974,            // +140 frames in ~5 s (~28 fps)
  "droppedFrames": 6,
  "buffered": "24021460.07-24021510.09",  // buffer end +6 s
  "liveLatency": "35.54",
  "isBuffering": true,
  "readyState": 3
}
```

Soft reload timing:

```
[SOFT-RELOAD] start
[SOFT-RELOAD] teardown done in 982ms
[SOFT-RELOAD] reinit done in 45ms
[SOFT-RELOAD] load done in 2771ms — total 3798ms
```

---

## Summary tables

### Per-rotation outcomes

| Rotation | t since load (s) | License resp duration | Reload total | Buffer behavior in 5 s window |
|---|---|---|---|---|
| #1 | 804.4 | 914 ms | 4023 ms | Buffer end grew +6 s (normal) |
| #2 | 896.3 | 788 ms | 3916 ms | **Buffer end did not advance** |
| #3 | 897.8 | 793 ms | 3798 ms | Buffer end grew +6 s (normal) |

### Soft-reload phase timing

| Phase | Min | Max | Avg |
|---|---|---|---|
| Teardown | 982 ms | 1236 ms | ~1084 ms |
| Reinit | 45 ms | 89 ms | ~71 ms |
| Load | 2708 ms | 2793 ms | ~2757 ms |
| **Total** | **3798 ms** | **4023 ms** | **~3912 ms** |

---

## Limitations of this evidence

The current log proves **correlation** (every rotation is followed within 5 seconds by a preventive reload) but does **not directly capture the freeze pattern** itself, because the pre-emptive reload tears down the player before the freeze can manifest.

To produce a direct capture of the freeze:

1. Temporarily disable the pre-emptive reload path (keep stall-loop fallback as a safety net).
2. Let one rotation happen and proceed to the freeze (typically 10–126 s after rotation in prior uninstrumented sessions).
3. Capture `[FREEZE-EVIDENCE]` snapshots showing `currentTime` stuck while `totalFrames` continues to grow.

---

## Source files

- `src2/shakaplayer/ShakaPlayer.ts` — `[KEY-EXCHANGE]` instrumentation in license request/response filters
- `src2/services/player/inlinePlayerImpl.ts` — `snapshotPlayerState()` exposing raw W3C VideoPlayer state
- `src2/hooks/videoPlayer/useLiveDecoderWatchdog.ts` — periodic + rotation/reload snapshots, watchdog trigger
- `src2/hooks/videoPlayer/useInitPlayer.ts` — `softReload()` with phase timing
