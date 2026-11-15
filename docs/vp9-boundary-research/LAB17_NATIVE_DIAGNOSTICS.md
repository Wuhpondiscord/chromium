# Lab 17: native D3D11 texture and DecoderStream fallback diagnostics

**Status:** Source instrumentation pushed; not compiled or executed in a Windows Chromium build. No native Lab 17 results or fixes are claimed.

## Lab 17 actual progress — stock Chrome alternative

**As of 2026-11-15, WinDbg successfully intercepted `d3d11!CDevice::CreateTexture2D` in stock Chrome.** The observed descriptor was **8×8, `DXGI_FORMAT_R8_UNORM` (61)**, not an odd NV12 texture; the single captured call's HRESULT was not traced. The instrumented source modifications described below are committed but **have never been compiled or executed**. The stock-Chrome method does not use this instrumented branch.

- [WinDbg evidence and exact breakpoint/descriptor transcript](LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md)
- For future targeted stock-Chrome debugging, condition the breakpoint on `DXGI_FORMAT_NV12` (103) and then inspect `960×129` before claiming the parity root cause.
- For HTML fallback, retain `chrome://media-internals` and software-only controls. Direct inspection of `decoder_produced_a_frame_` and the branch still requires adequate symbols or an instrumented build.

### Stock Chrome results now obtained

- A native `960×720` `DXGI_FORMAT_NV12` texture request was captured, and the **same call returned `S_OK`**.
- A later native logpoint captured **1,050 NV12 requests** (75 heights from 720 down to 128, each observed 14 times), all width 960; **no 960×120 or 960×129 NV12 request** during that original-WebM logging run.
- Two equivalent uploaded log exports were cross-checked; results are one test, **not independently replicated runs**.
- The optional source instrumentation described below **still has not been compiled or tested**.
- Full evidence: [Lab 17 NV12 allocation trace](LAB17_NV12_DIMENSION_TRACE.md).

## Files instrumented

- media/gpu/windows/d3d_decoder_configurator.cc: LAB17_D3D11_TEXTURE_REQUEST and LAB17_D3D11_TEXTURE_RESULT around ID3D11Device::CreateTexture2D. Records caller size, actual texture descriptor width/height/format, array size, mip levels, binding/miscellaneous flags, usage, CPU access, sample properties, shared-handle mode and HRESULT.
- media/filters/decoder_stream.cc: LAB17_DECODER_BEGIN_SELECTION, LAB17_DECODER_RESUME_SELECTION, LAB17_DECODER_REINITIALIZE, LAB17_DECODER_FIRST_OUTPUT, LAB17_DECODER_STATUS and LAB17_DECODER_FALLBACK_DECISION. Logs decision-state metadata, not frame data.
- All new records use VLOG(1); logging is opt-in via per-module verbose logging. No change to decoding or fallback logic.

## Build prerequisites

These patches require a full Chromium Windows checkout, depot_tools, installed Visual Studio and Windows SDK, GN/Ninja, synced dependencies, and build time/storage. Stock installed Chrome will NOT emit LAB17 markers.

From an appropriately prepared Windows checkout, with origin pointing to this fork:

~~~powershell
git fetch origin research/lab17-vp9-native-diagnostics
git checkout research/lab17-vp9-native-diagnostics
gn gen out\Lab17 --args="is_debug=false symbol_level=1"
autoninja -C out\Lab17 chrome
~~~

The commands are examples, not a claim that a Windows build succeeded here. Check Chromium's official Windows build instructions for workstation setup if needed.

## Launch a separate instrumented profile

~~~powershell
$chrome = (Resolve-Path '.\out\Lab17\chrome.exe').Path
$profile = Join-Path $env:TEMP 'chromium-vp9-lab17'
$log = Join-Path $env:TEMP 'chromium-vp9-lab17.log'
& $chrome --user-data-dir="$profile" --no-first-run --enable-logging --log-file="$log" --vmodule="d3d_decoder_configurator=1,decoder_stream=1" 'https://discord.com'
~~~

Depending on Chromium version and subprocess logging configuration, also inspect stderr and the test profile's chrome_debug.log if the specified log file is empty. Confirm verbose flags are active in the GPU process; an empty file is not evidence of success.

## Run sequence (bounded, sequential)

1. Capture a baseline fresh 960x128 VP9 hardware test and matching software control.
2. Run fresh hardware cases 960x129 and 960x130. Add 959x128, 961x128, and 962x128 for width parity.
3. Run original 960x128 -> 960x120 and synthetic 960x128 -> 960x129 with hardware; then software controls.
4. Run separate ordinary HTML playback of the original WebM; do not run WebCodecs simultaneously. Observe RendererImpl in chrome://media-internals.
5. Maintain the PowerShell GPU PID/crash-counter monitor; correlate every test using time and native logs.
6. Collect: Lab16-style JSON, media-internals export, Lab17 native logfile, GPU process monitor data. Redact personal browsing/account identifiers.

## Extract the diagnostic events

~~~powershell
$log = Join-Path $env:TEMP 'chromium-vp9-lab17.log'
Select-String -Path $log -Pattern 'LAB17_|GetVideoDecoderConfigCount|D3DVideoDecoderWrapper' |
  ForEach-Object { $_.Line } |
  Set-Content (Join-Path $env:TEMP 'lab17-events.txt')
~~~

## Hypothesis checks

### A: odd-coded dimensions

Prior Lab16 prediction: 960x129, 960x131, 959x128 and 961x128 return E_INVALIDARG; even neighbors succeed. Check LAB17_D3D11_TEXTURE_REQUEST's actual descriptor, especially DXGI_FORMAT_NV12 (DXGI enum 103), Width and Height. Microsoft documents that NV12 surface width and height must be even. If the recorded D3D texture descriptor is already even, investigate other invalid parameters instead of assuming an odd backing allocation. A padding prototype must separately validate downstream video decoder output and cropping, not merely CreateTexture2D.

### B: in-band resize fallback

Prior Lab16 prediction: an HTML player has already emitted video before its 960x120 failure. Inspect FIRST_OUTPUT, STATUS and FALLBACK_DECISION. On the observed DecoderStream error branch, fallback is feature-gated AND requires decoder_produced_a_frame_ to be false. A config-change reinitialization path separately exists, so do not claim Chromium has no mid-stream decoder re-selection.

## Evidence gates

- Prove the instrumented binary is actually running and the GPU selected adapter matches the earlier NVIDIA LUID {0,98569}.
- Preserve a successful even-size request/result and an odd-size request/result with failing HRESULT.
- Establish fallback-feature setting, first output, error, and eligible decision on normal HTML playback.
- Separate process-level restart evidence from handled decoder errors. Lab16 monitoring observed no full GPU PID change during the test.
- Do not claim a confirmed security vulnerability, native crash, or usable fix from observational logging.

## Previous documentation

See README.md, ISSUE_A_ODD_NV12.md, ISSUE_B_RECONFIG_FALLBACK.md, and EXPERIMENTS_AND_LAB17.md in this folder.