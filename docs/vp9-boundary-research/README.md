# VP9 / NVIDIA D3D11 boundary investigation — research index

**Research date:** 2026-11-15 (America/Detroit)  
**Status:** reproducible compatibility and decoder-recovery failures; security impact **not demonstrated**.  
**Target tested:** Google Chrome 157.0.8103.79, Windows 11 build 22631.4317, NVIDIA RTX 4080 SUPER, NVIDIA 32.0.16.1088, active adapter LUID `{0,98569}`; integrated AMD GPU also installed but *not* shown to decode these test cases.  
**Current active branch:** `research/lab17-vp9-native-diagnostics` (inherits `research/vp9-boundary-findings-labs15-16` and `research/lab15-vp9-config-diagnostics`). This is research documentation and opt-in source instrumentation, not a production patch. The WinDbg investigation uses **unmodified stock Chrome**.

## Reports

1. [Issue A — VP9 odd dimensions accepted but D3D11 NV12 output texture creation fails](ISSUE_A_ODD_NV12.md)
2. [Issue B — decoder failure following mid-stream 960×128→960×120 transition](ISSUE_B_RECONFIG_FALLBACK.md)
3. [Experimental record, reproducibility, limitations, and Lab 17 plan](EXPERIMENTS_AND_LAB17.md)
4. [Lab 17 stock-Chrome WinDbg evidence — actual 8×8 R8 texture descriptor, not odd NV12](LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md)
5. [Lab 17 optional native instrumentation and execution runbook](LAB17_NATIVE_DIAGNOSTICS.md)
6. [Lab 17 NV12 allocation trace — 1,050 native calls, heights 720→128](LAB17_NV12_DIMENSION_TRACE.md)
7. [Lab 18 two-frame VP9 transition — native D3D11 configuration-count comparison (128px: 2, 120px: 0)](LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md)

## Lab 17 progress update — 2026-11-15, 19:27 local (America/Detroit)

**Confirmed newly:** WinDbg attached to stock Chrome, loaded public D3D11 symbols, resolved `d3d11!CDevice::CreateTexture2D`, installed a real breakpoint, and intercepted a native texture creation call. At the function's entry point, `RDX` pointed to a valid 44-byte `D3D11_TEXTURE2D_DESC` whose values were **Width=8, Height=8, Format=61 (`DXGI_FORMAT_R8_UNORM`)**, MipLevels=1, ArraySize=1, SampleCount=1, Usage=DEFAULT, BindFlags=8, CPUAccess=0, MiscFlags=0. The call was from Chrome's GPU process, but **no evidence ties this specific 8×8 allocation to VP9 decoding**.

**Still missing:** the odd-size `960×129` + NV12 (Format 103) descriptor and the **same call's return HRESULT**; the live `decoder_produced_a_frame_` state/branch selection during the separate HTML playback 960×128→960×120 error. No final root-cause proof, native crash or security exploit is claimed. The 8×8 capture *validates the interception technique*, not the odd NV12 hypothesis.

See [full WinDbg evidence](LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md). A third-party statement that the 8×8 descriptor was the final “smoking gun” is incorrect.

## New Lab 17 evidence — after the initial 8×8 capture

Stock Chrome WinDbg **successfully** intercepted a **960×720 NV12** texture request and, after `gu`, recorded **`RAX=0` (`S_OK`)** for that same call. This is a successful even-dimension control; **not the odd-size failing call**.

The subsequent all-NV12 logpoint recorded **1,050 `CreateTexture2D` entry events** across **75 distinct dimensions**, every one width **960**, format **103/NV12**, and heights **720, 712, 704, ..., 136, 128**; **exactly 14 entries per height**, with **zero odd dimensions**, **zero 960×120** and **zero 960×129** requests in that recorded session. Both uploaded versions of this **same** log yield identical aggregates. Logging ended after a **manual debugger break** and `.logclose`, not a proven process crash. These entries do not themselves establish success HRESULTs.

This trace is **consistent with** Finding B's known 960×128→960×120 failure boundary but does not prove why the 120 configuration was rejected or show fallback-state internals. It was the original WebM, **not** the isolated Lab 16 synthetic 960×129 test, so it cannot settle Finding A's descriptor parity hypothesis. See [the full trace report](LAB17_NV12_DIMENSION_TRACE.md). WinDbg's compound `&&` expressions caused parser errors; accepted nested conditions and the working logpoint are documented there.

## Executive assessment

Two distinct hardware-decoder failure modes are independently reproduced:

- **Odd-dimension NV12 surface failure:** for VP9 profile 0, `960×129`, `960×131`, `959×128`, and `961×128` are accepted by the WebCodecs configuration-support probe and selected for native D3D11 decode, but produce no frame and fail with `D3DStatus::20`, HRESULT `0x80070057` (`E_INVALIDARG`), rooted at `media/gpu/windows/d3d_decoder_configurator.cc:199`. Adjacent even-size controls decode successfully and generate NV12 output. Software VP9 decodes all eight synthetic sizes.
- **Mid-stream decoder reconfiguration failure:** the source WebM and a two-keyframe WebCodecs reproduction successfully decode a `960×128` keyframe, then fail as the coded height changes to `960×120`. Native logs report `GetVideoDecoderConfigCount failed: The operation completed successfully. (0x0)`, followed by `D3DVideoDecoderWrapper is not created` / `D3DStatus::10`. This represents a **no-usable-configuration** outcome despite the success HRESULT, distinct from `E_INVALIDARG` texture creation.
- **Ordinary HTML playback impact:** a separate normal-playback media-internals record selects `D3D11VideoDecoder`, follows numerous changing resolutions, ends in `PIPELINE_ERROR_DECODE` at the low-resolution transition, and shows no subsequent successful software-decoder selection in that player record.
- **Stability monitoring:** a roughly 4m20s PowerShell sampling interval covering Lab 16 observed GPU process PID `48160` and `--gpu-recent-crash-count=1` unchanged. A separate earlier PID/counter change during WinDbg use is confounded and is **not evidence of a video-triggered crash**.

## Evidence strength / scope

| Conclusion | Assessment |
|---|---|
| Reproducible odd-width/odd-height failure on the tested NVIDIA D3D11 path | **Confirmed** |
| `DXGI_FORMAT_NV12` requires even width and height | **Confirmed by Microsoft documentation** |
| Failure arises from `CreateTexture2D` at configurator source location | **Confirmed by code path + logged HRESULT; exact descriptor still to record** |
| Chromium's WebCodecs capability check does not prevent the tested odd-dimension late failures | **Confirmed on tested Chrome/hardware** |
| Regular HTML playback can fail after the observed in-band 128→120 change | **Confirmed on tested Chrome/hardware** |
| “Chromium has no decoder reselection” | **False as a general architectural claim**; see Issue B |
| NVIDIA kernel-mode driver crashed / GPU process was restarted by Lab 16 | **Not observed** |
| Memory corruption, cross-context leak, UAF, arbitrary code execution | **Not observed** |
| Same issue on AMD, other NVIDIA SKUs, other drivers, Firefox or other OSes | **Not tested** |
| Both behaviors are previously unreported Chromium bugs | **Not established; duplicate search/review needed** |

## Critical correction to third-party write-up

An online explanation claimed `DecoderSelector` **never** re-selects after initialization. Current source in this repository contradicts that blanket statement: `media/filters/decoder_stream.cc` has `ReinitializeDecoder()`, `BeginDecoderSelection()`, `ResumeDecoderSelection()`, and a feature-gated decode-error fallback. The actual relevant condition is `!decoder_produced_a_frame_ && kFallbackAfterDecodeError`; after a frame has been produced the standard **decode-error** recovery branch generally reports an error. A separate *demuxer-driven config-change* path explicitly reinitializes decoder selection. See [Issue B](ISSUE_B_RECONFIG_FALLBACK.md) for important distinctions.

## Security and disclosure classification

This research currently documents **functional media compatibility/robustness defects** and suggests possible validation/error-recovery improvements. It does **not** establish sandbox escape, process crash, memory-safety compromise, or severity. Do not present a fatal media-pipeline error as a native process crash. Before a public Chromium issue, check for duplicates and consider whether any future evidence merits Chromium's private security-reporting channel.

## External primary references

- [Microsoft DXGI_FORMAT: NV12 dimensions must be even](https://learn.microsoft.com/en-us/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format)
- [Chromium upstream DecoderStream source](https://chromium.googlesource.com/chromium/src/+/main/media/filters/decoder_stream.cc)
- [Chromium upstream media feature flags](https://chromium.googlesource.com/chromium/src/+/main/media/base/media_switches.cc)
- Repository source: `media/gpu/windows/d3d_decoder_configurator.cc`; `media/gpu/windows/supported_profile_helpers.cc`; `media/filters/decoder_stream.cc`.

**Source provenance note:** URLs to Chromium tracker issues `40684757` and `41453795`, cited in the external write-up, were not accessible with enough issue content to verify their subject, status, fix, or relationship to these findings. They are **not** used as proof of an established duplicate or vulnerability.

## Lab 18 update — native configuration count established (2026-11-15)

A two-frame 960×128→960×120 source-packet-copy WebM reproduces the normal HTML player's `PIPELINE_ERROR_DECODE` under the D3D11 hardware decoder. WinDbg directly captured `d3d11!CDevice::GetVideoDecoderConfigCount` returning **`S_OK` and count=2 at 960×128** versus **`S_OK` and count=0 at 960×120** with the same VP9 decoder GUID and NV12 format. This confirms the *immediate configuration-count rejection* but does not establish a GPU crash, a driver defect, or a requirement for Chromium to recover midstream. The two-frame extract initially advertises 960×720 at the container-track level, which remains a confounder; single-frame A/C hardware-decoder selection has not been established. See [Lab 18 detailed evidence](LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md).
