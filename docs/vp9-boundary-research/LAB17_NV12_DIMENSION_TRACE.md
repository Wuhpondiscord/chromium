# Lab 17 — Stock Chrome NV12 allocation trace and debugger controls (2026-11-15)

**Status:** validated success HRESULT for a normal NV12 texture; full captured native dimension series 960×720 down to 960×128. **No 960×120 or odd-dimension NV12 texture request observed in this particular recording.** Lab 16 odd-size failure remains independently demonstrated by WebCodecs/media-internals, but the actual native descriptor for that *separate synthetic case* is still unverified.

## Source material and custody

Two user-uploaded near-duplicate exports of the same WinDbg session:
- `Pasted text(20261115-200822).txt` (reported 45,150 characters).
- `lab17-nv12-dimensions_c084_2026-11-15_15-06-15-532.txt` (reported 45,935 characters, includes header, breakpoint command, manual debugger break and `.logclose`).

Both exports were independently parsed by matching the exact pattern `LAB17_NV12 width=<int> height=<int> format=<int>`. Their **1,050 matched records agree exactly**. This is one trace in two exports, **not two independent runs**. The originals were **not** committed to the repository, preserving privacy and avoiding duplicate bulky logs; relevant excerpts and calculated aggregates are below.

Earlier material in the same session:
- `Pasted text(20261115-194700).txt` captured the first successful `960×720 NV12` entry.
- Later pasted console excerpts confirmed stepping out of that same call with `gu` and reading `RAX=0` (success).
- The later nested conditional breakpoint looking for `NV12 && Height=129` was accepted and then ran (`*BUSY*`) without a hit during playback of the **original** WebM. This is expected to be inconclusive for the *synthetic* 960×129 test, which must be run independently.

## Valid D3D11 control allocation plus return HRESULT

WinDbg at `d3d11!CDevice::CreateTexture2D` entry, before the function body:

```text
LAB17_NV12_CAPTURE
rdx=00002104000bddc8
00002104`000bddc8  000003c0 000002d0 00000001 00000001
00002104`000bddd8  00000067 00000001 00000000 00000000
00002104`000bdde8  00000208 00000000 00000802
0:000> gu
chrome!ChromeMain+0x2005b18:
00007ff8`7b6a3a68 8944244c mov dword ptr [rsp+4Ch],eax
0:000> r rax
rax=0000000000000000
```

Interpretation: width `0x3c0 = 960`, height `0x2d0 = 720`, `Format=0x67 = DXGI_FORMAT_NV12 (103)`. `gu` returned normally to the caller and `RAX=0` is `S_OK`. This is a **valid even-size success control** and confirms that descriptor+HRESULT capture works in stock Chrome. No failing HRESULT is captured from an odd-size request.

## Conditional breakpoint syntax corrections observed

1. Unconditional `bp d3d11!CDevice::CreateTexture2D` trapped an unrelated 8×8 R8_UNORM texture.
2. NV12-only command string `.if (dwo(@rdx+0x10) == 0x67) { ... } .else { gc }` successfully caught the 960×720 request.
3. Combining tests with C-style `&&` and `&` caused WinDbg **MASM expression** parsing errors, `Numeric expression missing from '& ...'`. These were debugger command errors, **not Chrome faults**.
4. Nested `.if` expressions (first format==`0x67`, then height==`0x81`) were accepted; WinDbg resumed with `*BUSY*`, and no height-129 NV12 event was observed during playback of the original clip. A missing synthetic test run cannot falsify Finding A.
5. The later all-NV12 **logpoint** captured allocations without manually stepping: `bp ... ".if (dwo(@rdx+0x10) == 0x67) { .printf \"LAB17_NV12 width=%u height=%u format=%u\\n\", dwo(@rdx), dwo(@rdx+4), dwo(@rdx+0x10); gc } .else { gc }"`. In practice, formatted events appeared on one long line: newline escaping did not produce independent lines. Parse by the full repeating `LAB17_NV12` marker, not line count.

## Full logged dimension series — independently verified from both exports

| Metric | Observation |
|---|---|
| Total `LAB17_NV12` log events | **1,050** |
| Distinct (width,height,format) tuples | **75** |
| Width of every event | **960** |
| Format of every event | **103 = NV12** |
| Maximum observed height | **720** |
| Minimum observed height | **128** |
| Heights | **720, 712, 704, ..., 144, 136, 128** |
| Height decrement | **8 pixels** across 74 transitions |
| Count per distinct height | **14** |
| Odd width/height among logged events | **0** |
| 960×120 NV12 events | **0** |
| 960×129 NV12 events | **0** |

All 75 observed heights satisfy `720 - 8*i` for integer `i ∈ [0,74]`; each height appears in a contiguous group of 14 events. All requests were **function-entry events**, not necessarily 1,050 successful allocations: this logpoint did **not** collect the HRESULT of each request. The fixed count of 14 per height should not be labeled 14 decoded video frames without separate output evidence.

Representative sequence from log (the omitted intermediate dimensions follow the exact progression):

```text
LAB17_NV12 width=960 height=720 format=103 [14 calls]
LAB17_NV12 width=960 height=712 format=103 [14 calls]
LAB17_NV12 width=960 height=704 format=103 [14 calls]
...
LAB17_NV12 width=960 height=144 format=103 [14 calls]
LAB17_NV12 width=960 height=136 format=103 [14 calls]
LAB17_NV12 width=960 height=128 format=103 [14 calls]
```

The complete export terminates after the height=128 group, then records a **manual debugger break** at `ntdll!DbgBreakPoint` and a user-issued `.logclose`. That break is not a proven crash. Lack of 120 in this trace does **not** prove that no 120 configuration was ever queried, only that no 120 NV12 texture request was **observed at the instrumented entry point** during this logging interval.

## Interpretation relative to Findings A and B

### Finding A — odd coded sizes

This logging run of the original descending-resolution WebM does **not** test the deliberately synthesized 960×129 VP9 keyframe used in Lab 16. It provides no direct confirmation or refutation that such a test causes a `960×129 NV12` texture descriptor or `0x80070057` from that same call. Lab 16's odd/even success/failure matrix and media-internals error record remain valid separate evidence. Run **explicitly identified** 960×128, 960×129, and 960×130 hardware-preferred synthetic cases in isolation; confirm each case in WebCodecs/media-internals and monitor the GPU PID.

### Finding B — in-band 128→120 transition

The observed native NV12 requests track the original WebM's descending-height progression down to **960×128**, without a 960×120 texture request in the collected window. This is **consistent with**, but does not independently prove, the previously observed `GetVideoDecoderConfigCount ... (0x0)`, `D3DStatus::10` and regular HTML `PIPELINE_ERROR_DECODE` at the 128→120 transition. No decoder capability-query count or actual `DecoderStream` fallback decision was captured in this WinDbg trace. Missing post-failure D3D11 calls are not evidence of a stale-pointer or memory-safety problem.

## Recommended follow-up

- **Finding A:** use exact synthetic 960×129 test; first broad NV12 logpoint to learn actual descriptor sizes. When a candidate request exists, stop at function entry and collect descriptor+stack, then `gu` and `r rax` to capture the **same call's** returned HRESULT. Filtering exclusively for 129 before confirming the case executes can hide relevant evidence.
- **Finding B:** synchronize normal HTML `chrome://media-internals` timing with native trace. Verify explicit software-only HTML playback (not just WebCodecs) and GPU adapter/PID. To establish exact fallback branch selection or lifetime errors, use adequate Chromium symbols or the already committed optional Lab 17 instrumentation; neither has produced a runtime fallback trace yet.
- Prefer a short run on an isolated Chrome profile. Document stop reasons and avoid interpreting a manual debugger interrupt, the `*BUSY*` state, or a media error as a GPU crash.

**Classification:** reproducible functional media compatibility / recovery issues; exact odd-dimension native causality remains open; no native memory-corruption finding or security exploit shown.

Related docs: [Lab 17 first breakpoint evidence](LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md), [Issue A](ISSUE_A_ODD_NV12.md), [Issue B](ISSUE_B_RECONFIG_FALLBACK.md), [Lab 17 instrumentation](LAB17_NATIVE_DIAGNOSTICS.md).
