# GForce VBT — Kapsel firmware releases

## Latest build

| File | Version | Description |
|---|---|---|
| `trikig_triki_v0.5.0_rtt.hex` | **v0.5.0** | Latest development build (RTT enabled) |

GIT commit: `e7b68ac` on branch `fix3/nrf52-batt-ble`. Built from sources at [Triki_GForce_fw](https://github.com/RafalB82/Triki_GForce_fw).

### SHA256

```
30cdbe6046ae68e0e586701a1a7593c226354390002fd747c43d717b04dfa68d  trikig_triki_v0.5.0_rtt.hex
```

---

## Implemented features (v0.5.0)

### IMU (LSM6DSL)

- Accelerometer 104 Hz ODR, ±16g full-scale (2048 LSB/g)
- Gyroscope 104 Hz ODR, ±2000 dps full-scale (16.4 LSB/dps)
- Block Data Update (BDU) + register auto-increment enabled
- FIFO bypass (poll-based reads)
- SW reset on init with 50ms timeout and readback verification
- WHO_AM_I check (0x6A) with retry and bus-clear recovery
- Readback validation of all configuration registers at boot (CTRL1/CTRL2/CTRL3)
- I2C: hybrid TWIM 400kHz (DMA, ~40µs) with automatic bit-bang fallback on fault
- TWIM banned after 3 consecutive faults (prevents re-init storm)

### VBT Engine (Velocity-Based Training)

- **Gravity tracking** — complementary filter (fixed-point q8.8/q16.16):
  - Gyro propagation: `dg = -(ω × g) × dt` each frame
  - ACC correction: gated when `|ω| < 15 dps` AND `||acc − g_est|| < 1.0 m/s²`
  - Permanent slow leak (1/2048) prevents dead-lock
  - Renormalization to `g_ref` every frame via integer `isqrt32`
- **Gyro bias learning** — online estimation in full rest (`||lin|| < 0.3 m/s²` AND `|ω − ω_bias| < 5 dps`), tau ~0.6s; disabled in IDLE mode
- **Rest detection** — first-order: `||lin|| < 0.3 m/s²` for 8 consecutive frames
- **Velocity integration** — 1-axis projection (default X = barbell, configurable), dt from TIMER1 timestamps
- **ZUPT** (Zero Velocity Update) — decay 1/32 per frame at rest with minimum step of 1 q8.8
- **Windup clamp** — velocity capped at ±15.6 m/s
- **IDLE mode** — velocity forced to 0 when gyro is in power-down
- **Force-fallback** — gravity re-estimated from LPF(acc) after ~5s without rest
- **Boot-hold** — velocity integration gated until first rest gravity estimate
- API: `vbt_velocity_mms()`, `vbt_moving()`, `vbt_flags()`, `vbt_reset()`, `vbt_set_axis()`, `vbt_idle()`

### Data Acquisition (DRDY)

- Hardware DRDY from LSM6DSL INT2 → P0.10 (confirmed by board measurement)
- Boot probe with drain-read (eliminates false-negative BDU bug from v0.3.1–v0.3.10)
- ISR: TIMER1 @1MHz timestamp only (no I2C in interrupt context)
- 4-slot timestamp FIFO from ISR → main loop
- Main loop processes I2C read + VBT + ring buffer write
- Watchdog fallback: 30ms (active) / 160ms (IDLE) — poll timer triggers fallback sample
- dt calculation from consecutive DRDY timestamps: clamp 4–40ms, gap >60ms → hard ZUPT
- RTC1-based gap detection (independent of 16-bit TIMER1 window)

### Bluetooth Low Energy

- Nordic UART Service (NUS) — vendor-specific UUID
- Advertising: fast mode 40ms, name "Triki GForce"
- MTU 247 (negotiated)
- **Wire protocol v1 (14B):** `[0x22][0x00]` + raw gyro6 + raw acc6 (legacy, default after boot)
- **Wire protocol v2 (19B):** `[0x22][0x01]` + seq16 + gyro6 + acc6 + vel16 + flags8 (switchable via `20 11 01`)
  - `seq` — sample counter (u16LE, continuous even during IDLE)
  - `vel` — VBT velocity snapshot (mm/s)
  - `flags` — bit0(active), bit1(rest), bit2(g-estimated), bit3(low-battery), bit4(g-forced)
- **RX commands:**
  - `20 10` — init/stream on
  - `20 11 01/00` — wire v2 on/off
  - `20 12` — FW info response (`22 03` + version + mode)
  - `20 15 01/00` — stream on/off
  - `20 16` — sleep now (SYSTEMOFF)
  - `20 17` — battery measurement response (`22 04` + mV)
  - `20 18 01/00` — **training mode** ON/OFF (v0.5.0)
- Connection parameters: 7.5–15ms (active), 150ms/latency 4 (IDLE)
- Single-try BLE send — no retry loop (drops counted in diagnostics)

### Power Management

- **IDLE-CONNECTED** (v0.4.0, HW inactivity via LSM6DSL INACT_EN):
  - After 6s rest: accelerometer → 12.5Hz low-power, gyroscope → power-down
  - HW auto-restores full ODR on motion detection (>250mg)
  - Activity detection via INT1/P0.09 (SLEEP_CHANGE) with GPIOTE TOGGLE
  - State synchronization via WAKE_UP_SRC readback
  - Connection parameters adjust automatically (7.5–15ms ↔ 150ms/lat4)
- **Training mode** (v0.5.0): `20 18 01` disables IDLE (keeps IMU at full 104Hz during sets)
  - `lsm6dsl_wake_force()` — explicit CTRL1/CTRL2 rewrite ensures gyro exit from power-down
  - Auto-exit on BLE disconnect (safety)
  - Full `vbt_reset()` on training start for clean calibration
- Sleep after 300s without BLE connection (SYSTEMOFF)
- Wake via BTN press (P0.25)

### Battery Measurement

- SAADC on AIN2 (P0.04), gain 1/6, internal reference 0.6V → FS 3.6V
- No voltage divider (AIN2 = direct Vbat, confirmed by HW measurement)
- Scale 1/1, offset 0mV, accuracy ~0.3%
- Software averaging of 4 samples
- Sampled every 1s from sleep timer
- Low-battery threshold: 2400mV (flag in wire v2 bit3)
- SAADC fault counting and zero-sum guard in diagnostics

### Diagnostics & Instrumentation

- Runtime counters (~40B BSS, always active): `imu_samples`, `ring_drops`, `seq_gaps`, `ble_drops`, `dt_faults`, `drdy_fallbacks`, `twim_faults`, `saadc_faults`, `drdy_mode`, `idle_state`, `idle_trans`, `idle_cp_fail`, `train_mode`
- Timing measurements (TIMER1 @1MHz): period min/avg/max, I2C burst, DSP total, gravity propagation, linear acc, velocity integration, BLE send
- RTT periodic print (all counters + timing, ~1s interval) — enabled in dev build (`_rtt.hex`)
- DSP profiling (compile-time) per-section microsecond timing
- Boot sequence logging: S1–S6 markers, register readback, mode changes

### Board Support

- **LED** — P0.28, active-low; `led_write()`, `led_blink(pat)`
- **Button** — P0.25, active-low with pull-up; debounced 27ms, edge-detected (counts press edges only)
- **3-press gesture** — resets idle sleep counter + 2x LED blink
- **Watchdog** — 12s reload, fed in main loop; SOS blink on fault → WDT reset
- **SYSTEMOFF** — deepest power-down, wake via BTN
- **Error handler** — logs code+line via RTT, SOS LED loop, then WDT reset

### Known Limitations (v0.5.0)

- Movement axis fixed in body frame (default X = barbell); host-side adaptation pending
- Gravity correction only in relative rest → gyro bias drift during long quasi-static holds (~2.4 deg/min)
- IDLE activity threshold 250mg (WK_THS floor at FS16g) may miss light movements — training mode available to override
- Velocity algorithm validated offline (harness 7/7) but not yet production-integrated at host level

---

## Programming via SWD

Hardware documentation, pinout, and SWD/JTAG programming instructions for the Triki kapsel are maintained by the community at:

**https://github.com/Piwencjusz/zabka-triki-hardware**

The repository covers:
- PCB pinout and SWD pad locations
- Debug probe wiring (Raspberry Pi Pico, J-Link, ST-Link)
- OpenOCD configuration and flashing commands
- Mechanical dimensions and case opening procedure

See `Hardware/README.md` in this repository for the GForce VBT-specific flashing procedure.