---
name: Local STT
description: Implement web application speech to text based in local models
---

# Local STT: Mooi implementation and replication specification

## 1. Scope and provenance

Extracted on 2026-10-28 from Mooi `main`, commit `2c347705f89c900f7ac606d4f9f8baa896c9d7b0`, after `git pull --ff-only` reported already up to date. This describes checked-in behavior, not an inspection of live environment overrides. No application was started or deployed during this extraction.

The goal is to reproduce Mooi's streaming dictation quality, responsiveness, and browser resilience in another application. Framework, language, and component names are incidental; audio processing, model/runtime selection, wire semantics, lifecycle ordering, and recovery behavior are material. Identical code cannot guarantee identical speed or accuracy across microphones, languages, hardware, browser processing, and host load.

“Local” means inference on the machine running the speech service, without a hosted transcription API. It does **not** mean browser-side inference. A phone or remote browser sends audio over the network to the Mooi host. Model installation needs network access unless artifacts are provisioned beforehand. Audio and transcripts are transient; the bridge persists only content-free metrics.

## 2. Architecture and essential boundaries

```text
Microphone
  → getUserMedia (mono, echo cancellation, noise suppression)
  → shared native-rate AudioContext
  → AudioWorklet: downsample, PCM16, 100 ms frames
  → browser WebSocket /api/stt/stream
  → same-origin gateway (WebSocket upgrade)
  → mic-speech bridge: admission, onset gate, protocol translation
  → loopback WebSocket /v1/audio/transcriptions/realtime
  → resident NeMo-Speech.cpp model
  → engine deltas/completions
  → bridge full-text snapshots
  → selection-owned live draft in the original text field
```

There is no MediaRecorder container upload, browser SpeechRecognition API, browser model download, cloud recognizer, or LLM cleanup step in this path. Native inference runs once per service process and stays loaded across dictations. Each dictation gets its own engine socket and transcript state. One process-wide lock admits one dictation at a time.

The current adapter is Python 3.13+, FastAPI/ASGI, Uvicorn, `websockets`, Pydantic and pydantic-settings, with exact Python dependencies in `mic-speech/uv.lock`. The browser is TypeScript/React with a plain JavaScript worklet. Another implementation can replace these languages/frameworks while retaining the contracts below.

## 3. Model, runtime, reproducibility, and alternatives

### Exact baseline

- Runtime: NVIDIA **NeMo-Speech.cpp 0.1.0**, native C++ inference.
- Model setting: **`nemotron-3.5`**, resolved through the pinned runtime model index, or an explicitly supplied local GGUF path.
- Language: **`es-ES`** by default; automatic punctuation requested on every session.
- Optional vocabulary phrases: comma-separated `SPEECH_CONTEXTS`, forwarded with boost **3.0**.
- Linux container: CPU archive, both amd64 and arm64 supported. Native Apple Silicon provisioning selects the Metal archive and defaults to `--device metal`; other systems default to `auto`. Running the Linux image on a Mac does not turn it into the native Metal build.
- Initial model download is described by the repository as approximately **742 MB**. It is cached persistently, not downloaded per dictation.

The runtime's index owns the model revision, file size, and checksum. Mooi does not separately declare the exact GGUF revision/checksum or quantization in its settings. For a reproducible handoff, preserve the 0.1.0 binary and its model index/cache, and record the resolved model artifact and checksum; do not resolve the alias using an arbitrary newer runtime. A local GGUF path bypasses indexed provisioning and therefore requires separately verifying provenance and checksum.

Runtime archive URL template:

```text
https://github.com/NVIDIA/NeMo-Speech.cpp/releases/download/v0.1.0/nemo-speech-0.1.0-{asset}.tar.gz
```

| Asset | SHA-256 pinned by Mooi |
| --- | --- |
| macos-aarch64-metal | f1dff4f9dd9c96214f8cb78b982812459132df8a4ad1a42409fd94de4a366244 |
| macos-x86_64-cpu | 042a4612e07460fab6a39b5d862aa1e39d0ac3eaedfdb979f3f5fc12de510c20 |
| linux-x86_64-cpu | 0f74131d631ad2c694cf0ec53490866bb6461147959589a69fb6fc231944065b |
| linux-aarch64-cpu | 0e4112255d566de7bdd142f239e984995c4447103ba8feb41f2bb5c559d561d3 |

### Current efficient-model context, separate from Mooi behavior

Upstream documentation checked on 2026-10-28 describes Nemotron 3.5 as a multilingual 0.6B cache-aware FastConformer-RNNT with language prompting. It also offers the English-only Nemotron Speech Streaming 0.6B. Parakeet TDT 0.6B v3 is an offline alternative and explicitly rejects streaming requests; it is not a drop-in replacement for this pipeline. These descriptions concern upstream documentation, which can evolve independently of the pinned release. [NVIDIA model guide](https://github.com/NVIDIA/NeMo-Speech.cpp/blob/main/docs/asr/models.md).

There is no evidence here establishing a universally “most efficient” current model. Preserve `nemotron-3.5` for behavioral parity. Evaluate alternatives against target language, first-word retention, punctuation, real-time factor, memory, first-partial latency and stop-to-final latency on the intended host. Upstream publishes hardware-specific throughput examples for another Nemotron streaming configuration; those are not Mooi benchmarks. [NVIDIA runtime overview](https://github.com/NVIDIA/NeMo-Speech.cpp).

The current upstream CLI can expose indexed model metadata through `nemo-speech --json model list`; confirm availability on the pinned binary before using it to inventory artifacts. Cache location is overridden with `NEMO_SPEECH_MODEL_DIR`. [NVIDIA CLI guide](https://github.com/NVIDIA/NeMo-Speech.cpp/blob/main/docs/cli.md).

## 4. Browser capture and processing

### Startup order is a requirement

1. Snapshot the target field, original value and selection before focusing it.
2. Acquire the optional browser audio-session lease: remember its previous type, set `play-and-record`, restore when the last lease ends. Unsupported API/type errors are ignored.
3. Request `getUserMedia` inside the initiating user gesture, with `audio: {channelCount: 1, echoCancellation: true, noiseSuppression: true}`. No explicit device sample rate or automatic-gain setting is supplied.
4. Only after permission resolves and the microphone is open, start the WebSocket and acquire/start the audio graph. Opening the mic changes audio routing; do not resume the graph before that change.
5. Reuse one page-wide `AudioContext({latencyHint: 'interactive'})` at the native rate. Load the worklet once. Serialize suspension transitions and reference-count users.
6. Resume with a 5-second bound and require `running`. Connect microphone source → mono explicit-channel worklet → destination. The output is silence; connecting to the destination keeps processing active without playing the microphone.
7. Queue audio captured before server `ready`; flush in capture order when it arrives.
8. Announce `recording` only after both server readiness and at least one nonzero PCM sample. Until then, show connecting.

Healthy contexts are suspended between uses, not closed each time: WebKit can deliver silence when repeatedly recreating contexts. Discard a failed context and rebuild once as a recovery exception. Late permission results after disposal must immediately stop their tracks.

### Worklet algorithm and audio contract

The worklet runs outside the UI thread. It reads the first input channel and keeps a fractional phase accumulator across render calls. For each native-rate sample, accumulate sum/count and increment phase. When phase reaches `nativeRate / 16000`, emit the average, subtract that ratio (repeating if necessary), and reset sum/count. This is box-filter averaging, not a high-quality general-purpose resampler; reproduce it for parity rather than silently changing the frontend signal.

Clamp each output sample to [-1, 1]. Quantize with `round(value * 32768)` for negative values and `round(value * 32767)` otherwise. Send **1,600 signed 16-bit samples per frame**: mono **16 kHz**, **100 ms**, **3,200 bytes**, nominal **32,000 bytes/s**. The wire contract is little-endian PCM, without WAV headers or compression. Mooi uses native `Int16Array` buffers on supported browser platforms; a portable implementation should explicitly preserve little-endian serialization.

Transfer full buffers through the worklet message port rather than copying them. On `stop`, emit any partial PCM frame and then `stopped`; the incomplete resampling accumulator is not separately flushed. On `close`, cease processing. Return false after stopping, or after 10 seconds without input, so orphan processors do not accumulate on the shared graph.

### Audio health, interruption, and backpressure

| Mechanism | Current value/behavior |
| --- | --- |
| First nonzero sample deadline | 1,500 ms after capture starts |
| Running zero/no-frame stall | 3,000 ms |
| Recovery | Once per dictation: fresh stream and fresh AudioContext, then startup deadline again |
| Audio resume | Maximum 5,000 ms; retry on interrupted/suspended state; closed context fails |
| Initial permission/start timer | 15,000 ms |
| WebSocket readiness timer | 15,000 ms, armed after mic opens |
| Finalization timer | 30,000 ms |
| Worklet tail wait | Maximum 250 ms |
| Socket unsent buffer | Fail when `bufferedAmount > 128000`, approximately 4 seconds |

The browser health check detects **digital zero**, not speech. Quiet room noise counts as a healthy input. This is distinct from the server RMS onset gate. A valid empty recognition result does not imply a broken audio engine.

On stop, release microphone tracks immediately, flush the worklet tail, release graph nodes, then send the stop control message. If recording was never ready, dispose without waiting for transcription. On every exit, clear timers, stop health checks and tracks, disconnect nodes, terminate/close the worklet port, release the routing lease, and close the socket. Guard callbacks by current capture/session identity so stale asynchronous operations cannot modify a later session.

The maximum recording timer subtracts all bytes already captured before recording was announced: `remainingMs = maxSeconds*1000 - capturedBytes/32`. The server independently enforces bytes received.

Current limitation: the pre-ready queue is time-bounded by connection handling, but has no separate byte cap; its flush does not apply the normal per-frame `bufferedAmount` check. Do not describe this as a fully bounded queue.

## 5. Browser ↔ bridge wire protocol

Resolve `VITE_SPEECH_BASE_URL` against the current page, append `api/stt/stream` without losing a preview path prefix, and map HTTP→WS or HTTPS→WSS. HTTPS or localhost is required for browser microphone access.

| Direction | Message | Meaning |
| --- | --- | --- |
| Server → browser | `{"type":"ready","maxSeconds":300}` | Engine session configured; browser may flush queued audio |
| Browser → server | Binary PCM frame | Nonempty, even byte count, at most 16,000 bytes (500 ms) |
| Browser → server | `{"type":"stop"}` | End input, finalize once |
| Server → browser | `{"type":"partial","text":"whole current text"}` | Replace the owned draft; not an incremental suffix |
| Server → browser | `{"type":"final","text":"definitive text"}` | Definitive result, possibly empty |
| Server → browser | `{"type":"error","message":"user-facing reason"}` | Failure followed by close |

The stop object must equal exactly `{"type":"stop"}`; unknown controls, extra keys, malformed JSON and invalid frames are rejected. After accepting stop, the bridge stops reading browser messages and waits for engine completion; clients must not send further input.

Close codes: **1000** normal, **1008** origin/protocol/timeout violation, **1011** transcription failure, **1013** busy. The bridge checks exact equality of the incoming `Origin` with `CORS_ORIGIN` before accepting. An absent Origin fails too.

## 6. Bridge ↔ engine protocol and concurrency

Start one engine child:

```text
nemo-speech serve --host 127.0.0.1 --port <free-loopback-port> --asr-model nemotron-3.5 --device <metal-or-auto-or-override> --no-ui
```

For each admitted dictation, connect to `ws://127.0.0.1:<port>/v1/audio/transcriptions/realtime`. Await `session.created`, send the configuration below, then require `session.updated` before telling the browser it is ready:

```json
{
  "type": "session.update",
  "session": {
    "sample_rate": 16000,
    "language": "es-ES",
    "automatic_punctuation": true,
    "speech_contexts": [{"phrases": ["Mooi"], "boost": 3.0}]
  }
}
```

Omit `speech_contexts` entirely when none are configured. Forward admitted audio as binary PCM. On browser stop send `{"type":"input_audio_buffer.commit"}`.

Accumulate `conversation.item.input_audio_transcription.delta` by appending its `delta` to a draft. On `conversation.item.input_audio_transcription.completed`, trim the `transcript`, append a nonempty value to finalized segments, and clear the draft. The current text is finalized segments plus trimmed draft joined with single spaces. Before stopping, both event types can produce a browser partial. After stopping, the first completion finishes the request. Ignore other event types; engine `error` fails the request. This event naming does not imply compatibility with the full OpenAI Realtime API. [Upstream client integration](https://github.com/NVIDIA/NeMo-Speech.cpp/blob/main/docs/clients.md).

Await browser and engine reads concurrently using first-completed semantics; do not wait for another browser frame before delivering an engine result. Engine connect and initial event waits are 15 seconds; incoming engine message limit is 1 MiB. Engine close timeout is 1 second to avoid holding admission while an abandoned engine session works through a backlog.

Admission waits up to 3 seconds for the process-wide lock before reporting busy. Start the overall deadline after ready: configured maximum duration plus 30 seconds. Each wait is capped at 30 seconds and the remaining overall duration. Count all browser audio, including onset-discarded silence, toward the byte limit. A disconnect observed while reading cancels the session.

Cleanup cancels outstanding read tasks, closes the engine socket, queues metrics, and releases the lock **before** sending final. Cleanup is idempotent and lock release happens in a finally block. Once stop is accepted, browser disconnect is no longer actively read; finalization is still bounded by deadlines.

## 7. Speech-onset gate: essential engine workaround

A near-silent initial stream can make the pinned engine ignore early speech for seconds. Browser capture recovery alone did not fix this. The bridge therefore delays audio delivery until a frame has RMS ≥ **500 PCM16 units**:

```text
RMS = sqrt(sum(sample²) / numberOfSamples)
pre-roll target = 6400 bytes = 200 ms at 16 kHz PCM16
```

Before onset, keep the newest frames in a deque. Remove oldest whole frames while the bytes remaining after removal would still be at least 6,400. On the triggering frame, send the retained pre-roll followed by the triggering frame, then permanently pass all later frames unchanged. Whole-frame retention can exceed 200 ms with nonstandard frame sizes. Track discarded leading bytes separately for diagnostics.

This is a one-time onset gate, not continual VAD, endpoint detection, or silence removal. Later silence must pass through. A silence-only dictation returns empty on stop without feeding or committing audio to the engine, although its engine session was already opened/configured.

RMS 500 is empirical, not universally safe: very soft syllables can fall below it; 200 ms of pre-roll mitigates clipping but cannot guarantee retention. Mouse clicks can open the gate early. Keep the workaround for this baseline and re-evaluate it on runtime/model upgrades using a near-silent-start recording.

## 8. Text insertion and user interaction

Use a four-state controller: `idle → connecting → recording → stopping → idle`. Keep session identity independent of asynchronous UI render timing.

A live transcript owns only the original selection range. Snapshot original value, start and end; reject detached, disabled, readonly or password fields and input types without selection APIs. Prefix dictated text with a space when immediately preceded by a non-whitespace character. Preserve original text before and after the selection.

Before each write, require that the field still equals the expected value: the original before the first write, then original prefix + last rendered slice + original suffix. If the user edits the value, refuse to overwrite it. Respect maxLength without truncating silently. Cursor movement alone does not change ownership. Every accepted update restores the caret to the end of the dictated slice and scrolls it into view, even if text is identical to the previous partial. Identical text does not dispatch a redundant value update.

Framework-controlled fields must receive their normal input notification. Mooi uses the native prototype value setter and a bubbling `input` event. A different framework may use its own state mechanism with equivalent semantics. Textarea caret visibility uses a hidden mirror with matching wrapping/font/box styles; at field end it scrolls to the bottom/right.

Escape restores the original selection/value only if the field still matches the owned draft. Empty finals and errors also attempt this guarded rollback. Final insertion refused because the field changed produces a user-facing error. The current equality guard is not a permanent conflict latch: if a value later exactly matches the expected value again, it can become writable again.

- Tap microphone to toggle. Hold for at least **350 ms** for push-to-talk; release to stop.
- Prevent pointer-down from stealing field focus; track matching pointer release on window capture listeners.
- `pointercancel` from a permission prompt does not stop the session. Pointer-up attempts audio resume after mic acquisition, accommodating mobile user activation.
- Keyboard: **Ctrl+Shift+.**, physical Period key, only with the bound field focused; release any chord key or window blur to stop. Ignore repeat. Escape is captured before surrounding dialogs.
- Keyboard button activation (Enter/Space click with detail zero) toggles.
- Hidden page: stop if recording; otherwise cancel. Unmount disposes and attempts rollback.
- Controller `finish()` disposes without rollback, retaining the draft; it differs from graceful `stop()` and cancel.
- Errors remain visible for 6 seconds. Display separate permission-denied, missing-device, device-busy, disconnected, no-signal, connection, backpressure and finalization errors.

## 9. Configuration and installation

Settings are cached per process. Precedence is environment over service `.env` over parent `../.env`; paths are relative to the working directory, not an automatic repository-root discovery. Unknown settings are ignored. Restart is needed for configuration changes.

| Setting | Default / constraints |
| --- | --- |
| `SPEECH_HOST` | `127.0.0.1`; settings accept only loopback names/addresses |
| `SPEECH_PORT` | `59100` |
| `CORS_ORIGIN` | `http://localhost:28471`; must start with HTTP(S), exact origin comparison |
| `SPEECH_LANGUAGE` | `es-ES`; regex `^[a-z]{2,3}-[A-Z]{2}$`; `auto` is not accepted by Mooi |
| `SPEECH_CONTEXTS` | Empty; split commas, trim, omit empty entries |
| `SPEECH_MAX_SECONDS` | 300; allowed 1–1800 |
| `SPEECH_RUNTIME_PATH` | Empty; explicit existing binary wins |
| `SPEECH_RUNTIME_DIR` | `.runtime`; expanded and resolved |
| `SPEECH_MODEL` | `nemotron-3.5`; `.gguf` suffix or slash treated as local path |
| `SPEECH_DEVICE` | Empty; Metal on native Darwin arm64, otherwise auto |
| `SPEECH_STARTUP_TIMEOUT_SECONDS` | 60, positive; dev Compose overrides to 120 |
| `SPEECH_METRICS_PATH` | `~/.cache/mooi/speech/metrics.sqlite` |
| `SPEECH_METRICS_RETENTION_DAYS` | 30, positive |
| `LOG_LEVEL` | INFO |
| `NEMO_SPEECH_MODEL_DIR` | Runtime model-cache override; supplied by container |
| `VITE_SPEECH_BASE_URL` | Required frontend setting; absolute URL or path prefix |

Mooi's slash heuristic means a Hugging Face repository ID is treated as a local path, even though upstream supports repository IDs. Use the short indexed name or a real GGUF file for this implementation.

Provision before starting the API: `python -m mic_speech.shared.engine provision`. Resolution is explicit binary, provisioned versioned binary, then PATH. If absent, download the platform archive, validate SHA-256, extract safely and normalize the archive root. For indexed models run `nemo-speech --quiet pull <model>`; for local paths require a file. A mistaken explicit binary path can lead to provisioning fallback; validate overrides during deployment.

The Dockerfile builds Python dependencies with frozen uv lock resolution, includes the pinned native archive, and runs as uid/gid 1001. Runtime requires curl and CA certificates for initial model pulling. Persist model/cache/metrics storage with writable ownership. Default image paths are under `/var/lib/mooi/speech`; dev uses `/home/mooi/speech` on a named volume.

Launch exactly one Uvicorn worker, with `--ws-max-size 16000 --ws-max-queue 8 --timeout-graceful-shutdown 5`. Multiple workers would create independent locks and engine children and violate intended admission control. These transport caps bound queued messages, not every buffer in the pipeline.

Engine readiness polls loopback TCP every 100 ms within startup timeout; it is not a synthetic transcription probe. Shutdown sends SIGTERM, waits 5 seconds, then SIGKILL. The code does not automatically restart a crashed child; service supervision must handle recovery.

## 10. Gateway, deployment, and security boundaries

The gateway preserves `/api/stt/`, forwards WebSocket Upgrade/Connection headers, disables proxy buffering, uses a 5-second connect timeout and 1,900-second read/send timeouts. Preserve public Origin through any outer proxy. Preview URL prefixes must route back to the same bridge path correctly.

Production image defaults bind the bridge to loopback and share the gateway's network namespace according to the Dockerfile. Dev Compose explicitly launches Uvicorn on `0.0.0.0:59100` inside its container and routes by service name `mic-speech:59100`, without publishing a speech host port. The engine itself always stays on loopback within that container. `SPEECH_HOST` validation does not enforce the actual Uvicorn CLI bind; do not confuse the setting with a network firewall. Existing Dockerfile comments claiming identical dev binding are stale relative to Compose.

The speech handler itself implements Origin checking, not user authentication or tenant authorization. There is no user/project/session identifier in its protocol or metrics. A single process lock is not tenant isolation. In a shared hosted replication, put authenticated, authorized access and rate/resource control at the trusted gateway or bridge and retain isolated per-session state. Origin filtering alone cannot authenticate non-browser clients. Do not expose engine, health or metrics endpoints publicly merely because they lack application content.

Keep TLS/WSS at the public edge. Do not log PCM or transcripts or persist temporary audio by default. Debug recording/replay is an explicit diagnostic exception requiring deliberate handling. Browser transcript content subsequently saved as chat is outside the speech service's content-free metrics guarantee.

Deployment files were read as evidence only; this document does not authorize changing Mooi's owner-managed deployment channels.

## 11. Observability and historical evidence

`GET /health` returns 200 with `{"status":"UP","engine":"<model-name>"}` while the child process is alive, otherwise 503/DOWN. It does not prove an unstalled engine or acceptable latency. `GET /metrics` returns retention, totals, outcome counts, cumulative audio seconds, mean partial count, and p50/p95 first-partial and stop-to-final latency.

SQLite stores timestamp, outcome (`final`, `empty`, `cancelled`, `error`), audio duration, first-partial milliseconds, partial count, and nullable stop-to-final milliseconds. File mode is 0600; newly created parent directory requests 0700. Database work runs off the event loop; prune on startup and insert. Metrics failures do not fail dictation. Admission failures before a transcription object exists are not included as dictation rows.

First-partial timing starts at creation of the admitted transcription, including engine handshake; it excludes permission time and admission waiting. Stop-to-final excludes later socket-release and delivery time. Silence-only sessions have null finalization latency to avoid distorting percentiles. Logs also include dropped leading-silence seconds, but this value is not a SQLite column. Percentiles use sorted linear interpolation, rounded to one decimal.

The repository issue report records production confirmation on 2026-10-27: five dictations had first partials at 1.5–2.6 seconds and stop-to-final at 0.24–0.43 seconds after the onset fix. Earlier failures delayed first partials 6.5–48 seconds or produced none. These are historical observations, not a service-level guarantee or measurements rerun for this document.

## 12. Failure diagnosis and preserved lessons

| Symptom | Distinction and remedy |
| --- | --- |
| Later recordings lose first words despite good PCM | Near-silent start can deafen pinned engine; retain onset gate and pre-roll |
| No frames / all-zero PCM | Browser graph or route problem; mic first, shared context, health deadline, one rebuild |
| First word clipped by gate | Check input level and pre-roll; very soft onsets remain a known limit |
| No words but quick final | Not necessarily backlog; compare audio signal and first-partial metrics |
| Busy after cancel under load | Queued audio delays disconnect; 3-second admission grace and 1-second engine close bound |
| Finalization slow under load | Host CPU contention; queue/frame changes alone do not add inference capacity |
| Increasing cost after repeated recordings | Worklets must return false/receive close, not merely disconnect ports |
| iOS permission popup stops dictation | Treat pointercancel differently from a genuine release |
| Phone call/Siri interruption | Bounded resume, then fail and release resources |
| Cursor moved and transcription disappears offscreen | Restore owned-slice caret and reveal even on identical snapshots |
| Selected text cannot be dictated over | Before first write compare against original value, not an already-replaced selection |
| User edit overwritten | Enforce expected-value ownership on update and rollback |
| Preview cannot connect | Verify base-prefix resolution, WSS, upgrade routing, and exact Origin |

`docs/issues/stt-issue.md` intentionally retains superseded hypotheses. Its initial conclusion that capture caused all missing words was disproved by deterministic replay; the later engine/onset investigation takes precedence. Its old statement that speech is absent from the dev stack is superseded by current Compose. Browser lifecycle fixes remain valuable even though they did not solve the primary engine fault.

The historical investigation observed four engine realtime slots in its tested configuration, but Mooi does not explicitly set that thread/slot count. Do not treat it as a configurable concurrency guarantee. Current application admission remains one request per service process; scaling needs measured engine capacity, memory, CPU budgeting and authenticated fairness, not simply extra workers.

## 13. Replication checklist and verification plan

1. Preserve runtime/model artifacts and language/context settings; record hardware/backend and checksums.
2. Implement native-rate browser capture, matching worklet PCM conversion and 100 ms framing.
3. Implement microphone-before-graph ordering, shared context, routing lease, bounded recovery and complete resource cleanup.
4. Reproduce both WebSocket contracts, full-duplex event handling, whole-transcript snapshots and finalization ordering.
5. Retain the one-time onset gate, pre-roll, byte/time limits, admission grace and bounded engine close.
6. Implement owned-range text updates, guarded rollback, caret visibility, keyboard/touch interaction and page-background handling.
7. Deploy a private resident engine and single-worker bridge behind the authorized gateway; persist only model/cache and content-free metrics.
8. Compare behavior on the intended browser/device/host matrix before claiming equivalent performance.

Framework/component structure, backend language, proxy brand, storage library and UI styling may vary. Changing the model, quantization, backend, resampler, capture processing, gate thresholds, streaming protocol or concurrency policy requires fresh quality/performance validation. Another transport can work, but must preserve ordered binary delivery, bidirectional partials, bounded buffering, explicit end-of-input and cancellation semantics.

When application testing is explicitly authorized, manually verify desktop Chrome, Safari and physical iPhone/iPad: repeated start/stop, first permission prompt, tap/hold/keyboard, near-silent onset, very quiet speech, long silence, empty results, cancellation then immediate restart, microphone unplug, audio interruption, page hiding, backlog under CPU load, maximum duration, selection replacement, user edits, long scrolling drafts and unchanged final text. For diagnostic replay, send the captured PCM with original timing both through the bridge and directly to the engine; compare gate enabled/disabled to isolate capture from recognition. These are future validation steps, not tests performed during extraction.

## 14. Implementation source map

All paths below are relative to the Mooi repository root. The preceding specification is intended to remain useful when copied without the repository.

| Concern | Source |
| --- | --- |
| Service lifecycle | `mic-speech/src/mic_speech/main.py` |
| Runtime provisioning, hashes, supervision | `mic-speech/src/mic_speech/shared/engine.py` |
| Settings | `mic-speech/src/mic_speech/shared/env.py` |
| Bridge, onset gate, limits, cleanup | `mic-speech/src/mic_speech/features/dictation.py` |
| Metrics persistence and summaries | `mic-speech/src/mic_speech/shared/metrics.py` |
| Health/metrics routes | `mic-speech/src/mic_speech/features/health.py`, `features/metrics.py` |
| Dependencies and image | `mic-speech/pyproject.toml`, `mic-speech/uv.lock`, `mic-speech/Dockerfile` |
| PCM worklet | `spa-mooi/public/dictation-worklet.js` |
| Capture/session/health | `spa-mooi/src/features/dictation/lib/DictationCapture.ts`, `DictationSession.ts`, `DictationAudioHealth.ts` |
| Shared graph and routing | `spa-mooi/src/features/dictation/lib/dictationAudio.ts`, `audioSession.ts` |
| Text ownership and visibility | `spa-mooi/src/features/dictation/lib/LiveTranscript.ts`, `captureTarget.ts`, `setNativeValue.ts`, `revealCaret.ts` |
| UI lifecycle and gestures | `spa-mooi/src/features/dictation/hooks/useDictation.ts`, `useDictationPointer.ts`, `useDictationHotkey.ts` |
| URL/messages | `spa-mooi/src/features/dictation/lib/dictationStreamUrl.ts`, `dictationMessages.ts` |
| Gateway and dev wiring | `spa-mooi/nginx.conf`, `spa-mooi/src/config/env.ts`, `compose.yml` |
| Investigation evidence | `docs/issues/stt-issue.md` |
