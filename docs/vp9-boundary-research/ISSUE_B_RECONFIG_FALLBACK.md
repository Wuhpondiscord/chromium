# Issue B: hardware VP9 decoder cannot reconfigure 960×128 → 960×120 and normal playback terminates

**Status:** decoder reconfiguration failure and player impact reproduced; scope of intended fallback behavior needs policy/maintainer review.  
**Platform:** Chrome 157.0.8103.79, Windows 11 build 22631.4317; NVIDIA RTX 4080 SUPER, NVIDIA driver 32.0.16.1088; active adapter LUID `{0,98569}`.  
**Do not conflate this issue with the separate odd NV12 allocation issue.** The failing `960×120` dimensions are both even.

## 1. Observation

A particular VP9 profile-0 WebM dynamically changes coded dimensions. A valid `960×128` keyframe decodes through the D3D11 hardware path, while a subsequent `960×120` keyframe causes the D3D11 wrapper to fail reinitialization, with a `GetVideoDecoderConfigCount` diagnostic. The same two keyframes decode successfully under software preference, and ordinary HTML video playback of the file reaches `PipelineStatus::PIPELINE_ERROR_DECODE`.

This is a **platform capability boundary plus downstream recovery question**, not an out-of-bounds-memory finding.

## 2. Independent reproductions

### Lab 15 / Lab 16 WebCodecs

- `VideoDecoder.isConfigSupported` reports `true` for fresh 960×128 with hardware preference and `false` for fresh 960×120 hardware preference.
- Fresh 960×128 hardware: NV12 frame produced, flush succeeds.
- Fresh 960×120 hardware: unsupported/rejected (Lab 15).
- Transition 960×128→960×120 hardware: first NV12 frame produced; second flush rejects `EncodingError: Decoding error.`; decoder closes.
- Transition 960×128→960×120 software: both keyframes successfully decode to I420.
- Lab 16 repeated the original transition using actual extracted source WebM blocks, rather than only synthetic keyframes.

### Normal HTML playback

The independent ordinary-player `chrome://media-internals` entry has `kRendererName: RendererImpl`, `kVideoDecoderName: D3D11VideoDecoder`, `kIsPlatformVideoDecoder: true`, `PipelineStatus::PIPELINE_ERROR_DECODE`, adapter LUID `{0,98569}`. It starts at an advertised/selected coded size of 960×720 and traverses many smaller even coded dimensions before the final transition. The player reports 960×192 in `Stats for Nerds`/the final resolution fields, but the **decoder's failing in-band coded size is 960×120**, which must not be replaced by the displayed player resolution.

Representative native event sequence:

```text
D3DVideoDecoder config change: coded_size: 960x128
... D3D11 backend / NV12 / shared handle ...
D3DVideoDecoder config change: coded_size: 960x120
GetVideoDecoderConfigCount failed:
  The operation completed successfully. (0x0)
D3DVideoDecoder: D3DVideoDecoderWrapper is not created
D3DStatus::10  "D3DVideoDecoderWrapper is not created"
DecoderStatus::1
PipelineStatus::PIPELINE_ERROR_DECODE
```

**Meaning of `0x0`:** it is the successful HRESULT `S_OK`, **not** proof that a usable D3D11 decoder configuration was available. The wrapper's implementation can reject a query yielding zero configs. The phrase “failed” in Chromium's log is therefore not synonymous with a failing HRESULT. The evidence supports no usable decoder configuration at 120px on this NVIDIA device.

### Hardware capability minimum

Repository `media/gpu/windows/supported_profile_helpers.cc` defines `kMinVP9ResolutionForNvidia(128,128)`. This is a Chromium NVIDIA VP9 policy guard. It does **not** prove NVIDIA's underlying hardware is physically incapable of every smaller stream in every software/firmware/API combination; it explains why a 120px transition falls outside the configured supported-size boundary.

## 3. DecoderStream source-level recovery analysis (critical nuance)

The online explanation claiming decoder selection **never** runs again after startup is not accurate. The repository's `media/filters/decoder_stream.cc` explicitly contains **two different recovery/reconfiguration mechanisms**:

**A. Demuxer-driven configuration-change reinitialization:** `ReinitializeDecoder()` clears remaining selector candidates, prepends the current decoder and calls `BeginDecoderSelection()`. On successful reinitialization, `CompleteDecoderReinitialization()` resets `decoder_produced_a_frame_ = false`. Therefore it is not correct to say Chromium universally discards all fallback paths once initial selection occurs.

**B. Unexpected decode-error fallback:** in `OnDecodeDone`'s default error case, the observed source reads:

```cpp
if (!decoder_produced_a_frame_ &&
    base::FeatureList::IsEnabled(kFallbackAfterDecodeError)) {
  // ... invalidate pending callbacks ...
  ResumeDecoderSelection(std::move(status));
} else {
  media_log_->NotifyError(status);
  MEDIA_LOG(ERROR, media_log_) << " decode error!";
  state_ = State::kStateError;
  ClearOutputs();
  // ... propagate error to reader ...
}
```

After the first decoded frame, `OnDecodeOutputReady()` sets `decoder_produced_a_frame_ = true` and calls `decoder_selector_.FinalizeDecoderSelection()`. The actual behavior is **fallback allowed for some initial decode failures, but not for the ordinary post-output decode-error branch**. Whether the in-band VP9 size change should be translated into a demuxer configuration event, or a new exception/error-recovery path should exist, is the engineering question.

The `kFallbackAfterDecodeError` feature exists and is enabled by default in upstream media switch definitions at the time examined. This feature does **not** promise arbitrary mid-stream recovery after a frame has already been output.

**WebCodecs distinction:** an application-controlled `VideoDecoder` is not equivalent to Chromium's normal `DecoderStream` pipeline. A WebCodecs rejection after in-band VP9 reconfiguration does not establish whether HTML-media fallback is intended. Use the separately captured HTML player log to assess the latter.

## 4. Hypotheses

- **H1 (most supported):** in-band VP9 keyframe coded-size changes are handled internally by the D3D11 decoder and produce a decode-time error when the new size is outside its capabilities; normal `DecoderStream` has already produced frames, so `decoder_produced_a_frame_` prevents the early-error fallback path. The player propagates fatal pipeline error.
- **H2:** a demuxer-level `OnConfigChanged` event should have triggered more graceful reselection, but the VP9 coded-size change did not (or should not) change the container-level track config. Requires demuxer/decoder trace.
- **H3:** fallback would work but an eligibility state or another feature prevents it, e.g. an alternate decoder not in the selector at the point of error. Needs internal tracing.
- **H4:** replaying from an appropriate keyframe with a software decoder would make the transition recoverable; feasibility depends on buffering, timestamp preservation, and decode dependencies. Software success on the two-keyframe test alone does not prove seamless recovery in the full source.

## 5. Bug impact and question for maintainers

**Observed:** regular playback ends in `PIPELINE_ERROR_DECODE` for a stream for which software VP9 decoding is viable in isolated tests.

**Candidate defect:** recoverability or format-selection policy when an in-band VP9 resize crosses an active hardware decoder's supported bound. The fallback behavior may be an acknowledged deliberate trade-off (e.g., avoiding decoder re-selection after output to preserve continuity); the evidence alone does **not** establish that Chromium violated an existing recovery guarantee.

**User experience:** a WebM posted in a normal communication application can display then stop at a particular point despite software decoding being able to handle the keyframe sequence.

**Security:** the media pipeline error is handled, no process crash, cross-decoder state leakage, UAF, corruption, or RCE demonstrated.

## 6. Lab 17 validation plan

1. Add native diagnostics at `DecoderStream::OnDecodeOutputReady`, `DecoderStream::OnDecodeDone`, `ReinitializeDecoder`, `ResumeDecoderSelection`, and `DecoderSelector` to record the current status, `decoder_produced_a_frame_`, feature state, selector candidates, coded size, and transition source.
2. Confirm whether `OnConfigChanged` is ever fired for **in-band VP9 coded-size** changes; distinguish container-track reconfiguration from decoder-internal output dimensions.
3. Run the original source under normal HTML playback and compare it with explicit software-only playback. Prefer an isolated test profile, not the everyday Discord browsing profile.
4. Isolate two transition orders: supported→unsupported (128→120) and unsupported→supported (120→128); trace recovery, buffering and re-selection.
5. Explore carefully scoped prototype recovery only in a local Chromium test build: drain/cancel outstanding work, preserve appropriate keyframe replay boundaries, and verify no stale frames/shared-image handles survive device switch.
6. Add a regression test asserting either documented graceful software recovery or a deliberate, well-explained and non-crashing error policy.

## 7. Primary references / limitations

- Repository source: `media/filters/decoder_stream.cc` (`OnDecodeOutputReady`, `OnDecodeDone`, `ReinitializeDecoder`, `CompleteDecoderReinitialization`).
- Chromium upstream source: https://chromium.googlesource.com/chromium/src/+/main/media/filters/decoder_stream.cc
- Fallback feature definition: https://chromium.googlesource.com/chromium/src/+/main/media/base/media_switches.cc
- Repository supported-profile checks: `media/gpu/windows/supported_profile_helpers.cc`.
- Reproduction source file SHA-256: `78589f127fe4436f71003f6652ade3b99a85a7691c7d4d1d70aed9f9b3666e10`.

A third-party summary linked Chromium issue 40684757 but issue contents were unavailable for verification. **No claim of an established duplicate, known unfixed bug, or official triage outcome is made here.**
