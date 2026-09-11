# GForce VBT — Hardware

[🇵🇱 Polski](#gforce-vbt---sprzęt-polski) | [🇬🇧 English](#gforce-vbt---hardware-english)

---

## GForce VBT — Sprzęt (Polski)

### Kapsel (nRF52810 + LSM6DSL)

Czujnik GForce VBT w wariancie Kapsel bazuje na produkcie **Triki** — tanim czujniku IMU dostępnym w polskiej sieci sklepów **Żabka**. Producentem jest polska spółka **HopX**.

Urządzenie po wyjęciu z pudełka działa jako oryginalny Triki. Aby używać go z oprogramowaniem GForce VBT, należy przeprogramować firmware przez złącze SWD.

**Parametry:**
- Mikrokontroler: nRF52810 (192KB flash, 24KB RAM, Cortex-M4)
- IMU: LSM6DSL (akcelerometr 16g + żyroskop 2000dps)
- Częstotliwość próbkowania: 104 Hz (po przeprogramowaniu)
- Bateria: CR2032 (~2h ciągłej pracy)
- Producent: HopX Sp. z o.o. (Polska)

### Pro (nRF52840 + BHI385)

Zaawansowany wariant z smart IMU BHI385 i mikrokontrolerem nRF52840 SuperMini. 400 Hz, kwaterniony, pełna orientacja w przestrzeni. Płytka nRF52840 SuperMini jest dostępna w sklepach internetowych (AliExpress, Amazon).

### Reprogramowanie kapsla przez SWD

Do wgrania własnego firmware na kapsel Triki potrzebujesz:

1. **Interfejs SWD** na bazie **Raspberry Pi Pico** (lub inny debugger zgodny z OpenOCD, np. J-Link, ST-Link)
2. **OpenOCD** — oprogramowanie do programowania przez SWD
3. **SoftDevice + firmware** — pliki `.hex` z `Firmware/triki/`

#### Schemat podłączenia Pico → kapsel Triki

| Funkcja | Pico (GPIO) | Kapsel (pad SWD) |
|---|---|---|
| SWDIO | GP3 | SWDIO |
| SWCLK | GP2 | SWCLK |
| GND | GND | GND |
| 3.3V | 3V3(OUT) | 3V3 |

Pady SWD na kapslu: najłatwiej dostępne są na krawędzi PCB obok baterii. Kolejność pinów (od krawędzi PCB): 3V3, GND, nRESET, SWDIO, SWCLK. Przed podłączeniem sprawdź multimetrem ciągłość do masy i 3.3V.

#### Alternatywne interfejsy SWD

- **J-Link EDU / EDU Mini** — droższy, ale w pełni obsługiwany przez nrfjprog
- **ST-Link V2** — tani, działa przez OpenOCD z konfiguracją `stlink.cfg`

#### OpenOCD — konfiguracja (Raspberry Pi Pico)

Zainstaluj OpenOCD z supportem dla Pico (wymaga kompilacji ze źródła lub najnowszej wersji):

```bash
# Pico jako CMSIS-DAP — użyj konfiguracji:
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/nrf52.cfg
```

#### Komendy flashowania

```bash
# 1. Skasuj cały flash
openocd -f interface/swd.cfg -f target/nrf52.cfg \
  -c "init" -c "nrf52 mass_erase" -c "exit"

# 2. Wgraj SoftDevice (S112) — najpierw SD!
openocd -f interface/swd.cfg -f target/nrf52.cfg \
  -c "init" -c "flash write_image erase s112_nrf52_7.2.0_softdevice.hex" -c "exit"

# 3. Wgraj aplikację
openocd -f interface/swd.cfg -f target/nrf52.cfg \
  -c "init" -c "flash write_image erase trikig_triki_v0.5.0_rtt.hex" -c "exit"
```

**Ważne:** Po pełnym skasowaniu (mass erase) zawsze wgrywaj najpierw SoftDevice, potem aplikację. Wgranie samej aplikacji po skasowaniu powoduje HardFault (pusty wektor przerwań).

---

## GForce VBT — Hardware (English)

### Kapsel (nRF52810 + LSM6DSL)

The GForce VBT Kapsel variant is based on the **Triki** product — an inexpensive IMU sensor sold in the **Żabka** convenience store chain in Poland. The manufacturer is the Polish company **HopX**.

Out of the box, the device runs the original Triki firmware. To use it with GForce VBT software, you need to re-flash it via the SWD debug connector.

**Specifications:**
- MCU: nRF52810 (192KB flash, 24KB RAM, Cortex-M4)
- IMU: LSM6DSL (accelerometer 16g + gyroscope 2000dps)
- Sample rate: 104 Hz (after reflashing)
- Battery: CR2032 (~2h continuous)
- Manufacturer: HopX Sp. z o.o. (Poland)

### Pro (nRF52840 + BHI385)

Advanced variant with a BHI385 smart IMU and nRF52840 SuperMini MCU. 400 Hz, quaternions, full 3D orientation. The nRF52840 SuperMini board is available from online retailers (AliExpress, Amazon).

### Flashing the Kapsel via SWD

To flash custom firmware onto the Triki kapsel you need:

1. **SWD interface** based on **Raspberry Pi Pico** (or any OpenOCD-compatible debugger: J-Link, ST-Link)
2. **OpenOCD** — open-source debugging/programming tool
3. **SoftDevice + firmware** — `.hex` files from `Firmware/triki/`

#### Pico → Kapsel wiring

| Function | Pico (GPIO) | Kapsel (SWD pad) |
|---|---|---|
| SWDIO | GP3 | SWDIO |
| SWCLK | GP2 | SWCLK |
| GND | GND | GND |
| 3.3V | 3V3(OUT) | 3V3 |

SWD pads on the kapsel are accessible on the PCB edge near the battery. Pin order (from PCB edge): 3V3, GND, nRESET, SWDIO, SWCLK. Verify continuity with a multimeter before connecting.

#### OpenOCD commands

```bash
# 1. Mass erase
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/nrf52.cfg \
  -c "init" -c "nrf52 mass_erase" -c "exit"

# 2. Flash SoftDevice (S112) — SD first!
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/nrf52.cfg \
  -c "init" -c "flash write_image erase s112_nrf52_7.2.0_softdevice.hex" -c "exit"

# 3. Flash application
openocd -f interface/cmsis-dap.cfg -c "adapter speed 1000" -f target/nrf52.cfg \
  -c "init" -c "flash write_image erase trikig_triki_v0.5.0_rtt.hex" -c "exit"
```

**Important:** After mass erase, always flash the SoftDevice first, then the app. App-only after full erase causes a HardFault.