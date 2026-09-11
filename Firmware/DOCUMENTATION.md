# GForce VBT — Firmware Documentation

## 1. Hardware variants

### 1.1 Kapsel (nRF52810 + LSM6DSL)

Based on the **Triki** product (available in Polish Żabka stores, manufactured by HopX). Re-flashed with GForce VBT firmware via SWD.

| Component | Detail |
|---|---|
| MCU | nRF52810 (192KB flash, 24KB RAM, Cortex-M4, no FPU) |
| SoftDevice | S112 7.2.0 @ 0x0, app @ 0x19000 |
| IMU | LSM6DSL, I2C addr 0x6A |
| External flash | MX25R8035F 1MB SPI (not used in current FW) |
| Crystal 32k | None — LFCLK = internal RC |
| Power | CR2032 3V directly on VDD |
| Battery measurement | SAADC AIN2 (P0.04), gain 1/6, ref 0.6V |

#### Pinout

| Pin | Function | Configuration |
|---|---|---|
| P0.05 | I2C SDA (bit-bang) | open-drain S0D1 + internal pullup |
| P0.06 | I2C SCL (bit-bang) | open-drain S0D1 + internal pullup |
| P0.09 | LSM INT1 / SLEEP_CHANGE | GPIOTE TOGGLE + readback WAKE_UP_SRC |
| P0.10 | LSM INT2 / DRDY_XL | GPIOTE, sample timestamp |
| P0.04 | SAADC AIN2 — Vbat | Battery voltage (no voltage divider) |
| P0.25 | BTN to GND | input PULLUP, active-low |
| P0.28 | LED | output, active-low |

### 1.2 Pro (nRF52840 + BHI385)

| Component | Detail |
|---|---|
| MCU | nRF52840 SuperMini (Cortex-M4F + FPU) |
| IMU | BHI385 (smart IMU, ARC EM4F + 6DoF, Bosch SensorAPI) |
| Stack | Zephyr RTOS / NCS v3.4.0 |
| Board | promicro_nrf52840/nrf52840/uf2 |
| BLE | MTU 247, wire 14B/sample (QUAT), packetizer 16 |
| Rate | 400 Hz |

---

## 2. Firmware architecture (Kapsel)

Bare-metal + nRF5 SDK 17 (no RTOS). 7 modules + main:

```
main.c              — boot (WDT, DRDY probe, BLE stack/timers), main loop
                      (DRDY acquisition → VBT → ring buffer → send, BTN, sleep)
trikig_bb_i2c.c/h   — bit-bang I2C master ~250kHz (init/bus-clear/fault-recovery)
trikig_lsm6dsl.c/h  — IMU init (WHO_AM_I retry, SW reset, config readback)
                      + read OUT 0x22 (TWIM 400kHz with fallback bb)
                      + DRDY enable (INT2_CTRL)
trikig_diag.c/h     — instrumentation: drop counters, dt min/avg/max, TIMER1 timing
trikig_vbt.c/h      — VBT DSP: gravity tracking, movement-axis velocity
trikig_batt.c/h     — SAADC battery measurement (CR2032)
trikig_board.c/h    — LED/BTN/SYSTEMOFF + RTT diagnostic printf
```

### Data flow (SPSC, single direction)

```
INT2 DRDY (LSM6DSL INT2_CTRL=DRDY_XL, P0.10)
  → GPIOTE ISR (timestamp TIMER1 @1MHz)
  → main loop: lsm6dsl_read_motion() [TWIM 400kHz, fallback BB I2C, 12B from 0x22]
  → VBT on-frame (dt = t[n]-t[n-1])
  → frame v1 14B / v2 19B → ring[16]
  → ble_nus_data_send → BLE notify
fallback (watchdog 30ms): poll 9ms with dup-guard

IDLE-CONNECTED: INT1 SLEEP_CHANGE (P0.09, TOGGLE)
  → readback WAKE_UP_SRC → state
  → DRDY/data still @12.5Hz, dt from RTC1, watchdog 160ms
```

### LSM6DSL configuration

| Register | Value | Meaning |
|---|---|---|
| CTRL1_XL (0x10) | 0x44 | ODR 104Hz, FS_XL=16g |
| CTRL2_G (0x11) | 0x4C | ODR 104Hz, FS_G=2000dps |
| CTRL3_C (0x12) | 0x0C | BDU + IF_INC |
| FIFO_CTRL5 (0x0A) | 0x00 | FIFO bypass |
| TAP_CFG (0x58) | 0xE1 | INTERRUPTS_ENABLE + INACT_EN=11 + LIR |
| WAKE_UP_THS (0x5B) | 0x01 | WK_THS=1 = 250mg @ FS 16g |
| WAKE_UP_DUR (0x5C) | 0x06 | SLEEP_DUR=6s inactivity timeout |
| MD1_CFG (0x5E) | 0x80 | INT1_SLEEP_CHANGE |

Sensitivity: acc 2048 LSB/g, gyro 16.4 LSB/dps.

---

## 3. VBT algorithm

### 3.1 Gravity tracking (complementary filter)

The velocity estimation uses a fixed-point complementary filter:

1. **Gyro propagation:** `dg = -(w × g) * dt` per frame (q16.16 × q8.8)
2. **Bias learning:** gyro bias learned only in full rest (s_rest AND |w-wbias| < 5dps, tau 0.6s)
3. **ACC correction:** gated (|w-wbias| < 15dps AND innovation < 1.0 m/s²) with slow leak 1/2048
4. **Renormalization** to |g| = 9.81 m/s²
5. **Linear acceleration:** `lin = LPF(acc - g_est)`
6. **Movement axis:** `a_move = dot(lin, axis)`, axis default X (barbell)
7. **Velocity integration:** `v += a_move * dt`

### 3.2 Rest detection

`||lin|| < 0.3 m/s²` for 8 consecutive frames — first-order detector.

### 3.3 ZUPT (Zero Velocity Update)

Decay 1/32 per frame at rest (tau ~0.31s) with minimum step of 1 q8.8.

### 3.4 IDLE-CONNECTED (low-power during BLE session)

| Aspect | ACTIVE | IDLE-CONNECTED |
|---|---|---|
| IMU acc ODR | 104Hz | 12.5Hz low-power |
| IMU gyro | 104Hz | power-down |
| BLE stream | 19B @104/s | 19B @12.5/s |
| Connection params | 7.5-15ms, lat 0 | 150ms, latency 4 |
| VBT | full (gyro+acc) | velocity = 0 by design |
| Entry | activity | 6s inactivity (SLEEP_DUR) |
| Exit | activity detected | motion > WK_THS (250mg) |

### 3.5 Training mode (v0.5.0)

RX command `20 18 01/00` to disable IDLE-CONNECTED during a set:

- **ON:** inactivity disabled, IMU forced to full ODR, VBT reset for clean calibration
- **OFF:** inactivity re-enabled, HW enters IDLE after SLEEP_DUR
- **Auto-OFF:** on disconnect (safety)

Motivation: dip/pullup exercises have movement class 140-246mg p95, below WK_THS 250mg hardware floor — HW never wakes from IDLE during these sets.

---

## 4. Firmware architecture (Pro / BHI385)

### Stack

- Zephyr RTOS / NCS v3.4.0
- Board: promicro_nrf52840/nrf52840/uf2
- Bosch SensorAPI via I2C (addr 0x29, SDA P0.17, SCL P0.20)

### Data flow

```
BHI385 FIFO @400Hz → ACQ workqueue (prio 1)
  → SPSC ring 512 (multi-reader)
  → BLE packetizer (16 samples → 227B ≤ MTU 244)
  → NUS notify
USB: printk preview (every 16th sample)
```

### Rotation modes

- **MODE=1 QUAT (default):** quaternion → rotation matrix R, LACC rotated to global frame
- **EULER:** optional, NaN risk at 360°
- **Wire format:** 14B/sample: seq4 + ts4 + gx + gy + gz (magic AB 03), milli-g scale

---

## 5. Known limitations (Kapsel)

1. **No FPU** — all DSP in fixed-point q8.8/q16.16
2. **No 32k crystal** — LFCLK = RC, dryf ~±1-2% typ.
3. **WK_THS floor at FS16g = 250mg** — cannot be lowered; training mode is the fix
4. **Velocity wander** ~±0.4 m/s under simultaneous rotation + oscillation
5. **No FIFO** — LSM6DSL FIFO not validated without logic analyzer
6. **Battery offset** — Vf diode offset requires per-unit calibration

## 6. Known limitations (Pro)

1. **BLE on rpi8gb** — BlueZ doesn't negotiate MTU 247; use Android phone
2. **USB CDC** — binary USB stream not yet implemented (printk preview only)
3. **FPU** — soft-float ABI required (FP_HARDABI fails)
4. **LACC saturation** — sat=N in DIAG indicates clamp at >3.34g (range 4g)

## 7. Flashing via SWD

See `Hardware/README.md` for SWD wiring and OpenOCD commands using a Raspberry Pi Pico.

### Version history (Kapsel)

| Version | Status | Key features |
|---|---|---|
| v19 | **PRODUCTION** | Stable baseline, 104Hz end-to-end |
| v0.1.0 | built | wire v2 (F1), RX commands (F2) |
| v0.2.0 | built | Battery measurement (F5) |
| v0.3.11 | **validated [P]** | DRDY end-to-end, TWIM 400kHz, VBT DSP |
| v0.4.0 | validated | IDLE-CONNECTED low-power |
| v0.4.3 | validated | velocity=0 in IDLE, training data regression |
| **v0.5.0** | built | **Training mode** `20 18`, SLEEP_DUR 6s |