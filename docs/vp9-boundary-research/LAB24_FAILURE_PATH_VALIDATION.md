# Lab 24 — D3D11 failure-path validation

**Date:** 2026-11-15 (America/Detroit)  
**Branch:** `research/lab17-vp9-native-diagnostics`  
**Status:** first unit test committed, **NOT yet compiled or executed**.  
**Goal:** Resolve candidate 23B (asynchronous initialization failure lacking a completion callback) and candidate 23C (`ProcessTexture()` returns `kOk` without checking `received_error_`) before the Lab 25 cutoff.

## Scope and an essential distinction

The original stock-Chrome VP9 960×128 → 960×120 reproducer fails when `D3D11VideoDecoderWrapper::Create()` receives `S_OK` but a zero `GetVideoDecoderConfigCount` result. That happens **before construction of the replacement decoder's picture buffers**. The asynchronous GPU-resource initialization defects hypothesized in Lab 23 are therefore **not demonstrated to cause the observed 960×120 failure**. They require independent deterministic test triggers; do not infer exploitability or a causal connection.

### Implemented now — characterization test

Added `D3DVideoDecoderTest.Lab24ZeroConfigsDuringRecreationReturnsDecodeError` to `media/gpu/windows/d3d_video_decoder_unittest.cc`. It:
1. Initializes a mock D3D11 decoder using H264, then supplies a new mock codec configuration change.
2. Simulates `GetVideoDecoderConfigCount` returning `S_OK` with count zero.
3. Asserts that another native D3D11 video decoder is **not** created.
4. Checks that the decode callback reports an error and no active picture buffers remain.

**Important:** This test uses the H264 mock fixture deliberately and does **not** replay VP9 or test the actual NVIDIA driver, GPU asynchronous initialization, or safety of `ProcessTexture()`. It is a narrow software-state regression check only. It is unbuilt/unrun at commit time.

### Local build/run (only if a compatible Chromium checkout and build output already exist)

```powershell
# From the root of a full Windows Chromium checkout:
autoninja -C out\Default media_unittests
.\out\Default\media_unittests.exe --gtest_filter=D3DVideoDecoderTest.Lab24ZeroConfigsDuringRecreationReturnsDecodeError
```

The exact existing GN output directory may differ. If there is **no local Chromium checkout**, do not confuse having the GitHub fork and stock Chrome binaries with a build environment. Do not edit the installed Chrome executable.

## Pending test A — asynchronous initialization failure (candidate 23B)

Use a dedicated mock-driven test with a `D3D11PictureBufferImpl` holding `DefaultD3DTextureWrapper`, a fake D3D11 texture, a valid mocked `ID3D11VideoDevice1`, and a `GetCommandBufferHelperCB` intentionally returning null to deterministically exercise `GpuResources::GpuResources`' early-return branch.

Record:
- `add_client_use()` before `InitD3D11()`;
- queued `SequenceBound<GpuResources>` constructor;
- `OnError` post-back;
- whether `OnGPUResourceInitDone` and the provided init-done callback fire;
- `in_client_use()` after `RunUntilIdle()` and after destruction;
- decoder error/status callback and actual state transition.

**Expected current-source behavior to verify, not yet runtime-confirmed:** On GPU init early-return the success callback is not run. Since the buffer's normal counter decrement happens in `D3DVideoDecoder::PictureBufferGPUResourceInitDone`, the counter might stay nonzero. Whether this causes a real stall depends on ownership, cleanup and reachable state; a broken counter alone does not prove a security issue. Explicitly examine that the `DefaultD3DTextureWrapper` error status is actually consumed.

## Pending test B — ProcessTexture valid-output contract (candidate 23C)

For `DefaultD3DTextureWrapper::ProcessTexture()`, test:
- normal successful initialization returns a non-null `ClientSharedImage`;
- injected GPU init failure is reflected by a failed `D3DStatus`, not `kOk` paired with a null output;
- teardown before delayed GPU init completion is safe;
- `D3DVideoDecoder::OutputResult()` cannot reach `shared_image->creation_sync_token()` without a valid shared image.

The implementation currently contains `TODO(liberato): Return the error.` and assigns `shared_image_dest = shared_image_` before returning `kOk` without checking `received_error_`. **That is a code-level contract concern, not a demonstrated reachable null dereference**. A direct call on an uninitialized texture wrapper is not sufficient proof: establish that the actual decoder can reach the call after the failure.

## Candidate 23A — scoped raw pointer

`ScopedD3D11DecoderBuffer` stores `raw_ptr<D3D11VideoDecoderWrapperImpl>`, and its destructor calls `Commit()` through that pointer. The wrapper intentionally destroys `bitstream_buffer_` before its D3D11 context and decoder members. Only escalate if the test can establish a scoped buffer actually outliving its owner, not merely because a raw pointer exists.

## Lab 24 pass/fail matrix

| Evidence | Conclusion | Lab 25 treatment |
|---|---|---|
| Zero-count test errors cleanly | Existing boundary is a handled decoder failure | Non-security issue unless other impact |
| Async failure leaves use count positive but buffer safely destroyed | Potential robustness/cleanup issue | Reliability triage |
| Async failure leaves decoder hung indefinitely | Reproducible DoS / availability issue | Determine scope, severity, and recoverability |
| A real path returns success with null shared image and dereferences it | Potential native process crash | Capture stack, evaluate boundary and recovery |
| Invalid memory access or lifetime invariant violation reproducible | Candidate memory safety issue | Confirm build parity, sanitizer/native evidence |
| No reachable violation by end of Lab 24 | No security issue demonstrated | Lab 25 final negative/inconclusive report |

## Caveats

- GitHub commits and source inspection do not execute code on the user's Windows host.
- Current user host has installed stock Chrome 157 and GPUView/xperf, but no demonstrated full Chromium build checkout or `out\Default\media_unittests.exe`.
- GPUView showed stable aggregate GPU memory and cannot prove CPU object lifetime or reachable UAF.
- Do not publish a CVE/VRP vulnerability claim based on a TODO or counter imbalance without evidence of actual security impact.

## Next step

1. Compile and run the new zero-config test if the full Chromium build exists.
2. Add a dedicated, deterministic asynchronous error-path test; if build is unavailable, use source audit to specify mock integration and avoid asserting runtime results.
3. Stop at Lab 25 with an evidence-backed final report whether a vulnerability was validated or not.
