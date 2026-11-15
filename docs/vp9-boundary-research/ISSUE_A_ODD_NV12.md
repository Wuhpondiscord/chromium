# Issue A: odd VP9 coded dimensions reach NVIDIA D3D11 and fail native NV12 surface allocation

**Status:** reproduced; root cause strongly indicated by Microsoft format contract and Chromium source, final validation/padding test outstanding.  
**Test date:** 2026-11-15.  
**Impact demonstrated:** VP9 decode error, zero frame output on odd-size cases; no GPU-process restart seen.  
**Test environment:** Chrome 157.0.8103.79, Windows 11 22631.4317, RTX 4080 SUPER / NVIDIA driver 32.0.16.1088, adapter LUID `{0,98569}`.

## 1. Problem statement

A standards-capable software VP9 implementation decodes odd-width and odd-height profile-0 streams. However, Chromium selects `D3D11VideoDecoder` with `hardwareAcceleration: "prefer-hardware"` for the same odd coded dimensions. Its initial WebCodecs support check returns `true`, yet first-frame decoding fails with `EncodingError: Decoding error.`. The media-internals native cause points to `D3DStatus::20`, `HRESULT 0x80070057` at a D3D11 output-texture allocation call.

This is **not** a claim that D3D11 must support odd-dimension NV12 textures. Microsoft documents that `DXGI_FORMAT_NV12` requires **even width and height**. It is a question of why valid VP9 source dimensions lead to a late unsupported D3D11 surface request rather than safe padding or earlier software selection.

## 2. Controlled Lab 16 matrix (synthetic VP9 keyframes)

Lab 16 used `VideoEncoder` with `codec: vp09.00.10.08`, software preference, realtime, 800,000 bps, 30 FPS, one keyframe generated from an 8-band canvas for each size. All eight source encodes completed, then independent `VideoDecoder` instances decoded one keyframe each using both preferences sequentially.

| Source coded size | Odd axis | `isConfigSupported` hardware | Hardware output | Native outcome | Software output |
|---|---|---:|---|---|---|
| 960×128 | none | true | NV12 960×128 | success | I420 success |
| 960×129 | height | true | none | `0x80070057` | I420 success |
| 960×130 | none | true | NV12 960×130 | success | I420 success |
| 960×131 | height | true | none | `0x80070057` | I420 success |
| 960×132 | none | true | NV12 960×132 | success | I420 success |
| 959×128 | width | true | none | `0x80070057` | I420 success |
| 961×128 | width | true | none | `0x80070057` | I420 success |
| 962×128 | none | true | NV12 962×128 | success | I420 success |

**Important controls:** software `VpxVideoDecoder` was selected (`kIsPlatformVideoDecoder:false`) and decoded every case, including odd dimensions. `D3D11VideoDecoder` was independently established for hardware cases (`kIsPlatformVideoDecoder:true`, `Selected D3DVideoDecoder adapter LUID:{0,98569}`, `D3DVideoDecoder producing NV12`).

The hardware failures are not merely capability-probe rejections: the probes returned supported and Chromium instantiated the platform decoder before its decode-time failure.

## 3. Exact relevant native error and source path

Representative `chrome://media-internals` errors for 960×129, 960×131, 959×128, and 961×128:

```text
D3DVideoDecoder is using vp9 profile0 / 4:2:0
D3DVideoDecoder is using D3D11 backend
D3DVideoDecoder producing NV12
D3DVideoDecoder: Selected NV12
D3DVideoDecoder config change: coded_size: 960x129
D3DStatus::20
HRESULT 80070057: The parameter is incorrect. (0x80070057)
media/gpu/windows/d3d_decoder_configurator.cc:199
media/gpu/windows/d3d11_video_decoder_backend.cc:335
media/gpu/windows/d3d_video_decoder.cc:772
DecoderStatus::1
media/gpu/windows/d3d_video_decoder.cc:890
```

At `media/gpu/windows/d3d_decoder_configurator.cc`, the repository's `D3DDecoderConfigurator::CreateD3D11OutputTexture` writes:

```cpp
output_texture_desc_.Width = size.width();
output_texture_desc_.Height = size.height();
output_texture_desc_.ArraySize = array_size;
...
HRESULT hr =
    device->CreateTexture2D(&output_texture_desc_, nullptr, &texture);
if (FAILED(hr)) {
  return base::unexpected(
      D3DStatus{D3DStatus::Codes::kCreateDecoderOutputTextureFailed, hr});
}
```

The same code sets shared-resource flags when `use_shared_handle` is true. The logs confirm `single textures`, `shared handle`, `NV12`, and `binding textures`, but **do not directly dump the full descriptor**, so a precise statement that `Width=959` or `Height=129` reached the D3D API remains an instrumentation target.

**Microsoft reference:** `DXGI_FORMAT_NV12` is 4:2:0 and its documentation states “Width and height must be even.”  
https://learn.microsoft.com/en-us/windows/win32/api/dxgiformat/ne-dxgiformat-dxgi_format

## 4. Leading causal model

1. A valid odd-coded VP9 frame passes Chromium's coarse profile/size support and hardware preference.
2. Chromium selects NVIDIA D3D11 VP9 profile-0 decode.
3. The frame triggers D3D11 decoder configuration, with NV12 as output format and shared single-texture allocation.
4. Texture allocation returns `E_INVALIDARG` under the odd-size inputs (all four tested odd-axis cases).
5. Chromium propagates a typed `D3DStatus::20` to `DecoderStatus::1`; WebCodecs reports `EncodingError`, closes the decoder and returns no frame.

**Inference:** the output allocation size is probably being passed without the even-dimension storage normalization mandated for NV12. This should be tested, not assumed proven. The actual D3D11 decoder descriptor, texture descriptor, and whether padded storage is accepted by the decoder must be inspected.

## 5. Hypotheses and alternatives

- **H1 — invalid odd NV12 backing texture (leading):** dimensions are unpadded; the D3D API rejects them. *Evidence:* perfect parity pattern, Microsoft format constraints, source allocation path and HRESULT.
- **H2 — other invalid descriptor field derived from dimensions:** perhaps texture flags, array size, view constraints, or a decoder-specific alignment requirement; identical HRESULT can arise from several invalid arguments.
- **H3 — NVIDIA-specific decoder constraint beyond NV12:** some decoder profiles could reject dimensions even if separately padded textures are valid. Needs native end-to-end trial.
- **H4 — synthetic-bitstream defect:** disfavored because software decoder succeeds on each sample and adjacent even sizes succeed on hardware; still worth using independent bitstream fixtures in regression tests.

## 6. Reportability / expected-versus-actual

**Expected user-visible behavior:** valid VP9 profile-0 odd-coded video should decode using a compatible backing representation or suitable software decoder when the platform pathway cannot handle its size. A support probe should not give the misleading impression that hardware use will definitely succeed; `isConfigSupported` only promises support for a configuration, not successful future bitstream decode, so **the mere support=true value is not automatically a spec violation**.

**Observed:** native hardware decoder selected; D3D11 output allocation fails for odd-coded sizes; software-only decoder succeeds.

**Candidate categories:** functional codec support, dimension-validation and output-surface padding, platform-decoder error handling. No crash or memory corruption demonstrated.

## 7. Lab 17 focused native verification

Instrument the *existing branch* without altering the Microsoft/NVIDIA API implementation:

1. Immediately before `CreateTexture2D`, record requested VP9 coded size, actual `size`, `output_texture_desc_.Width/Height/Format/ArraySize/MipLevels/BindFlags/MiscFlags/Usage/CPUAccessFlags/SampleDesc`, driver adapter LUID, shared-handle mode, and HRESULT. Avoid logging frame contents or unrelated browsing data.
2. Compare **fresh** 960×128, 960×129, 960×130, 959×128, 961×128, 962×128; keep one decoder alive at a time and preserve software controls.
3. In a purpose-built native test, investigate even backing dimensions `ceil(width/2)*2`, `ceil(height/2)*2` while preserving true coded/visible geometry; do not just rewrite the VP9 bitstream's header to claim an even coded size.
4. Check whether `CreateVideoDecoder`, texture views, resource-sharing, and first-frame decode still succeed with padding; texture creation alone is not an end-to-end fix.
5. Test an AMD hardware device independently and a second NVIDIA driver version before generalizing vendor impact.
6. Convert findings to a small Chromium regression test once a valid and safe approach is known.

**Success criterion:** odd-sized valid VP9 keyframes either decode correctly or consistently select a compatible path without a late D3D invalid-argument failure. **Failure criterion:** backing texture creation succeeds but downstream decoder cannot use the padded resource, in which case an explicit pre-selection/software path may be preferable.

## Lab 17 status — native odd-size request still unverified

Stock-Chrome WinDbg can now intercept the D3D11 texture descriptor and returned HRESULT. A **960×720 NV12 request returned `S_OK`**, while an earlier **8×8 R8_UNORM** request was unrelated to VP9 output. In a later original-WebM playback logging run, **1,050 NV12 requests** all had even heights from 720 to 128 and width 960; **none had height 129**. This run did **not** explicitly execute the *synthetic* 960×129 test from Lab 16, so it does not refute its reproducible hardware-path failure or establish the invalid-descriptor hypothesis. **The failing `960×129/NV12` entry plus same-call `E_INVALIDARG` return have not been obtained.** See [Lab 17 full trace](LAB17_NV12_DIMENSION_TRACE.md).

## 8. Reproduction / evidence provenance

- Original WebM: `funkyms_4.webm`, SHA-256 `78589f127fe4436f71003f6652ade3b99a85a7691c7d4d1d70aed9f9b3666e10`.
- Lab 15: 42 completed cases; 21 software-preferred successes; key hardware fresh 960×129 failure independently observed.
- Lab 16: `funkyms-vp9-boundary-v16.json`, 20 completed / 14 success / 6 hardware decode errors / zero unsupported; started 2026-11-15 18:14:45.668 UTC, ended seconds later. Four synthetic odd-dimension fresh cases reproduced.
- Raw `chrome://media-internals` export: player IDs 934 (129 height), 936 (131 height), 938 (959 width), 939 (961 width).
- Lab 16 sample outputs: 960×129 was 1573 encoded bytes; 960×131 was 1642; 959×128 was 1850; 961×128 was 2026.

Raw user-provided diagnostic exports were reviewed during report preparation but have **not been committed as binaries or personal browsing logs**; retain them in the secure test archive and redact unrelated page/account metadata before external reporting.
