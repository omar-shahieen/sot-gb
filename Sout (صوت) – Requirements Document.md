# Sout (صوت) – Requirements Document

Oct 17, 2026 · @omar

## 1. Overview

Sout is an Android phone app paired with an 8-channel forearm EMG armband and IMU sensor that turns Egyptian Sign Language into spoken Egyptian Arabic, so a Deaf or speech-impaired person can hold a conversation without a human interpreter. 

This version of the product uses an Android phone, an 8-channel EMG armband, and an IMU sensor. 

## 2. Hybrid architecture

Translation runs on a server in the normal (online) mode, and a small on-device model gives a limited offline mode with lower accuracy.


The armband streams to the phone. With a connection, the phone streams preprocessed signal chunks (via the native `libs/preprocessing` library) to the server in real time to minimize latency, and receives Arabic text as soon as signing completes along with high-quality server neural TTS speech; without one (or when server latency exceeds ~3 seconds), the small streaming model on the phone processes the stream locally and speaks using Android on-device TTS. Both paths end in the same preview and speech step.

| Situation | What runs | Features available |
| --- | --- | --- |
| Online, any phone | Large streaming model on the server for translation and speech-to-text; server neural TTS for Egyptian Arabic speech; phone does Bluetooth, native preprocessing (`libs/preprocessing`), audio playback, and screens | All features |
| Offline, capable phone | Small streaming model on the phone for a limited set of phrases; Android on-device TTS; phone’s own speech recognizer if it is good enough | F1 (limited vocabulary), F3, F4, F5 (limited), F6, F7 |
| Offline, weak phone | No translation model; phone runs only local features with on-device TTS | F6, F7, the channel check in F4, and the downloaded dictionary from F9 |

On first launch the app tests the phone and decides whether it can run the small model. At run time it uses the server when a connection exists and falls back to the on-device model when it does not or when server streaming latency exceeds ~3 seconds. The app shows the current mode and warns that offline vocabulary and accuracy are limited.

## 3. Features


### F1. Sign-to-speech translation

**What it is.** The user signs a sentence wearing the armband, and the app speaks the translation in Egyptian Arabic. To eliminate conversational delays, Sout uses chunked streaming: the phone continuously preprocesses sensor data via a native library (`libs/preprocessing`) and streams small feature chunks (every 100–200 ms) via WebSocket or gRPC over TLS 1.3 to the model while the user is signing. The model (a streaming BiLSTM + CTC architecture) updates its incremental encoder state in real time, allowing it to output the completed Egyptian Arabic sentence and deliver speech synthesis immediately after the user finishes the gesture. Online mode streams high-quality neural server TTS audio, while offline mode uses Android on-device TTS.

To boost accuracy, the user can provide optional context before signing. They can select or confirm a coarse location (e.g. pharmacy, bank), a conversation topic (e.g. medical, shopping), or specific expected phrases and keywords. These hints guide the model toward the right vocabulary, and any active context can be adjusted or cleared with one tap.

**Scenario.** Mona is Deaf and needs a headache medicine. At the pharmacy she signs her question. The phone sends the preprocessed signal chunks to the server, and immediately upon finishing her sentence, the phone plays the synthesized Egyptian Arabic translation. The pharmacist answers by voice and Mona reads the reply (see F5). Later, in a tunnel with no signal, the app switches to the offline model and still handles her most common phrases using on-device TTS.

**Key requirements**

- Translate whole sentences and output Egyptian Arabic text and speech (server neural TTS online; Android on-device TTS offline).
- Use the server streaming model when a connection exists; fall back to the on-device model when disconnected or when server latency exceeds ~3 seconds.
- Show which mode is active (online or offline) at all times.
- Allow optional user context (coarse location, topic, and phrase hints) to improve model accuracy, with one-tap editing and clearing; allow translation without context.

### F2. Sensor connection and hands-free gesture controls

**What it is.** The app pairs with the 8-channel EMG armband over Bluetooth Low Energy (BLE), receives the synchronized EMG and IMU stream, and time-stamps it. A status panel shows connection status, battery level, and electrode contact quality, with automatic reconnection if Bluetooth drops.

Translation control is completely hands-free via armband gestures. The user activates, pauses, and stops translation entirely through gestures performed with the band (e.g. start gesture, stop gesture, or fist-clench / natural pause to delimit sentences).

Translation only runs between start and stop gestures. This saves battery, minimizes data, and prevents accidental translations from everyday hand movements, while the phone screen reflects the active state in real time.

**Scenario.** Karim is at a bank counter with his phone propped on the desk. He performs a control gesture on his armband to activate translation, signs his request, and uses a stop gesture when finished. The phone screen reflects the state transitions instantly.

**Key requirements**

- Pair, stream, and auto-reconnect over Bluetooth with synchronized EMG and IMU streams.
- Hands-free translation control driven entirely by armband gestures (start, stop, sentence completion) with continuous gesture activity detection.
- Real-time visual status indicator and in-context warning banners on active screens for translation state, battery level, disconnection, and per-channel contact quality.

### F2b. Temporary Gesture Mode (Ephemeral Session)

**What it is.** A privacy-preserving ephemeral translation mode designed for private or sensitive conversations (e.g., medical diagnoses, legal consultations, financial transactions). When Temporary Gesture Mode is activated, the app enters an isolated session state marked by a distinct visual badge (`⚡ Temporary Gesture Mode`).

During temporary mode, all translated sentences, glosses, and partner replies are processed in volatile device memory (RAM) only. No records are written to the local Room (SQLCipher) conversation history, and no data is synced to cloud storage. Ending temporary mode prompts the user with a confirmation dialog before purging and zeroing the temporary memory buffer.

**Scenario.** Mona visits a specialist for a confidential health concern. She activates Temporary Gesture Mode. A pulsing amber badge confirms the session is ephemeral. After completing her consultation, she taps "End Session," confirms the modal prompt, and the app safely discards and zeroes out all dialogue without leaving any traces in her conversation history.

**Key requirements**

- One-tap activation of Temporary Gesture Mode from gesture controls and chat screens.
- Clear visual status indication (`⚡ Temporary Gesture Mode` banner/badge) during active ephemeral sessions.
- Strict isolation: zero persistence to local storage and zero cloud synchronization during temporary mode.
- Safe termination: confirmation dialog before ending the session and purging memory buffers.


### F3. Preview and confirm

**What it is.** After the user finishes a sentence, the translation appears as large text before it is spoken. The user then confirms or cancels. Confirm can be a short gesture or a button; cancel lets them sign the sentence again. This protects the user from a wrong sentence being said out loud, since no translation model is perfect.

The user picks one of three modes in settings:
1. **Manual confirm:** Never speaks without approval, suited for sensitive situations such as a doctor or lawyer.
2. **Auto-speak after a delay:** Plays the synthesized speech after 2–3 seconds unless the user cancels, keeping ordinary conversation moving.
3. **Instant speak:** Skips the preview and speaks immediately upon gesture completion, suited for short, repeated phrases.

In online mode, server neural TTS audio is received alongside the translated text and played according to the active preview mode; offline mode falls back to Android on-device TTS.

**Scenario.** Mona signs that her headache started two days ago. The preview reads “two months”. She cancels with a gesture, signs again, and the corrected sentence is spoken. Without the preview, the pharmacist would have heard wrong medical information.

**Key requirements**

- Display the translated sentence before speech in manual and auto modes.
- Confirm and cancel gestures must differ from normal signing, with on-screen buttons as backup.
- The user can change the mode at any time.

### F4. Sensor calibration

**What it is.** EMG signals differ between people (arm size, skin, muscle strength) and change if the armband sits a centimeter away from its last position. Calibration adapts the system to the current wearer. On first use, the app shows where and how to wear the band, checks that all 8 channels read a good signal, and asks for a few basic hand movements, each repeated two or three times. It then normalizes the signal to that user's range (applied by `libs/preprocessing` on the phone) and reports a quality result (good, acceptable, redo).

Calibration takes two to five minutes and can be repeated whenever the user re-wears the band. Light calibration (channel check and normalization) is part of the first release and runs locally on the phone. Fine-tuning the model on the user's own data is a possible later step and would run on the server with user consent.

**Scenario.** Omar borrows his friend's armband to try Sout. Before signing anything, the app walks him through placement and a short movement routine. Without it his first translations would be poor; after it, they are noticeably more accurate. We will measure that difference on new users before and after calibration.

**Key requirements**

- Guide placement, check channel quality across all 8 channels, and normalize the signal per user.
- Show a clear quality result and allow re-calibration at any time.
- Work offline, since it runs locally on the phone.


### F5. Hearing person’s speech as text

**What it is.** Sout so far goes one way: the user signs and the phone speaks. F5 adds the reply. The phone’s microphone listens to the person in front of the user, converts their speech to Egyptian Arabic text, and shows it in large type. The phone vibrates when a reply arrives so the user does not have to keep watching the screen.

Online, a server speech-to-text model on the GPU worker gives the best accuracy on Egyptian Arabic. Offline, the phone’s built-in speech recognizer is used where available; its accuracy on Egyptian dialect is likely lower and must be tested. If the sound is unclear or confidence is low, the app shows a clear audio warning notice so the user can ask the speaker to repeat.

**Scenario.** At the doctor’s office, Mona signs that she has a headache. The doctor answers, “Take it after meals, twice a day.” The sentence appears on the phone in large text and the phone vibrates. She reads it, signs her next question, and the conversation continues without a written note or an interpreter.

**Key requirements**

- Convert nearby speech to Egyptian Arabic text and display it in large type.
- Vibrate with distinct haptic feedback when a reply arrives.
- Warn when audio quality or recognition confidence is too low to trust.

### F6. Conversation history, storage preferences, and favorites

**What it is.** Every translated sentence and hearing partner reply can be viewed in a responsive, chat-style conversation interface grouped by date and searchable. The user has granular control over what gets saved and how long it remains:
- **History recording toggle:** Allows users to pause or resume recording conversation history at any time.
- **Storage content preferences:** Users choose between storing "Translated text only" (recommended privacy default) or "Full conversation dialogue" (both Deaf user translations and hearing partner replies).
- **Transcript export:** Conversations can be exported individually or in bulk as formatted plain text (`.txt`) transcripts or styled print-ready documents (`.pdf`).

Favorite phrases are sentences the user says often, such as “Please write it down, I am Deaf” or “I want to book an appointment”. They are grouped (doctor, bank, transport, work) and spoken with one tap, with no signing and no server. Favorites and history work fully offline, acting as a safety net when translation is unavailable.

**Scenario.** Karim opens his conversation history to review his doctor's advice. He exports a `.pdf` transcript to save with his medical records, uses quick search to locate an older note, and deletes an accidental entry, which is permanently erased on the spot. Later at the pharmacy, he uses his offline favorite phrases to request his prescription with one tap.

**Key requirements**

- Store conversations locally in encrypted Room database (SQLCipher AES-256) with date grouping, real-time search, and optional biometric/PIN lock.
- Toggle history recording on/off and select storage mode (translated text only vs. full dialogue).
- Immediately and permanently hard-delete messages from encrypted local storage with no leftover trace.
- Provide single and bulk transcript export as formatted `.txt` and printable `.pdf`.
- Create, categorize, and speak favorite phrases with one tap fully offline.
- Allow optional place labels and coarse context tags per conversation.

### F7. Emergency gesture

**What it is.** In danger, a Deaf user cannot call out for help. F7 lets them trigger an emergency action with one distinctive gesture or an always-visible on-screen button. The phone speaks a preset message aloud (for example “I am Deaf and need help”), shows it full screen so bystanders can read it, and sends an SMS with the user’s location link to saved contacts. The user chooses the contacts and the message during setup.

False alarms are the main risk. The gesture must be unusual enough not to happen by accident, and a short countdown with a large cancel button appears before sending, with a “send now” option. F7 uses the phone’s own speech (offline Android TTS), SMS and GPS, so it works offline and does not depend on the server. We do not call emergency services automatically in the first release; this is a scope decision to confirm.

**Scenario.** Omar is in a street accident and cannot speak. He makes the emergency gesture. His phone announces that he is Deaf and needs help, shows the same words on screen for the people around him, and his family receives his location by SMS within seconds.

**Key requirements**

- Trigger by gesture and by on-screen button.
- Speak and display the message, and SMS the location to saved contacts.
- Work with no internet and include a cancel countdown to prevent false alarms.

### F8. Offline mode

**What it is.** Offline mode keeps the app useful with no connection or when server streaming latency exceeds the threshold (~3 seconds). It has two levels. On a capable phone, the small on-device streaming model translates a limited set of phrases, and a local speech recognizer handles replies where it is good enough. On a weak phone, or when the small model is too slow, the app runs with no model at all and offers F6 (history and favorites), F7 (emergency) and the downloaded dictionary.

On first launch the app tests the phone (memory, speed on a sample input) and chooses a level. The app always tells the user which mode it is in and that offline accuracy and vocabulary are limited. If the connection drops or times out during a sentence, the app displays a clear banner and switches modes gracefully instead of failing silently.

**Scenario.** Karim travels to a village with weak coverage. The app shows “Offline mode: limited vocabulary”. His common phrases still translate on the phone, and his favorites and emergency button work on any device. When coverage returns, the app switches back to the server model by itself.

**Key requirements**

- Detect connectivity and device capability and pick the right mode automatically.
- Fall back to on-device streaming model if server latency exceeds ~3 seconds.
- Show the current mode and its limits clearly.
- Keep F6, F7 and F4 (channel check) fully working with no connection.


### F9. Sign dictionary and learning mode

**What it is.** The dictionary is a searchable list of Egyptian Sign Language words and phrases, grouped by topic (doctor, bank, family, numbers). Each entry has a short video showing how to sign it. Videos can be downloaded in advance so the dictionary works offline.

Learning mode uses the armband. The app shows a sign, the learner copies it, and the model tells them whether it recognized the sign. A “show what was recognized” view displays the glosses behind a translated sentence so family and coworkers can see how it was built. The hard part is the content: videos must be recorded and checked with a certified interpreter or a Deaf association, because signs vary between regions.

**Scenario.** Karim’s younger brother wants to talk to him without the phone. He practices ten signs a day in the app, gets feedback from the armband, and after a month can sign simple sentences himself.

**Key requirements**

- Searchable dictionary with a video per entry, downloadable for offline use.
- Practice mode that checks a learner’s sign with the armband.
- Content reviewed by qualified Egyptian Sign Language experts.

 
### F10. Text-to-sign reply * (NOT YET CONSIDERED)

**What it is.** Some Deaf users find written Arabic harder to read than signing, since sign language has its own grammar. F10 shows the hearing person’s reply as signs instead of text. The text from F5 is converted to sign order, then played either as a chain of short recorded videos or as a 3D avatar. The written sentence stays visible below for checking, and the user can slow down or repeat.

Video clips for a limited vocabulary are realistic for the first version. A full avatar is a much larger effort and is not planned for the first release. Both options need a second conversion step from Arabic text to sign order (rules or a model) and a recorded vocabulary.

**Scenario.** At a bank, the clerk says, “I need your ID card.” The phone, resting on the desk, plays short clips signing “ID card” and “need” for Karim, with the sentence written underneath.

**Key requirements**

- Convert reply text to a sequence of signs for a limited vocabulary.
- Play slowly or on repeat, with the written text always visible.
- Show a clear message when a word has no recorded sign.

## 4. Work division: Mobile, AI, Backend



| Feature | Mobile | AI | Backend |
| --- | --- | --- | --- |
| F1 Sign-to-speech translation | Receive sensor stream, run native preprocessing (`libs/preprocessing`), stream chunks in real time to the API, run small on-device streaming model, trigger audio playback (server neural TTS online / Android TTS offline), online/offline and latency fallback switching | Train streaming BiLSTM + CTC model architecture (incremental encoder + decoder); compress into small model; export both; maintain golden test vectors for Rust preprocessing library; measure latency and accuracy | Streaming Inference API: receive chunk stream, run incremental encoder, return Arabic text upon sentence end; stream server neural TTS audio; TLS 1.3 / WebSocket / gRPC; latency logging |
| F2 Sensor connection and gesture controls | Bluetooth pairing, streaming, time-stamps, auto-reconnect, status screen, simplified start/stop session states and in-context hardware warnings | Start and stop gesture detection and continuous gesture activity classification; test for false triggers | None |
| F2b Temporary Gesture Mode | Ephemeral session state, in-memory volatile buffer, exclusion from local DB and cloud sync, confirmation modal | None | Strict zero-retention on ephemeral session requests |
| F3 Preview and confirm | Preview screen, confirm and cancel gestures and buttons, the three modes, settings | Optional confidence score for each sentence | Return the confidence score with the response, if provided |
| F4 Calibration | Placement guide, channel check, per-user normalization parameters applied in `libs/preprocessing`, quality result | Choose the normalization method; compare accuracy before and after on new users | Optional: store per-user settings with consent; run fine-tuning later |
| F5 Speech-to-text | Microphone capture, large-text display, distinct haptic vibration, audio quality warnings, local recognizer fallback | Pick and test Egyptian Arabic speech-to-text models, online and offline | Speech-to-text endpoint on GPU worker (hosted model or proxy to a service) |
| F6 History, storage and chat | Local database (Room with SQLCipher AES-256), storage preference filters (text-only vs. full dialogue), immediate clean local hard delete, single/bulk TXT & PDF export engines, interactive chat UI for signed translation history, PIN/biometric lock | None | Optional encrypted cloud backup (E2EE AES-256-GCM) and account settings sync |
| F7 Emergency gesture | Gesture and button trigger, countdown, on-device speech (TTS), full-screen message, SMS, GPS | Choose a distinctive gesture; measure the false-trigger rate | None |
| F8 Offline mode | Detect connection and device capability, load small streaming model, manage latency timeout fallback (~3s), show mode banner | Deliver small streaming model and supported phrase list; set quality threshold for “device is capable” | Host versioned model files for download and update |
| F9 Dictionary and learning | Dictionary screens, video download and playback, practice mode | Score a learner’s sign for practice mode | Host videos; content API |
| F10 Text-to-sign reply (Post-MVP) | Play clips or avatar, speed and repeat controls | Convert Arabic text to sign order (rules or model) | Conversion service; host clips |

### Team summaries

**Mobile**

- Android app (Kotlin + Jetpack Compose), Bluetooth link to the armband, and all UI screens.
- Signal preprocessing via shared native Rust library (`libs/preprocessing`), on-device streaming model runtime, encrypted local database (Room with SQLCipher AES-256, storage mode filters, and immediate hard delete logic), in-context hardware warning banners, audio playback (server neural TTS stream online, on-device Android TTS offline), SMS and GPS.
- Ephemeral memory management for Temporary Gesture Mode (in-memory only, RAM zeroing).
- Formatted transcript export generators (plain text `.txt` and printable `.pdf`).
- Confirmation modal framework for destructive operations.
- Connectivity, latency timeout monitor (~3s), and device capability checks that choose the mode.

**AI**

- Data collection plan: many signers, several sessions, repeated armband placement.
- Streaming BiLSTM + CTC model architecture for the server and a compressed small model for the phone, evaluated separately.
- Preprocessing pipeline specification and golden test vectors (raw sensor input and expected preprocessed features) to validate `libs/preprocessing` in CI.
- Gesture control models for armband session inputs (start and stop gestures, continuous sign activity detection) and false-trigger suppression.
- Evaluation on new users, reported as word error rate and as the share of fully correct sentences.
- Speech-to-text model selection (F5) and text-to-sign conversion (F10, later).

**Backend**

- One unified GPU model worker hosting translation (BiLSTM + CTC streaming decoder), speech-to-text (F5), and server neural TTS for Egyptian Arabic.
- Spring Boot API gateway managing accounts, OAuth2 (JWT tokens), sync, right-to-erasure cascade deletion, and model manifest downloads.
- Model hosting and versioned downloads for the app via object storage.
- Production security: TLS 1.3 / HTTPS for in-transit encryption, AES-256 for at-rest storage, and OAuth2/JWT token authorization.
- Strict sensor privacy enforcement: zero persistent storage of raw high-frequency signals or camera feeds without explicit, segregated opt-in consent.
- Basic monitoring of delay and errors.

### Integration points between teams

- **Signal format:** sample rate, channel order, bit depth, and scaling are defined once and shared by Mobile, Hardware, and AI. EMG and IMU share one synchronized clock.
- **Preprocessing:** runs strictly on mobile via the shared native Rust library (`libs/preprocessing`) through JNI / UniFFI bindings. The server performs zero preprocessing on translation requests. Golden test vectors are maintained in contracts and verified in CI across Android and AI training repositories to guarantee mathematical parity.
- **API contract:** persistent streaming connection (WebSocket or gRPC over TLS 1.3) transporting preprocessed Protobuf signal chunks, user calibration normalization values, optional context hints (coarse place category, conversation topic, phrase keywords), and an end-of-sentence flag. Response streams Arabic text and optional confidence score, followed by server neural TTS audio stream. Precise coordinates are strictly excluded from the translation API.
- **Model files:** AI delivers a versioned on-device model; Backend hosts it in object storage with a signed manifest; Mobile downloads and checks the version and checksum before activating.
- **Sync & Backup API:** Mobile uploads opt-in end-to-end encrypted backup blobs (AES-256-GCM) and syncs user settings/favorites; Backend stores ciphertext blobs without access to user keys.

## 5. Non-functional requirements, roadmap, risks and open questions

The biggest unknowns are accuracy on new users and the delay of the online path, so both are measured early on a real phone.

### Non-functional requirements

- **Latency:** time from the end of a sentence to speech is measured on the real path (sensor, phone, network, server, speech). The target is set after the first measurement. A server timeout of ~3 seconds triggers fallback to the on-device model.
- **Privacy & Sensor Non-Retention:** Raw camera frames, video feeds, and high-frequency sEMG/IMU raw signals are strictly processed in transient memory and are never stored locally or transmitted for persistent storage. Location context is opt-in and coarse only. Ephemeral Temporary Gesture Mode guarantees zero persistence.
- **Security Architecture & Encryption:** All network communications with inference and synchronization backends enforce TLS 1.3 / HTTPS. Any persisted data (local Room database via SQLCipher or server backups) is encrypted at rest using AES-256 (AES-256-GCM for cloud backups). Cloud access is guarded by OAuth2 and JWT bearer tokens with scoped permissions.
- **Consent & AI Model Improvement:** Opting into AI model improvement data collection is an independent, granular, revocable consent setting completely decoupled from conversation history storage and user account identity.
- **Right to Erasure (Account & Data Deletion):** Users can permanently delete their account and purge all associated conversation data, preferences, and calibration profiles at any time, preceded by an export-first safeguard prompt.
- **Accessibility:** large text, high contrast, responsive chat styling, and vibration feedback in every screen.
- **Reliability:** Bluetooth auto-reconnect, clear messages when the connection drops, and a safe fallback to offline mode.
- **Battery:** translation runs only between a start and a stop.
- **Device support:** the minimum supported phone is decided from tests on a low-end device.
- **Accuracy reporting:** online and offline models are reported separately, on users not seen during training.

### Risks

| Risk | Effect | Mitigation |
| --- | --- | --- |
| Accuracy drops for new users or when the band shifts | Wrong translations | Calibration (F4); training data from many people and several wearing sessions; test on unseen users |
| Online delay is too long for conversation | Slow, awkward talk | Stream preprocessed signal chunks continuously during signing; compute encoder states incrementally; emit translation immediately upon sentence end |
| Small offline model is weak or slow on some phones | Offline mode unusable | Limited vocabulary; device capability test; fall back to model-free mode |
| Network or server fails during a demo | Failed presentation | Offline model as backup and a recorded demo |
| Signals and speech leave the phone | Privacy concern | Encrypted transport, consent, and an updated deck that says translation runs online |
| Little Egyptian Sign Language data and content exists | Weak models, no dictionary | Collect our own data; work with interpreters or a Deaf association |
| Emergency or confirm gestures trigger by accident | False alarms, lost trust | Distinct gestures, countdown with cancel, on-screen buttons |
| Location context is wrong or biases the model | Incorrect word choice | Show the active category; let the user correct, remove or disable it; never let context override strong sensor evidence |
