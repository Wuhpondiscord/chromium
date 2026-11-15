# Lab 18 — Two-frame VP9 transition and native D3D11 configuration-count comparison

**Date:** 2026-11-15 (America/Detroit)  
**Status:** native API configuration-count boundary **confirmed**; Chromium recovery policy and security implications **unresolved**.  
**Environment:** stock Chrome 157.0.8103.79, Windows 11, NVIDIA RTX 4080 SUPER, D3D11 hardware VP9 Profile 0 decoding. The process was attached in WinDbg using public D3D11 symbols.

## Reproducer and control cases

A Gradio/FFmpeg generator extracted original, encoded VP9 packets without re-encoding from the original descending-resolution WebM (previously recorded source SHA-256: `78589f127fe4436f71003f6652ade3b99a85a7691c7d4d1d70aed9f9b3666e10`). The generator validates frame dimensions with FFprobe.

| File | Expected encoded frames | User-observed playback |
|---|---|---|
| `test_A_960x128_only.webm` | One 960×128 keyframe | No equivalent error observed |
| `test_B_128_to_120.webm` | 960×128 keyframe then 960×120 keyframe | Reproduced `PIPELINE_ERROR_DECODE` |
| `test_C_960x120_only.webm` | One 960×120 keyframe | No equivalent error observed |

**Caution:** Test A/C's absence of an observed error does *not* establish that they selected hardware decoding. Capture their `kVideoDecoderName` before using them as hardware controls. Extracts were remuxed, which can change track metadata, timestamp normalization or initialization. Two-frame playback is not necessarily equivalent to a long source.

For Test B, `chrome://media-internals` reported a 936-byte (`0x3a8`) file, duration 0.066 s, `RendererImpl`, `D3D11VideoDecoder`, `kIsPlatformVideoDecoder=true`, VP9 profile 0, D3D11 backend, NV12 output and shared handles. The *container's initial track metadata* advertised 960×720 despite the extracted in-band frames being 960×128 and 960×120. The hardware decoder first reported an in-band configuration change to 960×128, then to 960×120, then:

```text
D3DVideoDecoder config change: ... coded_size: 960x128 ...
D3DVideoDecoder config change: ... coded_size: 960x120 ...
GetVideoDecoderConfigCount failed: The operation completed successfully. (0x0)
D3DVideoDecoder: D3DVideoDecoderWrapper is not created
D3DStatus::10: D3DVideoDecoderWrapper is not created
DecoderStatus::1
PipelineStatus::PIPELINE_ERROR_DECODE
```

The media-internals player displayed final `kResolution=960x128` even though its decoder log recorded the attempted 960×120 reconfiguration. The complete media-internals export was supplied in the research conversation as `Pasted text(20261115-211411).txt`; this report transcribes relevant fields rather than committing the entire export.

## Native WinDbg capture — same D3D11 function, same decoder GUID, different heights

WinDbg resolved `d3d11!CDevice::GetVideoDecoderConfigCount` at **`00007ff9\`0c008c50`** for this session. This address is *process/session-specific*, not stable across launches or versions. The Windows x64 COM method arguments at function entry were `RCX=this`, `RDX=D3D11_VIDEO_DECODER_DESC*`, `R8=UINT* pCount`. Descriptor DWORD offsets: `+0x10` width, `+0x14` height, `+0x18` output format. NV12 is `0x67` (103).

Both captures showed decoder GUID bytes (four DWORDs): `463707f8 4585a1d0 aa836d87 9eb8606d`. Both returned to the same caller location: `chrome!ChromeMain+0x26c84a9`, at `test eax,eax`.

| Native field | 960×128 control | 960×120 failure |
|---|---|---|
| Function | `d3d11!CDevice::GetVideoDecoderConfigCount` | same |
| GUID DWORDs | `463707f8 4585a1d0 aa836d87 9eb8606d` | same |
| Width | `0x3c0` = 960 | `0x3c0` = 960 |
| Height | `0x80` = 128 | `0x78` = 120 |
| Format | `0x67` = NV12 | same |
| Post-`gu` `RAX` | `0x00000000` = S_OK | `0x00000000` = S_OK |
| Post-`gu` `*pCount` | **2** | **0** |

### 960×120 native transcript (selected lines)

```text
LAB18_CONFIG_120
d3d11!CDevice::GetVideoDecoderConfigCount:
00007ff9`0c008c50 4053 push rbx
00002104`000c9384  463707f8 4585a1d0 aa836d87 9eb8606d
00002104`000c9394  000003c0 00000078 00000067
r8=0000000f661fd26c
0:000> r $t0 = @r8
0:000> dd @$t0 L1
0000000f`661fd26c  00000000
0:000> gu
chrome!ChromeMain+0x26c84a9: test eax,eax
0:000> r rax
rax=0000000000000000
0:000> dd @$t0 L1
0000000f`661fd26c  00000000
```

### 960×128 native transcript (selected lines)

```text
LAB18_CONFIG_128
d3d11!CDevice::GetVideoDecoderConfigCount:
00007ff9`0c008c50 4053 push rbx
00002104`00482de4  463707f8 4585a1d0 aa836d87 9eb8606d
00002104`00482df4  000003c0 00000080 00000067
r8=0000000f661fd2fc
0:000> r $t0 = @r8
0:000> gu
chrome!ChromeMain+0x26c84a9: test eax,eax
0:000> r rax
rax=0000000000000000
0:000> dd @$t0 L1
0000000f`661fd2fc  00000002
```

**Interpretation:** the native call succeeds in both cases, but reports two decoder configurations for 960×128 and zero for 960×120 under the same VP9 decoder GUID/NV12 format. Thus the media log `GetVideoDecoderConfigCount failed: ... (0x0)` reflects *no usable configurations with a successful HRESULT*, not a failed COM call. This explains the immediate D3D11 decoder-wrapper initialization failure. It does **not** prove the driver violated its contract: zero supported configurations can be the correct result for an unsupported resolution.

## Debugger stability note

During earlier broad `CreateTexture2D` breakpoint activity, the Chrome window became visually stale/transparent and unresponsive while WinDbg itself continued accepting commands. After `bc *` followed by `g`, Chrome became responsive again. This is consistent with debugger-induced suspension or breakpoint overhead; **not a confirmed Chrome process crash, GPU driver crash, or exploit**. Output such as `dd @rdx` after `gu` at an unrelated Chrome caller was not a valid texture descriptor read because `RDX` is volatile.

## Reproducible WinDbg procedure (session addresses must be re-resolved)

To capture 120px:

```text
bc *
bp d3d11!CDevice::GetVideoDecoderConfigCount ".if (dwo(@rdx+0x14) == 0x78) { .echo LAB18_CONFIG_120; r; dd @rdx L7; kb 8 } .else { gc }"
bl
g
```

When it stops:

```text
r $t0 = @r8
dd @$t0 L1
gu
r rax
dd @$t0 L1
```

Repeat in a fresh playback with `0x80` in place of `0x78` for the 128px control. Use `x d3d11!*GetVideoDecoderConfigCount*` to resolve the exact entry point for the currently running process.

## What remains open

1. Confirm A/C decoder selection in ordinary Chrome, including whether 960×120 alone uses software fallback.
2. Examine whether changing the WebM track's initially advertised 960×720 dimensions affects decoder selection and transition failure.
3. Trace `DecoderStream` post-output fallback and demuxer configuration changes using a symbolized/instrumented build; a COM count of zero alone does not prove a Chromium recovery-policy bug.
4. Reproduce without WinDbg, test another GPU/driver/browser build, and collect media-internals evidence.
5. Keep Finding A (odd-size `E_INVALIDARG` from texture path) separate from Finding B (zero decoder configurations on reinitialization).

**Security classification:** functional media decode / hardware fallback compatibility behavior. No process crash, memory corruption, UAF, sandbox escape, or RCE demonstrated.
