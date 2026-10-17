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

### F2. Sensor connection and wake gesture

**What it is.** The app pairs with the armband over Bluetooth, receives the EMG stream, and time-stamps it. A wake gesture starts or stops translation without touching the phone. The deck's example is raising the wrist and moving the hand up and down three times. A status panel shows connection, battery and electrode contact quality, and the app reconnects automatically if Bluetooth drops.

Translation only runs between a start and a stop. This saves battery and data, and it prevents ordinary hand movement from being translated by accident. An on-screen start/stop button is always available as a fallback, because a wake gesture that depends on a motion sensor may not be possible if the armband has none.

**Scenario.** Karim is at a bank counter with his phone propped on the desk. He makes the wake gesture, signs his request, and makes the gesture again to stop. When his band loses connection for a moment, the app shows a short “reconnecting” message and resumes without him opening any settings.

**Key requirements**

- Pair, stream, and auto-reconnect over Bluetooth.
- Start and stop translation by gesture and by on-screen button.
- Show battery level and per-channel contact quality.

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

### F6. Conversation history and favorite phrases

**What it is.** Every translated sentence and every reply is saved on the phone in a chat-style history, grouped by date and searchable. The user can delete a conversation or everything. The history can be locked with a PIN or fingerprint, or turned off, because it can hold health and money details.

Favorite phrases are sentences the user says often, such as “Please write it down, I am Deaf” or “I want to book an appointment”. They are grouped (doctor, bank, transport, work) and spoken with one tap, with no signing and no server. A favorite can also be linked to a gesture later. Favorites and history work fully offline, so they act as a safety net when translation is unavailable.

**Scenario.** Karim rides a minibus and needs to ask the driver to stop. He taps “Please stop here” in his Transport favorites and the phone says it at once. That evening he opens the history from his doctor visit to check the medicine name and dose the doctor mentioned.

**Key requirements**

- Store history locally, with search, delete and optional lock.
- Let users create, group and speak favorite phrases with one tap.
- Allow editing or deleting a wrong translation in the history.
- Allow an optional place label and coarse context category on each conversation so history can be searched by place without uploading precise location.

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

Mobile owns everything the user touches and everything that must work offline. AI owns the two translation models and their accuracy. Backend runs the large models and serves files to the app. F6 and F7 need no AI or Backend work.

| Feature | Mobile | AI | Backend |
| --- | --- | --- | --- |
| F1 Sign-to-speech translation | Receive the sensor stream, preprocess the signal, send it to the API, run the small on-device model, text-to-speech, online/offline switching | Train the large model; compress it into the small model; export both; measure accuracy for each | Inference API: receive signal, run the large model, return Arabic text; HTTPS; latency logging |
| F2 Sensor connection and wake gesture | Bluetooth pairing, streaming, time-stamps, auto-reconnect, status screen, start/stop gesture and button | Wake-gesture detection if it uses EMG; test for false triggers | None |
| F3 Preview and confirm | Preview screen, confirm and cancel gestures and buttons, the three modes, settings | Optional confidence score for each sentence | Return the confidence score with the response, if provided |
| F4 Calibration | Placement guide, channel check, per-user normalization, quality result | Choose the normalization method; compare accuracy before and after on new users | Optional: store per-user settings with consent; run fine-tuning later |
| F5 Speech-to-text | Microphone capture, large-text display, vibration, local recognizer fallback | Pick and test Egyptian Arabic speech-to-text models, online and offline | Speech-to-text endpoint (hosted model or proxy to a service) |
| F6 History and favorites | Local database, history and favorites screens, lock, one-tap speech | None | None |
| F7 Emergency gesture | Gesture and button trigger, countdown, speech, full-screen message, SMS, GPS | Choose a distinctive gesture; measure the false-trigger rate | None |
| F8 Offline mode | Detect connection and device capability, load the small model, switch modes, show the mode banner | Deliver the small model and its supported phrase list; set the quality threshold for “device is capable” | Host versioned model files for download and update |
| F9 Dictionary and learning | Dictionary screens, video download and playback, practice mode | Score a learner’s sign for practice mode | Host videos; content API |
| F10 Text-to-sign reply | Play clips or avatar, speed and repeat controls | Convert Arabic text to sign order (rules or model) | Conversion service; host clips |

### Team summaries

**Mobile**

- Android app, Bluetooth link to the armband, and all screens.
- Signal preprocessing, the on-device model runtime, local database, text-to-speech, SMS and GPS.
- Connectivity and device checks that choose the mode.

**AI**

- Data collection plan: many signers, several sessions, repeated armband placement.
- Large model for the server and a compressed small model for the phone, evaluated separately.
- Evaluation on new users, reported as word error rate and as the share of fully correct sentences.
- Speech-to-text model selection (F5) and text-to-sign conversion (F10, later).

**Backend**

- One inference service for translation, with speech-to-text added when F5 is built.
- Model hosting and versioned downloads for the app.
- Encrypted transport, no storage of raw signals without the user’s consent, and basic monitoring of delay and errors.

### Integration points between teams

- **Signal format:** sample rate, channel order and scaling are defined once and shared by Mobile and AI.
- **Preprocessing:** the same logic must run on the phone and on the server, or the two models will see different inputs.
- **API contract:** request carries the signal, the user’s calibration values and an optional coarse place category; response carries the Arabic text and, if available, a confidence score. Precise coordinates are not part of the translation API.
- **Model files:** AI delivers a versioned on-device model; Backend hosts it; Mobile downloads and checks the version.

## 7. Non-functional requirements, roadmap, risks and open questions

The biggest unknowns are accuracy on new users and the delay of the online path, so both are measured early on a real phone.

### Non-functional requirements

- **Latency:** time from the end of a sentence to speech is measured on the real path (sensor, phone, network, server, speech). The target is set after the first measurement.
- **Privacy:** signals and speech sent to the server are encrypted in transit and not stored without the user’s consent. History and precise location stay on the phone and can be locked or turned off. Location context is opt-in; only the visible coarse category is sent for inference.
- **Accessibility:** large text, high contrast and vibration feedback in every screen.
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
