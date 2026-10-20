
The **Preprocessing Layer** in **Sout (صوت)** sits between the raw Bluetooth sensor stream and the translation models. Its primary mission is to transform noisy, person-dependent muscle signals into a clean, standardized mathematical tensor that both the **Large Server Model** and the **Small On-Device Model** can understand identically.

---

## 1. What is Needed from the Preprocessing Layer

Every requirement in this layer directly maps to a functional or non-functional requirement in the Sout specification:

* **Shared Signal Format & Deterministic Math:** Sample rate, 8-channel ordering, and scaling must be defined once and run identically for both the phone and server models so neither model experiences input distribution skew.


* **Per-Channel Electrode Contact Check (F2, F4, F8b):** Continuously evaluates the signal quality of all 8 armband channels. It reports a clear status (`Good`, `Acceptable`, `Redo`) for **F4 Calibration** and identifies specific misaligned channels to trigger **F8b Warning Notifications**.


* **Signal Cleaning & Alignment (F1, F2):** Performs EMG smoothing, time-stamp alignment (fixing Bluetooth packet jitter), and sentence boundary splitting between the user's `Resume/Give Gesture` start and `Stop` triggers.


* **Per-User Calibration Normalization (F4):** EMG signals vary significantly by arm size, skin impedance, and muscle strength, and shift if the band moves by a centimeter. The layer must normalize the incoming signal against the current wearer's calibrated resting baseline and maximum range.



---

## 2. Available Implementation Choices & Tradeoffs

When deciding **how to build** this preprocessing layer across the Mobile (Kotlin), AI (Python), and Backend teams, there are four architectural choices:

| Implementation Choice | How It Works | Pros | Cons / Tradeoffs |
| --- | --- | --- | --- |
| **1. Shared Rust Crate** *(PyO3 + UniFFI)* **(Selected)** | Write DSP math once in Rust. Compile to a `.whl` package for Python (AI Training/Server) and `.so` + auto-generated Kotlin bindings for Android. | • **100% identical math** in Python & Kotlin.<br>

<br>• **Zero manual JNI/C++** boilerplate.<br>

<br>• Memory-safe with automatic RAM zeroing (`zeroize`) for **F2b/F6 privacy**.<br>

<br>• Native C-level speed. | • Requires Rust toolchain (`cargo-ndk`, `maturin`) in the build pipeline.<br>

<br>• Slight learning curve if team is new to Rust. |
| **2. Embedded Inside AI Model** *(TFLite / ONNX Graph)* | AI team writes preprocessing as TensorFlow/PyTorch tensor operations and exports it inside the `.tflite` / `.onnx` model file. | • Zero DSP code for the Mobile team.<br>

<br>• Guaranteed alignment with the AI model weights.<br>

<br>• Updates automatically when a new model is downloaded (**F8**). | • Harder to run standalone **F4 Channel Quality Checks** when the model isn't running (e.g., on a "Weak Phone" in **F8**).

<br>

<br>• TFLite ops for irregular Bluetooth timestamp alignment can be rigid. |
| **3. Shared C++ Library** *(Android NDK + JNI)* | Write math in C++. Bind to Android via manual `JNIEXPORT` and to Python via `pybind11`. | • Maximum low-level hardware control.<br>

<br>• Zero-copy `DirectByteBuffer` support.<br>

<br>• Shared codebase between AI and Mobile. | • High maintenance overhead: manual JNI pointers, memory leaks, segfault risks, and complex `CMakeLists.txt` debugging. |
| **4. Dual Native Code** *(Pure Kotlin + Pure Python)* | Mobile writes preprocessing in Kotlin; AI writes the "same" math in Python/NumPy. | • Fastest initial setup.<br>

<br>• No cross-compilation or native binaries needed on Android. | • **High risk of "Training-Serving Skew":** subtle differences in floating-point rounding or filter implementation between Kotlin and NumPy will degrade translation accuracy.

 |

---

## 3. Why Each Design Choice Was Taken for Sout

### Choice 1: Why Preprocess on the Phone (Even in Online Mode)?

* **The Requirement:** Section 2 states: *"The phone always handles the Bluetooth link, signal preprocessing, speech output and the screens. With a connection, the phone sends the processed signal to the server"*.


* **Why Taken:**
1. **Mid-Sentence Network Drop Recovery (F8):** If the connection drops while a sentence request is in flight, the phone already holds the preprocessed tensor in RAM and can immediately pass it to the local on-device model without failing silently or forcing the user to sign again.


2. **Privacy Compliance (F6 & NFRs):** Raw high-frequency sEMG/IMU signals never leave the phone; only cleaned, normalized feature vectors are transmitted over TLS 1.3.


3. **Weak Phone Offline Support (F8):** Even on a weak phone that cannot run the neural network offline, the phone still needs local preprocessing to run the **F4 Channel Check**.





### Choice 2: Why Choose Rust (UniFFI + PyO3) Over C++ or Dual Kotlin/Python?

* **Eliminating Training-Serving Skew (Section 6):** The AI team collects data from many signers across multiple wearing sessions in Python, while the Mobile team runs inference in Kotlin. Sharing a single Rust crate guarantees bit-for-bit identical normalization and smoothing across PyTorch training and Android inference.


* **No JNI Headaches for Mobile:** Unlike C++, Mozilla's `UniFFI` automatically generates clean Kotlin data classes (`CalibrationProfile`, `PreprocessedOutput`) and functions, saving the Android team from debugging JNI memory crashes.
* **Guaranteed Ephemeral Memory Wiping (F2b):** Rust's `#[zeroize(drop)]` trait guarantees at the compiler level that raw sensor arrays and temporary tensors are overwritten with zeros in RAM the instant they go out of scope, fulfilling the strict isolation requirement of **Temporary Gesture Mode (F2b)**.



### Choice 3: Why Separate Calibration Normalization (`F4`) from Model Weights (`F1`)?

* **The Requirement:** Light calibration (channel check and per-user normalization) must work **offline on the phone** in the first release and takes 2–5 minutes whenever the user re-wears the band.


* **Why Taken:** Neural network weights (`TFLite` on-device or `PyTorch` on-server) are static, whereas a user's skin contact and armband position shift every time they put the band on. By injecting `CalibrationProfile` (`baseline_mean` and `mvc_scale`) into the Rust preprocessing layer dynamically at runtime, the system adapts to a new wearer (like Omar borrowing his friend's armband in **F4**) immediately without retraining the AI model.

-------
