# Complete Archived Research Dossier: Discord-hosted WebM / Chromium VP9 / Windows D3D11 Investigation

**Researcher project:** Wuhpondiscord Chromium VP9 boundary research  
**Scope:** Consolidation of all available GitHub research reports and relevant investigation-chat observations through Lab 25  
**Research dates:** November 14–15, 2026 (US Eastern), with this archival edition assembled after the Lab 25 closeout  
**Repository:** https://github.com/Wuhpondiscord/chromium  
**Branch:** \`research/lab17-vp9-native-diagnostics\`  
**Archive status:** Closed investigation, preserved for later work on *other file types and processing surfaces*  
**Evidentiary status:** Reproduced compatibility/recovery issues; **no validated memory-safety or remote-execution vulnerability**

> **How to use this document:** Part I is a consolidated interpretation and reproducibility guide, including details discussed in the investigation chat. Part II reproduces all 11 source reports/index files in full as they existed when this archive was assembled. Do not treat the interpretation as a substitute for the underlying full report text or raw logs. A copied report that mentions an experiment as “planned” is historical—refer to the final status table before assuming it was run. Research fork source may differ from installed stock-browser binaries.

---

# Part I — Consolidated Research Narrative

## 1. Why this investigation started

The broader question was whether crafted or unusually encoded **Discord-hosted media files**—especially VP9 WebM—could trigger unusual behavior in Chromium/Discord playback beyond an ordinary decoding failure. Work therefore moved from browser-visible media errors through WebCodecs test matrices, media-internals logs, Windows D3D11 API tracing, GPU process and memory monitoring, GPUView/ETW, and source-level lifetime/error-path review.

**Threat-model boundary:** an application displaying or decoding an untrusted attachment exercises a parser/decoder stack with several trust boundaries (container reader, bitstream parser, video decoder, graphics APIs, GPU process, driver, and renderer-facing shared images). A failure in a stage does *not* show the attacker controls code or a protected context. This dossier does not demonstrate Discord-specific exploitability or a zero-click vulnerability. The media files were encountered through Discord, but the independently reproduced behavior was tested in the Chromium media stack as well.

The investigation eventually separated **two concrete, independent functional failure modes**:
- **Issue A — odd VP9 coded dimensions:** hardware decoder selected, NV12 output-surface creation fails with \`E_INVALIDARG\`; software decoder succeeds. The precise failing native texture descriptor plus same-call HRESULT were not captured in one session.
- **Issue B — mid-stream VP9 resize 960×128 → 960×120:** D3D11 hardware-decoder configuration count goes from 2 to 0; HTML video ends with \`PIPELINE_ERROR_DECODE\`. This is the stronger, native-confirmed immediate causal finding.

Other lifetime and error-propagation concerns identified in the source were **not demonstrated to be reachable through Issue B**.

## 2. Environment and identity controls

| Field | Recorded value |
|---|---|
| OS | Windows 11 Pro build 22631.4317 |
| Google Chrome | 157.0.8103.79 (stock browser; WinDbg testing) |
| Microsoft Edge | 157.0.4341.60 (separate Edge repetition) |
| Hardware | NVIDIA RTX 4080 SUPER, vendor 0x10de, device 0x2702 |
| NVIDIA driver | 32.0.16.1088 |
| Recorded NVIDIA adapter LUID | \`{0,98569}\` |
| Secondary GPU | AMD Radeon(TM) Graphics, vendor 0x1002, device 0x13c0; not verified as the decoding device in these cases |
| Hardware codec | VP9 profile 0, 4:2:0, D3D11 backend, NV12 video output; platform decoder flag true |
| Software codec | VpxVideoDecoder with I420 output; platform decoder flag false |
| Isolated Chrome profile | \`C:\Users\Admin\Downloads\Lab21\chrome-profile\` |
| Isolated GPU process | PID 53560 during Lab 21/22 capture; identity must be re-discovered each launch |
| Tools already used | Chrome DevTools/WebCodecs, chrome://media-internals, PowerShell/CIM/Get-Counter, WinDbg, Windows Performance Toolkit xperf/WPR, GPUView, GitHub research fork |
| Build tooling | \`autoninja\` and \`out\Default\media_unittests.exe\` unavailable on the user's host; source-level test commits were not run |

Never identify a process by PID alone after restart; use command line, executable, profile association, creation time and capture timestamps. Browser user-agent lines distinguish Edge from Chrome. The existence of an AMD adapter in device enumeration is not proof that AMD actually decoded a specific clip.

## 3. Source media / evidence chain of custody

Original source: **\`funkyms_4.webm\`**, a Discord CDN attachment. Entire file length **31,228 bytes**, SHA-256:

\`\`\`text
78589f127fe4436f71003f6652ade3b99a85a7691c7d4d1d70aed9f9b3666e10
\`\`\`

A prior parser catalogued **211 WebM blocks**. The keyframe pair that enabled the tiny transition reproducer is:

| Original block | Coded dimensions | Payload bytes | Payload SHA-256 |
|---|---|---:|---|
| 74 | 960×128 | 175 | \`1b958cd581b3f803189a4097a41a5d88aaca5bd6168af6d1b78cc762077960f0\` |
| 75 | 960×120 | 227 | \`c4006298875eafbcf269b67190a3bcd1c1a259be1fc929478c1cfa36903bdefe\` |

Lab 16 verified the full-file SHA-256 before extracting original VP9 packets. A Gradio/FFmpeg/FFprobe-based workflow remuxed selected packets **without re-encoding** and validated frame dimensions. The three minimal file roles were:
- \`test_A_960x128_only.webm\`: one 960×128 keyframe.
- \`test_B_128_to_120.webm\`: two consecutive keyframes, 960×128 then 960×120; **reproduces failure**.
- \`test_C_960x120_only.webm\`: one 960×120 keyframe.

Test B was described as **936 bytes** in the media-internals record and **0.066 seconds** duration. Its WebM **initial track metadata advertised 960×720**, despite in-band frame dimensions 960×128 and 960×120. That container/codec discrepancy is a **confounder**, not evidence of an exploit. A/C did not show the equivalent HTML error, but their *actual hardware decoder selection was not established*, so they cannot be used as fully matched hardware controls without further logs.

Preserve original compressed payload SHA-256 separately from output frame visual/readback fingerprints. FNV-1a image fingerprints from the browser harness are **not cryptographic hashes** and cross-format NV12/I420 bytes cannot be compared directly without pixel/plane/crop/colorspace normalization.

## 4. Chronological experimental record

### Lab 15 — broad hardware/software size boundary

42/42 cases completed with no harness timeouts. **21 software-preferred cases succeeded**. Fresh hardware VP9 cases with height 112 and 119–127 were unsupported or rejected; 128 succeeded; 129 was accepted by support probing but failed decode; 130, 136 and 144 worked. The original 128→120 transition failed in hardware preference and decoded in software preference.

The NVIDIA minimum VP9 capability guard examined in Chromium's \`supported_profile_helpers.cc\` is **128×128**. This is a Chromium policy/capability minimum, not proof of a fundamental hardware physics limit on all NVIDIA models and drivers.

### Lab 16 — odd vs even boundary

Synthetic VP9 profile-0 keyframes were created from eight-band canvas test images using software-preferred WebCodecs \`VideoEncoder\` (\`vp09.00.10.08\`, realtime, 800 kbps, 30 FPS). Eight fresh-size encodes succeeded. **20 decoder test cases, 14 successes, six decode errors, zero unsupported**. All ten software-preferred decoding cases passed. Six hardware failures comprised odd-height 960×129 and 960×131, odd-width 959×128 and 961×128, original 128→120 transition, and synthetic 128→129 transition.

| Fresh VP9 coded size | Hardware selection/support | Hardware output | Software output |
|---|---|---|---|
| 960×128 | Supported | NV12 success | I420 success |
| 960×129 | Supported | \`E_INVALIDARG\` / no output | I420 success |
| 960×130 | Supported | NV12 success | I420 success |
| 960×131 | Supported | \`E_INVALIDARG\` / no output | I420 success |
| 960×132 | Supported | NV12 success | I420 success |
| 959×128 | Supported | \`E_INVALIDARG\` / no output | I420 success |
| 961×128 | Supported | \`E_INVALIDARG\` / no output | I420 success |
| 962×128 | Supported | NV12 success | I420 success |

Relevant media-internals examples: players 934 (960×129), 936 (960×131), 938 (959×128), 939 (961×128) yielded \`D3DStatus::20\` and \`HRESULT 0x80070057\`; player 941 (128→120) yielded \`D3DStatus::10\` and a zero-config diagnostic; player 942 (synthetic 128→129) yielded the odd-size native allocation error. Software cases were players 943–952.

The associated source is \`D3DDecoderConfigurator::CreateD3D11OutputTexture\`; it copies \`size.width()\` / \`size.height()\` into \`D3D11_TEXTURE2D_DESC\` then calls \`CreateTexture2D\`. NV12 backing surfaces require even texture dimensions. This *strongly suggests* an odd backing-size normalization problem, but the exact failing 960×129 descriptor and corresponding native return were not jointly captured. Do **not** upgrade “consistent with” to “proven.”

### Labs 16–17 — browser playback and GPU monitoring

The ordinary HTML video path selected \`RendererImpl\` / \`D3D11VideoDecoder\`; multiple in-band coded-height changes were observed. At the later 960×128→960×120 transition, wrapper creation failed and the media pipeline reported \`PIPELINE_ERROR_DECODE\`. UI “Stats for Nerds” / last-known \`kResolution\` values of 960×192 or 960×128 are **not** the decoder's actual attempted 960×120 coded size.

A Lab 16 process sampler recorded ~245 samples over ~4 minutes 20 seconds. PID 48160 and the crash-count argument remained unchanged; a separate debugger-confounded PID transition could not be attributed to the WebM. Media errors are not by themselves native process crashes.

In Lab 17, WinDbg public D3D11 symbols and function breakpoints were confirmed. Early interceptions included an unrelated **8×8 \`DXGI_FORMAT_R8_UNORM\`** \`CreateTexture2D\` call—*not* a VP9 NV12 descriptor. Another correctly decoded **960×720 NV12** descriptor returned \`S_OK\`. A filtered logger then captured **1,050** native NV12 CreateTexture2D **entry events**, **75 sizes** (width 960; heights **720, 712, 704, ... 136, 128**) and **14 calls per size**. There were **zero 960×120** and **zero 960×129** NV12 allocation entry records in this capture. This supports the native progression of the original descending-resolution clip, but cannot establish that the synthetic odd-size case ran, that all calls succeeded, or that 14 calls means 14 output frames.

Debugger details preserved in the historical reports include Windows x64 register arguments, descriptor DWORD offsets, \`bp\`, \`gu\`, \`r\`, \`dd\`, MASM-expression parsing pitfalls, conditional breakpoints, and misleading \`*BUSY*\` after resuming. Session-specific instruction addresses must not be hardcoded for future process launches. A manual \`ntdll!DbgBreakPoint\`, debugger-induced window stalling, or user \`.logclose\` must not be called a browser crash.

### Lab 18 — decisive native config-count comparison

Native function \`d3d11!CDevice::GetVideoDecoderConfigCount\` was intercepted with **the same decoder GUID and output NV12 format**. The return value of the COM method was **\`S_OK\` in both cases**, while output configuration count differed:

| Native field | 960×128 | 960×120 |
|---|---:|---:|
| Descriptor width | 960 (\`0x3c0\`) | 960 (\`0x3c0\`) |
| Descriptor height | 128 (\`0x80\`) | 120 (\`0x78\`) |
| Descriptor NV12 format | 103 (\`0x67\`) | 103 (\`0x67\`) |
| HRESULT from method (\`RAX\`) | 0 (\`S_OK\`) | 0 (\`S_OK\`) |
| Output \`*pCount\` | **2** | **0** |

The recorded decoder GUID DWORDs were \`463707f8 4585a1d0 aa836d87 9eb8606d\`; the call returned to a Chrome wrapper routine. \`GetVideoDecoderConfigCount failed: The operation completed successfully. (0x0)\` is a confusing but interpretable log: **zero configurations** caused Chromium's wrapper-creation availability check to fail; it was **not** a failing HRESULT. A count of zero may correctly represent unsupported geometry; the driver is not proven to violate its API contract. Fallback behavior is the remaining product-design question.

### Labs 19–20 — repeatability, monitoring and lifecycle

The uploaded Edge 157 playback dataset reported **100 errors out of 100**, with **84** count-diagnostic messages and **16** generic video-decode errors. CPU telemetry sample sets were not a clean, synchronous hardware A/B control and were not necessarily attributable to the same player. During the available intervals there was no convincing sustained memory/handle growth, but neither a leak nor universal absence of leaks is proved.

Source review found important protections: wrapper destruction/teardown, reference-counted picture buffers, weak decoder pointers in frame-release callbacks, explicit output-view reset, backend resource release and a D3D11 \`Flush()\`. **Flush is not GPU-completion synchronization.** The resource and callback graph needed further review.

### Lab 21 — isolated Chrome control and rerun

The separate Chrome 157 isolated profile used \`--user-data-dir="C:\Users\Admin\Downloads\Lab21\chrome-profile"\`. Its GPU process was identified as PID **53560** for the measured interval (not 48160 regular Chrome or 57924 Edge). In Chrome, the playback harness produced **100/100 errors** in approximately 10.84 seconds; **86** count diagnostics and **14** generic decode errors.

The A-labeled CPU monitor had a reported net +3.22 MiB private memory and the B-labeled monitor +41.27 MiB. Their chronological order, initial state, profile/baseline and verified synchronized playback controls were insufficient for claiming a leaked resource. The originally provided CPU-time script used \`$Process.CPU.TotalSeconds\` although \`Get-Process.CPU\` is already a numerical CPU-seconds value; do not use the original CPU-derived trend as sound evidence.

### Lab 22 — GPU memory ETW and GPUView

The script \`C:\Users\Admin\Downloads\Lab22\monitor-gpu-memory.ps1\` used Windows \`GPU Process Memory\` dedicated/shared usage counters and sampled the isolated Chrome GPU process. B-transition data (EST 18:33:27.087–18:35:25.886) contained **57 valid counter rows**, ~2.1-second effective interval; dedicated first **85,295,104** bytes and last **66,953,216** bytes; shared first **13,115,392** bytes and last **10,973,184** bytes. Peak dedicated **92,770,304** bytes, peak shared **15,028,224** bytes. A single \`GPU_PROCESS_CHANGED\` event with empty OldPid and NewPid 53560 was the monitor's **initial discovery**, *not* a restart.

Manually recorded playback markers: \`START 2026-11-15T18:33:55.9293445-05:00\`, \`END 2026-11-15T18:34:38.0077334-05:00\`; the later \`START/END 19:03:20–19:03:21\` marker pair was **outside this monitor capture** and must not be used to interpret it.

The Windows Performance Toolkit had \`wpr.exe\`, \`xperf.exe\`, \`wpa.exe\`, GPUView and \`gpuview\log.cmd\`. Initial GPUView use failed because a single \`log.cmd\` invocation **starts** recording but did not create \`Merged.etl\` until a **second invocation** stopped/merged ETW. Separately, adjacent \`mkdir\` and \`copy\` commands had inadvertently been pasted as one, producing a command-line syntax error. Those were **capture workflow errors**, not browser/driver failures. The user eventually successfully opened \`Merged.etl\` in GPUView.

GPUView screenshots displayed an NVIDIA adapter, Video Decode and Chrome PID 53560 lanes, no reported lost ETW events or buffers in the viewed session, and a local/system memory chart which rose initially then flattened. The screenshot's graph was **not conclusively filtered to one decoder's allocations**, and there was no measured per-resource lifetime or exact overlap attribution for the decode activity. The final assessment was therefore **no accumulating GPU memory leak demonstrated**, not “all allocations safely freed.”

### Lab 23 — deeper source-level ownership and async initialization audit

The fork source was inspected in:
- \`media/gpu/windows/d3d11_video_decoder_wrapper.{cc,h}\`
- \`media/gpu/windows/d3d_texture_wrapper.{cc,h}\`
- \`media/gpu/windows/d3d_picture_buffer.{cc,h}\`
- \`media/gpu/windows/d3d11_video_decoder_backend.cc\`
- \`media/gpu/windows/d3d_video_decoder.cc\`
- \`media/gpu/windows/d3d_video_frame_mailbox_release_helper.cc\`
- \`media/filters/decoder_stream.cc\`

**Candidate 23A, scoped buffer pointer:** \`ScopedD3D11DecoderBuffer\` owns a \`raw_ptr<D3D11VideoDecoderWrapperImpl>\`; its destructor \`Commit()\` dereferences owner/context/decoder. In ordinary ownership, the wrapper destroys its stored bitstream buffer early, and the remaining scoped buffers appear local. No escaped scoped buffer or UAF has been demonstrated.

**Candidate 23B, async GPU init completion:** \`DefaultD3DTextureWrapper::Init()\` creates a \`SequenceBound<GpuResources>\`, with weak-pointer-posted completion/error callbacks. GPU initialization early returns for missing command-buffer helper, failed shared-handle creation, failed client shared-image creation, and failed video representation production. These branches run \`on_error_cb\` but **do not invoke the success callback** \`gpu_resource_init_cb\`. In backend initialization, \`add_client_use()\` occurs before the asynchronous call; \`PictureBufferGPUResourceInitDone()\` ordinarily decrements it. Hypothesis: failed async init might leave the use count nonzero, stall decode, or leave error propagation incomplete. **Not proven reachable as a user-visible stall, leak, or native bad access.** When a failed wrapper is discarded, ordinary reference-counting and callback cancellation may make the difference unobservable.

**Candidate 23C, ProcessTexture return contract:** \`DefaultD3DTextureWrapper::ProcessTexture()\` has a source comment \`TODO(liberato): Return the error.\`, then sets \`shared_image_dest = shared_image_\` and returns \`kOk\` without checking \`received_error_\`. Decoder output subsequently creates a \`VideoFrame\` and accesses shared image fields. A synthetic direct call against an uninitialized wrapper would *not* prove the real decoder reaches this path; the busy/use-counter gate may prevent output until successful initialization. **No reachable invalid dereference shown.**

**Crucial separation:** the observed Issue B 960×120 config-count failure happens **before new picture-buffer initialization**, so candidates 23B and 23C are **not demonstrated to explain the original video failure**.

### Lab 24 — committed unit test and stock-browser WinDbg check

A source-only test \`D3DVideoDecoderTest.Lab24ZeroConfigsDuringRecreationReturnsDecodeError\` was committed to \`media/gpu/windows/d3d_video_decoder_unittest.cc\` on commit \`756cce8b3ef3b7d6f4786c3390ce7f3cdf74102e\`. Its mock fixture uses H264 deliberately to exercise the state machine for \`S_OK\` + zero configurations. **It was not built, compiled, executed, or shown to pass.** The planned deterministic async-init error injection tests were **designed/documented but not implemented**.

The researcher did not have \`autoninja\` or \`out\Default\media_unittests.exe\` installed and chose not to download a full build at the planned deadline. Instead WinDbg was attached to stock Chrome. The uploaded file \`windbg-native-faults_e5cc_2026-11-15_19-48-48-502.txt\` contains essentially:
\`\`\`text
0:098> g
(d138.dcb4): Break instruction exception - code 80000003 (first chance)
ntdll!DbgBreakPoint:
00007ff9\`15833dc0 cc              int 3
0:098> .logclose
\`\`\`
This is **\`0x80000003\`**, a debugger breakpoint—not a \`0xC0000005\` access violation. The log is only eight lines; no \`.ecxr\`, \`!analyze -v\`, attributable fault stack, or proven playback coverage was included. Therefore it is **not evidence of native exploitation**, but also cannot prove that no fault occurred outside the logging window.

### Lab 25 — stop rule and disposition

The Lab 25 cutoff was deliberately imposed to avoid endlessly repeating stress or memory-monitor tests. The final source report was committed at \`docs/vp9-boundary-research/LAB25_FINAL_ASSESSMENT.md\`, commit \`ac6f379435c8acf5fa1ef6a8ca245e2eecebb6b7\`. Finding A and Finding B remain **functional compatibility/recovery candidates**. No reportable security vulnerability, UAF, native crash, or sandbox crossing was established. The cutoff is a scope decision, **not** mathematical proof of safety.

## 5. Root causes, uncertainty, and non-findings

| Claim | Final category | Important limitation |
|---|---|---|
| Valid odd-size VP9 can be software-decoded, yet tested NVIDIA D3D11 path fails | **Reproduced** | Exact same-call failing odd NV12 descriptor missing |
| Odd NV12 backing size is likely causal | **Strong hypothesis** | Same E_INVALIDARG could result from other invalid descriptor fields |
| 960×128→960×120 hardware transition fails | **Reproduced** | Other hardware/driver versions not tested |
| D3D11 query 128px→120px is S_OK with counts 2→0 | **Directly measured** | Zero count may be a correct indication of unsupported size |
| Chromium HTML playback returns PIPELINE_ERROR_DECODE | **Reproduced** | Fallback policy/expected behavior remains under review |
| Chromium never supports decoder reselection | **Incorrect** | Demuxer config changes have a separate selection path |
| A post-output decode error always triggers software fallback | **Incorrect** | Observed branch gates fallback on whether a prior frame was produced |
| 1,050 native allocations succeeded | **Not shown** | 1,050 entry events; HRESULTs not recorded for every call |
| The 8×8 R8 texture was the VP9 failing surface | **Disproved by descriptor** | It is not an NV12 VP9 output request |
| GPU process crashed due to the video | **Not established** | Some debugger/PID observations were confounded |
| GPU leaked persistently | **Not established** | Aggregate counters are not per-resource lifetime traces |
| An async completion path is suspicious | **Source-level hypothesis** | Independent trigger/instrumentation not run |
| Browser RCE, UAF, memory corruption, sandbox escape, cross-decoder contamination | **Not demonstrated** | Absence of a demonstration is not a guarantee |
| Discord itself has an attachment-processing security bug | **Not demonstrated** | Reproduction primarily characterized Chromium hardware path |

## 6. Recovery and fallback architecture

Chromium's normal HTML media \`DecoderStream\` is distinct from direct WebCodecs \`VideoDecoder\`. The former contains \`ReinitializeDecoder()\`, \`BeginDecoderSelection()\`, \`ResumeDecoderSelection()\`, and \`CompleteDecoderReinitialization()\`. One observed decode-error branch, subject to the \`kFallbackAfterDecodeError\` feature, permits re-selection **only when \`!decoder_produced_a_frame_\`**. Once a video frame has been produced, a later decoder error often leads to the terminal pipeline error path rather than automatic reselection.

Demuxer-driven configuration changes have different reinitialization semantics. In-band VP9 coded-size changes need not appear as demuxer container-track metadata changes. This distinction explains how a browser can support fallback in general and still stop on a later decoder-internal transition. Without matched internal state traces, whether a different policy is desirable or possible is an engineering question rather than a confirmed specification violation.

## 7. Reusable test protocol for other file types (future work)

For the next file format (including SVG, WebP, AVIF, APNG, PDF, GIF, MP4, alternate WebM codecs, or other encoded attachments), **do not automatically assume the VP9 D3D11 findings transfer**. Rebuild the test around the format's own parsing/execution/rendering path.

**7.1 Establish the input boundary.** Determine what the application does with the file: upload checks, MIME/content-type sniffing, storage/CDN transformations, attachment proxying, browser fetch, sandboxed document viewer, thumbnail extraction, decoder selection, external-app handoff, and final rendering. These may be different across desktop, web, mobile, and file preview; do not generalize from a Discord CDN URL to every Discord client.

**7.2 Preserve provenance.** Record original SHA-256, size, parser metadata, container headers, in-band dimensions, frame/chunk/table offsets, synthetic vs original payloads, tool/encoder versions, and whether any remux/transcode altered metadata. Keep unmodified input and minimized reproducer separately, and hash both. Do not confuse generated output/frame fingerprints with input-byte identity.

**7.3 Define independent controls.** Test minimal valid file, minimally changed suspicious file, adjacent boundary values, a software decoder where available, hardware-decoder selection verified in logs, a different browser/driver, and a clean profile. Avoid controls that silently change the underlying decoder or container configuration.

**7.4 Instrument by layer.** Browser DevTools/WebCodecs and \`chrome://media-internals\` establish application and decoder behavior; WinDbg can inspect Windows API arguments/returns; GPUView/ETW helps with GPU scheduling/aggregate allocation; source code and ASan-enabled builds test actual object lifetime. A UI freeze during debugger stop is not a native crash. Correlate every counter sample and event with profile, PID/creation time, timestamps and playback markers.

**7.5 Distinguish data/active content.** In particular, an **SVG document is an XML-based vector format with optional scripting and external resource features**, but script evaluation depends heavily on how it is embedded, CSP, sanitization, browser policies and origin restrictions. It is *not* equivalent to WebM decoder byte parsing, and the mere presence of \`<script>\` does not demonstrate script execution inside Discord. Testing should first map **rendering context and enforcement**; then examine safe, permissioned behavior and content type handling. A file format's broad feature set is not a finding.

**7.6 Model the threat boundary.** Enumerate who controls the bytes, the access rights required, whether preview is automatic, sandbox/process privileges, whether a failure is recovered, whether persistent state crosses documents/tabs/processes, and whether the result is simply visual degradation. Decoder errors and blocked scripts are not security-impact proof.

**7.7 Triage consequences by evidence.** Categories: expected rejection, compatibility bug, resource leak with repeated attributable growth, process crash with dump/stack, memory safety finding under sanitizer/debugger, information exposure across isolation boundary, privilege crossing, or code execution. A crash needs a causally linked stack and clean repro; security classification demands an actual security boundary/impact.

**7.8 Minimize carefully.** Strip unrelated metadata and frames incrementally, preserve the failure across exact payload reductions, recheck the parser and in-band fields after every remux, and record any reduced case that no longer reproduces. A single-frame control that selects software instead of hardware is not a matched test.

**7.9 Stop conditions.** Agree in advance on scope and maximum rounds, maintain evidence confidence, stop repeating stress loops if metrics stabilize, and conclude with a clear verified/non-verified disposition. Preserve promising hypotheses even if not tested before the cutoff.

## 8. Specific experiments worth revisiting later

These are **future work**, not tasks performed by this report:

1. **Issue A:** reproduce odd-source 960×129 in a truly isolated WinDbg session and capture one full \`D3D11_TEXTURE2D_DESC\` plus corresponding \`CreateTexture2D\` HRESULT, including format/bind/share flags and stack; test safe padded backing dimensions only in a suitable native test build.
2. **Issue B:** normalize the two-keyframe WebM track metadata and verify A/C decoder identity in HTML playback; contrast actual original and remuxed files; evaluate software fallback after already-output hardware frames; verify the \`decoder_produced_a_frame_\` gate.
3. **Version/vendor:** compare another NVIDIA driver and a genuinely selected AMD hardware decoder, second Chrome revision and non-Windows backend where practical.
4. **Lab 23B:** deterministic \`GpuResources\` helper failure, callback completion and picture-buffer usage accounting; identify whether object destruction safely cancels pending callbacks.
5. **Lab 23C:** test whether a legitimate decoder path ever calls \`ProcessTexture()\` when \`received_error_\` is set and \`shared_image_\` is null; require a stack or a unit/instrumented build before claiming a null dereference.
6. **Input-interface coverage:** for a new format identify which Discord client/viewer actually processes bytes and which transformations occur server side; do not infer client RCE from original upload bytes alone.
7. **Resource and data isolation:** if testing decoder A and B concurrently, use **distinct identifiable frame content** and distinct hashes; identical-content controls cannot detect output swapping/cross-contamination. Keep timing, per-decoder reference images, buffer identity and provenance separate.
8. **GPU pressure:** distinguish 5,000 concurrent synthetic clones or repeated errors from a genuine GPU residency/synchronization test; increase stress only when a bounded new hypothesis and observable are defined. “No visual flicker” is not a memory-safety guarantee.
9. **Repro archival:** save source file, minimized WebM, relevant unredacted local logs securely, sanitized report extracts, hash manifest, known-good controls and test harness versions; avoid embedding unrelated Discord conversations, tokens or credentials in public GitHub issues.

## 9. WinDbg / GPUView runbook retained from chat

**Find matching isolated GPU process (PowerShell):**

\`\`\`powershell
Get-CimInstance Win32_Process |
  Where-Object {
    $_.Name -eq "chrome.exe" -and
    $_.CommandLine -match '--type=gpu-process' -and
    $_.CommandLine -match 'Lab21\\chrome-profile'
  } |
  Select-Object ProcessId, ParentProcessId, CreationDate
\`\`\`

**Stock Chrome WinDbg basics:** \`.symfix\`, \`.reload\`, \`sxe av\`, \`sxe gp\`, \`sxe ii\`, \`.logopen /t <existing-directory>\`, \`g\`. If an attributable exception occurs: \`.exr -1\`, \`.ecxr\`, \`kv\`, \`!analyze -v\`. Finish with \`.logclose\` and \`.detach\`. Ordinary \`0x80000003\` debugger break is not a \`0xC0000005\` AV. Use public symbols to resolve native functions anew per process.

**D3D11 configuration-count capture (from Lab 18):**

\`\`\`text
bc *
bp d3d11!CDevice::GetVideoDecoderConfigCount ".if (dwo(@rdx+0x14) == 0x78) { .echo LAB18_CONFIG_120; r; dd @rdx L7; kb 8 } .else { gc }"
bl
g
\`\`\`
On hit, before stepping out, preserve the output pointer: \`r $t0 = @r8\`; then \`dd @$t0 L1\`; \`gu\`; \`r rax\`; \`dd @$t0 L1\`. Repeat with height \`0x80\` for 128px comparison. Registers are caller-volatile; reading \`@rdx\` at an unrelated caller *after* stepping out is not a safe way to recover the original descriptor.

**GPUView/ETW workflow:** run elevated \`cmd\`, \`cd /d "C:\Program Files (x86)\Windows Kits\10\Windows Performance Toolkit\gpuview"\`, \`log.cmd\` to start, reproduce once, **\`log.cmd\` again** to stop/merge, then \`dir Merged.etl\`. Copy the verified file with \`copy /Y\` to \`C:\Users\Admin\Downloads\Lab22\diagnostics\Merged.etl\`; open with \`GPUView.exe "<absolute-etl-path>"\`. Only analyze if actual file exists. Excessively long captures can be hard to attribute and may wrap circular buffers. GPUView alone does not expose all C++ object identities.

## 10. Files, exports, and repository landmarks

### Original or locally produced evidence (not necessarily checked into repository)

- \`funkyms_4.webm\` and its SHA-256 (above).
- \`funkyms-vp9-vendor-comparison-v15.json\`
- \`funkyms-vp9-boundary-v16.json\`
- \`test_A_960x128_only.webm\`, \`test_B_128_to_120.webm\`, \`test_C_960x120_only.webm\`
- \`lab19-playback-results (1).json\` and later Chrome playback JSON.
- \`chrome-gpu-monitor.csv\`; \`gpu-process-samples(1).csv\`, \`gpu-process-samples(3).csv\`, \`gpu-process-events(2).csv\`, \`gpu-process-events(3).csv\`.
- \`C:\Users\Admin\Downloads\Lab22\B-transition\gpu-memory.csv\`, \`events.csv\`, \`playback-markers.txt\`; ETW \`Merged.etl\` in Lab22 diagnostics (successfully opened in GPUView).
- GPUView screenshots dated 2026-11-15 ~19:18–19:30 local.
- \`C:\Users\Admin\Downloads\Lab24\windbg-native-faults_e5cc_2026-11-15_19-48-48-502.txt\`, eight-line debugger log.
- Various \`chrome://media-internals\` exports, including the Lab 18 two-frame media entry described in the original reports.
- \`LAB17\` optional source instrumentation and \`LAB24\` unit test committed in the fork but **not executed**.

These filenames are preserved as retrieval identifiers, *not claims that this consolidated document packages all binary payloads or raw ETL/JSON files*. The appended reports and code hyperlinks preserve evidence provenance; the original local files should be kept separately if available.

### Repo navigation

- Research index: \`docs/vp9-boundary-research/README.md\`.
- Source: \`media/gpu/windows/\` and \`media/filters/decoder_stream.cc\`.
- Lab 24 test: \`media/gpu/windows/d3d_video_decoder_unittest.cc\`.
- Historical final result: \`docs/vp9-boundary-research/LAB25_FINAL_ASSESSMENT.md\`.
- This **master dossier**: \`docs/vp9-boundary-research/COMPLETE_RESEARCH_ARCHIVE_LABS15-25.md\`.

## 11. Evidence-quality rules for the next investigation

- A media decoder returning an error is a **user-visible functional effect**; don't label it a crash unless a process fault is attributable.
- \`S_OK\` and *zero supported configurations* can coexist; check both HRESULT and output parameters.
- Decoder/GPU capability probing is not a guarantee every later bitstream or surface allocation will succeed.
- Container metadata, in-band codec settings, decoded visible rectangle, padded GPU storage and displayed UI resolution are **different quantities**.
- A debugger stop often pauses the whole target: windows can look frozen. Record \`g\` and capture intervals before making crash claims.
- Aggregate GPU memory trends cannot demonstrate absence of individual leaked allocations.
- Source containing a TODO, a \`raw_ptr\`, or a possible callback mismatch identifies an **audit target**, not necessarily a reachable vulnerability.
- The research branch contains local changes and may not match installed Chrome. Compile/run status must be explicit.
- Cross-format extensions should **start with file-format trust and rendering differences**, not assume VP9 buffer or D3D11 behavior applies to SVG or other document types.
- For security findings, preserve a short clean reproduction, exact version, stack/dump, impact boundary, and responsible disclosure path. For compatibility findings, provide expected/actual playback and a minimal file with source provenance.

## 12. Bottom-line archival conclusion

**Issue A:** reproducible hardware VP9 odd-coded-dimension failure on the tested NVIDIA D3D11 setup, with software decode success and a likely NV12 storage-dimension constraint; native descriptor-level causality not finally settled.

**Issue B:** reproducible hardware VP9 in-band 960×128→960×120 configuration failure, D3D11 **S_OK/config count 2→0**, leading to ordinary HTML \`PIPELINE_ERROR_DECODE\`; exact desired fallback behavior not adjudicated.

**Security:** no user-content-triggered native memory-safety failure, process compromise or code execution was validated through Lab 25. The investigation was intentionally closed with testable follow-up hypotheses and full evidence context intact.

---

# Part II — Full Historical Reports, Preserved in Source Order

The reports below are copied in full from the branch at archive creation time. Their individual title/date/status fields refer to each report's original stage, **not** to the status of this consolidated archive. Relative links within those reports resolve against the research directory when viewing the archive in GitHub. This attachment of the primary reports ensures their detailed tables, text excerpts, caveats and earlier plans remain recoverable even if a summary omits something.


---

## Historical source 1 of 11: `README.md`

Source blob SHA: `1b8741d08cbbebd86b7836675b3970cb8b21adb5`  
Original path: `docs/vp9-boundary-research/README.md`

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
8. [Lab 20 decoder lifetime and GPU-resource audit — source protections, monitoring limits, Lab 21 plan](LAB20_DECODER_LIFETIME_AUDIT.md)
9. [Lab 24 failure-path validation — zero-config characterization test, async-init and ProcessTexture test matrix](LAB24_FAILURE_PATH_VALIDATION.md)
10. [Lab 25 final verdict — reproducible compatibility failure; no demonstrated security vulnerability](LAB25_FINAL_ASSESSMENT.md)

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


---

## Historical source 2 of 11: `ISSUE_A_ODD_NV12.md`

Source blob SHA: `f6e4da283cc623689d185ccc5f751f6cbbe403b7`  
Original path: `docs/vp9-boundary-research/ISSUE_A_ODD_NV12.md`

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


---

## Historical source 3 of 11: `ISSUE_B_RECONFIG_FALLBACK.md`

Source blob SHA: `9865ef8cb9c27d4eb2f94642cfe49dedf9082a1e`  
Original path: `docs/vp9-boundary-research/ISSUE_B_RECONFIG_FALLBACK.md`

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

## Lab 17 stock-Chrome trace — 960×128 boundary observation

A working WinDbg `CreateTexture2D` NV12 logpoint captured **1,050** native request entries from original-WebM playback: width 960 throughout, heights 720→128 in exact 8-pixel decrements (75 distinct heights, 14 requests each). **No 960×120 NV12 allocation request was captured** before the manually stopped recording. This is consistent with, but **does not independently prove**, the existing media-internals diagnosis of an unusable decoder configuration at 960×120. These are allocation attempts, not a record of decoder configuration queries or the actual `DecoderStream` fallback branch. No resource-lifetime fault, GPU crash, or state desynchronization was observed. See [Lab 17 native trace](LAB17_NV12_DIMENSION_TRACE.md).

## 7. Primary references / limitations

- Repository source: `media/filters/decoder_stream.cc` (`OnDecodeOutputReady`, `OnDecodeDone`, `ReinitializeDecoder`, `CompleteDecoderReinitialization`).
- Chromium upstream source: https://chromium.googlesource.com/chromium/src/+/main/media/filters/decoder_stream.cc
- Fallback feature definition: https://chromium.googlesource.com/chromium/src/+/main/media/base/media_switches.cc
- Repository supported-profile checks: `media/gpu/windows/supported_profile_helpers.cc`.
- Reproduction source file SHA-256: `78589f127fe4436f71003f6652ade3b99a85a7691c7d4d1d70aed9f9b3666e10`.

A third-party summary linked Chromium issue 40684757 but issue contents were unavailable for verification. **No claim of an established duplicate, known unfixed bug, or official triage outcome is made here.**

## Lab 18 update — confirmed native D3D11 configuration-count boundary (2026-11-15)

A minimal source-packet-copy WebM containing two consecutive VP9 keyframes (960×128 → 960×120) reproduces the normal HTML player failure. In the uploaded `chrome://media-internals` log, **`D3D11VideoDecoder`** is selected and the two in-band config changes precede `GetVideoDecoderConfigCount failed ... (0x0)`, `D3DStatus::10`, and `PIPELINE_ERROR_DECODE`. The remuxed WebM's initial track metadata still advertises 960×720; this is a known confounder, and the A/C single-frame clips' decoder selection is not yet verified.

**Native WinDbg comparison using the same decoder GUID, NV12 output, and D3D11 function:**

| Request | HRESULT | Configuration count |
|---|---|---|
| 960×128 NV12 | `S_OK` / `0x00000000` | **2** |
| 960×120 NV12 | `S_OK` / `0x00000000` | **0** |

This **confirms the immediate native reason** for the wrapper failure: zero D3D11 configurations for the 120px height, despite API success. It does **not** establish that the driver is defective or that Chromium is obligated to fall back midstream. Prior Lab 17 evidence remains relevant and unchanged. See the complete [Lab 18 native transcript and reproduction notes](LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md).


---

## Historical source 4 of 11: `EXPERIMENTS_AND_LAB17.md`

Source blob SHA: `0d1af16eb315566186926497ae6eb6aed789dae5`  
Original path: `docs/vp9-boundary-research/EXPERIMENTS_AND_LAB17.md`

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


---

## Historical source 5 of 11: `LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md`

Source blob SHA: `49dba93b5bad10cab287abf6df0057bd5f01815c`  
Original path: `docs/vp9-boundary-research/LAB17_WINDBG_STOCK_CHROME_EVIDENCE.md`

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


---

## Historical source 6 of 11: `LAB17_NATIVE_DIAGNOSTICS.md`

Source blob SHA: `2415315af80126bf1206119e1d8ca84ca17b3aa7`  
Original path: `docs/vp9-boundary-research/LAB17_NATIVE_DIAGNOSTICS.md`

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


---

## Historical source 7 of 11: `LAB17_NV12_DIMENSION_TRACE.md`

Source blob SHA: `5e5fc0fd80b87542a9cd794f7d224623850ba829`  
Original path: `docs/vp9-boundary-research/LAB17_NV12_DIMENSION_TRACE.md`

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


---

## Historical source 8 of 11: `LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md`

Source blob SHA: `3e0c7f8bd170b43f51417064d489f33380be6731`  
Original path: `docs/vp9-boundary-research/LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md`

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


---

## Historical source 9 of 11: `LAB20_DECODER_LIFETIME_AUDIT.md`

Source blob SHA: `80a7c1de9c025e012c57e1c23200a282a44f00b5`  
Original path: `docs/vp9-boundary-research/LAB20_DECODER_LIFETIME_AUDIT.md`

# Lab 20 — VP9/D3D11 decoder lifecycle and resource-lifetime audit

**Date:** 2026-11-15 (America/Detroit)  
**Branch reviewed:** `research/lab17-vp9-native-diagnostics` in `Wuhpondiscord/chromium`  
**Status:** decoder incompatibility/recovery failure reproducible; no memory-safety violation demonstrated.  
**Hardware:** Windows 11, NVIDIA RTX 4080 SUPER. Lab 20 playback user agent identifies **Microsoft Edge 157**, not the stock Google Chrome 157 used for earlier Lab 18 evidence.

## Summary

Lab 20 tested whether the Lab 18 VP9 960×128 → 960×120 D3D11 decoder-configuration failure resulted in resource leaks, GPU-process restarts, stale callbacks or an invalid object lifetime. The new 100-iteration browser run returned 100 media decode errors; synchronized process telemetry did not show a clear sustained resource-growth pattern or process-disappearance event in the sampled processes. A static source audit found deliberate ownership and lifetime protections in Chromium's decoder, picture-buffer and callback paths. **No crash, UAF, memory corruption or exploitable security impact has been demonstrated.**

These conclusions must not be broadened: process identity was not conclusively mapped to the Edge test instance, the A/B control experiment was incomplete, and source on this research branch is not established to match the installed browser binary exactly.

## Prior confirmed trigger (Lab 18)

- A single VP9 960×128 keyframe (Test A) and a single 960×120 keyframe (Test C) were controls; the two-keyframe 960×128 → 960×120 file (Test B) reproducibly reported `PIPELINE_ERROR_DECODE`.
- WinDbg captured the same D3D11 decoder GUID and output format NV12. `GetVideoDecoderConfigCount` returned `S_OK` for both queries, with **2 configurations at height 128** and **0 configurations at height 120**.
- The error string `GetVideoDecoderConfigCount failed: The operation completed successfully. (0x0)` reflects the failed *availability check*, not a failed HRESULT. This is a handled API response, not by itself evidence of a native crash.
- See [Lab 18](LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md).

## Lab 20 playback dataset

Uploaded file: `lab19-playback-results (1).json` (generated by the Lab 19 HTML harness during Lab 20).

- 100 iterations, **100 `error` outcomes**, `mediaErrorCode=3`; no successes in the uploaded run.
- **84** `GetVideoDecoderConfigCount failed: ... (0x0)` diagnostics; **16** generic `video decode error!` diagnostics.
- Browser user agent: `Edg/157.0.0.0` / Chromium 157.
- Record timestamps: **2026-11-15 22:23:13.974Z through 22:23:24.879Z** = **17:23:13.974–17:23:24.879 EST**.
- Browser still reported 960×128 dimensions at error; this does not rule out the attempted in-band 960×120 transition.
- Error-string variation does not establish distinct native bugs.

**Control limitation:** there is no separate uploaded successful Test A run with comparable per-iteration JSON/decoder-selection evidence. The directory label `A-control` describes the *monitor session*, not proof that the synchronized playback file was a successful Test A control.

## PowerShell process telemetry

Uploaded datasets:
- A-labeled samples: `gpu-process-samples(1).csv`, **958 rows**, from **17:22:14.617 to 17:25:14.421 EST**; overlaps the playback run.
- B-labeled samples: `gpu-process-samples(3).csv`, **1030 rows** in final uploaded copy, from **17:24:27.320 to 17:27:34.650 EST**; starts about 63 seconds after the above playback run ended. Earlier console output showed 330 rows while this file was still growing.
- Events: `gpu-process-events(2).csv` for A and `gpu-process-events(3).csv` for B (the B event log repeats `GPU_PROCESS_SEEN` at 17:25:15 without a reported disappearance).
- Both monitors sampled the same identities: `48160|11/15/2026 13:10:09` and `35908|11/15/2026 14:12:54`. The monitors overlap with each other, not two isolated browser sessions.

### Whole A-labeled interval

| Metric | PID 48160 | PID 35908 |
|---|---:|---:|
| First private bytes | 728,817,664 (~695.1 MiB) | 176,050,176 (~167.9 MiB) |
| Last private bytes | 723,861,504 (~690.3 MiB) | 182,829,056 (~174.4 MiB) |
| Peak private bytes | 844,554,240 (~805.4 MiB) | 201,490,432 (~192.2 MiB) |
| First handles | 1186 | 1013 |
| Last handles | 1189 | 1017 |

There were no `GPU_PROCESS_DISAPPEARED` events in the logged intervals. This rules out only a **logged disappearance of those sampled identities during those windows**, not native faults elsewhere, missed sub-sample restarts, driver errors, or leaks of resources outside the CPU private-memory/handle counters.

The A-labelled interval contains the 100 playback errors, but the sample processes have not been conclusively attributed to that Edge test profile. Whole-window first/last differences are not a controlled measure of decoder-specific resource usage. There is no basis for claiming a leak, nor for proving that none is possible.

## WinDbg status

Lab 19 WinDbg logs contained only debugger `INT 3`/break instruction events, logging commands, and no recorded AV/guard-page/illegal-instruction fault. The user subsequently detached with `.detach` (`NoTarget` warning after detachment is expected). No new synchronized Lab 20 native exception log or instrumented-build trace was uploaded. Thus **no Lab 20 native crash was captured**, but lack of a trace is not proof of memory safety.

## Source-level lifecycle findings

Source reviewed: `media/gpu/windows/d3d_video_decoder.cc`, `d3d_video_decoder.h`, `d3d_picture_buffer.cc`, `d3d11_video_decoder_backend.cc`, `d3d_video_decoder_wrapper.cc`, and `media/filters/decoder_stream.cc`. These sources are on the research branch and include local LAB17 instrumentation.

### B1 — Destroy-then-recreate path

`D3DVideoDecoder::DoDecode()` processes a `kConfigChange` by updating profile/dimensions then calling `RecreateDecoderWrapper()` (`d3d_video_decoder.cc`, approximately lines 633–681). `RecreateDecoderWrapper()` (approximately 216–282) does the following **before** new wrapper creation:
```cpp
d3d_video_decoder_wrapper_.reset();
ClearPictureBuffers();
texture_selector_.reset();
decoder_configurator_.reset();
```
It then requests a configurator and decoder wrapper. Errors call `NotifyError` and return. It is therefore not transactional: the old wrapper is released before replacement creation succeeds. **This is not itself a UAF**; it defines the lifecycle boundary to verify.

### B2 — Ref-counted picture buffers and explicit output-view cleanup

`D3DPictureBuffer` inherits `RefCountedDeleteOnSequence` (`d3d_picture_buffer.cc`, ~22–35). The decoder destructor explicitly notes that GPU-resource initialization tasks and a client `VideoFrame` release callback can keep a buffer alive after `picture_buffers_.clear()` (`d3d_video_decoder.cc`, ~152–172). `ClearPictureBuffers()` (~827–837) calls `ResetOutputView()` for each buffer, clears the collection, and releases backend resources. `D3D11PictureBufferImpl::ResetOutputView()` (~130–132) resets its COM `ID3D11VideoDecoderOutputView` member. An individual buffer may remain alive while its former output view has been relinquished. Any remaining access must respect this split lifetime.

### B3 — Weak-pointer and callback ownership protection

In `D3DVideoDecoder::OutputResult()` (~896–904), the deferred frame release callback captures a `weak_factory_.GetWeakPtr()` for the decoder and a `scoped_refptr<D3DPictureBuffer>` for the picture buffer. This protects against a straightforward callback dereference of a destroyed decoder and retains the buffer object. `DecoderStream` uses weak-pointer factories for selection/fallback callbacks; its state machine counts pending decode requests and propagates terminal errors. `D3DVideoDecoder::PostDecoderStatus()` dispatches failure to pending decode callbacks (see ~935–967). **These are protective designs, not proof that every asynchronous backend path is correct.**

### B4 — Backend release and GPU execution

`D3D11VideoDecoderBackend::ReleaseResources()` (~42–49) resets `picture_buffer_texture_` and calls `device_context_->Flush()`. D3D11 `Flush()` queues/submits work; it is **not** a general guarantee that GPU execution has finished. A source audit must trace synchronization and ownership of GPU work and shared-image resources before concluding that a retained output resource is safe.

### B5 — Asynchronous buffer initialization has an explicit usage-counter ordering invariant

In `D3D11VideoDecoderBackend::CreatePictureBuffer()` (around lines 360–374), the source comments that `add_client_use()` must happen *before* `InitD3D11()` because GPU-thread initialization work might already be scheduled when the initializer returns an error. The callback later removes client use. This is directly relevant to teardown and async completion, and suggests the code explicitly accounts for late callbacks. Inspecting `D3DTextureWrapper`'s callback and internal resource lifetime is the next source-level step.

### B6 — Base wrapper destructor is not the concrete D3D11 implementation

`d3d_video_decoder_wrapper.cc` implements shared logic and has a defaulted base destructor; it is **not** the complete `D3D11VideoDecoderWrapper` implementation. Follow `media/gpu/windows/d3d11_video_decoder_wrapper.{cc,h}` and texture wrapper implementations before making assertions about the concrete COM/GPU object destruction path.

## Current hypothesis table

| Hypothesis | Evidence-based status |
|---|---|
| D3D11 configuration unavailable at height 120 | **Confirmed** in Lab 18 native capture |
| 100/100 handled media decode errors | **Confirmed** in Lab 20 playback |
| Old wrapper reset precedes replacement creation | **Confirmed** in reviewed source |
| Picture buffers can outlive decoder, with explicit ref-counting | **Confirmed**, intentionally supported |
| Stale decoder callback immediately implies UAF | **Weakened** by WeakPtr/ref-count safeguards |
| GPU resource/driver operation survives teardown unsafely | **Unresolved**, needs concrete wrapper/texture audit |
| Persistent process-memory or handle leak | **Not demonstrated**; monitoring not isolated/controlled |
| GPU-process restart | **Not logged** for sampled identities |
| Native UAF, AV, memory corruption, sandbox escape | **Not demonstrated** |
| VRP/CVE-worthy impact | **Not established** |

## Recommended Lab 21 — D3D11 resource ownership and decoder-generation validation

**Goal:** Establish, by exact source-to-build mapping and bounded instrumentation, whether pending async operations and GPU resources retain valid ownership across the failed 960×128→960×120 decoder recreation.

1. **21A: Version matching.** Identify the installed Chrome/Edge full version and matching Chromium source revision. Diff the research branch's lifetime-sensitive methods against the corresponding released implementation; explicitly separate existing upstream safeguards from locally added research code.
2. **21B: Concrete implementation audit.** Trace `D3D11VideoDecoderWrapper`, `D3DTextureWrapper` implementations (especially `DefaultD3DTextureWrapper::GpuResources`), `D3DPictureBuffer`, shared-image/mailbox release helper and fence/keyed-mutex paths. Produce an object ownership, last-use, and release/synchronization table. Determine what happens after `ResetOutputView()`.
3. **21C: Targeted instrumentation in a local debug build.** Record decoder generation, wrapper identity, buffer IDs, callback begin/end, pending decode counts, weak-pointer invalidation, output-view reset, COM-resource release, and completion/fence milestones. Use sequence assertions and bounded logs, without artificial sleeps or disruptive global breakpoints.
4. **21D: Matched A/B runtime test.** On separate, identified browser profiles with matching binaries and GPU PIDs, execute Test A and Test B with synchronized before/during/after logging; export both playback JSON files. Test only the minimal transition first, then a small bounded repetition count.
5. **21E: Security triage.** Escalate only on a reproducible invalid object access, broken lifetime/synchronization invariant, sanitizer finding, process fault attributable to the test, or security-boundary violation. Otherwise document an expected-handled decoder error/compatibility defect for a normal Chromium issue.

### Success criteria

- Per-event decoder generation and resource identity can reconstruct recreation/failure/cleanup ordering.
- Invariant assertions either hold across control and failing transitions or produce a precisely localized failure with an attributable stack.
- No CVE, exploitability, or bounty-impact claims without demonstrable security impact.

## Limitations and ethics

Only local, authorized testing is represented here. No remote Discord-delivery exploit, renderer escape, native code execution, UAF, or GPU process compromise has been shown. GPU-driver behavior and COM objects cannot be fully inferred from browser JS or process memory alone. A source branch containing new fixes is not proof those fixes exist in the installed binary. This report preserves the distinction between a robust reproducible failure and a security vulnerability.

## References

- [Lab 18 native D3D11 configuration-count evidence](LAB18_TWO_FRAME_D3D11_CONFIG_COUNT.md).
- [Issue B — reconfiguration and fallback](ISSUE_B_RECONFIG_FALLBACK.md).
- Source files listed in the source-level findings above, at `research/lab17-vp9-native-diagnostics`.
- Local artifacts: `lab19-playback-results (1).json`, `gpu-process-samples(1).csv`, `gpu-process-samples(3).csv`, `gpu-process-events(2).csv`, `gpu-process-events(3).csv`, prior Lab 19 WinDbg text logs. These raw logs are not asserted to have been committed with this report.


---

## Historical source 10 of 11: `LAB24_FAILURE_PATH_VALIDATION.md`

Source blob SHA: `e17422d1b5a7c3419100b29305e6c52c8af88f3d`  
Original path: `docs/vp9-boundary-research/LAB24_FAILURE_PATH_VALIDATION.md`

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


---

## Historical source 11 of 11: `LAB25_FINAL_ASSESSMENT.md`

Source blob SHA: `a96bb81c0d0b131f1bb0bf5e132b6e7b829e8524`  
Original path: `docs/vp9-boundary-research/LAB25_FINAL_ASSESSMENT.md`

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
