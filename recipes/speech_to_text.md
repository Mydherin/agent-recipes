---
name: Local push-to-talk dictation
description: App-wide floating microphone that dictates into any focused text field with live streaming partials, powered by a local NVIDIA Nemotron 3.5 streaming ASR model behind its own speech microservice.
---

# Local push-to-talk dictation

## Goal

Let the user dictate into **any** text field of the web application, from any page or dialog, by holding a
key combo or the floating microphone button. Text appears in the field **while speaking** (live partials) and
the definitive text replaces the draft almost immediately after stopping. All recognition runs **locally**:
no audio or text ever leaves the machine, nothing is translated, nothing is stored.

## Requirements

### Functional

- One floating microphone button, mounted once at the application root, visible on every page and above dialogs.
- Dictation targets the text field that has the cursor (`<input>` or `<textarea>`), inserting at the caret or
  replacing the current selection.
- Three ways to dictate:
  - **Keyboard push-to-talk:** hold `Ctrl+Shift+.` → records; releasing any of the three keys stops.
  - **Pointer/touch hold:** press and hold the button ≥ 350 ms → releasing stops (push-to-talk).
  - **Pointer/touch tap or keyboard activation (Enter/Space on the focused button):** toggles start/stop.
- Live partial transcript is written into the field and mirrored in a small status card next to the button.
- On stop, the final transcript from the **same stream** replaces the draft, exactly once.
- Cancel (card close button), errors or connection loss restore the field's original value — only if nobody
  else edited the field meanwhile.
- If the field was edited by something else during dictation, the dictation never overwrites it: the text is
  kept in the card (selectable, to copy) with an explanatory message.
- Empty result (no voice detected) restores the field and shows a message.
- Never writes into: other applications, `type="password"` inputs, disabled/read-only fields, or
  rich-text/`contenteditable` editors.
- Recording automatically stops at the configured maximum length (default 300 s).

### Non-functional

- **Local-only:** model, runtime and service run on the user's machine and bind to `127.0.0.1` only.
- **Privacy:** no audio or transcript is persisted or logged. Only content-free metrics (counts and timings).
- **Latency targets** (validation targets, measured on Apple M4 + Metal):
  first partial p95 ≤ 1 s; stop → final p95 ≤ 1.5 s; sustained real-time factor ≤ 0.7.
  Observed: first partial ≈ 0.6 s, stop → final ≈ 100–180 ms.
- **Language is a condition, not a detection:** the model is forced to `es-ES` (configurable). No translation,
  no automatic language detection, no second ASR/LLM pass.
- **One dictation at a time** per machine; a second concurrent request is rejected with a clear message.
- **Accessible:** `aria-label`, `aria-pressed`, `role="status"` messages, visible focus ring, keyboard operable,
  reduced-motion-aware animation.
- **Robust:** every step has a timeout; every resource (mic track, audio graph, sockets, engine session) is
  released on every exit path.

## Approach

```
┌──────────────────────── Browser (SPA) ──────────────────────────┐
│ DictationButton (UI + gestures)                                  │
│   ├─ targetSelection  → which field / caret / original value     │
│   ├─ LiveTranscript   → writes only the dictation-owned slice     │
│   └─ DictationSession → mic → AudioWorklet → WebSocket            │
│        AudioWorklet: Float32 → PCM16 mono 16 kHz, 500 ms frames   │
└──────────────┬────────────────────────────────────────────────────┘
               │ same-origin WS  /api/stt/stream   (reverse proxy)
┌──────────────▼──────── Speech microservice (loopback) ───────────┐
│ WS bridge: origin allowlist, single-session lock, validation,     │
│ full-duplex relay, timeouts, content-free metrics (SQLite)        │
│ Runtime supervisor: spawns, probes and stops the native engine    │
└──────────────┬────────────────────────────────────────────────────┘
               │ ws://127.0.0.1:<random>/v1/audio/transcriptions/realtime
┌──────────────▼──────── NeMo-Speech.cpp (native child process) ───┐
│ NVIDIA Nemotron 3.5 ASR Streaming 0.6B (Q8 GGUF), Metal on macOS │
└───────────────────────────────────────────────────────────────────┘
```

- Speech runs in an **independent microservice**, separate from the main backend, with its own
  lifecycle, port, health check and logs. The microservice owns the native engine as a child process.
- Browser ↔ service and service ↔ engine are both **WebSockets streaming raw PCM16** with JSON
  control/result messages. Reading audio and reading results happen **concurrently** (full duplex).
- The browser reaches the service through the **same origin** as the SPA (reverse proxy path
  `/api/stt/*` with WebSocket upgrade). The service is never exposed directly on a non-loopback interface.
- The service language/framework is not relevant as long as it supports async full-duplex WebSockets,
  child-process management and SQLite. Reference implementation: Python, FastAPI + uvicorn, `websockets`
  client, `pydantic-settings`.
- The frontend framework and styling are not relevant; follow the host project's design system. Reference:
  React + TypeScript + Tailwind + lucide icons. The audio pipeline is **Web Audio `AudioWorklet` + `WebSocket`**.

## Technical specification

### 1. Speech model and native runtime

| Item | Value |
|---|---|
| Model | NVIDIA **Nemotron 3.5 ASR Streaming 0.6B** (`nvidia/nemotron-3.5-asr-streaming-0.6b`), cache-aware streaming, 600M params |
| Quantization | Q8 GGUF (~742 MB on disk), resident server ≈ 1.0 GiB RAM |
| Runtime | **NVIDIA NeMo-Speech.cpp v0.1.0** (`nemo-speech` binary) |
| Licenses | Model: NVIDIA OpenMDW 1.1 (commercial use allowed). Runtime: Apache-2.0. Ship both notices when redistributing. |
| Language | `es-ES` fixed per session (configurable, e.g. `es-MX`, `es-US`). Never `auto`. |
| Punctuation | `automatic_punctuation: true` |
| Domain bias | Optional `speech_contexts` phrase list with conservative `boost: 3.0` |
| Device | `metal` on macOS, `auto` elsewhere |

Why this model: it is the only evaluated candidate that combined native streaming partials, an explicit language
condition (no Spanish→English confusion), fast close of the same stream (no second pass), and a commercial
license. Previously evaluated and discarded: Parakeet TDT 0.6B v3 via MLX (no partials, no language parameter,
re-decodes a long provisional window per chunk → RTF > 1 and long stop delay) and Moonshine Spanish Small
(lighter, but more visible errors).

**Model artifacts:** the service uses a local NeMo-Speech.cpp v0.1.0 binary (Metal build on Apple Silicon) and
the model resolved through the runtime's own pinned index (`nemo-speech pull nemotron-3.5`, which pins revision
and checksum), or a local Nemotron 3.5 GGUF file passed by path.

**Engine process:**

```
nemo-speech serve --host 127.0.0.1 --port <free-port> --asr-model <nemotron-3.5 | /path/model.gguf> \
                  --device <metal | auto> --no-ui
```

- Pick a free loopback port by binding a socket to `127.0.0.1:0` and reading the assigned port.
- Readiness: poll a TCP connect to `127.0.0.1:<port>` every 100 ms, up to 60 s. If the process exits during
  startup or the timeout expires → stop the process and fail startup.
- Shutdown: `SIGTERM`, wait 5 s, then `SIGKILL`.
- The engine starts once at service startup (model stays resident), **not** per dictation.

### 2. Engine protocol (service ↔ NeMo-Speech.cpp)

WebSocket `ws://127.0.0.1:<port>/v1/audio/transcriptions/realtime`, client max message size 1 MiB, open
timeout 15 s. One engine connection **per dictation**.

| Step | Direction | Message |
|---|---|---|
| 1 | engine → | `{"type":"session.created", ...}` (anything else → error) |
| 2 | → engine | `{"type":"session.update","session":{"sample_rate":16000,"language":"es-ES","automatic_punctuation":true,"speech_contexts":[{"phrases":["…"],"boost":3.0}]}}` (`speech_contexts` only when configured) |
| 3 | engine → | `{"type":"session.updated"}` (anything else → "engine rejected the language setting") |
| 4 | → engine | binary frames: raw PCM16 little-endian mono 16 kHz |
| 5 | engine → | `conversation.item.input_audio_transcription.delta` `{ "delta": "…" }` — append to draft |
| 6 | engine → | `conversation.item.input_audio_transcription.completed` `{ "transcript": "…" }` — a finalized segment |
| 7 | → engine | on stop: `{"type":"input_audio_buffer.commit"}` |
| — | engine → | `{"type":"error"}` → fail the dictation; any other type → ignore |

**Text assembly** (per dictation):

- `finalized: list[str]`, `draft: str`.
- On `delta`: `draft += delta` → emit partial.
- On `completed`: if `transcript.strip()` is non-empty, push it to `finalized`; reset `draft = ""`.
- Current text = join with single spaces of `[*finalized, draft.strip()]`, skipping empties, trimmed.
- `completed` may arrive **before** stop (segment boundary): emit it as a partial and keep going.
  Only the first `completed` received **after** the commit is the final result.

### 3. Browser ↔ service protocol

Endpoint: `WS /api/stt/stream` (same origin as the SPA; `wss:` when the page is `https:`).

Client → server:

- **Binary:** PCM16 LE mono 16 kHz frames. Valid frame: non-empty, even byte length, ≤ 16 000 bytes
  (≤ 8 000 samples = 500 ms).
- **Text:** exactly one `{"type":"stop"}`. Any other text, a second stop, or audio after stop → invalid.

Server → client (JSON):

| Message | Meaning |
|---|---|
| `{"type":"ready","sampleRate":16000,"maxSeconds":300}` | Engine session open; start sending audio |
| `{"type":"partial","text":"<full current text>"}` | Whole running transcript (not a delta) |
| `{"type":"final","text":"<full text>"}` | Definitive text; may be `""` |
| `{"type":"error","message":"<user-facing text>"}` | Followed by close |

Close codes: `1000` normal end · `1008` origin rejected / invalid or expired session · `1011` internal
transcription failure · `1013` busy (another dictation in progress).

### 4. Speech microservice

**Endpoints**

- `GET /health` → `{"status":"up","engine":"nemotron-3.5"}`; `503` if the engine process is missing or exited.
- `GET /metrics` → content-free summary (see §6).
- `WS /api/stt/stream` → the bridge.

**Server settings:** single worker process (the single-session lock and the engine are in-process), incoming
WebSocket max message size 16 000 bytes, max queue 8 messages.

**Bridge algorithm for `/api/stt/stream`:**

1. Check `Origin` header against the allowlist **before accepting**; mismatch → close `1008`.
2. Accept. If `busy` → send error `"El dictado está ocupado. Inténtalo de nuevo."`, close `1013`.
3. `busy = true`; create a `Transcription` (engine session + text assembly + counters).
4. Open the engine session (§2 steps 1–3). Send `ready`.
5. Hard deadline = now + `maxSeconds` + 30 s.
6. Loop with **two pending tasks at once**: `browser.receive()` and `engine.receive()`; wait for the first to
   complete with timeout `min(30 s, remaining deadline)`. No completion → timeout error.
   - Browser disconnect → outcome `cancelled`, exit.
   - Browser binary → validate (§3), add to received bytes, reject if total > `maxSeconds × 32000` bytes,
     forward to engine. Re-arm browser receive.
   - Browser `stop` → mark stopping, record stop time. If **no audio was ever received** → outcome `empty`,
     release, send `final ""`, exit. Otherwise send `input_audio_buffer.commit` to the engine and **stop
     reading from the browser**.
   - Engine `completed` while stopping → outcome `final` (or `empty` if text is blank), log timings,
     **release first, then send `final`**, exit.
   - Engine `partial`/pre-stop `completed` → send `partial` with the full current text. Re-arm engine receive.
7. Close `1000`.

**Error mapping:** validation / timeout / malformed JSON → send
`"La sesión ha expirado o el audio no es válido. Vuelve a intentarlo."`, close `1008`. Any other exception →
`"No se ha podido transcribir el audio."`, close `1011`. Sending to an already-closed socket is ignored.

**Release (idempotent, runs in `finally` on every path):** cancel and await both pending tasks, close the engine
connection, record metrics (failures here are logged, never raised), and **always** reset `busy = false`.
Releasing before sending `final` lets the user start the next dictation instantly.

**Logs:** one line per finished dictation with `audio_s`, `first_partial_ms`, `partials`, `stop_to_final_ms`.
Never log transcript text or audio.

### 5. Configuration

Environment variables with a service prefix (reference: `MIC_STT_`; names are illustrative):

| Setting | Default | Purpose |
|---|---|---|
| `HOST` / `PORT` | `127.0.0.1` / `59100` | Service bind address |
| `LANGUAGE` | `es-ES` | Forced ASR language |
| `RUNTIME_PATH` | — | Path to `nemo-speech` binary |
| `MODEL_PATH` | indexed `nemotron-3.5` | Local GGUF override |
| `SPEECH_CONTEXTS` | empty | Comma-separated domain phrases to bias |
| `METRICS_PATH` | `~/.cache/<app>/stt/metrics.sqlite` | Content-free metrics DB |
| `MAX_SECONDS` | `300` (allowed 1–1800) | Maximum recording length |
| `ALLOWED_ORIGINS` | `http://localhost:<spa-port>,http://127.0.0.1:<spa-port>` | WebSocket `Origin` allowlist |

### 6. Content-free metrics

- SQLite file, created on startup with permissions `0600`.
- Table `dictations(created_at INTEGER, outcome TEXT, audio_seconds REAL, first_partial_ms REAL NULL,
  partial_count INTEGER, stop_to_final_ms REAL NULL)`.
- `outcome ∈ {final, empty, cancelled, error}`. `audio_seconds = received_bytes / 32000`.
  `first_partial_ms` measured from session creation. `stop_to_final_ms` only for `final`/`empty` with audio.
- Rows older than 30 days deleted on startup and after every insert. Writes run off the event loop.
- `GET /metrics` returns `retentionDays`, `total`, per-outcome counts, total `audioSeconds`,
  `firstPartialMs.{p50,p95}`, `stopToFinalMs.{p50,p95}` (linear-interpolated percentiles, 1 decimal),
  `meanPartials`.

### 7. Audio capture in the browser

**AudioWorklet processor** (static file served by the SPA, e.g. `/dictation-worklet.js`, registered as
`'dictation'`), running off the UI thread:

```js
class DictationProcessor extends AudioWorkletProcessor {
  constructor() {
    super()
    this.buffer = new Int16Array(8000)          // 500 ms @ 16 kHz
    this.offset = 0
    this.stopped = false
    this.port.onmessage = ({ data }) => {
      if (data === 'stop') {
        this.stopped = true
        if (this.offset) this.port.postMessage(this.buffer.slice(0, this.offset).buffer)
        this.port.postMessage('stopped')
      }
    }
  }
  process(inputs) {
    if (this.stopped) return false
    const samples = inputs[0]?.[0]
    if (!samples) return true
    for (const value of samples) {
      const clamped = Math.max(-1, Math.min(1, value))
      this.buffer[this.offset++] = Math.round(clamped * (clamped < 0 ? 32768 : 32767))
      if (this.offset === this.buffer.length) {
        this.port.postMessage(this.buffer.buffer, [this.buffer.buffer])   // transfer, no copy
        this.buffer = new Int16Array(8000)
        this.offset = 0
      }
    }
    return true
  }
}
registerProcessor('dictation', DictationProcessor)
```

**DictationSession** (one instance per dictation; callbacks `onReady`, `onPartial`, `onStopping`, `onFinal`,
`onError`):

1. If `navigator.mediaDevices.getUserMedia` is missing → error "requires HTTPS or localhost".
2. **Request the microphone first, inside the user gesture**, before opening any server session:
   `getUserMedia({ audio: { channelCount: 1, echoCancellation: true, noiseSuppression: true } })`.
   Track `ended` → fail "microphone disconnected".
3. `new AudioContext({ sampleRate: 16000 })` — the browser resamples; if `context.sampleRate !== 16000` → fail.
   No client-side resampling code.
4. `audioWorklet.addModule(<base-url>/dictation-worklet.js)`, then `context.resume()`.
5. After every `await`, if the session was disposed meanwhile, release what was just acquired and return.
6. Open `WebSocket(<ws|wss>://<location.host>/api/stt/stream)`. Start a **15 s** "service not responding" timer.
7. On `ready`: clear timer, build the graph `MediaStreamSource → AudioWorkletNode(channelCount 1, explicit) →
   destination` (the worklet outputs silence; connecting to destination keeps it processing without playback),
   arm an auto-stop timer of `maxSeconds`, call `onReady`.
8. Worklet frame → send binary if the socket is open. **Backpressure:** if `socket.bufferedAmount > 128 000`
   bytes (~4 s of audio) → fail "connection cannot keep up".
9. `stop()` (idempotent): call `onStopping`, replace timers with a **30 s** finalize timeout, post `'stop'` to
   the worklet. When the worklet answers `'stopped'` (after flushing the tail frame): disconnect source, stop
   mic tracks, send `{"type":"stop"}`.
10. `partial` → `onPartial(text)`; `final` → dispose then `onFinal(text)`; `error` → fail with server message;
    unparsable message → fail. Unexpected `close`/`error` → fail.
11. `dispose()` (idempotent, guarded by a `closed` flag that also mutes all late callbacks): clear timers, stop
    tracks, disconnect nodes, close the worklet port, close the `AudioContext`, close the socket.

### 8. Target field and safe insertion

**Target capture** — returns `null` unless `document.activeElement` is an `HTMLInputElement` or
`HTMLTextAreaElement` that is not disabled, not read-only, not `type="password"`, and has a non-null
`selectionStart` (excludes number/email/etc. inputs that do not support selection). Snapshot:
`{ element, start: selectionStart, end: selectionEnd ?? selectionStart, original: element.value }`.

**Remember the last target:** while no dictation is active, listen to `focusin` and `selectionchange` on
`document` and keep the latest capture. Starting a dictation uses the currently focused field, falling back to
the remembered one; if none or it is no longer in the DOM → error "Place the cursor in a text field".
The button calls `preventDefault()` on `pointerdown` so it never steals focus from the field.

**LiveTranscript** — owns only the slice it wrote:

```ts
update(text): boolean
  expected = original[0:start] + rendered + original[end:]
  if (!element.isConnected || element.disabled || element.readOnly || element.value !== expected) return false
  if (text === rendered) return true
  available = maxLength < 0 ? ∞ : maxLength - (original.length - (end - start))
  if (text.length > available) return false
  setValue(original[0:start] + text + original[end:]); rendered = text; changed = true
  caret → start + text.length
  return true

rollback()
  only if changed && element.isConnected && element.value === expected:
    setValue(original); restore selection (start, end)
```

`setValue` must use the **native prototype setter** (`Object.getOwnPropertyDescriptor(HTMLInputElement|
HTMLTextAreaElement.prototype, 'value').set.call(element, value)`) followed by
`element.dispatchEvent(new Event('input', { bubbles: true }))`, so framework-controlled inputs (React etc.)
pick up the change as if the user typed it.

**Final handling:**

- Non-empty final and `update` succeeds → done; clear the card; re-capture the target (caret now after the text).
- Non-empty final and `update` fails → `rollback()`; keep the text in the card;
  error "The field changed or is no longer available. You can copy the transcription."
- Empty final → `rollback()`; error "No voice detected. Try again."
- Error / cancel → `rollback()`.

### 9. Button, gestures and status card

**States:** `idle → connecting → recording → stopping → idle`. Keep the state in a ref as well as in UI state
so global listeners read the current value.

**Keyboard push-to-talk:**

- Combo: `Ctrl+Shift+Period` matched on `event.code === 'Period'` (physical key, layout-independent), with
  `ctrlKey && shiftKey && !altKey && !metaKey`. **Not** `Ctrl+Shift+Space`: macOS uses it to switch input
  sources when several keyboard layouts are enabled.
- Listen on `window` in **capture** phase; `preventDefault()` on match; ignore `event.repeat` and presses while a
  session exists.
- `keyup` of `Period`, `Control` or `Shift` while push-to-talk is active → stop. `window` `blur` → stop.

**Pointer / touch:**

- `pointerdown` (primary button only, `preventDefault`): if a session exists → stop; else capture target, record
  `{pointerId, timestamp}`, start.
- `pointerup` / `pointercancel` tracked on `window` in capture phase (the finger may leave the button). Same
  pointer id: if held ≥ **350 ms** or `pointercancel` → stop (push-to-talk); shorter → keep recording
  (it was a tap → toggle).
- `click` with `event.detail === 0` (keyboard activation of the focused button) → toggle.
- Suppress context menu, `touch-action: none`, `user-select: none`, no iOS touch callout.

**Stop while connecting:** set a `stopWhenReady` flag; on `ready` stop immediately.

**Layout:** fixed bottom-right (20 px offsets), z-index above dialogs (reference `100`), column with the card
above the button, never wider than `100vw − 40px`. Button 56×56 px, rounded, white icon:

| State | Icon | Colour | Label (`aria-label` + `title`) |
|---|---|---|---|
| idle | microphone | indigo | "Iniciar dictado (mantén pulsado o Ctrl+Shift+.)" |
| connecting | spinner | indigo, 60 % opacity | "Conectando micrófono…" |
| recording | filled square | red | "Detener y finalizar dictado" |
| stopping | spinner, disabled | 60 % opacity | "Terminando transcripción…" |

`aria-pressed = (state === recording)`, visible focus ring (2 px outline, 4 px offset).

**Status card** (shown when there is text, an error, or state ≠ idle), width 320 px:

- `role="status"` line: error, or while recording a hint by gesture —
  keyboard: "Grabando… Suelta Ctrl+Shift+. para finalizar." · hold: "Grabando… Suelta para finalizar." ·
  tap: "Grabando… Pulsa para finalizar." — otherwise the state label.
- Close button "Cancelar dictado y cerrar": dispose session, rollback insertion, reset state, clear text/error.
- While recording: five red bars (heights 12/20/16/24/12 px) pulsing with staggered 150 ms delay, only under
  `prefers-reduced-motion: no-preference`, `aria-hidden`.
- Transcript text: max 192 px tall, scrollable, `white-space: pre-wrap`, selectable.

**Mounting:** render the component once at the application root, outside the router's page outlet, so it
survives navigation and is available everywhere. On unmount: dispose session and rollback.

**User-facing copy** (reference locale `es-ES`; translate as needed):

| Situation | Message |
|---|---|
| No target | "Coloca el cursor en un campo de texto antes de dictar." |
| No secure context | "El micrófono requiere HTTPS o localhost." |
| Mic denied / unavailable | "No se ha podido acceder al micrófono." |
| Mic unplugged | "El micrófono se ha desconectado." |
| 16 kHz unsupported | "Este navegador no admite audio a 16 kHz." |
| Connect timeout (15 s) | "El servicio de dictado no responde." |
| Socket error | "No se puede conectar con el servicio de dictado." |
| Socket closed early | "Se ha interrumpido la conexión de dictado." |
| Backpressure | "La conexión no puede seguir el ritmo del audio. Vuelve a intentarlo." |
| Finalize timeout (30 s) | "El servicio no ha completado la transcripción." |
| Bad server message | "Respuesta de dictado no válida." |
| Field changed | "El campo ha cambiado o ya no está disponible. Puedes copiar la transcripción." |
| Empty result | "No se ha detectado voz. Vuelve a intentarlo." |
| Busy (server) | "El dictado está ocupado. Inténtalo de nuevo." |
| Invalid/expired (server) | "La sesión ha expirado o el audio no es válido. Vuelve a intentarlo." |
| Engine failure (server) | "No se ha podido transcribir el audio." |

### 10. Routing

- Reverse proxy `/api/stt/*` → speech service with **WebSocket upgrade enabled**, declared **before** any
  generic `/api` rule. Preserve the browser `Origin` header (the allowlist depends on it).
  Reference (Vite dev server): `'/api/stt': { target: 'http://127.0.0.1:<stt-port>', ws: true }`.
- The origin allowlist must match the origin the SPA is served from.
- Microphone access requires a secure context: HTTPS, or `localhost`/`127.0.0.1`.

## Steps

1. **Speech service skeleton:** settings (§5), app factory with startup/shutdown hook, `/health`, single worker.
2. **Runtime supervisor:** resolve binary and model, free loopback port, spawn `nemo-speech serve`, TCP
   readiness probe, graceful stop (§1).
3. **Transcription session:** engine WebSocket handshake with language/punctuation/contexts, frame validation
   and size limits, commit on stop, delta/completed text assembly, timing counters (§2).
4. **WebSocket bridge:** origin check, busy lock, `ready`, full-duplex loop, deadlines, error mapping, idempotent
   release before `final`, content-free log line (§3, §4).
5. **Metrics store:** SQLite `0600`, 30-day retention, `/metrics` summary (§6).
6. **Reverse proxy** for `/api/stt` with WebSocket support (§10).
7. **AudioWorklet** static file producing 500 ms PCM16 frames with tail flush on stop (§7).
8. **DictationSession:** mic-first permission, 16 kHz context, worklet graph, socket protocol, timers,
   backpressure, idempotent dispose (§7).
9. **Target capture + LiveTranscript:** focus/selection tracking, owned-slice replacement, native setter +
   `input` event, rollback (§8).
10. **DictationButton:** states, keyboard push-to-talk, pointer hold/tap, status card, copy, accessibility;
    mount once at the app root (§9).

## Acceptance criteria

- `GET /health` returns `up`; killing the engine process makes it return `503`.
- Focus a textarea on any page or inside a dialog, hold `Ctrl+Shift+.` and speak Spanish: the card shows
  "Grabando…", words appear in the field while speaking (first partial ≈ ≤ 1 s), releasing inserts the final
  text within ≈ 1.5 s, caret ends after the inserted text.
- Dictating with a selection replaces exactly the selection; text before and after is untouched.
- Hold the button with the mouse/finger > 350 ms → releasing stops. A short tap starts; the next tap stops.
  Enter/Space on the focused button toggles.
- Typing in the field during dictation → the dictation never overwrites it; the final text stays in the card.
- Cancelling with the card's close button restores the original field value.
- Silence only → field restored, "No se ha detectado voz".
- React-controlled inputs keep the dictated value after re-render and the app state reflects it.
- Password, read-only, disabled fields and `contenteditable` editors are never written; no target → message.
- A second tab dictating simultaneously gets "El dictado está ocupado"; the first one is unaffected.
- A WebSocket from a non-allowlisted origin is rejected; the service is not reachable from another machine.
- Spanish speech is never returned translated to English.
- Logs and `/metrics` contain only numbers and outcomes; no audio or transcript exists on disk.
- Stopping the service stops the native engine (no orphan `nemo-speech` process).
