# Electron Feasibility Research

**Ticket:** https://github.com/lpridpath/tennis_MatchChartingProject_tagger/issues/2
**Retrieval date:** 2026-09-06
**Scope:** Fact-finding on the planned Electron stack (Stream Deck XL control + local video frame-stepping). Facts and showstoppers only — no product decisions.

Stack under test: Electron desktop app driving an Elgato Stream Deck XL over USB HID, plus HTML5 `<video>` playback of local YouTube-sourced tennis files (H.264/mp4, VP9/webm).

---

## Q1 — Stream Deck XL control in Electron

**Verdict: GREEN** (works; two operational caveats, both manageable)

### Findings

- **Library:** `@elgato-stream-deck/node`, maintained by Julusian (SuperFlyTV fork; the canonical Node library). **Current version 7.6.3, published 2026-06-02**, `engines.node >= 18.18`. It is a thin Node wrapper over the platform-agnostic `@elgato-stream-deck/core`.
  - Source: https://github.com/Julusian/node-elgato-stream-deck ; registry `https://registry.npmjs.org/@elgato-stream-deck/node/latest`.
- **HID access:** depends on **`node-hid ^3.2.0`** (native C++ addon over hidapi; IOKit HID on macOS). Full deps at 7.6.3: `node-hid ^3.2.0`, `@elgato-stream-deck/core`, `@elgato-stream-deck/node-lib`, `p-queue`, `eventemitter3`, `tslib`. There is also a WebHID variant (`@elgato-stream-deck/webhid`) and a bring-your-own-HID path via `core`.
- **(a) Per-key images — YES.** README Features: "Fill keys with images or solid RGB colors" and "Fill the entire panel with a single image, spread across all keys." For the **XL specifically**, install the optional `@julusian/jpeg-turbo@^2.0.0` — the README explicitly warns that without it, `jpeg-js` makes XL image transfers "noticably more cpu intensive and slower." Prebuilt binaries ship for jpeg-turbo.
- **(b) Key-press events — YES.** Emits `down` / `up` events with a `keyIndex` (README API + example: `myStreamDeck.on('down', keyIndex => …)`). Event-driven, so latency is USB-interrupt bound (sub-frame, well within "usable" for live tagging).
- **Process model:** `node-hid` is a native Node addon, so it must run where Node integration + native modules are available — i.e. the **main process** (or a Node-enabled utility/preload with `nodeIntegration`), **not a sandboxed renderer**. Bridge key events to the renderer via IPC. (The WebHID variant is the renderer-side alternative but requires a user gesture + `navigator.hid` permission and is a different code path.)
- **Native rebuild for Electron:** all native deps "ship with prebuilt binaries, so having a full compiler toolchain should not be necessary." However, Electron uses a different Node ABI than system Node, so the standard practice remains adding `@electron/rebuild` (formerly `electron-rebuild`) as a `postinstall` step to rebuild `node-hid` against the target Electron version. Toolchain fallback on macOS is just `xcode-select --install`.
- **macOS specifics:**
  - `node-hid` reaches the device through IOKit HID. On modern macOS the OS may prompt for **Input Monitoring** (TCC) the first time the app reads HID input; the user grants it in System Settings → Privacy & Security. For **direct distribution** (not Mac App Store) a non-sandboxed **Hardened Runtime** build needs **no special USB/HID entitlement**. Only a **sandboxed** build (MAS) requires `com.apple.security.device.usb`.
  - **Elgato desktop app / device claim:** macOS HID input reports are not exclusively locked, but in practice the Elgato Stream Deck app continuously renders key images and consumes presses, so it will fight a second controller. **Quit the Elgato Stream Deck app** while this tool owns the deck. Community reports confirm a resident controller (e.g. `streamdeck-ui`) "captures the USB device," blocking others.
  - Sources: https://github.com/node-hid/node-hid/issues/456 ; https://help.elgato.com/hc/en-us/articles/360028240651 ; https://github.com/Julusian/node-elgato-stream-deck (README).

### Implications for architecture
- Own the deck in the **main process**; expose an IPC surface (`renderKey(index, image)`, `on keyDown/keyUp`) to the renderer.
- Add `@electron/rebuild` postinstall + bundle/install `@julusian/jpeg-turbo` for XL image speed.
- Ship as a **non-sandboxed Hardened-Runtime** app (avoid MAS to skip USB entitlement + sandbox friction). Handle the Input-Monitoring prompt in first-run UX.
- Product/UX must tell the user to quit the Elgato app (or detect it).

---

## Q2 — Local video frame-step in Electron

**Verdict: GREEN** (Electron's bundled codecs are the key enabler; one accuracy caveat for VFR webm)

### Findings

- **Codec support — the load-bearing fact:** vanilla Chromium ships **without** H.264/AAC (open codecs only: VP8/VP9/WebM/Ogg). **Electron is different — official Electron builds compile ffmpeg with proprietary codecs enabled by default** (`proprietary_codecs=true`, `ffmpeg_branding="Chrome"` in Electron's `build/args/release.gn`), so **H.264/mp4 AND VP9/webm both play out of the box**. An alternate codec-free ffmpeg is available only if you deliberately need to distribute without proprietary codecs.
  - Sources: https://fossies.org/linux/electron/build/args/release.gn ; https://github.com/electron/electron/issues/633 ; https://github.com/electron/electron/issues/9534 . Contrast (vanilla Chromium lacks them): https://support.vuplex.com/articles/how-to-enable-proprietary-video-codecs/ .
  - **Licensing caveat (not technical):** H.264 is patent-encumbered; shipping it is a distribution/legal concern (AVC/Via LA licensing), though for a personal/internal tool this is not a runtime blocker.
- **Pause / frame-step / precise seek — supported.** Standard techniques both work in Chromium/Electron:
  - `HTMLVideoElement.requestVideoFrameCallback()` (rVFC) fires per presented frame with `metadata.mediaTime` (exact presentation timestamp). Frame-accurate stepping = nudge `currentTime` then read back `mediaTime` from rVFC. Callback rate = min(video fps, display refresh).
  - Fallback: `currentTime += 1/fps`. Simple but assumes constant fps.
  - Sources: https://developer.mozilla.org/en-US/docs/Web/API/HTMLVideoElement/requestVideoFrameCallback ; https://web.dev/articles/requestvideoframecallback-rvfc ; https://wicg.github.io/video-rvfc/ .
- **Seek precision limits / transcoding:** Chromium seeks to the exact `currentTime` (decodes forward from the prior keyframe), so seeks are frame-accurate in principle. **Caveat:** YouTube-sourced **VP9/webm is frequently variable-frame-rate (VFR)**, which breaks naive `1/fps` math — use **rVFC `mediaTime`** as ground truth instead of assuming a fixed fps. Very long GOPs can make backward single-stepping feel slow (decoder walks from the last keyframe). If a specific file steps unreliably, a one-time **remux/transcode to constant-fps H.264 mp4** (ffmpeg) fixes it — likely optional, not required across the board.

### Implications for architecture
- Rely on Electron's bundled proprietary codecs — **do not** assume vanilla-Chromium codec behavior. This is a concrete reason Electron beats an OS-webview stack (see Q3).
- Build frame-stepping on **rVFC `mediaTime`**, not `1/fps`, to survive VFR webm.
- Keep an optional **ffmpeg remux-to-CFR** escape hatch for pathological files; do not make it a hard pipeline step.
- Track and display fps/`mediaTime` for frame-accurate charting.

---

## Q3 — Showstoppers

**Verdict: GREEN — no RED showstopper.** Everything found is a known, documented cost, not a blocker.

- **Native module build/signing:** `node-hid` needs an Electron-ABI rebuild (`@electron/rebuild` postinstall). Prebuilt binaries exist; toolchain fallback is `xcode-select --install`. Routine.
- **HID entitlements / notarization:** direct-distribution **Hardened Runtime** app needs **no USB/HID entitlement**; standard codesign + notarize flow applies. Runtime **Input Monitoring** TCC prompt is expected. USB entitlement + sandbox pain only arise if targeting the Mac App Store — avoid MAS.
- **Codec licensing:** technical support is free (bundled); H.264 patent licensing is a *distribution* concern, immaterial for a personal/internal tool.
- **Operational:** must quit the Elgato Stream Deck app to release the deck — a UX note, not an engineering blocker.

**Strongest alternative if Electron were rejected:** **Tauri (Rust + `hidapi`)** — smaller binary, no bundled Chromium. **But** Tauri renders in the **OS webview (WKWebView on macOS)**, and **WKWebView does not decode VP9/WebM** — it would force transcoding every webm match to H.264, reintroducing exactly the pipeline Electron avoids. Native **Swift/AVKit** has the same VP9 gap (AVFoundation has no VP9). **Electron is the lower-risk choice specifically because it bundles both H.264 and VP9**, so no alternative is recommended.

---

## Summary verdicts

| Q | Topic | Verdict |
|---|-------|---------|
| 1 | Stream Deck XL control in Electron | **GREEN** |
| 2 | Local video frame-step in Electron | **GREEN** |
| 3 | Showstoppers | **GREEN** (none RED) |

No RED showstoppers found. Electron is technically feasible for the planned stack; the codec-bundling advantage actively favors Electron over Tauri/native for VP9/webm tennis footage.
