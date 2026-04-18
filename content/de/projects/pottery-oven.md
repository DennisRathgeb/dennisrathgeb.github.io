---
layout: article
title: "Töpferofen-Steuerung (STM32)"
date: 2024
description: "Eigenentwicklung eines elektrisch beheizten Brennofens inkl. Embedded-Regelung, Leistungselektronik und UI."
image: /assets/images/projects/kiln/cover.jpg
permalink: /projekte/pottery-oven/
lang: de
key: project-pottery-oven
sidebar:
  nav: project-de
---

> Eigenentwicklung eines elektrischen Brennofens (~25 L): Mechanik, Leistungselektronik, Embedded-Firmware und UI von Grund auf.

**Jahr:** 2024  ·  **Kontext:** Privatprojekt  ·  **Rolle:** End-to-end Einzelentwicklung

![Töpferofen im Brennvorgang – glühende Heizwicklung in hexagonaler Feuerfestkammer]({{ '/assets/images/projects/kiln/cover.jpg' | relative_url }})

**TL;DR**
- **Problem:** Einen selbstgebauten Brennofen mit reproduzierbaren, definiert gesteuerten Brennprofilen realisieren.
- **Meine Rolle:** Alles – Stahlbau, Feuerfest-Isolierung, 3-phasige Heizauslegung (~9 kW), Custom-PCB, Firmware, UI.
- **Ergebnis:** Lauffähiger Ofen mit kaskadierter Regelung (äusserer P + innerer PI auf EMA-Gradient), 20-s-Duty-Cycle auf 3× SSR, persistenter Speicher für 10 Brennprogramme × 10 Schritte, Hardware-Türkontakt als Failsafe, max. erreichte Temperatur ~1200 °C.
- **Stack:** STM32F030C8 (Cortex-M0), C (bare-metal), STM32 HAL, CMSIS-DSP, MAX31855 (Thermoelement), SSR-Schaltung, LCD1602 + Rotary Encoder, Custom-PCB.

---

## Kontext

Entwicklung und Bau eines vollständig eigenen **elektrischen Töpferofens (~25 L)** inklusive Leistungselektronik, Embedded-Firmware und Benutzerinterface.

Ziel war ein praxisnahes System, das reproduzierbare **Brennprogramme mit definierten Temperaturgradienten** ermöglicht und gleichzeitig eine einfache, robuste Bedienung bietet.

Neben der Firmware wurde auch die komplette Hardware umgesetzt – vom mechanischen Aufbau über die Heizkreise bis zur Steuerplatine.

---

## Meine Rolle

Einzelentwicklung, end-to-end:

- Thermischer Aufbau: Stahlgehäuse, Feuerfest-Isolierung, Heizwicklung (~9 kW, 3-phasig)
- Leistungselektronik: SSR-Ansteuerung, Duty-Cycle-Auslegung, Sicherheitsabschaltung
- Custom-PCB: MCU-Board mit Sensorik und Schnittstellen (SPI/I2C/USART/RTC/TIM)
- Firmware: kaskadierte Regelung, Kühlbrems-Regler, Interrupt-getriebenes Design, Flash-Persistenz
- UI: LCD + Encoder mit menügeführter Bedienung und Programm-Editor

---

## Architektur

Das System besteht aus drei Hauptkomponenten:

1. **Thermischer Aufbau**
   - Stahlgehäuse (geschweisstes Skelett)
   - Isolierung mit Hochtemperatur-Feuerfeststeinen
   - Heizwicklung aus Hochtemperatur-Heizdraht (~9 kW Gesamtleistung)
   - 3-phasige Auslegung zur gleichmässigen Lastverteilung

2. **Leistungselektronik**
   - Ansteuerung der Heizkreise über **Solid-State-Relais (SSR)**
   - Zeitfensterbasierte Leistungsregelung (Duty Cycle)
   - Sicherheitsabschaltung über Türkontakt

3. **Embedded Control System**
   - Custom-PCB mit Mikrocontroller, Sensorik und Schnittstellen
   - Temperaturmessung über Thermoelement + MAX31855 (SPI)
   - Lokales UI (LCD + Encoder + Taster)

---

## Regelung & Steuerung

### Kaskadierte Regelung

Zur stabilen Temperaturführung wurde eine zweistufige Regelung implementiert:

- **Äusserer P-Regler:**
  Temperatur → Soll-Gradient (°C/h)

- **Innerer PI-Regler:**
  Führt den gemessenen Temperaturgradienten (EMA-geglättet) auf den Sollwert

Vorteil:
Deutlich stabileres Verhalten als klassische PID-Regler bei trägen, nichtlinearen Systemen wie einem Brennofen.

---

### SSR-Leistungssteuerung

- Duty Cycle ∈ [0, 1] wird auf ein **20 s Zeitfenster** abgebildet
- Mindestschaltzeit: 5 s
- Clamping:
  - < 0.25 → AUS
  - > 0.75 → VOLLLAST

→ Verhindert schnelles Takten und reduziert Relaisbelastung

---

### Kühlbrems-Regler

- Separater P-Regler begrenzt die **maximale Abkühlrate**
- Verhindert Materialspannungen und Risse in Glasuren

---

## Firmware-Architektur

- **Interrupt-getriebenes Design**
  - Regelung: RTC-Alarm (1 Hz)
  - UI: EXTI + Timer Input Capture (Encoder)
- Event-basierte Entkopplung über einfache FIFO-Queue
- Deterministisches Verhalten trotz paralleler Eingaben

---

## Persistenz

- Nutzung des internen Flash (kein EEPROM):
  - Reservierte Flash-Seiten im Linker-Script
- Speicherung von:
  - Regelparametern
  - Bis zu 10 Brennprogrammen (je 10 Schritte)

→ System bleibt auch bei Stromausfall konsistent

---

## Benutzerinterface

- 16×2 LCD mit menügeführter Bedienung
- Drehencoder + Taster für Navigation
- Fokus auf:
  - einfache Bedienbarkeit
  - schnelle Programmauswahl
  - direkte Kontrolle während des Brennvorgangs

---

## Sicherheit

- Hardware-Türkontakt (Interrupt)
- Sofortige Abschaltung aller Heizkreise bei Öffnung
- Trennung von Steuer- und Leistungsebene

---

## Ergebnis & Impact

Ein vollständig funktionsfähiger, selbst gebauter Brennofen mit:

- reproduzierbaren Brennprofilen
- stabiler Regelung auch bei langsamer Dynamik
- robuster, eigenentwickelter Embedded-Architektur
- durchgängiger Integration von Mechanik, Elektronik und Software

---

## Tech-Stack

- **MCU:** STM32F030C8 (ARM Cortex-M0)
- **Sprache:** C (C11, Bare-Metal)
- **Libraries:** STM32 HAL, CMSIS-DSP
- **Peripherie:**
  - SPI (Temperaturmessung, MAX31855)
  - I2C (LCD1602 RGB)
  - USART (Debug)
  - RTC (1 Hz Regelzyklus)
  - TIM3 (Encoder Input)
- **Hardware:**
  - Custom-PCB (MCU + SSR-Ansteuerung + Sensorik)
  - 3× SSR für Drehstrom-Heizkreise
  - Türkontakt (Failsafe)
  - Drehencoder + Taster für UI

---

## Links

- Quellcode: [DennisRathgeb/PotteryOven](https://github.com/DennisRathgeb/PotteryOven)

---

## Reflexion

Das Projekt verbindet Embedded Software, Leistungselektronik und physikalisches Systemverhalten in einem realen Anwendungsfall. Besonders interessant war die Auslegung der Regelung für ein träges thermisches System sowie die praktische Umsetzung von Hardware und Firmware als Gesamtsystem.
