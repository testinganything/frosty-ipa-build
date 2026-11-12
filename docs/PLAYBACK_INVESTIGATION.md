# Frosty Playback Investigation

**Audit snapshot:** 2026-11-12 ~00:15 UTC  
**Upstream revision:** tommyxchow/frosty `main` (as of CI checkout)  
**Plugin:** tommyxchow/flutter-native-video-player `frosty-patches`

## Priority symptoms (user-reported)

1. Excessive playback latency
2. Video freezing while audio continues
3. Video jumping backward several seconds while audio continues forward
4. A/V desync (either track jumps independently)
5. Unreliable recovery after interruptions

## Coverage summary

| Scope | Status |
|-------|--------|
| Open issues (titles + key bodies) | Enumerated (~173); playback subset read in full |
| Closed issues | Enumerated (~219); playback-related sampled |
| Open/merged/draft PRs | Enumerated; video-related PRs read (575, 578, 567, 436, 414, 411, plugin #2) |
| Every comment on every issue/PR | **Not complete** — volume exceeds practical tool limits; prioritized playback threads covered |
| Native plugin iOS/Android live config | Inspected |

Inaccessible / truncated: full paginated comment threads on low-signal issues; some large file responses truncated by tooling.

## Compact audit ledger (playback-related)

| # | URL | Status | Problem / change | Findings | Disposition |
|---|-----|--------|------------------|----------|-------------|
| 573 | issues/573 | OPEN | Native latency +5s vs web | iPad ~18s native vs 3–4s web (Sep 2026). Plugin PR #2 merged for Android. | **Partial** — Android fixed in plugin; iOS residual |
| 418 | issues/418 | OPEN | 20–30s delay | Pre-native-player era; still cited | Duplicate of latency class |
| 608 | issues/608 | OPEN | Bad stream + chat delay | Recent, minimal body | Latency class |
| 598 | issues/598 | OPEN | 5–7s delay after update | May be *improved* latency after plugin | Monitor; may be expected |
| 391 | issues/391 | OPEN | Frame skip/jump landscape high fps | Android A22, 1080p60 | Likely decoder/HLS; needs device |
| 563 | issues/563 | OPEN | Pause on PiP | Native helps; #578 path claimed fix | Partially mitigated |
| 572 | issues/572 | OPEN | Freeze at ad points | Native has ad-break awareness | Investigate ad discontinuity |
| 561 | issues/561 | CLOSED | Landscape stops playback | Fixed by PR #575 (GlobalKey) | Fixed upstream |
| 575 | pulls/575 | MERGED | Keep native player across rotation | GlobalKey re-parent | Done |
| 578 | pulls/578 | DRAFT | Low-latency ~9s→~3s | Depends on plugin; mostly display + temp fork pin | Plugin work landed; app PR stale |
| plugin#2 | flutter-native-video-player#2 | MERGED | media3 prefetch + TARGETDURATION + LoadControl | Android ~3s verified on Pixel 8 | Done on frosty-patches |

## Confirmed causes

### 1. Latency (Android largely fixed; iOS still high)

- **Android:** Twitch advertises `TARGETDURATION:5` but emits ~2s segments. media3 reloaded too slowly → buffer drain → ratchet behind live. Fixed in plugin PR #2 (prefetch promotion, TARGETDURATION rewrite, low-latency LoadControl).
- **iOS:** Plugin already sets `automaticallyWaitsToMinimizeStalling = false`, `preferredForwardBufferDuration = 0`, `automaticallyPreservesTimeOffsetFromLive = true`, seek-to-live with **2s headroom**. Residual latency is mostly classic HLS hold-back + Twitch not serving LL-HLS partials on mobile paths.
- **App:** `highLatencyThresholdSeconds` was **30** — recovery almost never fired for 12–25s cases.

### 2. Video freeze while audio continues

- AVPlayer can drop rate to 0 on empty buffer while audio session continues.
- Stall recovery timer was **8s** before first `seekToLiveEdge()`.
- Comment in code explicitly notes this path.

### 3. Backward jumps / independent track motion

- Concurrent recovery: stall timer and high-latency poll both call `seekToLiveEdge()`.
- Seek targets ~2s before live edge (iOS); discontinuities at ad boundaries can jump.
- Full refresh recreates item → timeline reset can look like a jump.

### 4. Recovery after interruptions

- Bounded retries (`maxRefreshAttempts = 3`, backoff list).
- Ad-break flag can stick true when paused (time observer stops) — recovery deliberately does **not** gate on ad flag for this reason.
- User-paused suppresses auto-recovery; resume can leave a dead session unless `_pendingRefreshOnResume` is set.

## Changes applied in patched IPA builds

| File | Change | Rationale |
|------|--------|-----------|
| `video_timing_constants.dart` | `stallDetectionDelay` 8s → 5s | Faster recovery while still > iOS ~1s and < Android 12s watchdog |
| `video_timing_constants.dart` | `highLatencyThresholdSeconds` 30 → 12 | Engage recovery before users see 15–25s drift |
| `native_video_store.dart` | Latency string to 2 decimals | Observability (PR #578 intent) |

These do **not** claim to fix iOS glass-to-glass latency to ~3s; that requires plugin/Twitch path work.

## Recommended next code work (not yet in IPA)

1. **Coordinate recovery** — single recovery owner; suppress high-latency seek while stall timer is armed (and vice versa) to reduce double seeks/jumps.
2. **Plugin iOS** — experiment reducing the 2s seek headroom to 1s; optional `playImmediately(atRate:)` after seek.
3. **Ad freeze (#572)** — ensure ad-break end forces play even if flag stuck; consider discontinuity listener.
4. **Merge or close #578** — plugin low-latency is already on `frosty-patches`; app PR mainly needs display precision + cleanup of temp fork pin.

## Measurement (no device)

- `streamtop` / ffprobe against Twitch HLS can estimate **CDN edge** latency via `EXT-X-PROGRAM-DATE-TIME` if present.
- Player buffer policy and true glass-to-glass require device or instrumented simulator.
- App already exposes `getLatencyToLive()` — compare before/after on device with the patched IPA.

## Validation status

| Check | Result |
|-------|--------|
| Unsigned IPA builds (clean + patched) | **Pass** (CI macos-latest) |
| Patch application in CI | **Pass** |
| Dart analysis / unit tests | Not run in this environment |
| Device latency measurement | **Not performed** |
| iOS/Android native A/V sync | **Not performed** |

## PR title / description (ready to use upstream)

**Title:** `fix(video): tighter stall and high-latency recovery thresholds`

**Body:**

```
Reduces time-to-recovery for frozen video and high latency on the native player.

- stallDetectionDelay: 8s → 5s (still above iOS native ~1s path, below Android 12s watchdog)
- highLatencyThresholdSeconds: 30 → 12 (engages before users see 15–25s drift)
- Show latency to two decimal places for feedback

Does not change plugin live configuration. Android low-latency (~3s) already landed in flutter-native-video-player#2 on frosty-patches. iOS residual latency is largely classic HLS hold-back.

Related: #573, #418, #608, #598
```
