# Streaming tuning

Audit findings for `Artemide_Experimental_Async`, target setup:
TCL 65C6K (Pentonic 700) · `c2.mtk.hevc.decoder` · 4K60 95 Mbps HEVC ·
wired Gigabit · sub-1 ms RTT.

Existing installs keep their previous prefs — Android does not retroactively
rewrite SharedPreferences when an XML default changes. To pick up the new
defaults: either uninstall + reinstall, or flip the values manually in Settings.

## 1. App settings

| Setting | Value | Default |
|---|---|---|
| Resolution / FPS | 3840×2160 / 60 | ✅ |
| Bitrate | auto (~80 Mbps for 4K60); raise to 95 manually if you like | computed |
| Format | `forceh265` | ✅ |
| Frame pacing | `warp` | ✅ |
| Fast vsync | ON | ✅ |
| CPU boost | profile `medium`, core set `big` | ✅ |
| Prevent packet loss | ON | ✅ |
| Snappy input | ON | ✅ |
| Async decoder | ON | ✅ |
| HDR | ON | ✅ |
| Full-range RGB | ON | ✅ |
| Enforce display mode | ON | ✅ |
| Audio config | 5.1 surround | ✅ |
| Ultra-low-latency | OFF (no-op on Pentonic Codec2) | ✅ |
| Immediate frame delivery | OFF (TV-unsafe; saves ~2 ms but causes drops) | ✅ |
| GL upscaler / GPU path | OFF (4K source on 4K panel) | ✅ |

## 2. Sunshine / Vibepollo (PC)

| Setting | Value |
|---|---|
| Encoder | NVENC HEVC |
| Preset | **P5** or **P6** |
| Spatial AQ | **ON**, strength 8 |
| Temporal AQ | OFF (adds 1 frame) |
| Lookahead | OFF (adds frames) |
| B-frames | 0 |
| Reference frames | 2 |
| Slice count | 1 |
| Bitrate | 80–95 Mbps |
| HDR encoding | ON (Windows HDR must be on) |

## 3. TCL TV menu — per HDMI input

| Setting | Value |
|---|---|
| Picture mode | **Game**, or **PC** / **Computer** if exposed |
| Game Mode + ALLM | ON |
| Local Dimming / Dynamic Backlight | ON for HDR, OFF for SDR |
| HDR Color | Auto |
| Color Tuner / White Balance | Warm 1 or 2 |
| Sharpening | 10–30% or OFF |
| MEMC / Motion Smoothing | **OFF** |
| AI Picture / AI-PQ / AI Super-Resolution | **OFF** |
| Filmmaker / Cinema | **OFF** |
| Noise Reduction (digital + MPEG) | **OFF** |
| Eye Care / Auto Brightness | **OFF** |

## 4. Picture quality vs latency trade-offs

| Feature | Quality | Latency |
|---|---|---|
| Local Dimming | High (HDR contrast) | +5–15 ms |
| HDR10+ dynamic metadata | Per-scene tone-mapping | +2–5 ms |
| Sharpening (low) | Mild | +1–5 ms |
| AI Picture / AI-PQ | Marginal at 4K | +5–15 ms |
| MEMC / Motion Smoothing | Negative for games | +30–80 ms — **never** |
| AI Super-Resolution | None at 4K | +10–30 ms — **never** |

HDR10+ requires source bitstream SEI; stock NVENC produces HDR10 (static
metadata) only. `HDR Color → Auto` falls through to HDR10 when no HDR10+
payload is present.

Codec-stage PQ engine (`mtk-pq`, `mtk-aivision`, `mtk-camera`,
`mtk-watermark`, `mtk-codec2.is-filmmaker`) is suppressed by Tier 2 of the
c2.mtk preset. Local Dimming and HDR10+ operate at the panel-driver layer,
downstream of those hints — they're unaffected.

## 5. Code changes applied

| Commit | Change |
|---|---|
| `b643aa9b` | Pentonic c2.mtk preset (BSP-correct vendor namespaces) + new keys in `knownVendorLowLatencyOptions` |
| `4745cbd3` | GitHub Actions auto-build & release |
| `04a0046e` | `KEY_LATENCY=0` + `KEY_OUTPUT_REORDER_DEPTH=0` (Android R+) and `setSustainedPerformanceMode(true/false)` on the streaming surface |
| `c788e101` | Latency / quality / TV-class defaults |
| `33221d2e` | 4K60 / enforce-display-mode / 5.1-surround defaults |

## 6. Verification

```bash
adb logcat -d -s LimeLog:* | grep -E \
  "Decoder (input|output) format|Configuring with format|Selected (HEVC|AVC|AV1)|HDR mode|Applied (post-start|Surface)"
```

| Tag | Means |
|---|---|
| `Selected HEVC decoder: c2.mtk.hevc.decoder` | HEVC + Pentonic ✓ |
| `HDR mode: enabled` | Window in HDR colour mode ✓ |
| `Applied post-start low-latency parameter` | post-`start()` poke ✓ |
| `Applied Surface frame rate: 60 Hz` | frame-rate hint applied ✓ |
| `Decoder watchdog: no output >1.2s` | **bad** — codec is sleeping; preset too aggressive |

A key set in `Configuring with format:` but absent from `Decoder input format:`
was silently rejected by the BSP.

## 7. `c2.mtk.hevc.decoder` reference

- HW-accelerated; `FEATURE_LowLatency` advertised
- Max 100 Mbps / 4096×2304 / 64.5 fps at 4K
- Profiles: HEVC Main / Main10 / Main10 HDR10 / **Main10 HDR10+**
- Tunneled playback supported but unused (Moonlight is non-tunneled)

| Vendor namespace | Used for |
|---|---|
| `vendor.START.*` | start-time hints |
| `vendor.mtk-codec2.*` | decoder behaviour, frame rate, scan, error |
| `vendor.mtk-pq.*` | picture-quality engine |
| `vendor.mtk-aivision.*` | visual boost, DPTZ |
| `vendor.mtk-camera.*` | camera-pipeline PQ / AI-SR |
| `vendor.mtk-watermark.*` | overlay / watermark |
| `vendor.mtk-iview.*` | internal view manager |

The legacy `vendor.mtk.vdec.*` namespace does **not** exist on Pentonic and
was silently no-op'd before commit `b643aa9b`.
