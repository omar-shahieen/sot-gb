# Meeting prep: Sout (صوت)

## 1. Decisions made 

- **Mobile:** native Kotlin + Compose. The phone is a full offline runtime, with Room encrypted at rest (AES-256 SQLCipher).
- **Preprocessing:** runs on the phone only, implemented as a shared Rust library (`libs/preprocessing`) with golden test vectors checked in CI.
- **Translation:** one whole sentence per request (REST with a protobuf body over TLS 1.3). If the server doesn't answer in about 3 seconds, the phone falls back to the on-device model.
- **Backend:** Java Spring Boot API with OAuth2 (JWT tokens), TLS 1.3, and AES-256 encryption, calling a Python gRPC model worker (Translate, SpeechToText, TextToSpeech). Data lives in PostgreSQL and S3/R2 object storage. Supports full right-to-erasure cascade purge.
- **Sync:** local-first, per-record, last-write-wins. History backup is opt-in and end-to-end encrypted (AES-256-GCM).
- **Privacy & Modes:** precise location never leaves the phone (emergency SMS only). F2b temporary mode runs strictly in-memory (RAM) with memory zeroing on completion. Notifications and alerts (F8b) are managed via an in-app center and FCM push. Raw signals stored only with explicit consent.
- **Repos and tooling:** product monorepo plus a separate AI repo; Gradle + just + buf + uv; GitHub Actions with path filters.
- **Deployment:** Docker Compose on one GPU VM now, Kubernetes manifests as the production path.
- **Speech:** on-device Android TTS and SpeechRecognizer offline, server models online.




## 2. Requirements in brief

Sout is an Android app plus an 8-channel EMG/IMU armband that translates Egyptian Sign Language into spoken Egyptian Arabic. It works online with a large model and offline with a small model.

- **Core (F1–F4):** sentence translation, Bluetooth control with gestures, preview and confirm, calibration.
- **Safety and conversation (F5–F8b):** hearing person's speech as text, history and favorites, emergency gesture with SMS, offline mode, notification center (F8b). F2b is the temporary (ephemeral) mode.
- **Future (F9–F10):** sign dictionary and learning mode, text-to-sign replies.

## 3. What to do next

- **Mobile:** build the 8 screens with mock data, define the `SensorSource` interface and a fake sensor, and set up Room with encryption. Then add the emergency flow, the recorder mode, F2b ephemeral mode, and the device profiler.
- **Backend and infra:** set up the monorepo, buf, CI and Docker Compose. Build the Spring API with OAuth2 (JWT), right-to-erasure endpoint, then wire a stub Python worker that returns canned Arabic text, so the end-to-end path works early.
- **AI:** export a dummy-weight BiLSTM+CTC model for the runtime benchmark, build the Rust preprocessing crate (`libs/preprocessing`) with golden vectors, and write the data collection protocol. Contact interpreters and Deaf associations.
- **Hardware:** finalize the BLE GATT contract with Mobile and AI (sample rates, channel order, scaling, shared clock).

## 4. Things to raise (open questions)

1. **Emergency escalation:** does emergency (F7) ever call emergency services, or only send SMS alerts to saved contacts?
2. **Device support baseline:** what is the lowest supported Android device / chipset target for on-device BiLSTM+CTC inference?
3. **Armband hardware:** which microcontroller and IMU configuration will be chosen (ESP32 vs. nRF52, BLE throughput, battery constraints)?
4. **Sensor fusion:** how does the AI model fuse IMU with EMG signals (separate branch vs. feature concatenation), and how does that shape the BLE GATT contract?

## 5. How to run the meeting (about 45–60 min)

| Time | Item | Goal |
| --- | --- | --- |
| 5 min | Opening and goals | Align on the purpose |
| 10 min | Decisions recap | Confirm or challenge them |
| 10 min | Open questions (section 4) | Decide or assign an owner |
| 15 min | Task review per team | Mobile, Backend/Infra, AI, Hardware: blockers and dependencies |
| 5 min | Next 2 weeks | Pick the first milestone |
| 5 min | Wrap-up | Read back actions, owners and dates |

**Tips to keep it moving:**
- Share the architecture doc and tasks.md beforehand so people come prepared.
- Timebox each question and use a parking lot for side discussions.
- For each open question, ask: who decides, and by when?
- Suggested first milestone: the end-to-end skeleton (fake sensor → phone → API → stub worker → speech), which unblocks all four teams.
