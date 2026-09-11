# GForce VBT — Pro firmware releases (BHI385 + nRF52840)

## Latest builds

| File | Variant | Description |
|---|---|---|
| `bhi385_quat_audit.uf2` | QUAT (default) | Quaternion mode, 400Hz, MTU 247 |
| `bhi385_quat_equiv_audit.uf2` | QUAT + Euler | Same + Euler equivalent output |

Both firmware builds are based on commit `28d9595` (branch `fpu-experiment-v2`, audit-fixes merged).

### SHA256

```
b067159824ab6a9b0420255a63c8f783f6011b8667af9bb1071e66b3ca27756a  bhi385_quat_audit.uf2
2341f791540514be649643080aa8635cbc2b140633b2cfbd281495152cd3122d  bhi385_quat_equiv_audit.uf2
```

### Flashing

1. Enter UF2 bootloader mode on the nRF52840 SuperMini (double-click reset)
2. A mass storage device will appear — drag and drop the `.uf2` file
3. The device reboots and starts advertising

### Features

- 400 Hz IMU streaming over BLE NUS
- Quaternion-based orientation (Bosch SensorAPI)
- Rotation to global frame
- 14B/sample wire format: seq4 + ts4 + gx + gy + gz (milli-g)
- MTU 247, packetizer 16 (16 samples per notification)
- Integrated WDT, BLE retry, atomic connection flags
- Optional Euler output (NaN risk at 360°)