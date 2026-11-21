# Codie 1.6.0 HTTP API — Beta Integration Guide

For beta testers connecting to a Codie device running 1.6.0 over a local network, using scripts, desktop apps or mobile apps. This guide reflects the source code on the master branch. It does not mean every device has received this version or that every operation has been tested on a device.

## Using the package

Extract `Codie-HTTP-API-1.6.0.zip` and open `index.html`. Search all public endpoints offline, including parameters, request and response schemas, and capability requirements. See `manifest.json` for operation and path counts. No server or access to the device’s Laboratory page is required.

| File | Purpose |
| --- | --- |
| `index.html` | Complete offline API reference, with search and browser printing |
| `guide.zh-CN.html` / `GUIDE.zh-CN.md` | Chinese guide |
| `guide.ja.html` / `GUIDE.ja.md` | Japanese guide |
| `guide.en.html` / `GUIDE.en.md` | English guide |
| `guide.de.html` / `GUIDE.de.md` | German guide |
| `guide.fr.html` / `GUIDE.fr.md` | French guide |
| `openapi.json` | Default English OpenAPI 3.1 description; API tools must implement HMAC signing separately |
| `openapi.zh-CN.json` | Chinese OpenAPI; paths, fields, enums and signing requirements match the English contract |
| `examples/local_api.py` | Pairing and chat; text mode uses the Python 3 standard library, with microphone dependencies listed in the script |
| `examples/external_tools.py` | Register and connect external tools, then return invocation results |
| `examples/media_session.py` | Temporary image, audio and video sessions |
| `manifest.json` | Device version, source commit, export time and file SHA-256 hashes |

All five guides are included as HTML and Markdown. Choose a guide language at the top of `index.html`. The complete API reference uses Chinese navigation and English descriptions by default. OpenAPI is available in English as `openapi.json` and Chinese as `openapi.zh-CN.json`. Paths, field names, enum values and example code are unchanged across languages.

OpenAPI `info.version=1` is the existing API description version. The device software version is 1.6.0, recorded separately in `x-device-version` and the manifest.

## 1. Prepare the device

1. Check that the device runs 1.6.0 and is signed in to a user account.
2. Put the computer and device on a local network where they can reach each other. Obtain the device IP address. The default address is `http://DEVICE_IP:8081`; use the configured port if it differs.
3. Wake and unlock the device before starting a conversation. Requests use the device's current user and character slot.
4. Check connectivity, then run an example to pair.

```sh
curl http://DEVICE_IP:8081/v1/api/auth/ping
```

The HTTP base path is `/v1/api`. A complete URL is `http://DEVICE_IP:8081/v1/api/data/user/system`. For the example scripts, pass only the scheme, host and port, without `/v1/api`.

## 2. Run a text conversation

From the extracted package's root directory, run the Python example:

```sh
python examples/local_api.py http://DEVICE_IP:8081 "Hello" --credentials credentials.json
```

On the first run, enter the pairing code shown on the device into the terminal. The example saves credentials, reads the current slot, submits a text turn, polls its status and prints the character's reply. Later runs reuse the same credentials file.

To play the reply on the device, add `--speech device`:

```sh
python examples/local_api.py http://DEVICE_IP:8081 "Hello" --credentials credentials.json --speech device --timeout 180
```

The default is `--speech none`, with no spoken reply, and a 120-second wait. With `device`, `completed` means both generation and device playback have finished. A client timeout does not cancel the server-side turn. The example prints its ID so you can query it later.

Examples explicitly request `session` credentials: valid for 12 hours and invalid after the device service restarts. To request the existing long-term credentials, add `--auth-scope permanent` on first pairing. Session credentials allow public user APIs; admin APIs require permanent credentials. Device capability policies still apply. Existing files are not overwritten or upgraded automatically; use a new filename when pairing again. Keep credentials private; do not include them in feedback or redistribute them with this package.

## 3. Implement pairing

Only these five public HTTP operations do not require HMAC:

| Method | Path, including base path | Purpose |
| --- | --- | --- |
| GET | `/v1/api/auth/ping` | Connectivity check |
| GET | `/v1/api/auth/bootstrap` | Current `user_id`, sign-in status and languages |
| POST | `/v1/api/auth/pair/request` | Display a pairing code; returns `challenge_id`; valid for 5 minutes |
| POST | `/v1/api/auth/pair/confirm` | Submit the code and obtain credentials |
| POST | `/v1/api/auth/pair/cancel` | Cancel a pairing request |

Pairing confirmation JSON:

```json
{
  "challenge_id": "value returned by pair/request",
  "pairing_code": "code displayed on the device",
  "auth_scope": "session"
}
```

`auth_scope` accepts `session` or `permanent`. The endpoint defaults to `permanent` when omitted; examples explicitly use `session`. Save `data.local_secret`, `data.local_session_id`, `data.local_session_expires_at_ms`, `data.auth_scope`, and bootstrap's `data.user_id`. Scripts and native apps may omit `Origin/Referer`; browser clients remain subject to the device's existing origin checks.

## 4. Sign every business request

Join these values with `|`, with no trailing newline:

```text
METHOD|PATH|TIMESTAMP|NONCE|PAYLOAD_MODE|BODY_HASH|USER_ID
```

```text
signature = hex_lower(HMAC_SHA256(UTF8(local_secret), UTF8(canonical)))
```

| Header | Value |
| --- | --- |
| `X-Local-Sig` | Signature as 64 lowercase hexadecimal characters |
| `X-Local-Timestamp` | Unix time in seconds, as a string; device clock tolerance: 300 seconds |
| `X-Local-Nonce` | New random value for every request, preferably UUID; 1–128 characters; no pipe or newline |
| `X-Local-Payload-Mode` | Mode from the table below |
| `X-Local-Body-Hash` | Hash from the table below |
| `X-Local-User-Id` | Current `user_id` returned by bootstrap |
| `X-Local-Session-Id` | Required when pairing returned a nonempty `local_session_id` |

Use uppercase `METHOD`. `PATH` is the URL-decoded path including `/v1/api`, without the host or query string. Use the original `local_secret` string's UTF-8 bytes as the key; do not hex-decode it. Hash exactly the bytes sent on the wire. Do not reformat JSON after signing.

| Body | PAYLOAD_MODE | BODY_HASH |
| --- | --- | --- |
| No body | `empty` | Lowercase SHA-256 hex digest of empty bytes |
| JSON | `json` | Lowercase SHA-256 hex digest of the actual JSON bytes |
| File form | `multipart` | Literal `UNSIGNED-PAYLOAD` |
| zstd upload chunk | `zstd-stream` | Lowercase SHA-256 hex digest of the actual compressed bytes |
| gzip upload chunk | `gzip-stream` | Lowercase SHA-256 hex digest of the actual compressed bytes |

Empty-body hash: `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855`. Compressed chunks use `application/octet-stream`, only on endpoints that explicitly support them. Let the HTTP library generate the multipart `Content-Type`, including its boundary.

## 5. Unified conversation API

| Method | Path relative to `/v1/api` | Purpose |
| --- | --- | --- |
| POST | `/data/user/chat/turns` | Create a conversation turn |
| GET | `/data/user/chat/turns/{turn_id}` | Get status, transcript and reply |
| POST | `/data/user/chat/turns/{turn_id}/input/complete` | Finish device microphone input and continue generating; empty body |
| POST | `/data/user/chat/turns/{turn_id}/cancel` | Cancel input, generation and playback for this turn; empty body |

First call `GET /data/user/slot` and read `data.slot_number`. Use a new UUID as `request_id` for each new business turn, and replace `slot` with the current slot:

```json
{
  "request_id": "3f743580-bd45-4c24-96c0-61a67ac5f741",
  "slot": 1,
  "input": {"type": "text", "text": "Hello"},
  "output": {"speech": "none"}
}
```

| Input | Content-Type | Content |
| --- | --- | --- |
| `text` | `application/json` | Required `input.text`, nonempty after trimming; maximum 16000 characters |
| `device_microphone` | `application/json` | Record using the device microphone; omit `text`; finish with `input/complete` |
| `image_text` | `multipart/form-data` | `request` contains the configuration as a JSON string, with `input.type=image_text` and optional `text`; file field: `image` |
| `audio_file` | `multipart/form-data` | `request` contains the configuration as a JSON string, with `input.type=audio_file` and no `text`; file field: `audio` |

Image or WAV files are limited to 1 MiB. WAV must be PCM16, 16 kHz, mono and at most 30 seconds long. Required `output.speech` accepts `none`, `device` or `client`. Unknown request fields are rejected.

Optional `interrupt` defaults to `true`: if the device is in a conversation, speaking or running a task when the turn starts, that session is ended first. With `false` the turn does not interrupt: it ends `failed` with `error.code=DEVICE_BUSY` (`retryable=true`) and its user message is removed from history; retry later.

Optional `retention` sets what the turn leaves on the device and defaults to `full`: it is saved to chat history and updates the dialogue state, summary, long-term memory and diary (all extracted from the turn's chat history). `chat` saves it to chat history without those updates. `none` leaves nothing: no chat history, no stored image and no updates; the reply appears only in the turn's `assistant_text`, with empty `user_chat_ids` and `assistant_chat_ids`. With every value the reply is still generated from the existing chat history and memory.

Optional `tools` sets whether the model may call tools in this turn and defaults to `all`: the model can call the device tools and external tools available to the session. With `none` the model is offered no tools for the turn, which suits turns that only need a reply and must not trigger device actions or external tasks.

`input.type=audio_stream` also accepts JSON with `audio.format=pcm_s16le`, `sample_rate=16000`, `channels=1`, `container=raw`, up to 30 seconds. For streamed input or `speech=client`, connect `/v1/ws/chat/turns/{turn_id}/audio` by a signed WebSocket GET within 15 seconds and wait for `ready`. Receive MP3, WAV or Ogg Opus according to metadata; after `audio.end`, send `audio.ack` with the final sequence to acknowledge receipt, not playback. Disconnect ends the turn; reconnection and replay are unsupported. See OpenAPI for message schemas.

```sh
python -m pip install websocket-client sounddevice av
python examples/local_api.py http://DEVICE_IP:8081 --mic --seconds 10 --credentials credentials.json --timeout 180
```

Read `data.turn_id` from the accepted response and poll the turn, for example once per second. Terminal states are `completed`, `failed` and `cancelled`. Read `data.assistant_text` on completion or `data.error` on failure. `transcript` contains transcription; `user_chat_ids` and `assistant_chat_ids` link history records. Intermediate states are not final success.

Other states are `queued` (waiting), `starting` (opening a session), `receiving_input` (microphone or stream input), `thinking` (waiting for generation), `responding` (generating or delivering a reply), and `cancelling` (cancellation in progress). `input_complete` only marks the end of input; `transcript` may arrive before a terminal state. `device_microphone` automatically ends input after at most 30 seconds; `input/complete` can end it earlier. `speech=none` still runs the model without reading the reply aloud; `retention` controls whether the conversation is saved. Check `/data/user/device/state` before each request for current admission conditions.

`202` means accepted only. Within the retention window, replaying the same credentials, `request_id` and content returns the original turn; different content returns `409 REQUEST_ID_CONFLICT`. Every HTTP retry still needs a fresh signing nonce. With `interrupt=true`, a new turn stops the old session; an idempotent replay does not interrupt again.

Terminal turns are retained for 30 minutes, with at most 256 cached turns. Device service restart clears turn and idempotency records. Character/user changes, invalid credentials or cleared history can stop old turns or make them inaccessible. Cancellation applies only to the specified turn created by those credentials.

### Standalone text-to-speech (TTS)

Send text already determined by an external app, such as a Home Assistant action result, for the current character to read directly. This API does not call an LLM, generate a character reply or write chat history.

Submit JSON to `POST /data/user/tts/utterances`. Paths are relative to `/v1/api`; apply the HMAC signing described above:

```json
{
  "request_id": "e14eec98-5f67-445f-b605-bcc0b23f8f75",
  "slot": 1,
  "text": "The living room light is off."
}
```

Poll `GET /data/user/tts/utterances/{utterance_id}`. Cancel with `POST /data/user/tts/utterances/{utterance_id}/cancel` and an empty body.

202 means accepted; read `data.utterance_id`. States are `queued`, `synthesizing`, `playing`, `cancelling`, with terminal states `completed`, `failed`, `cancelled`. Completion requires both synthesis and device playback; failures appear in `data.error`. Text must contain 1–16000 characters after trimming; maximum JSON body 128 KiB. The current slot and its character voice, language and volume are used. The device must be awake and unlocked. A new utterance interrupts the foreground session; a new chat may interrupt speech. There is no playback queue.

An identical retry with the same credentials and request_id returns the original task (200); different content returns 409. The request_id namespace and 256 cached entries are shared with chat; terminal tasks are retained for 30 minutes and restart clears them. Invalid credentials, user/character changes or cleared history may stop or invalidate tasks. Maximum task lifetime: 180 seconds. No audio file or audio stream is returned.

```sh
python examples/local_api.py http://DEVICE_IP:8081 "The living room light is off." --credentials credentials.json --tts --timeout 180
```

`playing` requires a device-start acknowledgement for the current playback stream. When animations are enabled, one Talking animation is selected randomly. Cancellation returns 202 when accepted, 200 if already cancelling or cancelled, and 409 if completed or failed. A client wait timeout does not cancel the device task.

## 6. Other features and result confirmation

See the complete offline reference and OpenAPI for exact paths, fields and responses.

| Category | Features |
| --- | --- |
| Access and pairing | Connectivity, pairing, cancellation |
| Device and status | System information, capabilities, device/conversation status, AI language |
| Character and proactive interaction | Properties, slots, model/outfit upload and export, loading, reset, proactive settings |
| Conversation and history | Turns, history queries/deletion, chat images |
| Memory and diary | Queries and editing |
| Alarms and memos | Resource management, alarm enable/disable |
| Music | Upload, file management, playback and status |
| Voice and sound | Capabilities, standalone TTS, fixed-text audition, control, voice cloning |
| Lighting | Projects, preview, compilation, playback, apply, master switch |
| AI services | Provider catalog, profiles and tests, capability routes, system prompt, architecture/backend switching, latest runtime error |

Pairing does not grant extra device capabilities. Read `capabilities` from `/data/user/system`; individual operations list their `x-capabilities`. The Laboratory API documentation page and AI service configuration are available to all feature tags. Existing pairing signature authentication still applies.

A successful response can mean saved or queued, rather than completed. Query turns for conversations, task/loading status for model uploads/apply/export, playback status for music, and `/data/user/tts_audition/state` for voice auditions. Audition accepts only `voice_id` and optional `group_key`; it uses fixed text in the current AI language and does not accept arbitrary speech text.

Model files: at most 200 MiB. Voice-cloning files: at most 10 MiB. Batch music uploads: at most 200 MiB and 64 files per request. Ordinary signed bodies and compressed chunks: at most 8 MiB. Total multipart request sizes also include field overhead; consult each endpoint's constraints.

Optional status channel: `/v1/ws/events`. Use a signed GET handshake with query parameters `sig`, `timestamp`, `nonce`, `payload_mode=empty`, `body_hash`, `user_id`, and `session_id` when nonempty. Sign `/v1/ws/events` without its query string. This channel pushes status snapshots, not streaming generated chat text. Invalid credentials or user changes, among other conditions, close the connection.

After receiving `ready`, send `{"type":"subscribe","request_id":"sub-1","data":{"events":["chat_history","chat_turn"]}}`. Wait for a successful `result`. `events` replaces the complete subscription set; an empty array cancels all subscriptions. Resubscribe after reconnecting, then recover state through HTTP. Public contracts include only capabilities available to both Release builds and the RELEASE feature tag. Raw OpenAPI descriptions default to English.

## 7. Troubleshooting and feedback

| Symptom or code | Action |
| --- | --- |
| Cannot connect | Check IP, port and LAN connectivity; call ping first |
| `401 REQUEST_EXPIRED` | Align clocks; generate a new timestamp and signature |
| `401 NONCE_REUSED` | Use a new nonce for every request, including polling and retries |
| `401 LOCAL_SESSION_EXPIRED` | Pair again using a new credentials filename |
| `401 LOCAL_USER_MISMATCH` | Check current user; fetch bootstrap and pair again |
| `401 LOCAL_SECRET_MISMATCH` / `INVALID_SIGNATURE` | Check original secret, signing path, body bytes and headers |
| `403` | Check error fields and device capabilities; pairing cannot bypass permissions |
| `409 SLOT_MISMATCH` | Read the current slot again |
| `409 DEVICE_BLOCKED` / `AMBIENT_MODE_BLOCKED` | Read `reason`; wake/unlock the device and check conversation admission in the current mode |
| `413` / `415` | Check size, Content-Type, input/output type and WAV format |
| `503` | Use the error code to check service readiness or the cached-turn limit |

OpenAPI error-response `code.enum` lists local HTTP codes for chat, TTS, alarms, memos and device clone recording. Each operation exposes `x-error-codes.http`; `x-error-codes.task` lists known asynchronous errors and providers may add task codes. After HTTP 202, read `data.error` for failures. A rejected pairing source returns `403 PAIRING_SOURCE_NOT_ALLOWED`, conflicting idempotency content returns `409 REQUEST_ID_CONFLICT`, and a missing memo returns `404 MEMO_NOT_FOUND`.

`x-error-codes` also covers known AI provider, model task, character reset, memory and music errors. Diary legacy identifiers appear under `legacy_error` and use the response `error` field. Known task errors include `NO_SPEECH`; `DEVICE_INTERRUPTED.reason=vad_start` means speech start was detected. The reason string remains extensible.

Error structures differ by feature. Check HTTP status first, then the endpoint's `code`, `message`, `reason` or other documented fields. Do not blindly retry writes.

For feedback, provide the device version, source commit from `manifest.json`, event time and device time zone, method/path, HTTP status, redacted response, relevant `request_id` / `turn_id`, and observed device behavior. Do not send `local_secret`, credentials files or signing headers.

Admin endpoints, debug audio, diagnostic logs and internal maintenance are outside this public contract. This package verifies the source-level API contract; builds, device calls and cloud end-to-end behavior require separate validation.
