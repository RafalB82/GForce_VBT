# GForce VBT — Dokumentacja (Polski)

## Spis treści

1. [Wprowadzenie](#1-wprowadzenie)
2. [Warianty sprzętowe](#2-warianty-sprzętowe)
3. [Firmware](#3-firmware)
4. [Aplikacja mobilna](#4-aplikacja-mobilna)
5. [Protokół BLE](#5-protokół-ble)
6. [Algorytm VBT](#6-algorytm-vbt)
7. [Budowa i montaż](#7-budowa-i-montaż)
8. [Licencje](#8-licencje)

---

## 1. Wprowadzenie

**GForce VBT** to zestaw otwartego sprzętu i oprogramowania do treningu opartego na prędkości (Velocity-Based Training, VBT). System składa się z bezprzewodowego czujnika IMU mocowanego do sztangi oraz aplikacji mobilnej, które w czasie rzeczywistym mierzą prędkość, przemieszczenie i liczbę powtórzeń.

Projekt powstał w odpowiedzi na potrzebę dostępnego, zrozumiałego i możliwego do modyfikacji narzędzia VBT — zarówno dla sportowców, jak i deweloperów.

### Główne cechy

- Pomiar prędkości sztangi w czasie rzeczywistym (mm/s)
- Automatyczna detekcja powtórzeń
- Przesyłanie danych przez Bluetooth Low Energy
- Dwa warianty sprzętowe: Kapsel (104 Hz) i Pro (400 Hz)
- Aplikacja Flutter z historią treningów i estymacją 1RM
- Całkowicie otwarta specyfikacja (oprogramowanie + sprzęt)

---

## 2. Warianty sprzętowe

### Kapsel (nRF52810 + LSM6DSL)

Wariant Kapsel bazuje na produkcie **Triki** — czujniku dostępnym w polskiej sieci **Żabka** (producent **HopX**, Polska). Urządzenie po wyjęciu z pudełka działa jako oryginalny Triki; do użycia z GForce VBT należy przeprogramować je przez SWD.

| Parametr | Wartość |
|---|---|
| Mikrokontroler | nRF52810 (192KB flash, 24KB RAM, Cortex-M4 bez FPU) |
| IMU | LSM6DSL (akcelerometr 16g + żyroskop 2000dps) |
| Częstotliwość próbkowania | 104 Hz (po przeprogramowaniu) |
| Bateria | CR2032 (ok. 2h ciągłej pracy) |
| Protokół BLE | wire v1 (14B) / v2 (19B) przez NUS |
| Funkcje | gravity tracking VBT, IDLE-CONNECTED low-power, tryb treningowy |
| Producent | HopX Sp. z o.o. (Polska) |

**Zastosowanie:** Codzienny trening siłowy, ćwiczenia ze sztangą, trening w domu i na siłowni.

### Pro (nRF52840 + BHI385)

| Parametr | Wartość |
|---|---|
| Mikrokontroler | nRF52840 SuperMini (256KB RAM, Cortex-M4F + FPU) |
| IMU | BHI385 (smart IMU ARC EM4F, Bosch SensorAPI) |
| Częstotliwość próbkowania | 400 Hz |
| Bateria | Li-Po 300mAh (ok. 8h) |
| Protokół BLE | QUAT 14B, MTU 247, packetizer 16 |
| Funkcje | kwaterniony, obrót do ramy globalnej |

**Zastosowanie:** Badania, zaawansowana analiza ruchu, sport wyczynowy.

---

## 3. Firmware

### Kapsel — architektura

Firmware kapsla działa na bare-metal (bez RTOS) na stosie nRF5 SDK 17 + SoftDevice S112 7.2.0. Oparty jest na 7 modułach:

- **main.c** — inicjalizacja, pętla główna (akwizycja IMU → VBT → ring → BLE)
- **trikig_vbt.c** — algorytm VBT: gravity tracking, integracja prędkości, detekcja bezruchu, ZUPT
- **trikig_lsm6dsl.c** — sterownik IMU LSM6DSL (I2C TWIM 400kHz + fallback bit-bang)
- **trikig_bb_i2c.c** — bit-bang I2C (inicjalizacja, odtwarzanie magistrali)
- **trikig_batt.c** — pomiar baterii (SAADC AIN2)
- **trikig_diag.c** — instrumentacja i diagnostyka RTT
- **trikig_board.c** — LED, przycisk, SYSTEMOFF

### Pro — architektura

Firmware Pro działa na Zephyr RTOS / NCS v3.4.0 i wykorzystuje Bosch SensorAPI do komunikacji z BHI385.

- Osobna workqueue ACQ (prio 1) dla akwizycji danych z BHI385
- SPSC ring 512 dla buforowania próbek
- BLE packetizer (16 próbek na notyfikację)
- MODE=1 QUAT domyślnie (kwaternion → macierz obrotu → akceleracja w ramie globalnej)
- Streaming 400 Hz, MTU 247

### Wersje firmware (Kapsel)

| Wersja | Status | Opis |
|---|---|---|
| v19 | **Produkcja** | Stabilna wersja produkcyjna |
| v0.5.0 | Zbudowany | Tryb treningowy (`20 18`), SLEEP_DUR 6s |
| v0.4.3 | Zwalidowany | IDLE-CONNECTED, velocity=0 w IDLE |
| v0.3.11 | Zwalidowany | DRDY end-to-end, VBT DSP |

---

## 4. Aplikacja mobilna

**Triki_G** to aplikacja Flutter dla Androida. Komunikuje się z czujnikiem przez BLE i zapewnia:

### Funkcje

- **Live velocity** — prędkość sztangi w czasie rzeczywistym z kolorowymi strefami VBT (Velocity Zones)
- **Detekcja powtórzeń** — pipeline: V2 FSM → Set-Level Segmentation → PhaseDetector → RepValidator → SetResult
- **Podsumowanie serii** — MCV, MPV, GPE, liczba powtórzeń, wykres prędkości
- **Load-Velocity Profile** — estymacja 1RM na podstawie serii z różnymi obciążeniami
- **Historia treningów** — przeglądanie, eksport CSV
- **Plan treningowy** — definiowanie ćwiczeń i serii
- **Statystyki** — trendy, progresja obciążeń
- **Integracja z czujnikiem** — automatyczne łączenie, wyświetlanie wersji FW i stanu baterii

### Wymagania

- Android 8.0+ (API 26)
- BLE 4.0+

---

## 5. Protokół BLE

### wire v1 (14B, domyślny po boot)

```
[0x22][0x00] [gyroX_l gyroX_h gyroY_l gyroY_h gyroZ_l gyroZ_h]
[accX_l accX_h accY_l accY_h accZ_l accZ_h]
```

Wszystkie wartości i16LE, raw z sensora (2048 LSB/g, 16.4 LSB/dps).

### wire v2 (19B, przełączalny)

```
[0x22][0x01] [seq_l seq_h] [gyro6] [acc6] [vel_l vel_h] [flags]
```

- **seq** — licznik próbek (u16LE, wrap 65536)
- **vel** — prędkość (i16LE, mm/s)
- **flags** — bit0: moving, bit1: rest, bit2: g-estimated, bit3: low-battery, bit4: g-forced

### Komendy RX

| Komenda | Opis |
|---|---|
| `20 11 01` | Włącz wire v2 |
| `20 11 00` | Wyłącz wire v2 |
| `20 12` | Informacje o FW |
| `20 15 00/01` | Stream off/on |
| `20 16` | Uśpij teraz |
| `20 17` | Zmierz baterię |
| `20 18 01/00` | Tryb treningowy ON/OFF |

---

## 6. Algorytm VBT

Algorytm gravity tracking na kapslu używa filtru komplementarnego (fixed-point q8.8/q16.16) do estymacji wektora grawitacji i wyznaczenia prędkości liniowej w osi ruchu:

1. **Propagacja gyro:** `dg = -(w × g) * dt` — obróć estymowany wektor grawitacji zgodnie z prędkością kątową
2. **Nauka biasu gyro:** w pełnym bezruchu (rest) — tau 0.6s, tylko przy `|w-wbias| < 5dps`
3. **Korekcja ACC:** jeśli ruch jest mały (`|w-wbias| < 15dps` i innowacja `< 1.0 m/s²`), popraw estymację grawitacji na podstawie odczytu akcelerometru
4. **Renormalizacja:** przywróć długość wektora do `g = 9.81 m/s²`
5. **Przyspieszenie liniowe:** `lin = LPF(acc - g_est)` — odfiltruj grawitację
6. **Rzutowanie na oś ruchu:** `a_move = dot(lin, axis)`
7. **Integracja prędkości:** `v += a_move * dt`
8. **ZUPT:** w bezruchu tłumienie prędkości (decay 1/32 na ramkę)

### IDLE-CONNECTED

Od wersji 0.4.0 kapsel obsługuje tryb niskiego poboru mocy podczas połączenia BLE:
- Po 6s bezruchu IMU przechodzi w low-power (akcelerometr 12.5Hz, żyroskop wyłączony)
- Po wykryciu ruchu (>250mg) HW automatycznie przywraca pełną częstotliwość
- Stream BLE biegnie dalej (wolniejsze tempo), parametry połączenia dostosowane do 150ms/lat4

### Tryb treningowy

Dla ćwiczeń o niskiej amplitudzie przyspieszenia (dipy, pull-upy) wprowadzono tryb treningowy (`20 18 01`), który wyłącza IDLE na czas serii, zapewniając 100% pokrycia próbkami.

---

## 7. Budowa i montaż

### Programowanie kapsla przez SWD

Kapsel Triki programuje się przez złącze SWD (5 padów na krawędzi PCB). Zalecany interfejs: **Raspberry Pi Pico** z OpenOCD (CMSIS-DAP).

1. Podłącz Pico do kapsla: SWDIO→GP3, SWCLK→GP2, GND→GND, 3.3V→3V3(OUT)
2. Zainstaluj OpenOCD (wymaga wsparcia CMSIS-DAP)
3. Skasuj flash: `nrf52 mass_erase`
4. Wgraj SoftDevice S112 7.2.0
5. Wgraj firmware GForce VBT (`.hex`)

Szczegółowe instrukcje i schemat połączeń: `Hardware/README.md`.

**Uwaga:** Po pełnym skasowaniu zawsze wgrywaj najpierw SoftDevice, potem aplikację.

---

## 8. Licencje

| Komponent | Licencja |
|---|---|
| Firmware | GPL-3.0 + Commons Clause |
| Sprzęt (BOM, instrukcje) | CC BY-NC-SA 4.0 |
| Aplikacje | MIT |
| Specyfikacja BLE (BLE_PROTOCOL.md) | CC0 (domena publiczna) |

**Licencje w skrócie:**
- Możesz używać, modyfikować i udostępniać niekomercyjnie
- Sprzedaż urządzeń na podstawie tych plików wymaga zgody autora
- Aplikacje możesz wykorzystać komercyjnie (MIT)
- Specyfikację BLE może implementować każdy bez ograniczeń (CC0)