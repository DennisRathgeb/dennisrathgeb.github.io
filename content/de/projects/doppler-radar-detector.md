---
layout: article
title: "Radar-Detektor (STM32 + ESP32)"
date: 2025
description: "24-GHz-Radarsystem: Echtzeit-Signalverarbeitung auf dem Mikrocontroller ohne FPGA, WiFi-Livestream an eine PC-UI und ML-basierte Winkelerkennung aus I/Q-Daten."
image: /assets/images/projects/doppler-radar/cover.jpg
permalink: /projekte/doppler-radar-detector/
lang: de
key: project-doppler-radar-detector
sidebar:
  nav: project-de
---

![Prototyping-Setup – Python-Live-UI, Signalmessungen auf dem Scope, Radar-Board auf dem Tisch]({{ '/assets/images/projects/doppler-radar/cover.jpg' | relative_url }})

## Einleitung

Ein **24-GHz-Radar**, das Geschwindigkeit, Distanz **und** Winkel eines Ziels misst — Echtzeitverarbeitung vollständig auf dem Mikrocontroller, ohne FPGA. Die Winkelschätzung übernimmt ein **eigenes, auf I/Q-Rohdaten trainiertes neuronales Netz**; die Detektionen werden live über WiFi auf eine PC-UI gestreamt. Teamprojekt im ZHAW-Modul **PM4 (FS25)** mit Bryan Uhlmann und Benjamin Tschopp.

---

## Überblick

Das System misst drei Grössen aus dem reflektierten Radarsignal:

- **Geschwindigkeit** — via CW-Doppler (Frequenzverschiebung durch bewegte Ziele)
- **Distanz** — via FMCW (Beat-Frequenz zwischen gesendetem Chirp und Reflexion)
- **Winkel** — über ein eigenes **Machine-Learning-Modell**, das direkt aus den komplexen I/Q-Rohdaten den Einfallswinkel schätzt

Die zentrale Design-Constraint war der Verzicht auf ein FPGA: die gesamte Signalerfassung und -verarbeitung läuft auf einem STM32F429. Das bedeutet, der komplette Pfad — Dual-ADC-Abtastung, DMA, FFT, Feature-Extraktion — muss deterministisch und ohne Sample-Verluste durchgetaktet sein.

**YOLOv8 wurde nicht für die Winkelschätzung verwendet**, sondern ausschliesslich für die Erzeugung von **Ground-Truth-Labels**: eine Kamera nimmt das Ziel parallel zum Radar auf, YOLO bestimmt seine Position im Bild, daraus ergibt sich der wahre Winkel — mit dem dann das Radar-ML-Modell trainiert wurde.

---

## Challenges & Designentscheidungen

| Problem | Lösung | Warum |
|---|---|---|
| Hohe Sample-Rate ohne FPGA | Dual-simultaneous ADC + zirkulärer DMA + Ping-Pong-Verarbeitung | Garantiert null verlorene Samples, CPU kann parallel FFT rechnen |
| Hardware-Limits der Radarmodule | Zwei getrennte Module (CW + FMCW) statt einem umgeschalteten Modul | Keine Mode-Switch-Totzeit; beide Messpfade parallel |
| Schwaches Basisbandsignal | Eigenes Analog-Frontend mit dreistufigem Bandpass | Rauschen reduziert, Signal vor dem ADC sauber gefiltert |
| Winkel ohne phasengesteuertes Antennenarray | **ML-Modell auf I/Q-Daten**, mit kamerabasiertem Labeling trainiert | Keine zusätzliche Antennen-Hardware nötig |
| Live-Visualisierung + Debugging | WiFi-Livestream zu einer Python-PC-UI | Grosser Bildschirm, volle Datenauflösung, einfache Erweiterung für ML-Experimente |

---

## Systemarchitektur

![System-Blockdiagramm: STM32 Dev Board, Radarmodul mit Q&I-Ausgang, Radar-Board mit Bandpass-Filtern und Leistungselektronik]({{ '/assets/images/projects/doppler-radar/block-diagram.png' | relative_url }})

Vollständiger Schaltplan der Radarplatine: [`schematic.pdf`](https://github.com/DennisRathgeb/doppler-radar-detector/blob/main/docs/schematic.pdf)

- **RF-Frontend** — zwei 24-GHz-Module liefern komplexes Basisband (I/Q)
- **Analog-Board** — eigene Platine mit zwei Mehrfach-Rückkopplungs-Bandpässen (~100 dB Gain @ 12 kHz), Spannungsreglern und VCO-Ansteuerung (Zero-Order-Hold aus dem DAC)
- **STM32F429** — das Herzstück: Datenerfassung und Signalverarbeitung in Echtzeit
- **ESP32 (WLAN-Modul)** — aktuell WiFi-Bridge; mittelfristig Plattform für die ML-Inferenz
- **PC-Anwendung** — Haupt-UI, ML-Training, Datenlabeling

---

## Tech-Stack

- **MCU:** STM32F429 (Discovery Board), ESP32 (ESP-IDF)
- **Sprachen:** C (Bare-Metal + FreeRTOS), C++ (TouchGFX), Python 3
- **Libraries:** STM32 HAL, **CMSIS-DSP** (`arm_cfft_f32`), TouchGFX, Ultralytics YOLOv8 (für Labeling), FastAPI, OpenCV, PyTorch
- **Hardware:** eigenes Analog-/Radar-Board (KiCad), 24-GHz-Radarmodule (CW + FMCW), Power-Latch-Schaltung

---

## MCU-Datenpfad (STM32F429)

Der technische Kern: hohe Datenrate sauber verarbeitet **ohne FPGA**, ohne dabei die UI oder das WiFi-Streaming auszubremsen.

![Datenpfad auf dem STM32: I/Q-Output → Dual-ADC → DMA → SRAM → FFT → Frequenz-zu-Geschwindigkeit / Distanz / Audio. Separater Sendezweig: SRAM → DMA → DAC → VCO]({{ '/assets/images/projects/doppler-radar/mcu-data-path.png' | relative_url }})

### Dual-ADC im I/Q-Lock-Step

Zwei On-Chip-12-Bit-ADCs laufen im `ADC_DUALMODE_REGSIMULT`-Modus. ADC1 sampelt den Q-Kanal, ADC2 den I-Kanal — beide auf derselben Taktflanke. **Warum**: nur mit exakter Phasengleichheit zwischen I und Q ergibt sich nach der FFT ein korrektes komplexes Spektrum, das positive und negative Frequenzen (Annäherung vs. Entfernung) unterscheiden kann.

### DMA + Ping-Pong

Ein zirkulärer DMA (DMA2 Stream 0) schreibt die gepaarten Samples in einen Ringpuffer im SDRAM — CPU-frei, in Hardware. Der Puffer ist zweigeteilt:

- `ConvHalfCpltCallback` feuert, sobald die erste Hälfte voll ist
- `ConvCpltCallback` feuert, sobald die zweite Hälfte voll ist

Pro Interrupt weckt ein `xTaskNotifyFromISR` + `portYIELD_FROM_ISR` sofort die Sampling-Task. Während diese die fertige Hälfte verarbeitet, füllt der DMA die andere. **Warum**: klassisches Double-Buffering — bei hoher Sample-Rate und ohne FPGA die einzige Möglichkeit, auf einem 168-MHz-MCU keine einzige Sample-Grenze zu verlieren.

### CMSIS-DSP FFT

Pro Halbpuffer: DC-Offset entfernen → 16-Bit-Codes in `float32_t` konvertieren → `arm_cfft_f32` (komplexe FFT, in-place, bit-reversed). **Warum CMSIS-DSP**: die Library ist für den Cortex-M4 mit FPU hand-optimiert und nutzt SIMD-Instruktionen — das macht Echtzeit-FFTs auf diesem Rechenbudget überhaupt erst praktikabel.

### FreeRTOS + eigenes Modul-Design

Zwei produktive Tasks (GUI + Sampling mit hoher Priorität) plus ein Default-Task; dazu eigene Module für `measurement`, `vco`, `timing`, `esp` (SPI), `shutdown_handler`, `buzzer`, `log` und `error_handler`. Der Error-Handler wickelt jeden HAL-Call in `RETURN_OK()` / `RETURN_ERROR("grund")`-Makros — Fehler landen mit Kontext in einer prioritätsbasierten Log-Queue.

---

## Live-UI — WiFi-Streaming zur PC-Anwendung

**Das primäre User-Interface ist die PC-Anwendung**, nicht das eingebaute Display. Die Radar-Daten werden komplett über WiFi gestreamt und in einem Python-Dashboard auf dem Laptop dargestellt: grosser Bildschirm, volle Zeitauflösung, direkt anbindbar an die ML-Pipeline.

Das Display auf dem STM32F429 Discovery Board (**TouchGFX**) gibt es zusätzlich als **Debug- und Zusatzanzeige** — eine Bequemlichkeit, die das Devboard mitbringt. Interessant ist hier der Designpunkt, dass der FFT-Ausgangspuffer im SDRAM direkt der Quellpuffer des GUI-Modells ist: keine Queue, kein `memcpy`, Latenz ≈ ein Render-Frame. Aber: diese lokale Anzeige ist nicht der Produktivpfad — die volle UI läuft auf dem PC.

---

## WiFi-Bridge (ESP32)

Der ESP32 ist **SPI-Slave** zum STM32-Master. Auf STM32-Seite: SPI3, 8-Bit, Mode 0, Hardware-NSS, DMA-TX mit Busy-Flag. Auf ESP32-Seite: HSPI-Host mit Handshake-Leitung — der ESP setzt die Leitung auf High, sobald der Empfangspuffer bereit ist (Flow-Control ohne geteilten Takt).

**Zukünftige Rolle**: der ESP32 ist der designierte Einsatzort für das **Embedded-ML-Modell**. Das trainierte Modell wurde bereits auf dem PC validiert, und die Grössen- und Latenzmessungen zeigen, dass es auf einem ESP32 laufen kann. Die Portierung wurde aus Zeitgründen (Semesterumfang) zurückgestellt, ist aber der nächste Schritt.

---

## ML-Winkelerkennung

Radar allein liefert Geschwindigkeit und Distanz — aber keinen Winkel, zumindest nicht ohne phasengesteuertes Antennenarray. Die Lösung: **ein eigenes neuronales Netz, trainiert auf den komplexen I/Q-Rohdaten**, das den Einfallswinkel schätzt.

### Pipeline

1. **Datensammlung**: Radar + Kamera laufen zeitgleich; Radar liefert I/Q-Samples, Kamera liefert das Bild des Ziels.
2. **Labeling (hier kommt YOLOv8 ins Spiel)**: YOLOv8n läuft auf dem Kamerabild, detektiert das Ziel, und die horizontale Position der Bounding-Box wird in einen Winkel umgerechnet (Pixel-Position × Field-of-View). Dieser Winkel ist die **Ground Truth** für das Sample.
3. **Training**: Supervised Learning — Input: I/Q-Feature-Tensor, Label: Winkel in Grad.
4. **Inferenz**: auf dem PC bereits validiert; Portierung auf den ESP32 geplant.

**Ergebnis**: Genauigkeit im Bereich **~10° Abweichung** zum kamerabasierten Ground Truth — ohne Zusatz-Hardware am Radar.

![Angle Waterfall im Python-Live-Dashboard – oben die Winkelschätzung des Modells über der Zeit, unten die zugehörige Peak-Waterfall der FFT]({{ '/assets/images/projects/doppler-radar/angle-waterfall.png' | relative_url }})

**Warum dieser Ansatz**: ein phasengesteuertes Antennenarray war im Rahmen des Projekts nicht umsetzbar. Machine Learning ist hier die clevere Abkürzung — die Information über den Winkel steckt implizit in der Phasenlage der I/Q-Daten, und ein Netz lernt diese Beziehung direkt aus Daten statt sie analytisch modellieren zu müssen.

---

## Desktop-Anwendung

Die Python-Anwendung auf dem PC ist **nicht nebensächlich**, sondern das zentrale Werkzeug für drei Zwecke:

- **Live-Visualisierung** der gestreamten Radar-Daten (Spektrum, Detektionen, rohe I/Q) — die eigentliche Haupt-UI
- **Debugging**: komplette Rohdaten einsehbar, viel höhere Zeitauflösung als die eingebaute Anzeige
- **ML-Workflow**: Datensammlung, YOLOv8-basiertes Labeling der Kamerabilder, Training des NN, PC-seitige Inferenz zur Validierung

Die Architektur: **FastAPI-Backend** empfängt die vom ESP32 gestreamten Daten, Python-Client visualisiert sie und steuert die ML-Pipeline.

---

## Validierung

| Test | Erwartet | Gemessen | Ergebnis |
|---|---|---|---|
| LDO 3,3 V | 3,3 V ± 50 mV | 3,289 V | i.O. |
| DC-Offset | 1,65 V ± 50 mV | 1,64 V | i.O. |
| Supply-Ripple | < 10 mVpp | 200 µV | i.O. |
| AC-Gain (I, Q) @ 7 kHz | 58,28 dB ± 0,6 dB | 57,8 – 58,2 dB | i.O. |
| VCO-Sweep | 2 V / 1 ms / 0,5 ms | wie erwartet | i.O. |
| Power-Latch + Auto-Shutdown | — | funktioniert | i.O. |
| ML-Winkelschätzung | Abweichung minimieren | ~10° | validiert |

---

## Ergebnis

- Funktionierende **Echtzeit-Signalpipeline ohne FPGA** — das war der Kern-Beweis des Projekts.
- **ML-basierte Winkelbestimmung** aus I/Q-Rohdaten, validiert mit Kamera-Ground-Truth, ~10° Genauigkeit.
- Komplette End-to-End-Signalkette vom 24-GHz-Basisband bis zur PC-UI mit Live-Streaming über WiFi.
- Sauber modularisierte Firmware, reproduzierbare Hardware, dokumentierte Testkampagne.

---

## Links

- Quellcode: [DennisRathgeb/doppler-radar-detector](https://github.com/DennisRathgeb/doppler-radar-detector)

---

## Reflexion

Der interessante Teil war der Umgang mit Systemgrenzen: Wie bringt man einen Signalverarbeitungspfad, der an sich nach einem FPGA ruft, sauber auf einem MCU unter? Die Antwort war eine Mischung aus Architekturdisziplin (DMA, Ping-Pong, streng getrennte Tasks, deterministische Callbacks) und dem bewussten Einsatz hardware-nutzender Libraries (CMSIS-DSP mit FPU und SIMD).

Der zweite spannende Punkt war der Tradeoff **Hardware vs. Software** bei der Winkelerkennung: statt das Problem mit zusätzlicher RF-Hardware (phasengesteuertes Array) zu lösen, haben wir es in die Software verlagert und mit ML bearbeitet. Das Ergebnis — ~10° Genauigkeit aus reinen I/Q-Daten — zeigt, wie viel Information ein neuronales Netz aus einem Signal ziehen kann, das analytisch nicht trivial handhabbar wäre.

Das Projekt war insgesamt interdisziplinär: **RF + Analog + DSP + Embedded + ML** in einem Gerät vereint.
