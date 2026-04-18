---
layout: article
title: "Radar Detector (STM32 + ESP32)"
date: 2025
description: "24 GHz radar system: real-time signal processing on the microcontroller without an FPGA, WiFi live-stream to a PC UI, and ML-based angle estimation from I/Q data."
image: /assets/images/projects/doppler-radar/cover.jpg
permalink: /en/projects/doppler-radar-detector/
lang: en
key: project-doppler-radar-detector
sidebar:
  nav: project-en
---

> 24 GHz radar that measures a target's speed, distance and angle in real time on a microcontroller — without an FPGA, and with ML-based angle estimation from raw I/Q.

**Year:** 2025  ·  **Context:** ZHAW module PM4 (spring 2025), 3-person team  ·  **Role:** Sampling + DSP on the STM32, ML angle estimation, desktop app; contributions on the analog front-end

![Prototyping setup – Python live UI, signal plots on the scope, radar board on the bench]({{ '/assets/images/projects/doppler-radar/cover.jpg' | relative_url }})

**TL;DR**
- **Problem:** Measure speed, range *and* angle — on an STM32 instead of an FPGA, and without a phased antenna array.
- **My role:** Sampling and signal processing on the STM32 (dual-ADC/DMA/FFT, FreeRTOS), ML angle estimation, desktop app; contributions on the analog front-end.
- **Outcome:** Working end-to-end chain from 24 GHz baseband to PC UI; ML angle estimation at ~10° error vs. camera ground truth; MCU DSP pipeline (dual-ADC I/Q lock-step, circular DMA, CMSIS-DSP FFT) drop-free on a 168 MHz Cortex-M4.
- **Stack:** STM32F429 (FreeRTOS, CMSIS-DSP, TouchGFX), ESP32 (ESP-IDF), C/C++, Python (FastAPI, PyTorch, OpenCV, YOLOv8 for labeling), custom analog/RF board (KiCad).

---

## Context

A **24 GHz radar** that measures a target's speed, distance **and** angle — real-time signal processing entirely on a microcontroller, without an FPGA. Angle estimation is done by a **custom neural network trained on I/Q baseband data**; detections are streamed live over WiFi to a PC UI. Team project for ZHAW module **PM4 (spring 2025)** with Bryan Uhlmann and Benjamin Tschopp.

---

## Problem & Goal

Three quantities are extracted from the reflected radar signal:

- **Speed** — via CW Doppler (frequency shift caused by moving targets)
- **Distance** — via FMCW (beat frequency between transmitted chirp and reflection)
- **Angle** — via a **custom machine-learning model** that estimates the angle of arrival directly from the complex I/Q baseband

The central design constraint was that there is **no FPGA**: all acquisition and processing has to run on an STM32F429. That means the entire path — dual-ADC sampling, DMA, FFT, feature extraction — has to be deterministic and drop-free under real-time load.

**YOLOv8 was not used for angle estimation.** It is used purely for generating **ground-truth labels**: a camera records the target alongside the radar, YOLO locates the target in the image, its pixel position is converted into a true angle, and that angle is the supervision signal for training the radar ML model.

---

## My Role

Three-person team project. Ownership split:

- **Me (Dennis):** Sampling and signal processing on the STM32 (dual-ADC I/Q lock-step, circular DMA, CMSIS-DSP FFT, FreeRTOS task layout), ML angle estimation (training on raw I/Q, camera-based YOLOv8 labeling), desktop application (live visualization, FastAPI backend), contributions to the analog front-end.
- **Bryan Uhlmann & Benjamin Tschopp:** Filter stage on the analog board, hardware testing / validation campaign.

---

## Challenges & design decisions

| Problem | Solution | Why |
|---|---|---|
| High sample rate without an FPGA | Dual-simultaneous ADC + circular DMA + ping-pong processing | Guarantees zero dropped samples; the CPU can crunch the FFT in parallel |
| RF module limitations | Two dedicated modules (CW + FMCW) instead of one mode-switched module | No mode-switch dead time; both measurement paths run concurrently |
| Weak baseband signal | Custom three-stage analog front-end (bandpass + gain) | Noise cut early, clean signal reaches the ADC |
| Angle without a phased antenna array | **ML model on raw I/Q**, trained against camera-based labels | No additional antenna hardware required |
| Live visualization + debugging | WiFi stream to a Python PC UI | Big screen, full data resolution, easy ML-pipeline integration |

---

## Architecture

![System block diagram: STM32 dev board, radar module with Q&I output, radar board with bandpass filters and power electronics]({{ '/assets/images/projects/doppler-radar/block-diagram.png' | relative_url }})

Full schematic of the radar board: [`schematic.pdf`](https://github.com/DennisRathgeb/doppler-radar-detector/blob/main/docs/schematic.pdf)

- **RF front-end** — two 24 GHz modules, complex I/Q baseband output
- **Analog board** — custom PCB with two multi-feedback bandpass chains (~100 dB gain @ 12 kHz), regulators, and VCO drive (zero-order hold from the DAC)
- **STM32F429** — the core: real-time acquisition and signal processing
- **ESP32 (WLAN module)** — today a WiFi bridge; the intended platform for the ML inference going forward
- **PC application** — primary UI, ML training, data labeling

---

## MCU data path (STM32F429)

The technical heart: a high data rate processed cleanly **without an FPGA**, without starving the UI or the WiFi stream.

![STM32 data path: I/Q output → dual ADC → DMA → SRAM → FFT → frequency-to-velocity / distance / audio. Separate transmit branch: SRAM → DMA → DAC → VCO]({{ '/assets/images/projects/doppler-radar/mcu-data-path.png' | relative_url }})

### Dual-ADC I/Q lock-step

Two on-chip 12-bit ADCs run in `ADC_DUALMODE_REGSIMULT`. ADC1 samples the Q channel, ADC2 samples I — both on the same clock edge. **Why**: exact phase alignment between I and Q is what produces a correct complex spectrum after FFT, distinguishing positive from negative frequencies (target approaching vs. receding).

### DMA + ping-pong

A circular DMA (DMA2 Stream 0) writes paired samples into a SDRAM ring buffer — CPU-free, in hardware. The buffer is split in two halves:

- `ConvHalfCpltCallback` fires when the first half is full
- `ConvCpltCallback` fires when the second half is full

Each interrupt wakes the sampling task via `xTaskNotifyFromISR` + `portYIELD_FROM_ISR`. While the task processes the completed half, the DMA is already filling the other. **Why**: classic double-buffering — at a high sample rate and without an FPGA, this is the only reliable way to preserve every sample boundary on a 168 MHz MCU.

### CMSIS-DSP FFT

Per half-buffer: remove the DC offset → convert 16-bit codes to `float32_t` → `arm_cfft_f32` (complex FFT, in-place, bit-reversed). **Why CMSIS-DSP**: the library is hand-tuned for the Cortex-M4 FPU and uses SIMD instructions — that is what makes real-time FFTs practical at this compute budget.

### FreeRTOS + custom module layout

Two productive tasks (GUI + high-priority sampling) plus a default task; custom modules for `measurement`, `vco`, `timing`, `esp` (SPI), `shutdown_handler`, `buzzer`, `log`, and `error_handler`. The error handler wraps every HAL call in `RETURN_OK()` / `RETURN_ERROR("reason")` macros — errors land in a priority log queue with full context.

---

## Live UI — WiFi streaming to the PC app

**The primary user interface is the PC application**, not the built-in display. Radar data is streamed over WiFi and visualized in a Python dashboard on the laptop: big screen, full time resolution, directly pluggable into the ML pipeline.

The **TouchGFX display** on the STM32F429 Discovery board is present as a **debug / auxiliary view** — a convenience afforded by the dev board. There is one interesting design point worth noting: the FFT output buffer in SDRAM is the GUI model's source buffer — no queue, no `memcpy`, latency ≈ one render frame. But this local display is not the production path; the full UI lives on the PC.

---

## WiFi bridge (ESP32)

The ESP32 is the **SPI slave** to the STM32 master. STM32 side: SPI3, 8-bit, mode 0, hardware NSS, DMA TX with a busy flag. ESP32 side: HSPI host with a handshake line — the ESP raises the line when its receive buffer is queued, giving the master flow control without sharing a clock.

**Future role**: the ESP32 is the intended home for the **embedded ML model**. The trained model has already been validated on the PC, and size / latency measurements show it fits the ESP32's budget. Porting was deferred for time-box reasons (end-of-semester project) but is the next step.

---

## ML angle estimation

Radar alone gives you speed and range — not angle, at least not without a phased antenna array. The solution: **a custom neural network trained on the raw I/Q baseband** that estimates the angle of arrival.

### Pipeline

1. **Data collection**: radar and camera run simultaneously; radar yields I/Q samples, camera yields a video frame of the target.
2. **Labeling (this is where YOLOv8 belongs)**: YOLOv8n runs on the camera frame, localizes the target, and the horizontal bbox position is converted into an angle (pixel position × field of view). That angle is the **ground truth** for the sample.
3. **Training**: supervised learning — input: I/Q feature tensor, label: angle in degrees.
4. **Inference**: validated on the PC; ESP32 port planned.

**Result**: accuracy in the **~10°** range compared to the camera-based ground truth — achieved without any extra radar hardware.

![Angle Waterfall in the Python live dashboard – top: model's angle estimate over time, bottom: corresponding FFT peak waterfall]({{ '/assets/images/projects/doppler-radar/angle-waterfall.png' | relative_url }})

**Why this approach**: a phased antenna array was not feasible within the project scope. Machine learning is the clever shortcut here — the angle information is implicit in the phase relationship of the I/Q data, and a neural network learns this mapping directly from data instead of having to model it analytically.

---

## Desktop application

The Python PC app is **not an afterthought** — it is the central workbench with three jobs:

- **Live visualization** of the streamed radar data (spectrum, detections, raw I/Q) — the actual primary UI
- **Debugging**: full raw data accessible, higher time resolution than the on-board display
- **ML workflow**: data collection, YOLOv8-based labeling of camera frames, training the neural network, PC-side inference for validation

Architecture: **FastAPI backend** receives the stream forwarded by the ESP32; Python client visualizes it and drives the ML pipeline.

---

## Outcome & Impact

- A working **real-time signal pipeline without an FPGA** — the core proof of the project.
- **ML-based angle estimation** from raw I/Q data, validated against camera ground truth, ~10° accuracy.
- Complete end-to-end signal chain from the 24 GHz baseband to a PC UI with WiFi live streaming.
- Cleanly modularized firmware, reproducible hardware, documented test campaign.

### Validation

| Test | Expected | Measured | Verdict |
|---|---|---|---|
| LDO 3.3 V | 3.3 V ± 50 mV | 3.289 V | OK |
| DC offset | 1.65 V ± 50 mV | 1.64 V | OK |
| Rail ripple | < 10 mVpp | 200 µV | OK |
| AC gain (I, Q) @ 7 kHz | 58.28 dB ± 0.6 dB | 57.8 – 58.2 dB | OK |
| VCO sweep | 2 V / 1 ms / 0.5 ms | as expected | OK |
| Power latch + auto-shutdown | — | works | OK |
| ML angle estimation | minimize error | ~10° | validated |

---

## Stack

- **MCUs:** STM32F429 (Discovery board), ESP32 (ESP-IDF)
- **Languages:** C (bare-metal + FreeRTOS), C++ (TouchGFX), Python 3
- **Libraries:** STM32 HAL, **CMSIS-DSP** (`arm_cfft_f32`), TouchGFX, Ultralytics YOLOv8 (for labeling), FastAPI, OpenCV, PyTorch
- **Hardware:** custom analog/radar board (KiCad), 24 GHz radar modules (CW + FMCW), power-latch circuit

---

## Links

- Source: [DennisRathgeb/doppler-radar-detector](https://github.com/DennisRathgeb/doppler-radar-detector)

---

## Reflection

The interesting part was dealing with system limits: how do you take a signal-processing path that really asks for an FPGA and run it cleanly on an MCU? The answer was a mix of architectural discipline (DMA, ping-pong, strictly separated tasks, deterministic callbacks) and deliberate use of hardware-aware libraries (CMSIS-DSP on the FPU with SIMD).

The second interesting point was the **hardware-vs-software tradeoff** in the angle estimation: rather than solving the problem with additional RF hardware (a phased antenna array), we moved it into software and solved it with ML. The result — ~10° accuracy from pure I/Q data — is a small demonstration of how much information a neural network can extract from a signal that would be non-trivial to handle analytically.

The project ended up genuinely interdisciplinary: **RF + analog + DSP + embedded + ML** in a single device.
