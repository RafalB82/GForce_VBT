# GForce VBT — Documentation (English)

## Table of Contents

1. [Introduction](#1-introduction)
2. [Hardware variants](#2-hardware-variants)
3. [Firmware](#3-firmware)
4. [Mobile app](#4-mobile-app)
5. [BLE protocol](#5-ble-protocol)
6. [VBT algorithm](#6-vbt-algorithm)
7. [Building and assembly](#7-building-and-assembly)
8. [Licenses](#8-licenses)

---

## 1. Introduction

**GForce VBT** is an open-source Velocity-Based Training system consisting of a wireless IMU sensor and a companion mobile app. The sensor clips onto a barbell and measures bar velocity, displacement, and rep count in real time over Bluetooth Low Energy.

The project was created to provide an accessible, understandable, and modifiable VBT tool — for athletes and developers alike.

### Key features

- Real-time bar velocity measurement (mm/s)
- Automatic rep detection
- Bluetooth Low Energy data streaming
- Two hardware variants: Kapsel (104 Hz) and Pro (400 Hz)
- Flutter companion app with training history and 1RM estimation
- Fully open specification (software + hardware)

---

## 2. Hardware variants

### Kapsel (nRF52810 + LSM6DSL)

The Kapsel variant is based on the **Triki** product — an inexpensive sensor available in the Polish **Żabka** convenience store chain (manufacturer **HopX**, Poland). Out of the box it runs the original Triki firmware; to use it with GForce VBT, re-flash it via SWD.

| Parameter | Value |
|---|---|
| MCU | nRF52810 (192KB flash, 24KB RAM, Cortex-M4 without FPU) |
| IMU | LSM6DSL (accelerometer 16g + gyroscope 2000dps) |
| Sample rate | 104 Hz (after reflashing) |
| Battery | CR2032 (~2h continuous) |
| BLE protocol | wire v1 (14B) / v2 (19B) over NUS |
| Features | VBT gravity tracking, IDLE-CONNECTED low-power, training mode |
| Manufacturer | HopX Sp. z o.o. (Poland) |

**Use case:** Daily strength training, barbell exercises, home and gym use.

### Pro (nRF52840 + BHI385)

| Parameter | Value |
|---|---|
| MCU | nRF52840 SuperMini (256KB RAM, Cortex-M4F + FPU) |
| IMU | BHI385 (smart IMU ARC EM4F, Bosch SensorAPI) |
| Sample rate | 400 Hz |
| Battery | Li-Po 300mAh (~8h) |
| BLE protocol | QUAT 14B, MTU 247, packetizer 16 |
| Features | Quaternions, rotation to global frame |

**Use case:** Research, advanced motion analysis, competitive sports.

---

## 3. Firmware

### Kapsel — architecture

The Kapsel firmware runs bare-metal (no RTOS) on the nRF5 SDK 17 + SoftDevice S112 7.2.0 stack. It consists of 7 modules:

- **main.c** — boot, main loop (IMU acquisition → VBT → ring buffer → BLE)
- **trikig_vbt.c** — VBT algorithm: gravity tracking, velocity integration, rest detection, ZUPT
- **trikig_lsm6dsl.c** — LSM6DSL driver (I2C TWIM 400kHz + bit-bang fallback)
- **trikig_bb_i2c.c** — bit-bang I2C (init, bus recovery)
- **trikig_batt.c** — battery measurement (SAADC AIN2)
- **trikig_diag.c** — instrumentation and RTT diagnostics
- **trikig_board.c** — LED, button, SYSTEMOFF

### Pro — architecture

The Pro firmware runs on Zephyr RTOS / NCS v3.4.0 using the Bosch SensorAPI to communicate with the BHI385.

- Dedicated ACQ workqueue (prio 1) for BHI385 data acquisition
- SPSC ring 512 for sample buffering
- BLE packetizer (16 samples per notification)
- MODE=1 QUAT (quaternion → rotation matrix → acceleration in global frame)
- 400 Hz streaming, MTU 247

### Firmware versions (Kapsel)

| Version | Status | Description |
|---|---|---|
| v19 | **Production** | Stable production baseline |
| v0.5.0 | Built | Training mode (`20 18`), SLEEP_DUR 6s |
| v0.4.3 | Validated | IDLE-CONNECTED, velocity=0 in IDLE |
| v0.3.11 | Validated | DRDY end-to-end, VBT DSP |

---

## 4. Mobile app

**Triki_G** is a Flutter Android application. It communicates with the sensor over BLE and provides:

### Features

- **Live velocity** — real-time bar velocity display with VBT Velocity Zones
- **Rep detection** — pipeline: V2 FSM → Set-Level Segmentation → PhaseDetector → RepValidator → SetResult
- **Set summary** — MCV, MPV, GPE, rep count, velocity chart
- **Load-Velocity Profile** — 1RM estimation from sets at different loads
- **Training history** — review, CSV export
- **Training plan** — define exercises and sets
- **Statistics** — trends, load progression
- **Sensor integration** — auto-connect, FW version and battery display

### Requirements

- Android 8.0+ (API 26)
- BLE 4.0+

---

## 5. BLE Protocol

### wire v1 (14B, default after boot)

```
[0x22][0x00] [gyroX_l gyroX_h gyroY_l gyroY_h gyroZ_l gyroZ_h]
[accX_l accX_h accY_l accY_h accZ_l accZ_h]
```

All values i16LE, raw sensor data (2048 LSB/g, 16.4 LSB/dps).

### wire v2 (19B, switchable)

```
[0x22][0x01] [seq_l seq_h] [gyro6] [acc6] [vel_l vel_h] [flags]
```

- **seq** — sample counter (u16LE, wraps at 65536)
- **vel** — velocity (i16LE, mm/s)
- **flags** — bit0: moving, bit1: rest, bit2: g-estimated, bit3: low-battery, bit4: g-forced

### RX Commands

| Command | Description |
|---|---|
| `20 11 01` | Enable wire v2 |
| `20 11 00` | Disable wire v2 |
| `20 12` | Query FW info |
| `20 15 00/01` | Stream off/on |
| `20 16` | Sleep now |
| `20 17` | Measure battery |
| `20 18 01/00` | Training mode ON/OFF |

---

## 6. VBT Algorithm

The Kapsel gravity tracking algorithm uses a fixed-point (q8.8/q16.16) complementary filter to estimate the gravity vector and compute linear velocity along the movement axis:

1. **Gyro propagation:** `dg = -(w × g) * dt` — rotate estimated gravity by angular velocity
2. **Gyro bias learning:** in full rest only — tau 0.6s, requires `|w-wbias| < 5dps`
3. **ACC correction:** if movement is small (`|w-wbias| < 15dps` and innovation `< 1.0 m/s²`), correct gravity estimate from accelerometer reading
4. **Renormalization:** restore vector length to `g = 9.81 m/s²`
5. **Linear acceleration:** `lin = LPF(acc - g_est)` — filter out gravity
6. **Project onto movement axis:** `a_move = dot(lin, axis)`
7. **Velocity integration:** `v += a_move * dt`
8. **ZUPT:** dampen velocity at rest (decay 1/32 per frame)

### IDLE-CONNECTED

Since v0.4.0 the Kapsel supports low-power mode while connected over BLE:
- After 6s of rest the IMU enters low-power (accelerometer 12.5Hz, gyro off)
- On motion detection (>250mg) the HW automatically restores full ODR
- BLE stream continues (slower rate), connection params adjusted to 150ms/lat4

### Training mode

For exercises with low acceleration amplitude (dips, pull-ups), the training mode (`20 18 01`) disables IDLE during the set, ensuring 100% sample coverage.

---

## 7. Building and assembly

See `Hardware/README.md` for SWD wiring and flashing instructions using a Raspberry Pi Pico.

**Note:** After mass erase, always flash the SoftDevice first, then the application.

---

## 8. Licenses

| Component | License |
|---|---|
| Firmware | GPL-3.0 + Commons Clause |
| Hardware (BOM, instructions) | CC BY-NC-SA 4.0 |
| Applications | MIT |
| BLE protocol specification (BLE_PROTOCOL.md) | CC0 (Public Domain) |

**In plain language:**
- You may use, modify, and share non-commercially
- Selling devices built from these files requires author permission
- Applications can be used commercially (MIT)
- The BLE spec can be implemented by anyone without restriction (CC0)