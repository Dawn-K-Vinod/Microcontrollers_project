# Voice-Controlled Music Player with Two-Stage Wake Detection (STM32-Based)
## Complete Project Reference Document

---

## 1. Project Summary

A fully offline, STM32-based voice-controlled music player. The system is woken and put to sleep using a dedicated wake/sleep keyword, and once awake, responds to four fixed voice commands: **play, pause, next, previous**. Unlike naive always-on voice classifiers, this system uses a **two-stage detection pipeline** — a lightweight Voice Activity Detection (VAD) gate runs continuously, and only wakes the heavier MFCC + neural network classifier when actual speech-like sound is detected. This reduces power consumption and false triggers compared to running full classification continuously.

---

## 2. System Architecture

```
                        +----------------------+
                        |   INMP441 I2S Mic     |
                        |   (Continuous Audio)  |
                        +-----------+-----------+
                                    |
                                    v
                        +----------------------+
                        |   STAGE 1: VAD        |
                        |  (always-on, cheap    |
                        |   energy/spectral      |
                        |   check)               |
                        +-----------+-----------+
                                    |
                        Speech-like activity detected?
                                    |
                     -------------------------------
                     |                              |
                    NO                             YES
                     |                              |
                     v                              v
              (loop back, classifier         +----------------------+
               stays dormant)                |   STAGE 2: MFCC +     |
                                              |   Neural Network       |
                                              |   Classifier            |
                                              |   (Edge Impulse model) |
                                              +-----------+-----------+
                                                          |
                                          Compare against trained commands:
                                          wake / sleep / play / pause /
                                          next / previous
                                                          |
                                    -------------------------------------
                                    |                                    |
                              No confident match                  Match found
                                    |                                    |
                                    v                                    v
                             (ignore, return           +----------------------+
                              to Stage 1)               | Check system state:   |
                                                         | SLEEPING or AWAKE      |
                                                         +-----------+-----------+
                                                                     |
                                                    -----------------------------------
                                                    |                                  |
                                             SLEEPING:                          AWAKE:
                                        only "wake" accepted            all commands accepted
                                                    |                                  |
                                                    v                                  v
                                          Switch to AWAKE state          Execute command:
                                                                          - play / pause / next / previous
                                                                            -> sent to music playback module
                                                                          - "sleep" -> switch to SLEEPING state
                                                                          - LED/display confirms action
```

---

## 3. Exact Hardware Components

| Component | Exact Recommendation | Why This One | Approx. Cost (INR) |
|---|---|---|---|
| **Microcontroller** | **STM32 Nucleo-F446RE** (STM32F446RET6, Cortex-M4F, 180 MHz, 128 KB SRAM, 512 KB Flash) | Needs Cortex-M4F with hardware DSP extensions and sufficient RAM/flash to run an Edge Impulse–exported classifier. Basic Blue Pill (F103) does not have enough RAM. | ₹750–1,000 |
| **Microphone** | **INMP441** (I2S digital MEMS microphone) | Digital I2S output integrates cleanly with STM32 I2S/SAI peripheral and is well-documented for Edge Impulse audio projects. | ₹150–250 |
| **Audio output** | Small speaker (8 ohm, 0.5-1W) + PAM8403 mini amplifier module, or a DFPlayer Mini MP3 module for simplified playback | Handles actual music playback triggered by voice commands; DFPlayer Mini is simplest since it handles MP3 decoding from an SD card independently, controlled via simple UART commands from STM32 | ₹100–250 (DFPlayer Mini + SD card) |
| **Status LED** | Standard 5mm LED (or 2-3 for state indication: sleeping/awake/executing) | Visual confirmation of system state and command execution | ₹5–15 |
| **Small display (optional)** | 16x2 I2C LCD | Shows current state (Sleeping/Awake) and last recognized command — useful for demo clarity | ₹150–200 |
| **Breadboard + jumper wires** | — | Prototyping | ₹200–300 |
| **Power** | USB power via Nucleo's onboard ST-LINK USB connector | Sufficient for development and demo; no separate power supply needed | ₹0 |

### Total Estimated Cost
**₹1,350 – ₹2,000** (with optional LCD; ₹1,200–₹1,800 without)

---

## 4. Software Tools

- **Edge Impulse Studio** (studio.edgeimpulse.com, free tier) — audio data collection, MFCC processing block, neural network training, C++ library export
- **STM32CubeMX** — peripheral configuration (I2S/SAI for mic, UART for DFPlayer/LCD, GPIO for LEDs)
- **STM32CubeIDE** — firmware development, debugging, flashing
- **STM32 HAL Libraries** — peripheral driver abstraction
- **Embedded C / C++** — firmware (Edge Impulse export requires C++ for some components)

---

## 5. Data Collection Plan (For Training)

You need labeled audio samples for **6 classes**: wake, sleep, play, pause, next, previous — plus a "noise/background" class for the VAD and classifier to learn what to ignore.

| Class | Suggested Sample Count | Notes |
|---|---|---|
| Wake word | 100–150 | Multiple team members, varied distance/tone |
| Sleep word | 100–150 | Same |
| Play | 80–120 | |
| Pause | 80–120 | |
| Next | 80–120 | |
| Previous | 80–120 | |
| Background/noise | 150–200 | Room noise, silence, unrelated speech/chatter — critical for reducing false positives |

Recording can be done with phone voice recorders or a laptop mic, then uploaded to Edge Impulse — no special hardware needed for data collection itself.

---

## 6. Firmware Logic (Pseudocode)

```
GLOBAL:
  system_state = SLEEPING

INITIALIZE:
  configure I2S peripheral for INMP441 mic input
  configure UART for DFPlayer Mini / LCD
  initialize Edge Impulse classifier library
  initialize VAD parameters (energy threshold)

MAIN LOOP:
  audio_frame = read_from_mic_I2S()

  IF VAD_check(audio_frame) == SPEECH_DETECTED:
      result = run_classifier(audio_frame)   // Stage 2: MFCC + NN

      IF result.confidence > THRESHOLD:
          command = result.top_label

          IF system_state == SLEEPING:
              IF command == "wake":
                  system_state = AWAKE
                  update_LED(AWAKE)
                  update_LCD("Awake - Listening")
              // all other commands ignored while sleeping

          ELSE IF system_state == AWAKE:
              SWITCH command:
                  CASE "play":     send_UART_command(DFPlayer, PLAY)
                  CASE "pause":    send_UART_command(DFPlayer, PAUSE)
                  CASE "next":     send_UART_command(DFPlayer, NEXT)
                  CASE "previous": send_UART_command(DFPlayer, PREV)
                  CASE "sleep":
                      system_state = SLEEPING
                      update_LED(SLEEPING)
                      update_LCD("Sleeping")
              update_LCD("Last command: " + command)

  ELSE:
      // No speech detected - Stage 2 classifier not invoked, saving power
      continue loop
```

---

## 7. Team Task Allocation (5-6 People)

| Role | Responsibilities | People |
|---|---|---|
| **Data Collection & Model Training** | Record all 6+ command classes and background noise, run Edge Impulse pipeline (MFCC block + classifier), tune accuracy, export C++ library | 1-2 |
| **Firmware — Audio Pipeline & Classifier Integration** | I2S mic setup, VAD Stage 1 implementation, integrate exported classifier for Stage 2, main detection loop | 1-2 |
| **Firmware — Playback & State Control** | DFPlayer Mini UART control, wake/sleep state machine, command execution logic | 1 |
| **Display & Status Feedback** | LCD/LED integration for state and command confirmation | 1 |
| **Testing, Simulation & Documentation** | Proteus simulation of non-audio logic (state machine, UART, LED/LCD), real-hardware testing of audio pipeline, false-positive/negative rate testing, report writing | 1 |

---

## 8. Suggested Timeline (6-8 Weeks)

| Week | Milestone |
|---|---|
| 1 | Order components, set up STM32CubeIDE for all members, begin recording training audio samples |
| 2 | Complete data collection (all 6 command classes + background noise), begin Edge Impulse pipeline setup |
| 3 | Train initial model, evaluate accuracy, identify weak classes and record additional samples if needed |
| 4 | Export classifier, begin firmware integration: I2S mic reading + Stage 1 VAD implementation |
| 5 | Integrate Stage 2 classifier, implement wake/sleep state machine and command routing |
| 6 | Integrate DFPlayer Mini playback control and LCD/LED feedback, full end-to-end testing |
| 7 | Real-world testing (varied distances, noise conditions), tune thresholds, fix false triggers |
| 8 | Documentation, Proteus simulation finalization, report writing, demo rehearsal |

---

## 9. Simulation Plan (Proteus)

Since Proteus cannot realistically simulate live microphone/audio input, structure your simulation submission as follows:
- **Simulate the non-audio logic fully in Proteus**: the state machine (Sleeping/Awake), UART commands to a simulated DFPlayer/serial device, LED state indicators, and LCD status display — all standard Proteus library components.
- **Represent the audio classification stage as a simulated trigger**: e.g., a button press standing in for "classifier detected command X," feeding into the same state machine and output logic used in the real system.
- **Clearly document in your report** that audio input and classification are validated separately on real hardware (with recorded/live audio and Edge Impulse's own accuracy reporting), while the control logic is what's demonstrated in Proteus. This is an honest, defensible split — not a workaround, but the correct way to simulate a system with a real-world sensing component.

---

## 10. Anticipated Questions & How to Answer Them

| Likely Question | Your Answer |
|---|---|
| "Doesn't this already exist (like Alexa)?" | "Yes, commercial voice assistants exist, but they depend on cloud connectivity. Our system runs entirely offline on the STM32 itself — no internet, no data leaving the device. Additionally, we use a two-stage detection pipeline to reduce power consumption and false triggers compared to a naive always-on classifier, which is a real design choice, not present in most beginner-level voice recognition projects." |
| "Why can't you easily add a new keyword?" | "Our command set (wake, sleep, play, pause, next, previous) is intentionally fixed, like the buttons on a physical remote control — this is standard for embedded control interfaces and isn't meant to be end-user configurable. This differs from a general-purpose assistant, where flexible vocabulary would be expected." |
| "What if the classifier misfires?" | "The two-stage VAD gate reduces false triggers by filtering out non-speech noise before it ever reaches the classifier. Additionally, our wake/sleep state machine means commands are only accepted while the system is explicitly Awake, providing a second layer of protection against accidental activation." |
| "Why not just use a phone app for music control?" | "This is a demonstration of an embedded, standalone, hands-free control interface — useful in contexts where phone interaction isn't convenient (e.g., hands occupied, phone not nearby), and shows practical application of on-device DSP and embedded ML rather than relying on external app processing." |

---

## 11. Honest Notes on Difficulty & Scope

- The heavy DSP/ML math (MFCC extraction, neural network training) is handled by Edge Impulse — your team's real work is data collection, firmware integration, state machine logic, and testing. This is realistic for a team starting from basic C.
- Board choice is non-negotiable: must be STM32F4-class or higher for adequate RAM/flash to run the exported model.
- Expect to iterate on training data 2-3 times to reduce false positives/negatives — budget real time for this, not just the initial recording session.
- The two-stage VAD architecture is the project's genuine, defensible point of originality — keep it central to your pitch, not a footnote.
