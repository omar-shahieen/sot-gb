# Sout (صوت) – Project Tasks 

---

##  Mobile  (Android)

- [ ] **Android Core App & UI (Without AI)**
  - [ ] Implement the 8 designed screens with mock data (Home, Calibration, Signing, Preview, Conversation, Phrases, Emergency, Offline Home).
  - [ ] Setup Room database with encryption at rest (SQLCipher) for conversation history and favorite phrases (F6).
  - [ ] Implement emergency workflow with countdown timer, GPS coordinate capture, and background/silent SMS dispatch (F7).
  - [ ] Implement translation interaction modes: Preview and Confirm modes (F3).
  - [ ] Build online/offline connectivity mode manager and local settings storage.
- [ ] **Sensor Abstraction & Simulator** -
  - [ ] Define the `SensorSource` interface in Kotlin for 8-channel EMG + IMU data streams.
  - [ ] Build a mock/fake `SensorSource` that replays synthetic signals or public EMG/IMU datasets.
- [ ] **Data-Collection Recorder Mode**
  - [ ] Build an in-app recording tool that captures labeled EMG and IMU sessions.
  - [ ] Include prompt list UI, session IDs, repeated placement notes, and consent confirmation.
- [ ] **Temporary Gesture Mode (F2b)**
  - [ ] Ephemeral in-memory translation session state bypassing Room DB and cloud sync.
  - [ ] Status badge, memory zeroing on session end, and termination confirmation dialog.
- [ ] **History Privacy & Export (F6)**
  - [ ] Tombstone soft-delete (`⊘ This message was deleted`) with permanent purge.
  - [ ] Storage preferences toggle ("Translated text only" vs "Full dialogue").
  - [ ] Single and bulk transcript export to `.txt` and printable `.pdf`.
  - [ ] Biometric / PIN lock for local history archive.
  - [ ] Client-side E2EE backup encryption (AES-GCM derived from user passphrase/PIN).
  - [ ] Selective link sharing UI and one-tap revocation action.
- [ ] **Notification Center & Alerts (F8b)**
  - [ ] Screen 20 with category tabs (`Reminders`, `Warnings`, `Updates`) and unread badges.
  - [ ] Deep-link action cards and reusable confirmation modal framework.
- [ ] **Location Context & Privacy Abstraction (F1)**
  - [ ] Coarse place category selector and custom user place labels.
  - [ ] GPS isolation ensuring coordinates stay on-device for emergency SMS only.
- [ ] **Device Capability Profiler & Fallback (F8)**
  - [ ] First-launch benchmark classifying device as Capable vs Weak.
  - [ ] 3-second server translation timeout fallback.
- [ ] **Haptic Feedback Engine**
  - [ ] Tactile vibration patterns for partner reply (F5), emergency countdown (F7), and alerts.

Later:
  
- [ ] **On-Device Evaluation & Benchmarking (Co-owned with AI Team)**
  - [ ] Integrate LiteRT and ONNX Runtime Mobile runners into Android.
  - [ ] Benchmark BiLSTM+CTC runtime latency, memory, and binary footprint on low-end test phones.
  - [ ] Test Android on-device offline Egyptian Arabic TTS voice quality (critical for F7).
  - [ ] Benchmark Android offline `SpeechRecognizer` for Egyptian Arabic.
- [ ] **Native Preprocessing Integration**
  - [ ] Integrate the shared Rust preprocessing library (`libs/preprocessing`) via JNI / UniFFI bindings.

---

##  Backend  (API & Server Worker)

- [ ] **Spring Boot API Gateway (`apps/api`)**
  - [ ] Initialize Spring Boot service with Spring Security and OAuth2 JWT authentication (short-lived access token, rotating refresh token, TLS 1.3, AES-256).
  - [ ] Implement account management endpoints and PostgreSQL persistence.
  - [ ] Build sync endpoints for favorites, settings, and end-to-end encrypted history backup blobs.
  - [ ] Build model manifest endpoint serving signed version manifests, checksums, and download URLs.
  - [ ] Implement the translation REST endpoint accepting preprocessed binary Protobuf signals.
  - [ ] Implement selective sharing API (tokenized signed read-only links, public view endpoint, instant revocation).
  - [ ] Implement security baseline (refresh token revocation/blacklisting, password reset, email verification, endpoint rate limiting).
  - [ ] Implement right-to-erasure account deletion and complete user data purge endpoint.
- [ ] **gRPC Worker Gateway Integration**
  - [ ] Implement gRPC client calling worker RPC methods (`Translate`, `SpeechToText`, `TextToSpeech`).
  - [ ] Create stub Python gRPC worker returning canned Arabic text responses.
  - [ ] Wire end-to-end translation flow in Docker Compose.
- [ ] **API & Service Contracts**
  - [ ] Maintain OpenAPI specifications for REST endpoints.
  - [ ] Maintain Protobuf schemas for translation requests and worker RPCs.

---

## Infra / DevOps 

- [ ] **Monorepo Scaffolding & Build System**
  - [ ] Configure root Gradle multi-project build covering `apps/android` and `apps/api`.
  - [ ] Setup `justfile` command surface (`just build`, `just test`, `just proto`, `just lint`).
  - [ ] Setup `buf` for Protobuf schema linting, formatting, and breaking-change detection.
  - [ ] Setup `uv` for Python virtual environments and dependency management (`apps/model-worker`).
- [ ] **CI/CD Pipelines**
  - [ ] Setup GitHub Actions with path filtering to ensure components build only when their directories change.
  - [ ] Add automated contract compatibility checks (Protobuf breaking change checks via `buf` & OpenAPI validation).
  - [ ] Add CI check for preprocessing golden test vectors.
- [ ] **Local & Cloud Infrastructure**
  - [ ] Configure `docker-compose.yml` for local development (PostgreSQL, Spring Boot API, Python gRPC worker stub).
  - [ ] Setup S3 / Cloudflare R2 object storage buckets for model weights and encrypted backups.

---

## Hardware / Firmware Team

- [ ] **BLE GATT Contract Specification**
- [ ] Define BLE GATT contract characteristics with Mobile & AI teams (synchronized clock for 8-channel EMG and IMU, sample rates, scale factors, bit depth, channel order, battery, contact quality indicator).


---

##  AI / Research 

- [ ] **Model Architecture & On-Device Export**
  - [ ] Export real-size BiLSTM + CTC model with representative dummy/random weights to LiteRT and ONNX Runtime Mobile formats for mobile benchmarking.
  - [ ] Profile model parameter count, FLOPs, and quantizability (INT8 vs FP16).
- [ ] **Preprocessing Pipeline (Rust Shared Library)**
  - [ ] Design signal filtering, notch filter, normalization, and IMU feature fusion pipeline.
  - [ ] Implement shared signal processing crate in Rust (`libs/preprocessing`).
  - [ ] Generate golden test vectors (raw signal inputs and expected preprocessed outputs) to guarantee identical processing across Android and server training.
  - [ ] Test and validate preprocessing pipeline using public EMG datasets.
- [ ] **Speech-to-Text & TTS Benchmarks**
  - [ ] Benchmark Egyptian Arabic STT models online (e.g. Whisper fine-tunes) on GPU worker.
  - [ ] Evaluate neural server TTS voice quality for Egyptian Arabic.
- [ ] **Data Collection & ESL Domain Preparation**
  - [ ] Write data collection protocol: sentence list, target vocabulary, signer count, session counts, repeated band placement protocol, and consent form.
  - [ ] Contact Egyptian Sign Language (ESL) interpreters and Deaf associations for vocabulary review and sign ambiguity validation.
