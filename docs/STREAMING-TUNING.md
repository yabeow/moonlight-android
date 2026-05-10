# Streaming tuning guide

This document captures the optimization findings from the audit of the
`Artemide_Experimental_Async` branch against a real-world target setup:

- **Client:** TCL 65C6K TV — Pentonic 700 (MT9689) — Android TV
- **Codec:** `c2.mtk.hevc.decoder` (HEVC Main / Main10 / Main10 HDR10 / Main10 HDR10+)
- **Stream:** 4K @ 60 fps, 95 Mbps, HEVC, Sunshine/Vibepollo on the host
- **Network:** wired Gigabit (USB dongle on the TV), sub-1 ms RTT
- **Goal:** minimise end-to-end latency **and** maximise picture quality (including HDR)

Findings are sorted by **independence** — knobs that improve one axis without
costing the other come first; trade-offs come last.

---

## 1. Independent latency wins

These reduce latency at zero or negligible quality cost. Apply all of them.

### App settings

The first six rows are **now the app's defaults** — fresh installs and clean
preferences pick them up automatically. Existing installs keep whatever the
user previously configured (Android does not retroactively rewrite shared
preferences when a default value changes in XML).

| Setting (preference) | Value | Default? | Why |
|---|---|---|---|
| **Frame pacing** (`framePacing`) | `warp` | **Default** | Tightest output gating; NanoPacer's spin/park delivers frames within 2–100 µs of the display deadline. |
| **Fast vsync** (`fastVsync`) | ON | **Default** | Activates NanoPacer. Has no effect outside WARP / WARP2 / MIN_LATENCY / GPU_RAW pacing. |
| **CPU boost** (`pref_cpu_warmup_boost`) | **MEDIUM** | **Default** | Pre-scales DVFS so frame-1 and burst-decode are stable. Has a thermal gate built in. Master switch (`pref_cpu_warmup_enable`) also defaults ON. |
| **CPU boost core set** (`pref_cpu_warmup_core_set`) | **big** | **Default** | A73 cluster only — lower thermal load than `all`. |
| **Prevent packet loss** (`preventPacketLoss`) | ON | **Default** | Tiny FEC/jitter guard on the receive path. Effectively free at <1 ms RTT. |
| **Snappy input** (`snappyInput`) | ON | **Default** | Routes gamepad events off the main thread; cuts ~1–3 ms of input lag. |
| **Async decoder** (`asyncDecodeEnabled`) | ON | **Default** | MediaCodec async-callback mode. Eliminates `dequeueInputBuffer`/`dequeueOutputBuffer` polling latency. |
| **Ultra-low-latency** (`enableUltraLowLatency`) | **OFF** | Default | Only affects the legacy `omx.mtk` path. On Pentonic / `c2.mtk.*` it's a no-op now that the BSP-correct preset is in place. |
| **Immediate frame delivery** (`immediateFrameDelivery`) | **OFF** | Default | Forces 0 µs dequeue + URGENT priority on a TV. Saves ~2 ms but can cause occasional drops/tearing. The 500 µs WARP timeout is plenty. |
| **GL upscaler** (`videoUpscaleEnable`, `gpuPathMode`) | **OFF** | Default | 4K source on a 4K panel — the GL upscaler is a render+copy pass with no benefit. |

### Code-level changes (proposed)

These are independent of any user setting and apply on *every* MediaCodec
configure call:

1. **`MediaFormat.KEY_LATENCY = 0`** (Android M+)
   Standard cross-vendor "no buffering" hint. Currently not set anywhere in the
   tree.

2. **`MediaFormat.KEY_OUTPUT_REORDER_DEPTH = 0`** (Android Q+)
   Standard cross-vendor "no output reordering" hint. Independent of the per-vendor
   `vendor.qti-ext-dec-picture-order.enable` already set for Qualcomm.

3. **`Window.setSustainedPerformanceMode(true)`** on the streaming activity.
   Was present on `moonlight-noir` (Game.java:3817) but is missing on
   `Artemide_Experimental_Async`. Stops the SoC governor from idling between frames.

---

## 2. Independent picture-quality wins

These improve image quality at zero or negligible latency cost.

### Sunshine / Vibepollo (PC encoder)

| Setting | Value | Why |
|---|---|---|
| NVENC preset | **P5 or P6** (slow / quality) | Encode work happens on the *PC*, which has the headroom. Higher preset = better quality at the same bitrate. Latency cost on the encoder side is sub-millisecond. |
| **Spatial AQ** | **ON**, strength 8 | Adaptive Quantization spends bits where humans notice (textures, faces, dark regions). Zero latency cost. |
| Temporal AQ | **OFF** | Adds 1 frame of buffering. |
| Lookahead | **OFF** | Adds multiple frames of buffering. |
| HDR encoding | **ON** (HEVC Main 10) | Wider colour volume, deeper blacks/whites. Requires HDR enabled in Windows display settings. |
| Slice count per frame | **1** | Matches `getDecoderOptimalSlicesPerFrame()` for Pentonic-class decoders. |
| B-frames | **0** | Sunshine default. Each B-frame adds a frame of latency. |
| Reference frames | 2 (Sunshine default) | More refs help compression at the cost of decode complexity; 2 is a good balance. |
| CABAC (HEVC) | always-on (built into HEVC) | n/a |
| Bitrate | **80–95 Mbps** | At 4K60 HEVC the codec ceiling is 100 Mbps; leaving headroom prevents bitstream spikes from triggering decoder recovery on action scenes. |

### App settings

| Setting (preference) | Value | Default? | Why |
|---|---|---|---|
| **HDR** (`enableHdr`) | ON | **Default** | The existing pipeline plumbs HDR static info via the CTA-861.3 InfoFrame (`MediaCodecDecoderRenderer.configureAndStartDecoder`); the activity calls `setColorMode(COLOR_MODE_HDR)` (`Game.updateHdrWindowMode`); the codec advertises `HEVCProfileMain10HDR10` and `HDR10Plus`. The runtime checks `Display.HdrCapabilities` and falls back to SDR (with a Toast) on HDR-incapable displays, so default-ON is safe. |
| **Full-range RGB** (`fullRange`) | ON | **Default** | Configure Sunshine to match (encode full-range YUV). Mismatch produces washed-out blacks (full→limited) or crushed whites (limited→full). |
| **Format** (`videoFormat`) | **Force HEVC** | **Default** | AV1 caps at 40 Mbps on this codec spec, well below a 95 Mbps stream. Forcing HEVC prevents accidental AV1 selection on 10-bit streams. Devices without an HEVC decoder will fall through; this build targets HEVC-capable TV-class hardware. |

### TV menu (TCL on Pentonic)

| Setting | Value | Why |
|---|---|---|
| Picture mode | **Game** (or **PC** if available) | PC mode often has even less PQ pipeline than Game while keeping low latency. |
| Game Mode + ALLM | **ON** | Auto-low-latency mode signals the TV to bypass most processing. |
| Color Tuner / White Balance | **Warm 1 or 2** | Calibrates closer to D65. Pure picture/quality knob, no latency cost. |
| HDR Color | **Auto** | Lets the TV apply its HDR pipeline appropriately. |
| Game-Mode-gated colour enhance / sharpening (low: 10–30%) | optional ON | Most TCLs gate these to a low-latency sub-pipeline in Game mode. Try it; if decode-time number ticks up in the perf overlay, turn it off. |

---

## 3. The trade-off zone

These have real picture-quality benefits but also real latency costs. Pick where
you want to land.

| Feature | Quality benefit | Latency cost |
|---|---|---|
| **Local dimming / dynamic backlight** (HDR essential) | Huge — proper blacks, HDR contrast | ~5–15 ms |
| **HDR10+ dynamic metadata** (vs static HDR10) | Noticeable per-scene tone-mapping | ~2–5 ms |
| **Sharpening** (low setting only) | Mild edge clarity | ~1–5 ms |
| **AI Picture / AI-PQ engine** | Marginal at native 4K source | ~5–15 ms |
| **MEMC / Motion Smoothing** | **Negative for games** (input lag + soap-opera) | **30–80 ms — never enable for streaming** |
| **AI Super-Resolution** | None at 4K source | **10–30 ms — never** |
| **Filmmaker / Cinema mode** | Wrong colour targets for games | Adds processing — never |

**Practical sweet spot for HDR gaming on this TV:**

> Game mode ON, Local Dimming ON, HDR Color Auto, low Sharpening (10–30%),
> Game-Mode-gated Colour Enhance ON, **everything else OFF**.

---

## 4. Proposed code changes (status)

| # | Change | Status | File / location |
|---|---|---|---|
| 1 | Split `c2.mtk` from `omx.mtk` in `setDecoderLowLatencyOptions`; apply BSP-correct Pentonic Codec2 / mtk-pq / mtk-aivision / mtk-camera / mtk-watermark / mtk-iview keys | **Applied** (`b643aa9b`) | `MediaCodecHelper.java:649-742` |
| 2 | Add Pentonic Codec2 advertisement keys to `knownVendorLowLatencyOptions` | **Applied** (`b643aa9b`) | `MediaCodecHelper.java:227-229` |
| 3 | GitHub Actions workflow: build + release on every push | **Applied** (`4745cbd3`) | `.github/workflows/build-and-release.yml` |
| 4 | Make Tier 2 (PQ disable) conditional on a new `disablePostDecodePq` preference | **Skipped** — current default already disables codec-stage PQ, which is what we want. TV-menu Local Dimming + HDR10+ are independent of these codec hints. |
| 5 | Add `MediaFormat.KEY_LATENCY = 0` and `KEY_OUTPUT_REORDER_DEPTH = 0` to `setDecoderLowLatencyOptions` | **Applied** | `MediaCodecHelper.java` near the existing `low-latency=1` block |
| 6 | Re-add `setSustainedPerformanceMode(true)` on the streaming activity | **Applied** | `Game.java` `surfaceCreated` / `surfaceDestroyed` |

### Picture-quality intent (post-processing)

Post-decode picture-quality features fall into three categories:

- **Codec-stage PQ** — what the decoder's PQ engine adds before handing frames
  to the display engine (sharpening, NR, AI-SR, Visual Boost, watermarking,
  Filmmaker, etc.). Tier 2 of the c2.mtk preset suppresses these via codec
  hints, which is **the current default behaviour and is intended**.
- **TV-menu PQ** — settings the TV applies inside its display engine. The
  codec hints don't override these. This is where the user sets exactly which
  features they want (see §5 TV menu checklist).
- **Panel/backlight features** (Local Dimming, HDR10+ dynamic metadata) — these
  operate on the actual luminance of the displayed frame at the panel-driver
  layer, separate from the PQ engine and the menu-driven enhancement features.
  Disabling codec-stage PQ does **not** affect them.

The intended end state for this setup:

| Stage | Status |
|---|---|
| Codec-stage PQ engine (mtk-pq, mtk-aivision, mtk-camera, mtk-watermark, filmmaker) | **OFF** (Tier 2 codec hints) |
| TV-menu MEMC, AI-SR, AI-PQ, Filmmaker, Noise Reduction | **OFF** (TV menu) |
| TV-menu Sharpening, Color Enhance | **OFF** or very low (TV menu) |
| **Local Dimming / Dynamic Backlight** | **ON** (TV menu — panel feature, latency cost ~5–15 ms, accepted) |
| **HDR10+ dynamic metadata** | **ON** (HDR Color → Auto in TV menu — applied automatically when the bitstream contains HDR10+ SEI) |

Note: HDR10+ requires the source bitstream to actually contain HDR10+ SEI
metadata. Stock NVENC encoding via Sunshine produces HDR10 (static metadata
only) — HDR10+ would need either content that already carries it (e.g.
playback of HDR10+ media on the host) or an encoder that synthesises it.
Setting "HDR Color → Auto" on the TV is correct in either case: it falls
through to HDR10 static metadata when no HDR10+ payload is present.

---

## 5. Quick configuration checklists

### Moonlight (Artemide app on the TV)

- [ ] Format → **Force HEVC**
- [ ] Resolution → 3840×2160, FPS → 60, Bitrate → 80–95 Mbps
- [ ] Frame pacing → **Warp** (or Warp2)
- [ ] Fast vsync → **ON**
- [ ] CPU boost → **ON**, profile **MEDIUM**, **BIG cores**, **4 workers**
- [ ] Async decoder → **ON** (default)
- [ ] Prefer big cores → **ON**
- [ ] Prevent packet loss → **ON**
- [ ] HDR → **ON** (only with HDR-encoded stream and HDR-mode TV)
- [ ] Full-range RGB → match Sunshine output
- [ ] GL upscaler / GPU path mode → **OFF**
- [ ] Immediate frame delivery → **OFF**
- [ ] Ultra-low-latency → **OFF** (no-op on Pentonic)

### Sunshine / Vibepollo (PC)

- [ ] Encoder: **NVENC HEVC**
- [ ] Preset: **P5** or **P6**
- [ ] Spatial AQ: **ON** (strength 8)
- [ ] Temporal AQ: **OFF**
- [ ] Lookahead: **OFF**
- [ ] B-frames: **0**
- [ ] Reference frames: 2 (default)
- [ ] Slice count: **1**
- [ ] Bitrate: **80–95 Mbps** for 4K60
- [ ] HDR encoding: **ON** (only if Windows HDR display setting is ON)

### TCL TV picture menu (per-input or per-app)

- [ ] Picture mode → **Game** (or **PC**)
- [ ] Game Mode → **ON** (ALLM auto-detect)
- [ ] Local Dimming / Dynamic Backlight → **ON** (HDR), OFF (SDR)
- [ ] HDR Color → **Auto**
- [ ] Color Tuner / White Balance → **Warm 1 or 2**
- [ ] Sharpening → low (10–30%) or off
- [ ] **MEMC / Motion Smoothing → OFF**
- [ ] **AI Picture / AI-PQ → OFF**
- [ ] **AI Super-Resolution → OFF**
- [ ] **Filmmaker → OFF**
- [ ] **Noise Reduction (digital + MPEG) → OFF**
- [ ] **Eye Care / Auto Brightness → OFF**

---

## 6. Verifying the configuration

After installing a CI APK, capture logcat with:

```bash
adb logcat -d \
  -s LimeLog:* \
  | grep -E "Decoder (input|output) format|Configuring with format|Selected (HEVC|AVC|AV1)|Applied (post-start|Surface) (low-latency|frame)|HDR mode|RendererAffinity"
```

The `Configuring with format:` line shows every key the app *attempted* to set.
The `Decoder input format:` and `Decoder output format:` lines (logged after
`videoDecoder.start()`) show what the codec *accepted* — keys present in the
input format dump but missing from the output format dump were silently
rejected by the BSP.

Useful tags to look for:

- `Selected HEVC decoder: c2.mtk.hevc.decoder` — confirms HEVC + Pentonic
- `HDR mode: enabled` — confirms `setColorMode(COLOR_MODE_HDR)`
- `RendererAffinity: ... allowed_after=...` — confirms big-core pinning
- `Applied post-start low-latency parameter` — confirms the post-`start()` poke
- `Applied Surface frame rate: 60 Hz` — confirms `Surface.setFrameRate(60)`
- `Decoder watchdog: no output >1.2s` — should **never** appear; if it does, the
  codec is going to sleep and the preset is too aggressive

---

## 7. Codec context (for reference)

`c2.mtk.hevc.decoder` (Pentonic 700 / MT9689):

- Hardware-accelerated, low-latency feature: **true (FEATURE_LowLatency)**
- Max bitrate: **100 Mbps** (you stream 95 → near ceiling, 80–95 recommended)
- Max resolution: 4096×2304
- 4K max fps: 64.5 (60 fps fits comfortably)
- Profiles: HEVC Main / Main10 / **Main10 HDR10** / **Main10 HDR10+**
- Adaptive playback: yes
- Tunneled playback: supported but **not used** (Moonlight is non-tunneled)

Vendor parameter namespaces (Codec2 / TV-class media stack):

- `vendor.START.*` — start-time hints
- `vendor.mtk-codec2.*` — core decoder behaviour, frame rate, scan, error
- `vendor.mtk-pq.*` — picture-quality engine
- `vendor.mtk-aivision.*` — AI vision (visual boost, DPTZ)
- `vendor.mtk-camera.*` — camera-pipeline PQ / AI-SR
- `vendor.mtk-watermark.*` — overlay/watermark detection
- `vendor.mtk-iview.*` — internal view manager (real-time hints)

The pre-Pentonic legacy namespace `vendor.mtk.vdec.*` does **not** exist on
this BSP. Any keys set there are silently ignored — that was the situation
before the `c2.mtk` split applied in commit `b643aa9b`.
