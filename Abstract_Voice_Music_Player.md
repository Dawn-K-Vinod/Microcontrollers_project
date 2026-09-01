# Abstract: Voice-Controlled Music Player with Two-Stage Wake Detection (STM32-Based)

## Project Title
**Low-Power Voice-Assisted Music Player Using Two-Stage Keyword Spotting on STM32**

---

## Abstract

Voice-controlled assistants have become a common interface for hands-free device control, but most commercial implementations (Alexa, Google Assistant, Siri) depend on continuous cloud connectivity, raising concerns around privacy, latency, and reliability in offline environments. Additionally, always-on voice classifiers running continuously on embedded hardware consume unnecessary power and are prone to false activations from ambient noise.

This project proposes a **fully offline, on-device voice-controlled music player** built on an STM32 microcontroller, capable of being woken and put to sleep using a dedicated wake/sleep keyword, and controlled via a fixed set of voice commands — **play, pause, next, and previous** — without requiring an internet connection or cloud-based processing.

To address the inefficiency of continuously running a full keyword classifier, the system employs a **two-stage detection architecture**: a lightweight, low-power **Voice Activity Detection (VAD)** stage continuously monitors the microphone input for the mere presence of sound, while the computationally heavier **MFCC feature extraction and neural network classification** stage remains dormant and only activates when the VAD stage detects speech-like activity. This gatekeeping approach meaningfully reduces both power consumption and false command triggers compared to a naive always-on classification design, while preserving full responsiveness once genuine speech is detected.

The system is trained using Edge Impulse, generating a compact, quantized model deployable on an STM32F4-series microcontroller with onboard DSP acceleration. The result is a low-cost, private, offline, and power-conscious voice interface, demonstrating a genuine, small-scale engineering optimization over naive always-on embedded voice recognition — appropriate for resource-constrained, real-world embedded applications.

---

## Problem Statement

- Commercial voice assistants require internet connectivity for command processing, introducing latency, privacy concerns, and dependency on external infrastructure.
- Naive embedded keyword-spotting implementations run the full detection pipeline (feature extraction + neural network inference) continuously, which is computationally and energy expensive for a battery-powered or always-listening device.
- There is a genuine need for a **fully local, low-power, offline voice control interface** for simple, fixed-command applications (such as media playback control), where cloud dependency and continuous full-pipeline inference are both unnecessary overhead.

## Objectives

1. Implement offline voice command recognition on STM32 with **no cloud dependency**.
2. Support a **wake/sleep keyword** to explicitly control when the system is listening for commands.
3. Support four playback commands: **play, pause, next, previous**.
4. Implement a **two-stage detection pipeline** (VAD gate → MFCC/classifier) to reduce power consumption and false triggers compared to continuous full-pipeline inference.
5. Demonstrate real-time, low-latency response suitable for practical media control use.

---

## System Architecture

### High-Level Block Diagram

```
                        +----------------------+
                        |   INMP441 I2S Mic     |
                        |   (Continuous Audio)  |
                        +-----------+-----------+
                                    |
                                    v
                        +----------------------+
                        |   STAGE 1: VAD        |
                        |  (Voice Activity       |
                        |   Detection - always   |
                        |   running, lightweight)|
                        +-----------+-----------+
                                    |
                        Speech-like activity?
                                    |
                     -------------------------------
                     |                              |
                    NO                             YES
                     |                              |
                     v                              v
              (keep monitoring,          +----------------------+
               classifier stays          |   STAGE 2: MFCC +     |
               dormant - power saved)    |   Neural Network       |
                                         |   Classifier (Edge     |
                                         |   Impulse, on-demand)  |
                                         +-----------+-----------+
                                                     |
                                          Match against known commands:
                                          [Wake] [Sleep] [Play] [Pause]
                                          [Next] [Previous]
                                                     |
                                    -------------------------------------
                                    |                                    |
                              No match / low                      Match found
                              confidence                                 |
                                    |                                    v
                                    v                       +----------------------+
                             (ignore, return                | Execute Action:      |
                              to Stage 1)                    | - Toggle listening   |
                                                              |   state (wake/sleep) |
                                                              | - Send play/pause/   |
                                                              |   next/prev command  |
                                                              |   to music module    |
                                                              | - LED/Display        |
                                                              |   confirmation       |
                                                              +----------------------+
```

### System State Diagram (Wake / Sleep Behavior)

```
        +-------------+          "wake" detected           +-------------+
        |   SLEEPING   | ---------------------------------> |   AWAKE     |
        | (only VAD +  |                                     | (listening  |
        |  wake-word   | <--------------------------------- | for play/   |
        |  check active)|         "sleep" detected            | pause/next/ |
        +-------------+                                     |  previous)  |
                                                              +-------------+
```

- **SLEEPING state**: Stage 2 classifier only checks for the "wake" keyword (ignores play/pause/next/previous to avoid accidental triggers while not actively in use).
- **AWAKE state**: Stage 2 classifier checks against the full command set (play, pause, next, previous, sleep).
- This state-based restriction is itself a small additional safeguard against false positives — commands are only "live" when the system is explicitly awake.

---

## The Core Innovation (Stated Plainly)

The individual components of this project — keyword spotting, MFCC feature extraction, embedded neural network inference — are established, existing techniques. The genuine, small-scale contribution of this project is the **two-stage gating architecture**: rather than continuously running the full, computationally expensive classification pipeline on every audio frame (the naive/default approach in most beginner embedded voice-recognition tutorials and projects), this system inserts an inexpensive, always-on **Voice Activity Detection filter** ahead of it. The heavier classifier only activates when speech-like energy is actually present.

This is a genuine, defensible engineering optimization:
- **Reduces average power consumption**, since the expensive classification stage is dormant during silence (which, realistically, is most of the time for a personal device)
- **Reduces false command triggers**, since random ambient noise (which is not speech-shaped) is filtered out before ever reaching the classifier
- **Does not sacrifice responsiveness** — once real speech is detected, Stage 2 activates immediately, so there is no perceptible delay in genuine use

This is a small, honest innovation — not a claim of inventing keyword spotting, but a claim of applying a known systems-design principle (gate expensive computation behind a cheap filter) to make the implementation meaningfully more efficient and robust than the "obvious" naive version.

---

## Expected Outcome

A working prototype where the user can say a wake word to activate listening, issue playback commands (play/pause/next/previous) via voice, and say a sleep word to deactivate listening — all processed entirely on-device, with no internet dependency, and demonstrably lower false-trigger rate than a single-stage always-on classifier due to the VAD gating mechanism.
