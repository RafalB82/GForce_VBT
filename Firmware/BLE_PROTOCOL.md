# GForce VBT — BLE Protocol Specification

> **License: CC0 1.0 Universal (Public Domain)**
>
> This specification is released under CC0. Anyone may implement a compatible
> application, tool, or service without any restrictions.

---

## 1. General

| Parameter | Value |
|---|---|
| GAP name | **"Triki GForce"** |
| Service | Nordic UART Service (NUS): `6e400001-b5a3-f393-e0a9-e50e24dcca9e` |
| RX characteristic | `6e400002-...` (host → device, write) |
| TX characteristic | `6e400003-...` (device → host, notify) |
| MTU | 247 (negotiated) |
| Connection params | 7.5–15ms, latency 0, supervision timeout 4s |

---

## 2. Wire protocol v1 (14B, default after boot)

```
Offset  Size  Field     Description
─────────────────────────────────────────────
0       1     0x22      Frame magic
1       1     0x00      Version indicator (v1)
2-7     6     gyro      Gyroscope XYZ (i16LE each): raw LSB, scale 16.4 LSB/dps
8-13    6     acc       Accelerometer XYZ (i16LE each): raw LSB, scale 2048 LSB/g
─────────────────────────────────────────────
Total: 14 bytes
```

Conversion to SI:
- `gyro_dps = raw / 16.4`
- `acc_mps2 = raw / 2048 * 9.80665`

---

## 3. Wire protocol v2 (19B, switchable)

Enable: RX `20 11 01` · Disable: RX `20 11 00`

```
Offset  Size  Field     Description
─────────────────────────────────────────────────────
0       1     0x22      Frame magic
1       1     0x01      Version indicator (v2)
2-3     2     seq       Sample sequence counter (u16LE, wraps at 65536)
4-9     6     gyro      Gyroscope XYZ (i16LE each)
10-15   6     acc       Accelerometer XYZ (i16LE each)
16-17   2     vel       Velocity (i16LE, mm/s, movement axis)
18      1     flags     Status flags (u8, see below)
─────────────────────────────────────────────────────
Total: 19 bytes
```

### Flags byte

| Bit | Name | Description |
|---|---|---|
| 0 | moving | Movement detected |
| 1 | rest | Device at rest |
| 2 | g-estimated | Gravity being estimated (calibration not yet stable) |
| 3 | low-battery | Battery voltage < 2400 mV |
| 4 | g-forced | Gravity calibration forced after ~5s without rest |
| 5-7 | — | Reserved |

---

## 4. Battery status frame (4B)

Trigger: RX `20 17`

```
Offset  Size  Field     Description
────────────────────────────────────────
0       1     0x22      Frame magic
1       1     0x04      Status: battery
2-3     2     mv        Battery voltage (u16LE, mV), 0 = measurement unavailable
────────────────────────────────────────
Total: 4 bytes
```

---

## 5. RX Commands (host → device)

All commands use prefix `0x20` in the first byte.

| Command | Payload | Description |
|---|---|---|
| `20 10 00 D0 07 68 00 03` | 8B | Init / stream on (stock compatible) |
| `20 11 01` | 3B | Enable wire v2 (19B frames) |
| `20 11 00` | 3B | Disable wire v2, revert to v1 (14B) |
| `20 12` | 2B | Request FW info → response `22 03 | maj | min | pat | mode` |
| `20 15 00` | 3B | Stream off |
| `20 15 01` | 3B | Stream on |
| `20 16` | 2B | Sleep now (SYSTEMOFF, wake with BTN) |
| `20 17` | 2B | Request battery measurement → response `22 04 | mv` |
| `20 18 01` | 3B | **Training mode ON** (disable IDLE-CONNECTED, reset VBT) |
| `20 18 00` | 3B | **Training mode OFF** (re-enable IDLE-CONNECTED) |

### Training mode (`20 18`) semantics

- **ON:** `lsm6dsl_inactivity_enable(false)`, force IMU to full ODR, `vbt_reset()` for clean calibration, connection params back to 7.5-15ms.
- **OFF:** re-enable hardware inactivity. HW will enter IDLE after SLEEP_DUR=6s of rest.
- **Auto-OFF:** training mode is automatically disabled on BLE disconnect (safety).

---

## 6. Pro variant (BHI385) wire format

```
Offset  Size  Field     Description
─────────────────────────────────────────────
0-3     4     seq       Sample counter (u32LE)
4-7     4     ts        Timestamp (u32LE, device ticks)
8-9     2     gx        Acceleration X (i16LE, milli-g)
10-11   2     gy        Acceleration Y (i16LE, milli-g)
12-13   2     gz        Acceleration Z (i16LE, milli-g)
─────────────────────────────────────────────
Total: 14 bytes, magic prefix AB 03
```

- Scales: `acc_mps2 = raw_milli_g / 1000 * 9.80665`
- 16 samples packetized into one BLE notification (224B data ≤ MTU 244)