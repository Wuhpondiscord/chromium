# Experiments, evidence custody, reproducibility, and Lab 17 verification plan

Research date: 2026-11-15. This document captures **what was run**, **what it demonstrates**, and **what it cannot demonstrate**, preserving the distinction between JavaScript, media-internals, OS-process monitoring, and hypothetical native tests.

## A. Test system and controlled conditions

- Chrome: `157.0.8103.79` (user agent reported `Chrome/157.0.0.0`).
- Windows: `Windows NT 10.0.22631.4317` (Windows 11).
- Active adapter: `NVIDIA GeForce RTX 4080 SUPER`, vendor `0x10de`, device `0x2702`, driver `32.0.16.1088`, adapter LUID `{0,98569}`.
- Secondary adapter listed but **not selected for the logged tests**: AMD Radeon(TM) Graphics, vendor `0x1002`, device `0x13c0`.
- Hardware decoder: `D3D11VideoDecoder`, `kIsPlatformVideoDecoder:true`, NV12 single-texture output with shared handle.
- Software decoder: `VpxVideoDecoder`, `kIsPlatformVideoDecoder:false`, I420 output; software `codedWidth` often rounded up because of internal stride/alignment, so compare **display/visible dimensions** when assessing decoded geometry.
- Reproduction invoked from Discord page via DevTools JavaScript; tests used WebCodecs directly. A separate ordinary Discord `<video>` player was captured in media-internals.

## B. Original source and keyframe provenance

- Video source: `funkyms_4.webm` (Discord CDN attachment).
- File SHA-256: `78589f127fe4436f71003f6652ade3b99a85a7691c7d4d1d70aed9f9b3666e10`.
- File length: `31,228` bytes.
- Parsed WebM blocks: `211`.
- Relevant original keyframes from prior analysis:
  - Block 74, coded `960×128`, payload `175` bytes, SHA-256 `1b958cd581b3f803189a4097a41a5d88aaca5bd6168af6d1b78cc762077960f0`.
  - Block 75, coded `960×120`, payload `227` bytes, SHA-256 `c4006298875eafbcf269b67190a3bcd1c1a259be1fc929478c1cfa36903bdefe`.
- Lab 16 downloaded the original and checked whole-file SHA-256 before extracting these block payloads. Do not rely only on block numbers if a different file version is used.

## C. Lab 15 — broader boundary comparison

Previously completed `funkyms-vp9-vendor-comparison-v15.json`:

- 42/42 tests completed, no timeout cases.
- 21 software-preferred tests succeeded.
- Hardware fresh sizes below 128px height (112,119–127, and original 120) unsupported/rejected.
- Fresh 128 hardware decoded; 129 reported supported but failed; 130,136,144 decoded.
- Original 128→120 transition failed hardware and succeeded software.
- `prefer-hardware` alone is not a vendor probe. Hardware identity was verified independently from native logs and active adapter LUID.

## D. Lab 16 — fine odd/even boundaries

**Run start:** `2026-11-15T18:14:45.668Z`; all eight synthetic encodes succeeded, 20 completed decoding tests, 14 successes, 6 decode errors, 0 unsupported, no harness fatal error.

All **ten** software-preferred tests succeeded; all six errors were **hardware**:
1. Fresh `960×129` — odd height.
2. Fresh `960×131` — odd height.
3. Fresh `959×128` — odd width.
4. Fresh `961×128` — odd width.
5. Original keyframe transition `960×128→960×120`.
6. Synthetic transition `960×128→960×129`.

Hardware success cases: `960×128`, `960×130`, `960×132`, `962×128`.

Native media-internals entries include:
- Player 934: 960×129, `D3DStatus::20`, HRESULT `0x80070057`.
- Player 936: 960×131, same error.
- Player 938: 959×128, same error.
- Player 939: 961×128, same error.
- Player 941: original 128→120, `GetVideoDecoderConfigCount ... (0x0)` and `D3DStatus::10`.
- Player 942: synthetic 128→129, `D3DStatus::20`, `0x80070057`.
- Players 943–952: software test instances.

**Frame hashes** from Lab 16 are simple FNV-1a readback fingerprints and should not be confused with SHA-256 of raw VP9 payloads or cryptographic security attestations. NV12 and I420 bytes/hashes cannot be compared directly without normalizing color format, visible rectangle, planes, and color-space conversion.

## E. GPU stability / monitoring

A PowerShell `Get-CimInstance Win32_Process` sampler recorded approximately once per second from `2026-11-15T13:11:02-05:00` through `13:15:22-05:00` (245 samples). During the monitored Lab 16 window, GPU PID `48160` and command-line crash counter `1` remained unchanged. This excludes an **observed full GPU-process restart at the monitor's sampling granularity**; it does **not** rule out shorter transient GPU device loss, internally recovered driver problems, or thread-level faults.

An earlier WinDbg run coincided with a PID transition `45700→44612` and crash-counter `1→2`; debugger attach/pause/detach and other intervening browser activity confound causal attribution. `chrome://crashes` had no recent matching visible crash report. **Do not call this a confirmed exploit or driver crash.**

## F. Normal HTML playback

A separate `RendererImpl` media-internals player selected hardware `D3D11VideoDecoder`, initialized with high-resolution 960×720 video, handled multiple in-band height changes, then ultimately logged 960×128→960×120, `GetVideoDecoderConfigCount failed ... (0x0)`, wrapper creation failure, and `PipelineStatus::PIPELINE_ERROR_DECODE`.

The player UI showed last-known `960×192`; that is **not** the failing decoder coded size. The key unanswered question is whether a software fallback is *expected or feasible* after previous successfully output frames in the exact pipeline.

## G. Source inspection checkpoints

- `media/gpu/windows/d3d_decoder_configurator.cc`: texture descriptor width/height copied from caller `gfx::Size size`; failure at `device->CreateTexture2D`; typed `kCreateDecoderOutputTextureFailed`.
- `media/gpu/windows/d3d11_video_decoder_backend.cc`: creates output texture in single-texture path.
- `media/gpu/windows/supported_profile_helpers.cc`: NVIDIA minimum VP9 size `128×128`; initial coarse supported-profile/format/range behavior is distinct from a complete decoder+texture allocation guarantee.
- `media/filters/decoder_stream.cc`: genuine reinitialization and selection support; initial-error fallback gated by `!decoder_produced_a_frame_`; post-output decode errors can become terminal.
- Microsoft DXGI NV12 documentation: even texture dimensions required.

## H. Lab 17 instrumentation checklist (SOURCE IMPLEMENTED; NOT COMPILED OR EXECUTED)

The optional native tracing patch was committed on `research/lab17-vp9-native-diagnostics`, but there is no compiled browser build or native trace from that patch. Separately, stock Chrome WinDbg interception **did** succeed and captured an **8×8 R8_UNORM** texture; see [Lab 17 WinDbg evidence](LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md). The `960×129` NV12 descriptor, HRESULT return, and fallback gate trace remain pending.

**Native texture record** (at `CreateTexture2D`):
- Coded VP9 size, visible rect, supplied texture-size argument, `D3D11_TEXTURE2D_DESC` full fields, adapter identity, return HRESULT.
- Record only metadata; avoid storing decoded video, URL tokens, Discord messages, or credentials.
- Run parity controls `960×128/129/130/131/132` and `959/961/962×128`; verify logs correlate with known JS case IDs.
- After validating caller-versus-descriptor equality, test properly aligned output backing storage **only in a dedicated local build/test**; validate first-frame decode and visible crop, not just allocation.

**Fallback record** (HTML pipeline):
- Trace `OnDecoderOutputReady`, `decoder_produced_a_frame_`, `OnDecodeDone` status, `kFallbackAfterDecodeError`, config-change callbacks, selector candidate set and reinitialize attempts.
- Establish when first frame was delivered relative to 120px error.
- Check whether decoding with software *from an appropriate random-access point* can recover the entire original stream, not merely two isolated keyframes.
- Do not substitute WebCodecs error behavior for `DecoderStream` semantics.

**Cross-vendor repeat:**
- Verify target device is truly AMD by observing LUID and decoder name for every test; presence in `chrome://gpu` is insufficient.
- Test another NVIDIA driver/release if possible, taking care that differences reflect decoder firmware and Chromium version.

**Classification gates before reporting:**
1. Reproduction on clean profile (no WinDbg, no concurrent harness).
2. Isolated minimal test case and reliable expected-vs-actual behavior.
3. Duplicate search / tracker verification; no fabricated issue-history claims.
4. Decide if it's an implementation bug, deliberate limitation, or a security issue based on actual impact.
5. For any future native crash or sanitizer report, preserve the stack/minidump and investigate separately; no present claim of such a crash.

### Lab 17 stock-Chrome debugger progress (subsequent to the initial report)

A stock-Chrome debugger returned **`RAX=0` (`S_OK`)** after stepping out of a native **960×720 NV12** texture allocation. Its later NV12-only logging run showed **1,050 calls**, **75** distinct heights from **720** to **128** in decrements of **8**, with **14 requests per height** and **no odd dimensions, 120, or 129** in that one session. The original descending-resolution WebM was played; the isolated synthetic 960×129 Lab 16 test has **not** been documented as run within this debugger capture. Thus this trace supports texture-boundary observation for Finding B but **does not establish** the odd-texture descriptor/root cause for Finding A. An earlier 8×8 R8_UNORM capture was unrelated to video. See [NV12 trace](LAB17_NV12_DIMENSION_TRACE.md) and [initial debugger capture](LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md). Neither the logs nor the manual WinDbg breaks establish GPU process crashes or memory unsafety.

## I. Evidence location and privacy

Source logs were supplied as exported JSON and text in the research conversation. This branch records **derived technical observations**, not raw browsing or user account metadata. Suggested local archive names: `funkyms-vp9-vendor-comparison-v15.json`, `funkyms-vp9-boundary-v16.json`, `chrome-gpu-monitor.csv`, and `media-internals-normal-playback.json`. These are *evidence labels*, not a claim that the raw files have been committed to GitHub.

## J. Reporting draft skeleton

**Title A:** Valid VP9 odd coded dimensions selected for NVIDIA D3D11 decode, then fail `CreateTexture2D` with `E_INVALIDARG` instead of decoding/falling back.  
**Title B:** VP9 in-band resize 960×128→960×120 ends HTML playback with `PIPELINE_ERROR_DECODE` after NVIDIA D3D11 decoder loses usable config.  
**Category:** media compatibility / decoder selection and recovery, security impact unknown and not demonstrated.  
**Expected:** valid VP9 remains decodable by a supported path or reports an appropriately scoped unsupported configuration.  
**Observed:** software success; hardware-path failure; normal video playback terminates at reconfiguration.  
**Repro:** hardware adapter identity, browser build, file hashes, JS control matrix, native error stacks, temporal logs.  
**Proposed next evidence:** the actual **odd NV12** texture descriptor plus matching return HRESULT and the live error branch condition in `DecoderStream`. The initial WinDbg capture was **8×8 R8_UNORM**, not VP9 NV12.
