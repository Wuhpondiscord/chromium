# Lab 17 — Stock Chrome WinDbg first interception evidence (2026-11-15)

**Status: D3D11 function interception confirmed; VP9-specific native texture descriptor NOT yet captured.** This document records the *actual* debugger transcript supplied after Labs 15–16 and corrects an external claim that this initial 8×8 allocation proved the odd-height NV12 mismatch.

## Subsequent session update

This file records the **first** 8×8 R8_UNORM interception. Further experiments have now achieved a **960×720 NV12 request**, captured its **`S_OK`** return, and logged **1,050 NV12 request-entry events** along a complete 960×720→960×128 descending series. See [Lab 17 NV12 dimension trace](LAB17_NV12_DIMENSION_TRACE.md) for validated aggregates and exact limits. **No odd 960×129 NV12 descriptor or corresponding failing HRESULT has been captured.**

## Scope and chronology

- Tested installed Chrome: `C:\Program Files\Google\Chrome\Application\157.0.8103.79\chrome.exe`; Windows 11; NVIDIA user-mode graphics driver present.
- Two concurrent Chrome GPU process entries were initially observed (PIDs **48160** and **39516**). PID **39516** was suggested as the likely new isolated test GPU process, but the provided partial command-line information did **not** independently establish its profile association. No unwarranted assertion about the exact debugged PID is made.
- WinDbg attached with Windows x64 process context, paused at `ntdll!DbgBreakPoint`. The user reported Chrome not interactive until entering `g`. `*BUSY*` after `g` indicated **normal resumed execution**, not debugger failure.
- The original WebM had **not been opened during the first symbol-resolution/session-setup logs**; the later breakpoint triggered after continuing but no evidence correlates that specific hit to the VP9 input.
- Public Microsoft `d3d11.pdb` symbol resolution succeeded. Both `d3d11.dll` and NVIDIA `nvwgf2umx.dll` were present.
- Initial commands `bl` (empty), bare `d3d11!CDevice::CreateTexture2D` (unrecognized command), `r rcx rdx r8 r9` (syntax error), and `dd @rdx` at `ntdll!DbgBreakPoint` were **not** valid evidence of D3D11 descriptor data. These mistakes were corrected: `bp` was used and `r` printed registers after the breakpoint genuinely triggered.

## Validated symbol and breakpoint

```text
lm m d3d11
00007ff9`0c000000 00007ff9`0c257000 d3d11 (pdb symbols)

lm m nvwgf2umx
00007ff9`01200000 00007ff9`067a5000 nvwgf2umx (deferred)

x d3d11!*CreateTexture2D*
00007ff9`0c019610 d3d11!CDevice::CreateTexture2D (void)

0:094> bp d3d11!CDevice::CreateTexture2D
0:094> bl
0 e Disable Clear 00007ff9`0c019610 0001 (0001) 0:**** d3d11!CDevice::CreateTexture2D
0:094> g
Breakpoint 0 hit
d3d11!CDevice::CreateTexture2D:
00007ff9`0c019610 4055 push rbp
```

**Important:** address `00007ff9`0c019610` is specific to this process/load and should **not** be hardcoded in future sessions. Use the symbol to set breakpoints.

## Captured argument registers and descriptor

The breakpoint stopped at the entry point, so under the Windows x64 ABI `RDX` held the `D3D11_TEXTURE2D_DESC*` argument:

```text
0:000> r
rax=00007ff90c019610
rcx=0000018eb8fab430
rdx=0000000f661fb94c
r8=0000000000000000
r9=0000000f661fb7d0
rip=00007ff90c019610

0:000> dd @rdx Lb
0000000f`661fb94c  00000008 00000008 00000001 00000001
0000000f`661fb95c  0000003d 00000001 00000000 00000000
0000000f`661fb96c  00000008 00000000 00000000
```

For the standard `D3D11_TEXTURE2D_DESC`, the 11 DWORD fields decode to:

| Offset | Field | Hex | Interpreted value |
|---|---|---|---|
| +0x00 | Width | 0x8 | **8** |
| +0x04 | Height | 0x8 | **8** |
| +0x08 | MipLevels | 0x1 | 1 |
| +0x0C | ArraySize | 0x1 | 1 |
| +0x10 | Format | 0x3d | **61 — DXGI_FORMAT_R8_UNORM** |
| +0x14 | SampleDesc.Count | 0x1 | 1 |
| +0x18 | SampleDesc.Quality | 0 | 0 |
| +0x1C | Usage | 0 | D3D11_USAGE_DEFAULT |
| +0x20 | BindFlags | 0x8 | D3D11_BIND_SHADER_RESOURCE |
| +0x24 | CPUAccessFlags | 0 | 0 |
| +0x28 | MiscFlags | 0 | 0 |

**Interpretation:** this is an **8×8 R8_UNORM shader resource**. It is **not** a 960×129 NV12 texture, does **not** prove the VP9 output allocator issued a malformed descriptor, and was **not** accompanied by a captured HRESULT. The observed immediate caller was inside Chrome's large `chrome.dll`; public symbol names such as `ChromeMain+...`, `IsSandboxedProcess+...`, and `sqlite3_dbdata_init+...` are coarse/unreliable for identifying the real media-specific caller. Do not treat that call stack alone as proof of VP9 ownership.

## Exactly what has been verified

1. WinDbg can successfully attach to an active Chrome GPU process and load symbols for D3D11.
2. `d3d11!CDevice::CreateTexture2D` resolves and a software breakpoint at its actual entry point triggers automatically on a real call.
3. The x64 `RDX` descriptor pointer is readable and the supplied data has the expected 44-byte `D3D11_TEXTURE2D_DESC` layout.
4. The *single documented captured call* is 8×8 `DXGI_FORMAT_R8_UNORM`, showing our method works, **but not Finding 1's descriptor**.
5. No observed native method return status from this call was recorded.
6. No Lab 17 WinDbg capture of `decoder_produced_a_frame_` or fallback branch exists.
7. The Labs 15–16 hardware-vs-software findings remain independently reproducible, but their root-cause verification remains outstanding.

## Correcting external interpretation

A separate commentary described this breakpoint as the “smoking gun,” claimed that it showed Chromium passing odd VP9 dimensions to the NVIDIA driver, and recommended stopping debugging. **That conclusion does not follow from the actual capture.** The DWORD values `00000008 00000008 ... 0000003d` correspond to 8×8 R8_UNORM, and the 960×129 NV12 request was merely *predicted*, not captured. Also, seeing a native D3D11 method called within Chrome does not prove the intercepted caller was the video decoder.

**Security:** a handled `E_INVALIDARG`, a media-pipeline error, or a successful native breakpoint do not establish memory corruption or RCE. No native crash is evidenced here.

## Minimal next step on stock Chrome (no modified Chromium build)

With the GPU process paused and the existing breakpoint installed, replace the broad breakpoint with one that logs only the desired NV12 texture requests. The D3D11 descriptor `Format` field is at offset `0x10`; `DXGI_FORMAT_NV12 == 103 (0x67)`.

Example WinDbg condition (verify support for `bp /w` in the installed WinDbg version; stop if the debugger rejects the syntax):

```text
bc 0
bp /w "dwo(@rdx+0x10) == 103" d3d11!CDevice::CreateTexture2D
bl
g
```

Run a **single** known 960×129 VP9 hardware-preferred keyframe test in the browser (ideally compare with fresh 960×128 and 960×130 controls), and inspect `r`, `dd @rdx Lb`, `kb 8` only after the conditional breakpoint hits. If numerous valid NV12 textures trigger the breakpoint first, refine the condition further to odd width/height using `dwo(@rdx)` and `dwo(@rdx+4)`. This procedure has not yet been executed in the supplied logs.

**Strong proof threshold for Finding 1:** capture descriptor `Width=960`, `Height=129`, `Format=103` at the entry point; obtain the **return HRESULT from the same call**, showing `0x80070057`, and correlate it with the identical test and native Chromium error. This will validate the precise parity hypothesis. A matching descriptor without HRESULT validates the submitted parameters, not the return outcome.

**Strong proof threshold for Finding 2:** capture a separate **normal HTML** player's 960×128→960×120 failure, preferably with software-only playback control, then use symbols/instrumentation to establish `decoder_produced_a_frame_ == true`, `kFallbackAfterDecodeError`, and the actual terminal branch selection. WebCodecs results alone are not this trace.

## Artifact provenance

Original user-supplied WinDbg text exports, in chronological order:

- `Pasted text(20261115-191649).txt`: symbol lookup, module map, initial attach.
- `Pasted text (2)(20261115-191928).txt`: continued module loading after `g`.
- `Pasted text(20261115-192612).txt`: unsuccessful attempted symbol command and descriptor dump from manual `ntdll!DbgBreakPoint`; no useful texture data.
- `Pasted text(20261115-192743).txt`: first successful `CreateTexture2D` breakpoint, registers, actual 8×8 R8_UNORM descriptor and stack.

Raw logs have **not** been committed; this report captures the decisive excerpts without user path, account, and unrelated log content.

See [Issue A](ISSUE_A_ODD_NV12.md), [Issue B](ISSUE_B_RECONFIG_FALLBACK.md), [Lab 15–16 experiments](EXPERIMENTS_AND_LAB17.md), and [Lab 17 instrumentation runbook](LAB17_NATIVE_DIAGNOSTICS.md).
