# Sout (صوت) – Requirements Document

Oct 17, 2026 · @omar

## 1. Overview

Sout is a phone app paired with a forearm EMG armband that turns Egyptian Sign Language into spoken Egyptian Arabic, so a Deaf or speech-impaired person can hold a conversation without a human interpreter. The user signs a full sentence, the system translates the whole sentence (not sign by sign, because Egyptian Sign Language has its own grammar and word order), and the phone speaks the result aloud.

This version of the product uses a phone and an armband only; there is no smartwatch. Translation runs on a server in the normal (online) mode, and a small on-device model gives a limited offline mode with lower accuracy. The two together are called the Hybrid approach in this document.

## 2. Hybrid architecture

Sout runs two translation models: a large one on the server for the best accuracy, and a small one on the phone that keeps working without a connection. The phone always handles the Bluetooth link, signal preprocessing, speech output and the screens.

&#91;embedded content: hybrid architecture · phone, server, two models\]

The armband streams to the phone. With a connection, the phone sends the processed signal to the server and receives Arabic text; without one, the small model on the phone produces the text. Both paths end in the same preview and speech step.

| Situation | What runs | Features available |
| --- | --- | --- |
| Online, any phone | Large model on the server for translation and speech-to-text; phone does Bluetooth, preprocessing, speech and screens | All features |
| Offline, capable phone | Small model on the phone for a limited set of phrases; the phone’s own speech recognizer if it is good enough | F1 (limited vocabulary), F3, F4, F5 (limited), F6, F7 |
| Offline, weak phone | No model; the phone runs only local features | F6, F7, the channel check in F4, and the downloaded dictionary from F9 |

On first launch the app tests the phone and decides whether it can run the small model. At run time it uses the server when a connection exists and the on-device model when it does not. The app shows the current mode and warns that offline vocabulary and accuracy are limited.

## 3. Core features (F1–F4)

These four features make up the minimum usable product: capture the signal, translate it, let the user check the result, and adapt to the person wearing the band.

### F1. Sign-to-speech translation

**What it is.** The user wears the armband, signs a full sentence, and the app speaks the translation in Egyptian Arabic. The signal is cleaned (EMG smoothing, alignment, splitting into sentences) and passed through a CNN with four branches that are fused into one feature vector. A BiLSTM encoder reads the sequence, CTC helps the model learn sign glosses, and an attention decoder writes the Arabic sentence. The text then goes to text-to-speech.

With the user’s permission, Sout also uses location as optional context for inference. The phone converts the current place into a coarse category such as pharmacy, clinic, bank, transport or other. The active category is always visible and can be corrected, removed or disabled before signing. The category may help the model choose between otherwise ambiguous meanings, but it must not override a translation strongly supported by the EMG signal. Precise coordinates stay on the phone and are not sent with translation requests; only the selected category and an optional user-created place label are used.

In the Hybrid approach there are two models behind this one feature. Online, a large model on the server translates with an open vocabulary and the highest accuracy. Offline, a small compressed model on the phone translates a limited set of phrases  with lower accuracy. Because the encoder reads the whole sentence, speech starts after the user finishes signing.

**Scenario.** Mona is Deaf and needs a headache medicine. At the pharmacy she signs her question. The phone sends the signal to the server, and a second later it says the sentence aloud in Egyptian Arabic. The pharmacist answers by voice and Mona reads the reply (see F5). Later, in a tunnel with no signal, the app switches to the offline model and still handles her most common phrases.

**Key requirements**

- Translate whole sentences and output Egyptian Arabic text and speech.
- Use the server model when a connection exists; fall back to the on-device model when it does not.
- Show which mode is active (online or offline) at all times.
- With permission, include a visible and editable coarse place category in online and offline inference; allow translation without location context.

### F2. Sensor connection, controls and wake gesture

**What it is.** The app pairs with the armband over Bluetooth, receives the EMG stream, and time-stamps it. A status panel shows connection, battery and electrode contact quality, and the app reconnects automatically if Bluetooth drops.

Translation operates via a synchronized dual-control scheme accessible through both on-screen buttons and armband muscle/motion gestures:
- **Stop:** Pauses or stops translation stream. Controlled by an on-screen "Stop" button or an armband **fist clench gesture**.
- **Resume:** Resumes translation after a pause. Controlled by an on-screen "Resume" button or an armband **double-tap gesture**.
- **Give Gesture (Sign):** Explicitly triggers a signing motion burst with real-time 8-channel EMG capture and gloss detection into Arabic text. Controlled by an on-screen "Give Gesture" button or a physical **sign motion gesture**.

Translation only runs between a start and a stop. This saves battery and data, and it prevents ordinary hand movement from being translated by accident. Both phone UI controls and armband gesture shortcuts are kept synchronized in real time.

**Scenario.** Karim is at a bank counter with his phone propped on the desk. He uses the double-tap resume gesture on his armband to activate translation, performs a sign motion, and clenches his fist to stop. The on-screen controls reflect the changes instantly, allowing him to interact without touching the phone.

**Key requirements**

- Pair, stream, and auto-reconnect over Bluetooth.
- Provide a 3-button control cluster (Stop, Resume, Give Gesture) on phone screens.
- Support dual-control via armband gestures (fist clench for stop, double-tap for resume, sign motion for give gesture) and on-screen buttons.
- Show battery level and per-channel contact quality in real time.

### F2b. Temporary Gesture Mode (Ephemeral Session)

**What it is.** A privacy-preserving ephemeral translation mode designed for private or sensitive conversations (e.g., medical diagnoses, legal consultations, financial transactions). When Temporary Gesture Mode is activated, the app enters an isolated session state marked by a distinct visual badge (`⚡ Temporary Gesture Mode`).

During temporary mode, all translated sentences, glosses, and partner replies are processed in volatile device memory only. No records are written to the local SQLite/IndexedDB conversation history, and no data is synced to cloud storage. Ending temporary mode prompts the user with a confirmation dialog before purging the temporary memory buffer.

**Scenario.** Mona visits a specialist for a confidential health concern. She activates Temporary Gesture Mode. A pulsing amber badge confirms the session is ephemeral. After completing her consultation, she taps "End Session," confirms the modal prompt, and the app safely discards all dialogue without leaving any traces in her conversation history.

**Key requirements**

- One-tap activation of Temporary Gesture Mode from gesture controls and chat screens.
- Clear visual status indication (`⚡ Temporary Gesture Mode` banner/badge) during active ephemeral sessions.
- Strict isolation: zero persistence to local storage and zero cloud synchronization during temporary mode.
- Safe termination: confirmation dialog before ending the session and purging memory buffers.


### F3. Preview and confirm

**What it is.** After the user finishes a sentence, the translation appears as large text before it is spoken. The user then confirms or cancels. Confirm can be a short gesture or a button; cancel lets them sign the sentence again. This protects the user from a wrong sentence being said out loud, since no translation model is perfect.

The user picks one of three modes in settings. Manual confirm never speaks without approval and suits sensitive situations such as a doctor or lawyer. Auto-speak after a delay speaks after 2–3 seconds unless the user cancels, which keeps ordinary conversation moving. Instant speak skips the preview and suits short, repeated phrases.

**Scenario.** Mona signs that her headache started two days ago. The preview reads “two months”. She cancels with a gesture, signs again, and the corrected sentence is spoken. Without the preview, the pharmacist would have heard wrong medical information.

**Key requirements**

- Display the translated sentence before speech in manual and auto modes.
- Confirm and cancel gestures must differ from normal signing, with on-screen buttons as backup.
- The user can change the mode at any time.

### F4. Sensor calibration

**What it is.** EMG signals differ between people (arm size, skin, muscle strength) and change if the armband sits a centimeter away from its last position. Calibration adapts the system to the current wearer. On first use, the app shows where and how to wear the band, checks that all 8 channels read a good signal, and asks for a few basic hand movements, each repeated two or three times. It then normalizes the signal to that user's range and reports a quality result (good, acceptable, redo).

Calibration takes two to five minutes and can be repeated whenever the user re-wears the band. Light calibration (channel check and normalization) is part of the first release. Fine-tuning the model on the user's own data is a possible later step and would run on the server.

**Scenario.** Omar borrows his friend's armband to try Sout. Before signing anything, the app walks him through placement and a short movement routine. Without it his first translations would be poor; after it, they are noticeably more accurate. We will measure that difference on new users before and after calibration.

**Key requirements**

- Guide placement, check channel quality, and normalize the signal per user.
- Show a clear quality result and allow re-calibration at any time.
- Work offline, since it runs on the phone.

## 4. Conversation and safety features (F5–F8)

These features turn the translator into a two-way conversation tool and make it safe and usable when the connection is poor.

### F5. Hearing person’s speech as text

**What it is.** Sout so far goes one way: the user signs and the phone speaks. F5 adds the reply. The phone’s microphone listens to the person in front of the user, converts their speech to Egyptian Arabic text, and shows it in large type. The phone vibrates when a reply arrives so the user does not have to keep watching the screen.

Online, a server speech-to-text model gives the best accuracy on Egyptian Arabic. Offline, the phone’s built-in speech recognizer is used where available; its accuracy on Egyptian dialect is likely lower and must be tested. If the sound is unclear, the app shows a notice so the user can ask the speaker to repeat.

**Scenario.** At the doctor’s office, Mona signs that she has a headache. The doctor answers, “Take it after meals, twice a day.” The sentence appears on the phone in large text and the phone vibrates. She reads it, signs her next question, and the conversation continues without a written note or an interpreter.

**Key requirements**

- Convert nearby speech to Egyptian Arabic text and display it in large type.
- Vibrate when a reply arrives.
- Warn when audio quality is too low to trust.

### F6. Conversation history, storage preferences, and chat interaction

**What it is.** Every translated sentence and hearing partner reply can be viewed in a responsive, chat-style conversation interface grouped by date and searchable. The user has granular control over what gets saved and how long it remains:
- **History recording toggle:** Allows users to pause or resume recording conversation history at any time.
- **Storage content preferences:** Users choose between storing "Translated text only" (recommended privacy default) or "Full conversation dialogue" (both Deaf user translations and hearing partner replies).
- **Strict sensor privacy guarantee:** Raw video, camera feeds, and raw high-frequency sEMG/IMU sensor signals are never stored locally or remotely; only decoded text and glosses are retained.
- **Deleted message tombstones:** In accordance with modern messaging privacy patterns, when a message or translation is deleted, it is replaced with a tombstone indicator (`⊘ This message was deleted / تم حذف هذه الرسالة`) with timestamp and dashed border styling, with an option for permanent purge.
- **Selective sharing & link revocation:** Users can generate read-only shareable links for individual conversation threads, view active links, and revoke all links instantly with one tap.
- **Transcript export:** Conversations can be exported individually or in bulk as formatted plain text (`.txt`) transcripts or styled print-ready documents (`.pdf`).
- **Interactive chat interface:** Provides an active messaging screen with a quick text input bar, preset quick chips, simulated speech partner responses in Egyptian Arabic, and an armband `⚡ Sign` quick shortcut.

Favorite phrases are sentences the user says often, such as “Please write it down, I am Deaf” or “I want to book an appointment”. They are grouped (doctor, bank, transport, work) and spoken with one tap, with no signing and no server. Favorites and history work fully offline, acting as a safety net when translation is unavailable.

**Scenario.** Karim opens his conversation history after a clinic visit. He reviews the advice given by his doctor, exports a `.pdf` transcript for his records, and deletes an accidental translation, which now displays as deleted. Later, he generates a secure share link for his family and revokes it once they have read the summary.

**Key requirements**

- Store history locally with real-time search, date grouping, and optional PIN/biometric lock.
- Toggle conversation history on/off, and select storage preferences (Translated text only vs. Full conversation).
- Absolute guarantee: never retain raw sensor or camera data.
- Display deleted messages as deleted with tombstone indicators and permanent purge option.
- Provide single and bulk transcript export to `.txt` and printable `.pdf`.
- Provide selective link sharing with one-tap revocation.
- Let users create, group, and speak favorite phrases with one tap.
- Allow an optional place label and coarse context category on each conversation.

### F7. Emergency gesture

**What it is.** In danger, a Deaf user cannot call out for help. F7 lets them trigger an emergency action with one distinctive gesture or an always-visible on-screen button. The phone speaks a preset message aloud (for example “I am Deaf and need help”), shows it full screen so bystanders can read it, and sends an SMS with the user’s location link to saved contacts. The user chooses the contacts and the message during setup.

False alarms are the main risk. The gesture must be unusual enough not to happen by accident, and a short countdown with a large cancel button appears before sending, with a “send now” option. F7 uses the phone’s own speech, SMS and GPS, so it works offline and does not depend on the server. We do not call emergency services automatically in the first release; this is a scope decision to confirm.

**Scenario.** Omar is in a street accident and cannot speak. He makes the emergency gesture. His phone announces that he is Deaf and needs help, shows the same words on screen for the people around him, and his family receives his location by SMS within seconds.

**Key requirements**

- Trigger by gesture and by on-screen button.
- Speak and display the message, and SMS the location to saved contacts.
- Work with no internet and include a cancel countdown to prevent false alarms.

### F8. Offline mode

**What it is.** Offline mode keeps the app useful with no connection. It has two levels. On a capable phone, the small on-device model translates a limited set of phrases, and a local speech recognizer handles replies where it is good enough. On a weak phone, or when the small model is too slow, the app runs with no model at all and offers F6 (history and favorites), F7 (emergency) and the downloaded dictionary.

On first launch the app tests the phone (memory, speed on a sample input) and chooses a level. The app always tells the user which mode it is in and that offline accuracy and vocabulary are limited. If the connection drops during a sentence, the app says so and switches modes instead of failing silently.

**Scenario.** Karim travels to a village with weak coverage. The app shows “Offline mode: limited vocabulary”. His common phrases still translate on the phone, and his favorites and emergency button work on any device. When coverage returns, the app switches back to the server model by itself.

**Key requirements**

- Detect connectivity and device capability and pick the right mode automatically.
- Show the current mode and its limits clearly.
- Keep F6, F7 and F4 (channel check) fully working with no connection.

### F8b. Notifications, reminders, and system alerts

**What it is.** A centralized notification and alert center (Screen 20) that delivers important reminders, warnings, and system notices without cluttering the main translation screens. Notifications are categorized into:
- **Reminders:** Prompts for critical actions, such as reviewing cloud synchronization settings, backing up favorite phrases, or re-verifying conversation history preferences.
- **Warnings:** Hardware and security alerts, including poor electrode contact quality, low armband battery, Bluetooth disconnection, and offline capability downgrades.
- **System Updates:** Notifications regarding downloaded offline vocabulary updates, model improvements, or app feature additions.

Users can filter notifications by category tab (`All`, `Reminders`, `Warnings`, `Updates`), view unread notification counters, dismiss individual notifications, or mark all as read. Each notification card includes direct action links (e.g., `Review Sync Settings →` linking to Privacy Settings, `Check Calibration →` linking to Calibration). Confirmation modals protect users before any destructive actions (clearing history, deleting accounts, revoking shares).

**Scenario.** Karim connects his armband and receives a warning notification indicating low battery (15%) and electrode channel 4 misalignment. Tapping the notification takes him directly to the Sensor Calibration screen. Later, after updating his privacy settings, a reminder alerts him to review his cloud sync preferences.

**Key requirements**

- Dedicated Notification Center screen with category filters and unread badges.
- Dismissal of individual notifications and "Mark all as read" capability.
- Deep-link action buttons within notification cards pointing to relevant settings or calibration tools.
- Non-intrusive global access points: header bell icon (`🔔`), bottom navigation bar (`🔔 Alerts`), and settings page shortcut.
- Reusable confirmation modal dialogs for all destructive user actions.

## 5. Future features (F9–F10)

These two features depend on sign content that does not exist yet (videos, a reliable dictionary). They are planned after the core product works.

### F9. Sign dictionary and learning mode

**What it is.** The dictionary is a searchable list of Egyptian Sign Language words and phrases, grouped by topic (doctor, bank, family, numbers). Each entry has a short video showing how to sign it. Videos can be downloaded in advance so the dictionary works offline.

Learning mode uses the armband. The app shows a sign, the learner copies it, and the model tells them whether it recognized the sign. A “show what was recognized” view displays the glosses behind a translated sentence so family and coworkers can see how it was built. The hard part is the content: videos must be recorded and checked with a certified interpreter or a Deaf association, because signs vary between regions.

**Scenario.** Karim’s younger brother wants to talk to him without the phone. He practices ten signs a day in the app, gets feedback from the armband, and after a month can sign simple sentences himself.

**Key requirements**

- Searchable dictionary with a video per entry, downloadable for offline use.
- Practice mode that checks a learner’s sign with the armband.
- Content reviewed by qualified Egyptian Sign Language experts.

### F10. Text-to-sign reply

**What it is.** Some Deaf users find written Arabic harder to read than signing, since sign language has its own grammar. F10 shows the hearing person’s reply as signs instead of text. The text from F5 is converted to sign order, then played either as a chain of short recorded videos or as a 3D avatar. The written sentence stays visible below for checking, and the user can slow down or repeat.

Video clips for a limited vocabulary are realistic for the first version. A full avatar is a much larger effort and is not planned for the first release. Both options need a second conversion step from Arabic text to sign order (rules or a model) and a recorded vocabulary.

**Scenario.** At a bank, the clerk says, “I need your ID card.” The phone, resting on the desk, plays short clips signing “ID card” and “need” for Karim, with the sentence written underneath.

**Key requirements**

- Convert reply text to a sequence of signs for a limited vocabulary.
- Play slowly or on repeat, with the written text always visible.
- Show a clear message when a word has no recorded sign.

## 6. Work division: Mobile, AI, Backend

Mobile owns everything the user touches and everything that must work offline. AI owns the translation and control models and their accuracy. Backend runs the large models, authentication, and secure sharing endpoints.

| Feature | Mobile | AI | Backend |
| --- | --- | --- | --- |
| F1 Sign-to-speech translation | Receive the sensor stream, preprocess the signal, send it to the API, run the small on-device model, text-to-speech, online/offline switching | Train the large model; compress it into the small model; export both; measure accuracy for each | Inference API: receive signal, run the large model, return Arabic text; HTTPS; latency logging |
| F2 Sensor connection and gesture controls | Bluetooth pairing, streaming, time-stamps, auto-reconnect, status screen, 3-button control cluster (Stop, Resume, Give Gesture), synchronized armband gesture state | Control gesture detection (fist clench for stop, double-tap for resume, sign motion); test for false triggers | None |
| F2b Temporary Gesture Mode | Ephemeral session state, in-memory volatile buffer, exclusion from local DB and cloud sync, confirmation modal | None | Strict zero-retention on ephemeral session requests |
| F3 Preview and confirm | Preview screen, confirm and cancel gestures and buttons, the three modes, settings | Optional confidence score for each sentence | Return the confidence score with the response, if provided |
| F4 Calibration | Placement guide, channel check, per-user normalization, quality result | Choose the normalization method; compare accuracy before and after on new users | Optional: store per-user settings with consent; run fine-tuning later |
| F5 Speech-to-text | Microphone capture, large-text display, vibration, local recognizer fallback | Pick and test Egyptian Arabic speech-to-text models, online and offline | Speech-to-text endpoint (hosted model or proxy to a service) |
| F6 History, storage and chat | Local database (SQLite/IndexedDB), storage preference filters (text-only vs. full dialogue), tombstone rendering (`⊘ This message was deleted`), single/bulk TXT & PDF export engines, interactive chat UI with partner simulation, PIN/biometric lock | None | Selective sharing API (tokenized read-only links, instant revocation endpoint), optional encrypted backup |
| F7 Emergency gesture | Gesture and button trigger, countdown, speech, full-screen message, SMS, GPS | Choose a distinctive gesture; measure the false-trigger rate | None |
| F8 Offline mode | Detect connection and device capability, load the small model, switch modes, show the mode banner | Deliver the small model and its supported phrase list; set the quality threshold for “device is capable” | Host versioned model files for download and update |
| F8b Notifications and alerts | Dedicated Screen 20, unread badge counters, category filtering, deep-link navigation triggers, persistent alert manager, modal dialog system | None | Push notification service for server alerts and model updates (optional) |
| F9 Dictionary and learning | Dictionary screens, video download and playback, practice mode | Score a learner’s sign for practice mode | Host videos; content API |
| F10 Text-to-sign reply | Play clips or avatar, speed and repeat controls | Convert Arabic text to sign order (rules or model) | Conversion service; host clips |

### Team summaries

**Mobile**

- Android app, Bluetooth link to the armband, and all UI screens.
- Signal preprocessing, on-device model runtime, local database (with storage mode filters and tombstone deletion logic), text-to-speech, SMS and GPS.
- Ephemeral memory management for Temporary Gesture Mode.
- Formatted transcript export generators (plain text `.txt` and printable `.pdf`).
- Notification center, category filtering, deep-link shortcuts, and confirmation modal framework.
- Connectivity and device checks that choose the mode.

**AI**

- Data collection plan: many signers, several sessions, repeated armband placement.
- Large model for the server and a compressed small model for the phone, evaluated separately.
- Gesture control models for armband inputs (fist clench, double tap, sign motion) and false-trigger suppression.
- Evaluation on new users, reported as word error rate and as the share of fully correct sentences.
- Speech-to-text model selection (F5) and text-to-sign conversion (F10, later).

**Backend**

- One inference service for translation, with speech-to-text added when F5 is built.
- Model hosting and versioned downloads for the app.
- Production security: TLS 1.3 / HTTPS for in-transit encryption, AES-256 for at-rest storage, and OAuth2/JWT token authorization.
- Tokenized selective conversation sharing service with instant revocation capabilities.
- Strict sensor privacy enforcement: zero persistent storage of raw high-frequency signals or camera feeds without explicit, segregated opt-in consent.
- Basic monitoring of delay and errors.

### Integration points between teams

- **Signal format:** sample rate, channel order and scaling are defined once and shared by Mobile and AI.
- **Preprocessing:** the same logic must run on the phone and on the server, or the two models will see different inputs.
- **API contract:** request carries the signal, the user’s calibration values and an optional coarse place category; response carries the Arabic text and, if available, a confidence score. Precise coordinates are not part of the translation API.
- **Model files:** AI delivers a versioned on-device model; Backend hosts it; Mobile downloads and checks the version.
- **Sharing & Sync API:** Mobile requests temporary, cryptographically signed read-only links for specific conversation threads; Backend routes public view requests and handles revocation requests.

## 7. Non-functional requirements, roadmap, risks and open questions

The biggest unknowns are accuracy on new users and the delay of the online path, so both are measured early on a real phone.

### Non-functional requirements

- **Latency:** time from the end of a sentence to speech is measured on the real path (sensor, phone, network, server, speech). The target is set after the first measurement.
- **Privacy & Sensor Non-Retention:** Raw camera frames, video feeds, and high-frequency sEMG/IMU raw signals are strictly processed in transient memory and are never stored locally or transmitted for persistent storage. Location context is opt-in and coarse only. Ephemeral Temporary Gesture Mode guarantees zero persistence.
- **Security Architecture & Encryption:** All network communications with inference and synchronization backends enforce TLS 1.3 / HTTPS. Any persisted data (local SQLite/IndexedDB or server backups) is encrypted at rest using AES-256. Cloud access is guarded by OAuth2 and JWT bearer tokens with scoped permissions.
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
| Online delay is too long for conversation | Slow, awkward talk | Send the whole sentence once; use a nearby server; measure early |
| Small offline model is weak or slow on some phones | Offline mode unusable | Limited vocabulary; device capability test; fall back to model-free mode |
| Network or server fails during a demo | Failed presentation | Offline model as backup and a recorded demo |
| Signals and speech leave the phone | Privacy concern | Encrypted transport, consent, and an updated deck that says translation runs online |
| Little Egyptian Sign Language data and content exists | Weak models, no dictionary | Collect our own data; work with interpreters or a Deaf association |
| Emergency or confirm gestures trigger by accident | False alarms, lost trust | Distinct gestures, countdown with cancel, on-screen buttons |
| Location context is wrong or biases the model | Incorrect word choice | Show the active category; let the user correct, remove or disable it; never let context override strong sensor evidence |

### Open questions

- Does the armband include a motion sensor (IMU)? This decides how the wake gesture is detected.
- Speech-to-text (F5): hosted model, a cloud service, or the phone’s own recognizer for offline?
- Will F5 and F10 be built in this project or presented as future work?
- Where is the server hosted (university, cloud credits, or a simple platform)?
- Should the emergency feature ever call emergency services, or only message saved contacts?
- Is the signal sent as one whole sentence (simpler) or streamed (faster but heavier)?
- What is the lowest phone we support?
