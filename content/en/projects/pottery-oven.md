---
layout: project
title: "Pottery Kiln Controller (STM32)"
date: 2024
description: "End-to-end build of an electrically heated ceramic kiln, including embedded control, power electronics and UI."
image: /assets/images/projects/kiln/cover.jpg
permalink: /en/projects/pottery-oven/
lang: en
key: project-pottery-oven
---

> End-to-end build of an electric ceramic kiln (~25 L): mechanics, power electronics, embedded firmware and UI all from scratch.

**Year:** 2024  ·  **Context:** Private project  ·  **Role:** Solo end-to-end

![Kiln during firing – glowing heating coils in a hexagonal firebrick chamber]({{ '/assets/images/projects/kiln/cover.jpg' | relative_url }})

**TL;DR**
- **Problem:** Build a self-made kiln that runs reproducible, precisely controlled firing profiles.
- **My role:** Everything — steel frame, refractory insulation, 3-phase heating layout (~9 kW), custom PCB, firmware, UI.
- **Outcome:** Working kiln with cascaded control (outer P + inner PI on an EMA gradient), 20 s duty cycle on 3× SSRs, persistent storage for 10 firing programs × 10 steps each, hardware door interlock as failsafe, max temperature reached ~1200 °C.
- **Stack:** STM32F030C8 (Cortex-M0), C (bare-metal), STM32 HAL, CMSIS-DSP, MAX31855 (thermocouple), SSR drive, LCD1602 + rotary encoder, custom PCB.

---

## Context

Design and build of a fully self-made **electric pottery kiln (~25 L)** — power electronics, embedded firmware and user interface all done from scratch.

The goal was a practical system that allows **reproducible firing programs with defined temperature gradients** while remaining simple and robust to operate.

In addition to the firmware, the full hardware was built too — from the mechanical enclosure, through the heating circuits, to the controller PCB.

---

## My Role

Solo project, end-to-end:

- Thermal assembly: welded steel frame, refractory insulation, heating element winding (~9 kW, three-phase)
- Power electronics: SSR drive, duty-cycle design, safety cutoff
- Custom PCB: MCU board with sensing and peripherals (SPI/I2C/USART/RTC/TIM)
- Firmware: cascaded control, cooling-brake controller, interrupt-driven design, flash persistence
- UI: LCD + encoder with menu-driven navigation and program editor

---

## Architecture

The system comprises three main components:

1. **Thermal assembly**
   - Welded steel frame
   - Refractory firebrick insulation
   - High-temperature heating element winding (~9 kW total)
   - Three-phase layout for balanced load distribution

2. **Power electronics**
   - Heating circuits switched via **solid-state relays (SSRs)**
   - Time-proportioning power control (duty cycle)
   - Safety cutoff via door interlock

3. **Embedded control system**
   - Custom PCB with microcontroller, sensing and interfaces
   - Temperature sensing via thermocouple + MAX31855 (SPI)
   - Local UI (LCD + rotary encoder + buttons)

---

## Control & actuation

### Cascaded control

For stable temperature tracking, a two-stage control scheme was implemented:

- **Outer P controller:**
  Temperature → target gradient (°C/h)

- **Inner PI controller:**
  Drives the measured temperature gradient (EMA-filtered) to the setpoint

Benefit:
Markedly more stable behaviour than a flat PID on sluggish, nonlinear systems like a kiln.

---

### SSR power control

- Duty cycle ∈ [0, 1] is mapped onto a **20 s time window**
- Minimum switching time: 5 s
- Clamping:
  - < 0.25 → OFF
  - > 0.75 → FULL

→ Prevents rapid toggling and reduces relay wear

---

### Cooling-brake controller

- A dedicated P controller caps the **maximum cooling rate**
- Prevents material stress and cracks in the glazes

---

## Firmware architecture

- **Interrupt-driven design**
  - Control loop: RTC alarm (1 Hz)
  - UI: EXTI + timer input capture (encoder)
- Event-based decoupling through a small FIFO queue
- Deterministic behaviour despite concurrent inputs

---

## Persistence

- Uses internal flash (no external EEPROM):
  - Flash pages reserved in the linker script
- Stores:
  - Controller parameters
  - Up to 10 firing programs (10 steps each)

→ State survives power loss

---

## User interface

- 16×2 LCD with menu-driven navigation
- Rotary encoder + buttons for input
- Focus on:
  - ease of use
  - quick program selection
  - direct control during firing

---

## Safety

- Hardware door interlock (interrupt-driven)
- Immediate shutdown of all heating circuits on door open
- Clean separation between control and power stages

---

## Outcome & Impact

A fully working, hand-built kiln with:

- reproducible firing profiles
- stable control even in slow thermal dynamics
- a robust, self-developed embedded architecture
- end-to-end integration of mechanics, electronics and software

---

## Stack

- **MCU:** STM32F030C8 (ARM Cortex-M0)
- **Language:** C (C11, bare-metal)
- **Libraries:** STM32 HAL, CMSIS-DSP
- **Peripherals:**
  - SPI (temperature sensing, MAX31855)
  - I2C (LCD1602 RGB)
  - USART (debug)
  - RTC (1 Hz control tick)
  - TIM3 (encoder input capture)
- **Hardware:**
  - Custom PCB (MCU + SSR drive + sensing)
  - 3× SSRs for three-phase heating circuits
  - Door interlock (failsafe)
  - Rotary encoder + buttons for UI

---

## Links

- Source: [DennisRathgeb/PotteryOven](https://github.com/DennisRathgeb/PotteryOven)

---

## Reflection

The project ties together embedded software, power electronics and physical system dynamics in a real application. What made it particularly interesting was designing a controller for a sluggish thermal system and bringing hardware and firmware together into a working whole.
