# GForce VBT — Firmware

[🇵🇱 Polski](#gforce-vbt---firmware-polski) | [🇬🇧 English](#gforce-vbt---firmware-english)

---

## GForce VBT — Firmware (Polski)

Firmware dla czujnika GForce VBT dostępny jest w dwóch wariantach sprzętowych:

### Kapsel (nRF52810 + LSM6DSL)
- **MCU:** nRF52810 (192KB flash, 24KB RAM, Cortex-M4 bez FPU)
- **IMU:** LSM6DSL (akcelerometr 16g + żyroskop 2000dps)
- **Stos:** nRF5 SDK 17, SoftDevice S112 7.2.0, bare-metal
- **Częstotliwość:** 104 Hz
- **Protokół BLE:** wire v1 (14B) / v2 (19B) przez NUS
- **Funkcje:** gravity tracking VBT, IDLE-CONNECTED low-power, training mode (`20 18`)
- **Wersje:** v0.1.0 – v0.5.0 (produkcja: v19)

### Pro (nRF52840 + BHI385)
- **MCU:** nRF52840 (256KB RAM, Cortex-M4F + FPU)
- **IMU:** BHI385 (smart IMU, ARC EM4F, Bosch SensorAPI, fusion/kwaterniony)
- **Stos:** Zephyr RTOS / NCS v3.4.0
- **Częstotliwość:** 400 Hz
- **Protokół BLE:** QUAT 14B, MTU 247, packetizer 16
- **Funkcje:** kwaterniony, obrót do ramy globalnej, streaming 400Hz

### Binarki

Gotowe pliki `.hex` do flashowania znajdują się w katalogu `triki/`.

### Dokumentacja

- `DOCUMENTATION.md` — architektura firmware, algorytm VBT, moduły
- `BLE_PROTOCOL.md` — specyfikacja protokołu Bluetooth (bajt po bajcie)

---

## GForce VBT — Firmware (English)

Firmware for the GForce VBT sensor is available in two hardware variants:

### Kapsel (nRF52810 + LSM6DSL)
- **MCU:** nRF52810 (192KB flash, 24KB RAM, Cortex-M4 without FPU)
- **IMU:** LSM6DSL (accelerometer 16g + gyroscope 2000dps)
- **Stack:** nRF5 SDK 17, SoftDevice S112 7.2.0, bare-metal
- **Rate:** 104 Hz
- **BLE protocol:** wire v1 (14B) / v2 (19B) over NUS
- **Features:** VBT gravity tracking, IDLE-CONNECTED low-power, training mode (`20 18`)
- **Versions:** v0.1.0 – v0.5.0 (production: v19)

### Pro (nRF52840 + BHI385)
- **MCU:** nRF52840 (256KB RAM, Cortex-M4F + FPU)
- **IMU:** BHI385 (smart IMU, ARC EM4F, Bosch SensorAPI, sensor fusion / quaternions)
- **Stack:** Zephyr RTOS / NCS v3.4.0
- **Rate:** 400 Hz
- **BLE protocol:** QUAT 14B, MTU 247, packetizer 16
- **Features:** quaternion output, rotation to global frame, 400Hz streaming

### Binaries

Pre-built `.hex` files are available in the `triki/` directory.

### Documentation

- `DOCUMENTATION.md` — firmware architecture, VBT algorithm, modules
- `BLE_PROTOCOL.md` — byte-exact Bluetooth protocol specification