# Lab 25 — Final VP9/D3D11 security and compatibility assessment

**Research period:** Labs 15–25, completed 2026-11-15 (America/Detroit)  
**Status:** **Closed at the planned Lab 25 cutoff.** Reproducible functional hardware-decoding failure; **no validated security vulnerability**.  
**Research branch:** `research/lab17-vp9-native-diagnostics` in `Wuhpondiscord/chromium`.

## Executive verdict

The VP9 960×128 → 960×120 transition reproducibly fails under D3D11 hardware decoding on the tested Windows 11 / NVIDIA RTX 4080 SUPER configuration. Native WinDbg evidence in Lab 18 established that `GetVideoDecoderConfigCount` returned `S_OK` for both sizes but configuration counts **2 for 960×128** and **0 for 960×120**, with the same decoder GUID and NV12 output format. The media pipeline then emits `PIPELINE_ERROR_DECODE`.

**No attributable native access violation, use-after-free, memory corruption, native process crash, GPU process compromise, renderer/sandbox escape, or code execution was demonstrated.** There is insufficient evidence to classify this as a security vulnerability or claim CVE/VRP impact.

A legitimate non-security compatibility/recovery issue remains. The zero configuration count need not mean the driver violated its API contract. Whether Chromium should select another decoder for this specific in-band transition requires testing the demuxer/decoder fallback state and applicable policy. Treat as a *functional bug candidate*, subject to duplicate search and cross-version reproduction.

## Environment and provenance

- Stock Google Chrome **157.0.8103.79**; separate Microsoft Edge **157.0.4341.60** run.
- Windows 11 Pro **22631.4317**.
- NVIDIA RTX 4080 SUPER driver **32.0.16.1088**; secondary AMD Radeon Graphics installed but not demonstrated to decode the reproducer.
- Lab 21 isolated Chrome profile `C:\Users\Admin\Downloads\Lab21\chrome-profile`; identified GPU process **PID 53560** during the recorded window. PIDs are ephemeral.
- Source-level analysis refers to **the research fork branch** with opt-in LAB15/LAB17 edits, not a proven byte-identical match to the installed Chrome/Edge binaries.

## Evidence ledger

| Evidence | Observation | Inference limit |
|---|---|---|
| Lab 17 | NV12 `CreateTexture2D` interception worked; logged 1,050 calls with heights 720→128, no 120 or odd sizes in that trace | Cannot attribute absent dimensions to a vulnerability or infer HRESULT for all calls |
| Lab 18 native WinDbg | `GetVideoDecoderConfigCount`: `S_OK`, count 2 (960×128) vs `S_OK`, count 0 (960×120), same GUID/NV12 | Confirms configuration availability boundary, **not** memory corruption or driver contract violation |
| Lab 18 normal HTML media logs | `D3D11VideoDecoder` selected, two in-band size changes, wrapper creation fails, `PIPELINE_ERROR_DECODE` | Did not establish exact behavior of every software fallback policy |
| Lab 20 Edge repetition | 100/100 media errors; 84 count-diagnostic, 16 generic errors | Diagnostic variation need not imply a second native bug |
| Lab 21 Chrome repetition | 100/100 media errors; 86 count-diagnostic, 14 generic errors | Repeatability, not severity |
| Lab 21 CPU-process samples | One A-labeled trace +3.22 MiB net private bytes; B-labeled +41.27 MiB; different starting states/order and unsynchronized control identity | **Not** evidence of a leaked allocation |
| Lab 22 GPU process counters | 57 valid samples across 18:33:27–18:35:25 EST, PID 53560; GPU dedicated 85,295,104→66,953,216 bytes; shared 13,115,392→10,973,184 bytes | Net decreases do not prove every individual resource was freed |
| Lab 22 manually recorded markers | First start/end 18:33:55–18:34:38 EST, within the counter interval; second pair 19:03:20–19:03:21, outside it | Markers are manual, not frame-exact |
| Lab 22 GPUView | ETW trace loaded, PID 53560 visible, no lost events/buffers reported in inspected screenshots, memory chart appeared stable after initial activity | Native C++ resource lifetime and decoder-specific allocation ancestry not established |
| Lab 24 WinDbg | Log contains a first-chance `0x80000003` at `ntdll!DbgBreakPoint`, followed by log close | Breakpoint not an access violation; short record insufficient to prove playback coverage or memory safety |

## Direct causal explanation of the observed decode failure

The two-frame Test B contains an in-band transition from 960×128 to 960×120. Chrome initially selected a hardware D3D11 VP9 decoder. When the decoder received the second configuration, it attempted wrapper recreation. Native D3D11 reported **zero valid configurations** for the 960×120 descriptor (despite `S_OK`). Wrapper creation therefore failed, and the media pipeline reported a handled decode error. This explains the immediate failure. It does not establish why no configuration exists or whether Chromium should have performed different fallback/recovery behavior.

The two-frame extract's container track metadata initially advertised 960×720, so future compatibility triage should preserve this caveat and verify a normalized track header. Single-frame A/C hardware-decoder selection was not confirmed in the earlier record. Do not conflate the separate odd-dimension `E_INVALIDARG` issue with the 960×120 zero-configuration issue.

## Source-level ownership audit (Lab 20/23)

Reviewed:
- `media/gpu/windows/d3d_video_decoder.cc`: `RecreateDecoderWrapper()`, `ClearPictureBuffers()`, `OutputResult()`, `PostDecoderStatus()`.
- `media/gpu/windows/d3d11_video_decoder_backend.cc`: picture-buffer initialization, `ReleaseResources()`.
- `media/gpu/windows/d3d11_video_decoder_wrapper.cc`: scoped decoder buffers and decoder lifetime.
- `media/gpu/windows/d3d_texture_wrapper.{cc,h}`: asynchronous `SequenceBound<GpuResources>` initialization.
- `media/gpu/windows/d3d_picture_buffer.{cc,h}`: reference counting and decoder output views.
- `media/gpu/windows/d3d_video_frame_mailbox_release_helper.cc`: sync token wait on frame release.
- `media/filters/decoder_stream.cc`: decode/fallback state handling.

Protective invariants observed:
- Old D3D11 wrapper is destroyed before picture buffers are cleared.
- Picture-buffer objects are ref-counted; old output views are explicitly `Reset()` before backend teardown.
- Video-frame release callback holds a ref-counted picture buffer and a weak decoder pointer.
- Concrete wrapper releases `bitstream_buffer_` before D3D11 context/decoder destruction.
- `GpuResources` holds a ref-counted picture buffer while asynchronous initialization runs; callbacks back to `DefaultD3DTextureWrapper` use weak pointers.
- Mailbox helper waits for sync tokens; D3D11 `Flush()` is **not** a GPU completion wait by itself.

**Unresolved candidate A — scoped raw pointer.** `ScopedD3D11DecoderBuffer` stores `raw_ptr<D3D11VideoDecoderWrapperImpl>` and uses it in `Commit()`. No path was demonstrated that lets a scoped buffer outlive its owning wrapper. Raw pointer presence alone does not establish a UAF.

**Unresolved candidate B — async init early-return.** `DefaultD3DTextureWrapper::GpuResources` error branches may return without normal success/completion callback, while the backend has incremented `add_client_use()` before initialization. It is not proven that this produces a persistent stall, resource leak, or unsafe access in a real decoder instance.

**Unresolved candidate C — `ProcessTexture()` status contract.** Implementation contains `TODO(liberato): Return the error.` and can return `kOk` with `shared_image_dest = shared_image_` without explicitly checking `received_error_`. This is a useful robustness-review lead but **no actual reachable `OutputResult()` dereference of a null shared image was shown**.

**Critical causal separation:** The 960×120 wrapper creation failure occurs *before new picture-buffer GPU-resource initialization*. Therefore candidates B and C are not established causes of the original reproducer. They require separate deterministic failure injection.

## Lab 24 test implementation status

Committed `D3DVideoDecoderTest.Lab24ZeroConfigsDuringRecreationReturnsDecodeError` in `media/gpu/windows/d3d_video_decoder_unittest.cc`; it uses a mocked H264 configuration change to characterize `S_OK`/zero count handling. It was **never compiled or run**. The user did not have `autoninja` or a built `media_unittests.exe` and elected not to install Chromium build dependencies before the Lab 25 cutoff. Do not count this as a passing test.

The planned asynchronous initialization failure-injection tests (23B/23C) were documented but **not implemented or executed**.

## Final classification

| Question | Outcome |
|---|---|
| Reproducible input-dependent decoder error? | **Yes, confirmed** on tested hardware/browser |
| Native D3D11 count boundary known? | **Yes: S_OK + 2 vs S_OK + 0** |
| Persistent GPU resource leak? | **Not demonstrated** |
| Unhandled native access violation? | **Not demonstrated** |
| Memory safety violation / UAF? | **Not demonstrated** |
| Security boundary compromise / RCE? | **Not demonstrated** |
| Security severity / CVE / VRP eligibility? | **Not established** |
| Potential compatibility/recovery bug? | **Yes, reportable for normal triage subject to duplicates and further controls** |

## Responsible next action (outside the Lab 25 cutoff)

For a regular Chromium issue, include exact browser/driver versions, the minimal two-keyframe Test B, the media-internals excerpt showing D3D11 decoder selection, the `S_OK`/zero-count comparison, and exact expected vs actual user-visible playback behavior. Avoid describing it as a driver crash or UAF. Before filing, check for an existing issue and verify A/C decoder selection and normalized container metadata if feasible.

If future source instrumentation independently proves the asynchronous callback error path is reachable with native memory corruption or a security-boundary violation, treat it as a **new separate security investigation** with concrete reproduction, faulting stack, and version-matched evidence.

## Final disposition

**Research CLOSED at Lab 25.** Functional decode/recovery failure characterized. No working security vulnerability identified or validated. Remaining source hypotheses recorded, not presented as vulnerabilities.

## Related reports

- [Lab 18 two-frame D3D11 configuration-count comparison](LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md)
- [Lab 20 ownership and resource-lifetime audit](LAB20_DECODER_LIFETIME_AUDIT.md)
- [Lab 24 test design and unexecuted characterization test](LAB24_FAILURE_PATH_VALIDATION.md)
- [Issue B compatibility/recovery analysis](ISSUE_B_RECONFIG_FALLBACK.md)
