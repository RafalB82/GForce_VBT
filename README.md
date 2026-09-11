# GForce VBT

**Velocity-Based Training sensor — open hardware & firmware**

[🇵🇱 Polski](#gforce-vbt---polski) | [🇬🇧 English](#gforce-vbt---english)

---

## GForce VBT — Polski

**GForce VBT** to zestaw otwartego sprzętu i oprogramowania do treningu opartego na prędkości (VBT). System składa się z bezprzewodowego czujnika IMU mocowanego do sztangi oraz aplikacji mobilnej, które w czasie rzeczywistym mierzą prędkość, przemieszczenie i liczbę powtórzeń.

Projekt jest dostępny w dwóch wariantach sprzętowych:

| Wariant | MCU | IMU | Częstotliwość | BLE | Bateria |
|---|---|---|---|---|---|
| **Kapsel** | nRF52810 | LSM6DSL | 104 Hz | 14B/19B wire | ~2 h |
| **Pro** | nRF52840 | BHI385 | 400 Hz | QUAT 14B, MTU 247 | ~8 h |

### Repozytorium

```
GForce_VBT/
├── Firmware/     — dokumentacja firmware, binarki, narzędzia
├── Apps/         — aplikacje mobilne i desktopowe
├── Hardware/     — BOM, instrukcje montażu
└── docs/         — pełna dokumentacja (EN/PL)
```

### Licencje

- **Firmware**: GPL-3.0 + Commons Clause — możesz używać, modyfikować i udostępniać niekomercyjnie. Sprzedaż urządzeń na podstawie tych plików wymaga zgody autora.
- **Sprzęt (BOM, instrukcje)**: CC BY-NC-SA 4.0
- **Aplikacje**: MIT
- **Specyfikacja protokołu BLE**: CC0 (domena publiczna) — każdy może implementować własną aplikację bez ograniczeń.

### Jak zacząć

1. **Zbuduj lub kup** — zobacz `Hardware/BOM/` po listę części
2. **Wgraj firmware** — pobierz `.hex` z `Firmware/triki/`
3. **Zainstaluj aplikację** — zobacz `Apps/`
4. **Kalibruj i trenuj** — podłącz, skalibruj, zacznij serię

---

## GForce VBT — English

**GForce VBT** is an open-source Velocity-Based Training system consisting of a wireless IMU sensor that clips onto a barbell and a companion mobile app. It measures bar velocity, displacement, and rep count in real time over Bluetooth Low Energy.

Two hardware variants are available:

| Variant | MCU | IMU | Rate | BLE | Battery |
|---|---|---|---|---|---|
| **Kapsel** | nRF52810 | LSM6DSL | 104 Hz | 14B/19B wire | ~2 h |
| **Pro** | nRF52840 | BHI385 | 400 Hz | QUAT 14B, MTU 247 | ~8 h |

### Repository layout

```
GForce_VBT/
├── Firmware/     — firmware documentation, binaries, tools
├── Apps/         — mobile & desktop applications
├── Hardware/     — BOM, assembly instructions
└── docs/         — full documentation (EN/PL)
```

### Licenses

- **Firmware**: GPL-3.0 + Commons Clause — non-commercial use, modification, and redistribution allowed; selling devices built from these files is reserved to the author.
- **Hardware (BOM, instructions)**: CC BY-NC-SA 4.0
- **Applications**: MIT
- **BLE protocol specification**: CC0 (public domain) — third-party apps can be built against it without restriction.

### Getting started

1. **Build or buy** — see `Hardware/BOM/` for the parts list
2. **Flash the firmware** — download `.hex` from `Firmware/triki/`
3. **Install the app** — see `Apps/`
4. **Calibrate and lift** — connect, calibrate while still, start your set