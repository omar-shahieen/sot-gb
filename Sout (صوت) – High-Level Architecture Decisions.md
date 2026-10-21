# Sout (صوت) – High-Level Architecture Decisions

Oct 19, 2026 · @omar

## Context and architecture drivers

Sout is an Android app paired with an 8-channel EMG armband that turns Egyptian Sign Language into spoken Egyptian Arabic, and five requirements from the requirements doc shape every decision below.

- **Offline-first phone.** F2 (sensor link), F2b (temporary mode), F3 (preview), F4 (calibration), F6 (history, favorites), F7 (emergency) and F8 (offline mode) must work with no server, so the phone is a full runtime, not a thin client.
- **Two models, one input.** A large server model and a small on-device model must receive identically preprocessed signals (EMG and IMU together), or their results diverge.
- **Streaming inference for low latency.** Signal data is preprocessed and streamed to the models in real time as the user signs rather than buffered and sent in a single bulk request after signing ends. The encoder processes features incrementally on the fly, allowing the translated Egyptian Arabic sentence to be emitted almost immediately when the signing gesture completes, drastically reducing conversational latency.
- **Privacy by design and Right-to-Erasure.** Signals are encrypted in transit via TLS 1.3 and stored only with explicit consent. Local databases and E2EE backups use AES-256 encryption. History and precise location stay on the phone; only a coarse place category is ever used for inference. Users have full right-to-erasure to completely purge their accounts and cloud data. F2b provides an ephemeral temporary mode that operates strictly in-memory without persistence.
- **Parallel teams.** Mobile, AI and Backend (plus the armband) work in parallel, so every boundary needs an explicit, versioned contract.

The designed screens (Home, Calibration, Signing, Preview, Conversation, Phrases, Emergency, Offline home) match this split: the Offline home screen lists only the features that run locally.

## System overview


The phone owns sensing, speech and personal data and can translate offline; the cloud runs the large models behind a Spring Boot API.

## Technology stack

The stack is native on the phone, Java on the API, and Python only where the models live.

| Component | Technology | Why |
| --- | --- | --- |
| Android app | Kotlin, Jetpack Compose, Room with AES-256 encryption at rest (SQLCipher), foreground service for BLE | Best BLE reliability, background streaming, silent emergency SMS, offline TTS, and F2b in-memory zeroing |
| Preprocessing library | Rust shared library (`libs/preprocessing`), JNI / UniFFI bindings | Runs on mobile only; guarantees identical signal math to AI training pipelines with zero server preprocessing |
| On-device model runtime | Not chosen yet; candidates LiteRT, ONNX Runtime Mobile, ExecuTorch, behind one runtime interface | BiLSTM and CTC conversion and low-end phone speed must be benchmarked first |
| Speech output (F1, F7) | Android on-device TTS for Egyptian Arabic; server neural TTS online | F7 must work offline; online voice is higher quality |
| Speech input (F5) | Server speech-to-text on the GPU worker online; Android SpeechRecognizer offline | Matches the requirements doc |
| API gateway | Java, Spring Boot, Spring Security with OAuth2 (JWT tokens), TLS 1.3, AES-256 | Accounts, OAuth2 auth, sync, right-to-erasure account deletion, model downloads, routing to worker |
| Model worker | Python, gRPC server, PyTorch or ONNX on GPU | Translation, speech-to-text and server TTS next to the models |
| Database | PostgreSQL | Users, devices, favorites, settings, sync cursors, right-to-erasure cascade support |
| Object storage | S3-compatible bucket \|\| R2 Bucket | Encrypted backup blobs (AES-256-GCM), on-device model files, dictionary videos |
| Armband | Hardware undecided, but it has an IMU beside the EMG electrodes; hardware-agnostic BLE GATT contract with EMG and IMU streams | Open decision, see the last section |
| Contracts | Protobuf with buf; OpenAPI for REST | Versioned, checked for breaking changes in CI |
| Build | Gradle multi-project, just, buf, uv | Native Android and Spring builds, one command surface |
| CI/CD | GitHub Actions with path filters | Each component builds only when it changes |
| AI training | Separate repository, Python | Datasets and checkpoints stay out of the product repo |

## Communication and contracts

Three links carry data, and each has one versioned contract: BLE to the phone, streaming/REST from the phone, gRPC to the worker.

| Link | Protocol | Payload | Notes |
| --- | --- | --- | --- |
| Armband to phone | BLE GATT, custom EMG service | Time-stamped 8-channel EMG samples and IMU samples, battery, per-channel contact quality | Hardware undecided, so the contract fixes sample rates, channel order and scaling once, shared by Mobile and AI. EMG and IMU share one clock so they can be aligned. |
| Phone to API: accounts, sync, downloads, erasure | REST with OpenAPI over TLS 1.3 | JSON | OAuth2 JWT: short-lived access token, rotating refresh token. AES-256 for encrypted payloads. Right-to-erasure endpoint completely purges user account and cloud data. |
| Phone to API: translation | Streaming (WebSocket / gRPC over HTTP/2) with Protobuf over TLS 1.3 | Real-time stream of preprocessed EMG and IMU chunks, calibration values, optional coarse place category, sentence completion flag | Preprocessed on mobile only by `libs/preprocessing`. Signal chunks are streamed continuously during signing to reduce end-to-end latency. Ephemeral mode (F2b) sessions remain strictly in RAM. |
| API to worker | gRPC client/bidirectional streaming | Streamed protobuf signal chunks | Methods: StreamTranslate (streaming chunks $\rightarrow$ translated Arabic text upon end-of-sentence flag), SpeechToText, TextToSpeech. |
| Phone to object storage | HTTPS download, signed manifest | Model files, dictionary videos | The app checks the manifest version and checksum before activating a model. |

**Request flow.** The phone continuously cleans and extracts features from the EMG and IMU stream via `libs/preprocessing` and streams these preprocessed chunks in real time to the server over a persistent connection. The model's encoder ingests frames incrementally as the user signs. When the user finishes the sentence (signified by a fist-clench gesture or natural signing pause), an end-of-sentence flag triggers the decoder, which emits the final Egyptian Arabic text immediately to the preview screen. If the streaming connection drops or server latency exceeds the fallback threshold, the phone falls back seamlessly to the local on-device streaming model. In F2b temporary mode, the translation session bypasses disk persistence and cloud sync entirely.

**Preprocessing on mobile only via shared library.** Preprocessing runs strictly on the mobile device only, executed by the shared native Rust library (`libs/preprocessing`) integrated into Android via JNI / UniFFI bindings. The server performs zero preprocessing on translation requests. Parity is guaranteed because the AI team defines the signal processing pipeline and supplies golden test vectors (raw sensor input and expected preprocessed output). Automated CI checks on both Android and the AI training repo validate against these golden vectors, guaranteeing both on-device and server models receive identical features without duplicate codebase drift.

## Data placement and offline modes

The phone is the source of truth for everything personal; the cloud holds accounts, consented data and files the phone downloads.

| Data | Phone | Cloud | Rule |
| --- | --- | --- | --- |
| Conversation history | Yes, Room (AES-256 SQLCipher), PIN or biometric lock | Only as an opt-in, end-to-end encrypted blob (AES-256-GCM) | The server cannot read it. |
| Ephemeral / Temporary mode (F2b) | In-memory (RAM) only; memory zeroed on exit | Never | Bypasses Room SQLite database and cloud sync completely; leaves zero disk traces. |
| Favorite phrases | Yes | Yes, synced per record | Low sensitivity. |
| Settings (confirm mode, contacts, message) | Yes | Yes, synced | Emergency contacts are sensitive, so they sync only with the authenticated account. |
| Calibration profile | Yes | Yes, as a starting hint | Band placement changes, so a restored profile needs a quick channel check. |
| Precise location | Used for the emergency SMS only | Never | Not part of any API. |
| Coarse place category and place labels | Yes, user-chosen | Category only, with a translation request | No automatic lookup; the user picks or confirms it. |
| Raw EMG and IMU signals | Buffered during a sentence | Only with explicit consent, for training | Off by default. |
| Accounts and devices | OAuth2 token cache | PostgreSQL | OAuth2 JWT auth over TLS 1.3; subject to complete right-to-erasure cascade deletion. |
| On-device models, dictionary videos | Downloaded copies | Object storage with a signed manifest | Versioned, checksummed. |

## Sync design

Sync is local-first: Room on the phone is the source of truth, and the app works the same with or without a connection.

- **Favorites and settings.** Each record has an ID, a modified time and a deleted flag. The phone queues changes while offline and pushes them when connected. On conflict the latest edit wins, and deletions propagate as tombstones.
- **History backup (opt-in).** The phone encrypts new conversation entries with AES-256-GCM using a key derived from the user's PIN or passphrase, then uploads them as append-only encrypted chunks. The server stores ciphertext only. A forgotten passphrase means the backup cannot be recovered, and the app must say so before the user enables it.
- **Temporary gesture mode (F2b).** Operates completely in volatile memory (RAM). Translations and dialogue during an ephemeral session bypass Room SQLite persistence and cloud sync entirely. When the session terminates, all buffered data is actively zeroed out in memory.
- **Right-to-erasure account deletion.** A dedicated REST endpoint initiates an immediate cascading purge across PostgreSQL and object storage, removing user accounts, credentials, device tokens, synced settings, and all encrypted backup blobs, accompanied by a clean local database reset.
- **Models and dictionary.** One-way download from the signed manifest, over Wi-Fi by default.

## Repositories, build and CI/CD

The product lives in one monorepo built with Gradle, just and buf; AI training lives in its own repository and ships versioned models.

```plain
sout/                      (product monorepo)
  contracts/               protobuf, OpenAPI, signal spec, golden test vectors
  apps/android/            Kotlin, Compose (Gradle)
  apps/api/                Java, Spring Boot (Gradle)
  apps/model-worker/       Python, gRPC (uv)
  libs/preprocessing       Rust preprocessing library
  firmware/                armband firmware, once hardware is chosen
  infra/                   Docker Compose, Kubernetes manifests, CI
  docs/                    architecture, ADRs, requirements
  justfile                 one command surface: build, test, proto, lint

sout-ai/                   (separate repo)
  data, training, evaluation, export; publishes versioned model artifacts
```

- **Build tool.** Gradle builds Android and Spring natively with one protobuf setup. `just` wraps uv and firmware commands, and `buf` lints contracts and blocks breaking changes. Bazel was rejected: its setup cost and Android friction outweigh its cross-language caching for a team this size. Revisit only if builds become slow.
- **Model release path.** The AI repo's CI exports a model, uploads it to object storage, and updates the signed manifest. The monorepo only consumes the manifest.
- **CI.** GitHub Actions with path filters. Contract checks (buf breaking, golden vectors) run on every change.

## Deployment, security and privacy

**Deployment.** Docker Compose on one cloud GPU VM runs the Spring API, the Python worker, PostgreSQL and a TLS reverse proxy for development and the demo. Kubernetes manifests in `infra/` are the production path, so the same images move over unchanged.

**Security baseline (OAuth2, AES-256, TLS 1.3):**

- OAuth2 framework with short-lived JWT access tokens and rotating, revocable refresh tokens.
- TLS 1.3 enforced across all external network transport and API communications.
- AES-256 encryption at rest: SQLCipher for Room database on Android; AES-256-GCM for client-side E2EE conversation backups (server stores ciphertext only and never holds keys).
- Passwords hashed with Argon2 or bcrypt, never stored in plain form.
- Email verification, password reset, and rate limiting on login and translation endpoints.
- Right-to-erasure: GDPR-compliant account deletion endpoint initiating complete, irrevocable cascading purge of all user records, synced data, and cloud backup blobs.
- Ephemeral session security (F2b): translation session data held strictly in RAM with zeroing upon completion; zero disk retention and zero cloud sync.
- HTTPS/TLS 1.3 everywhere; no raw signals stored without explicit recorded consent.

**Privacy rules carried from the requirements.** History and precise location stay on the phone. Location context is opt-in, user-chosen and visible. Only the coarse category is sent, and it must never override strong sensor evidence.

## Decision log

Twenty-two decisions are made, one is deferred to a benchmark, and one waits on hardware.

| # | Decision | Chosen | Alternatives rejected | Main reason |
| --- | --- | --- | --- | --- |
| 1 | Mobile stack | Native Kotlin + Compose | Flutter, React Native | BLE, DSP, on-device ML and silent SMS are native work; iOS cannot send SMS silently |
| 2 | Preprocessing location | Phone only, via shared Rust library (`libs/preprocessing`) | Server-side preprocessing, dual native codebases | Parity guaranteed by shared library and golden test vectors; eliminates server compute load |
| 3 | Translation payload | Real-time streaming chunks over persistent connection (WebSocket/gRPC) | Whole-sentence single batch request | Streaming allows the model encoder to process features incrementally as the user signs, reducing end-to-end latency so translation emits immediately when signing completes |
| 4 | Hosting | Containerized cloud GPU VM | University server, serverless | GPU access and a realistic deployment |
| 5 | Backend language | Java, Spring Boot | Go, Node | Team choice |
| 6 | API to worker | gRPC streaming | HTTP/JSON, unary calls, queue | Real-time chunk streaming with typed Protobuf contract directly to model encoder |
| 7 | Model worker | Separate Python service | In-process in the API | Models are PyTorch or ONNX and need the GPU |
| 8 | Authentication & Security | OAuth2 with JWT, TLS 1.3, AES-256 encryption | Self-built non-standard auth, basic auth | Industry standard, robust token lifecycle, matches requirements |
| 9 | Cloud data | Accounts, settings, calibration hint, consented training data, optional backup | Minimal-only, full cloud history | Restore on a new phone without reading conversations |
| 10 | History backup | Opt-in, end-to-end encrypted (AES-256-GCM) | Server-readable, favorites only | Matches the privacy requirement; server stores ciphertext only |
| 11 | Client API | Streaming Protobuf (WebSocket/gRPC) for translation; REST (OpenAPI) for accounts/sync | Unary REST for translation, JSON everywhere | Minimizes latency during live signing while keeping CRUD and account endpoints simple and debuggable |
| 12 | Sync | Local-first, per-record, last-write-wins | Manual snapshot, CRDT | Single-user edits do not need more |
| 13 | Repositories | Product monorepo + separate AI repo | Single monorepo, polyrepo | Atomic contract changes without bloating the repo |
| 14 | Build tool | Gradle + just + buf (+ uv) | Bazel&#32; | Native Android and Spring builds; low setup cost |
| 15 | Data stores | PostgreSQL + S3-compatible storage | Postgres only, MongoDB | Relational records, large blobs in object storage |
| 16 | Deployment | Docker Compose now, Kubernetes for production | Kubernetes only | Simple start, production path kept open |
| 17 | Text-to-speech | On-device Android TTS + server neural TTS online | On-device only, own model | Offline for F7, better voice online |
| 18 | Speech-to-text (F5) | Server model on the worker + Android SpeechRecognizer offline | Third-party API only | Matches requirements; audio stays in our infrastructure |
| 19 | Model delivery | Object storage + signed manifest | Bundled in APK, Play asset packs | Model updates without app releases |
| 20 | Place category | User-chosen, saved place labels, no automatic lookup | Places API, offline map | Coordinates never leave the phone |
| 21 | On-device runtime | Deferred until benchmarking | LiteRT, ONNX Runtime Mobile, ExecuTorch | BiLSTM and CTC conversion must be tested on a low-end phone |
| 22 | Armband hardware | Deferred | ESP32, nRF52, vendor device | Decides BLE throughput for the EMG and IMU streams, and the firmware toolchain |
| 23 | Temporary mode (F2b) | In-memory only ephemeral session state with memory zeroing | Persisting with temporary flags | Strict privacy guarantee; leaves zero trace in local storage or cloud |
| 24 | Right to erasure | Cascade hard delete endpoint (PostgreSQL + S3/R2 blobs) | Soft-delete only, manual DB cleanups | Fulfills user privacy rights and GDPR compliance |

## Open questions, risks and next steps

**Open questions**

- Armband hardware: the band has an IMU that feeds inference, but the chip, IMU axes and sample rates are open. How the AI team fuses IMU with EMG (extra model branch or feature concatenation) fixes the signal spec.
- On-device runtime: which of LiteRT, ONNX Runtime Mobile and ExecuTorch runs the compressed BiLSTM model acceptably on the lowest supported phone?
- Does emergency ever call services, or only message saved contacts?

**Assumed defaults to confirm.** These were not discussed and are proposed here:

- Room database encrypted at rest with AES-256 (SQLCipher), unlocked by the app PIN or fingerprint.
- Translation timeout of about 3 seconds before falling back to the on-device model.
- Server neural TTS and speech-to-text hosted on the same GPU worker as translation.

**Main risks**

| Risk | Effect | Mitigation |
| --- | --- | --- |
| Preprocessing differs between training and phone | Accuracy drops silently | One spec, shared Rust library, golden vectors checked in CI on both sides |
| OAuth2 token leakage or implementation flaw | Account or data exposure | Follow OAuth2 RFCs, short-lived JWTs, rotating refresh tokens, rate limiting, review before release |
| E2EE passphrase forgotten | Backup unrecoverable | Warn before enabling; keep backup optional |
| Model conversion fails for BiLSTM or CTC | Offline mode delayed | Benchmark spike first; keep the runtime interface swappable |
| Single GPU VM fails during a demo | Failed presentation | Offline model as backup, recorded demo |
