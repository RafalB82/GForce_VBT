# Triki_G — Functional Documentation

**Version:** 0.1.0+11 · **Platform:** Android 8.0+ (API 26)

**GitHub:** [Triki_G](https://github.com/RafalB82/Triki_G)

---

## Overview

Triki_G is a Flutter Android application for Velocity-Based Training (VBT). It connects to a GForce VBT sensor (Kapsel or Pro) over Bluetooth Low Energy and provides real-time bar velocity feedback, automatic rep detection, and training analytics.

---

## 1. BLE / Connection

| # | Feature | Detail |
|---|---|---|
| 1.1 | BLE Scanning | Scan for Triki devices with configurable timeout (default 8s); auto-detection by advertising service UUID or name prefix `Triki` |
| 1.2 | Device Discovery | `TrikiScanDevice` model with name, MAC, RSSI, and `isTriki` flag |
| 1.3 | Auto-Connect | Automatic find-and-connect to the first Triki device; scan can abort on first discovery |
| 1.4 | Manual Pairing | `connectTo(TrikiScanDevice)` for connecting to a specific device from the scan list |
| 1.5 | NUS Service | Nordic UART Service UUIDs for RX/TX characteristics |
| 1.6 | BLE Permissions | Runtime requests for `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` (Android 12+), fallback to `ACCESS_FINE_LOCATION` (<12) |
| 1.7 | Streaming ON/OFF | Start/stop IMU data streaming at configured ODR |
| 1.8 | Wire v2 (default) | GForce protocol: 19-byte frames (seq, velocity, flags) vs legacy 14-byte; auto-enabled on connect |
| 1.9 | Firmware Info | Request FW version via `20 12` |
| 1.10 | Battery Monitoring | Periodic polling every 3 min via `20 17` (decodes `22 04` reply); low-battery flag at <2400 mV |
| 1.11 | Sleep Command | Immediate SYSTEMOFF via `20 14` |
| 1.12 | Packet Diagnostics | Real-time telemetry: packet size min/avg/max, measured sample rate, gap detection, loss estimation |
| 1.13 | Packet Reassembly | `PacketReassembler` for split packets across MTU boundaries |
| 1.14 | Nominal Timestamps | Timestamps built from `index / sampleFreq` (not wall clock) to avoid transport jitter |

---

## 2. IMU Pipeline — Rep Detection

Two parallel detector pipelines, switchable at runtime.

### 2A. PhaseDetector (G-Path Yaw-based, default)

| # | Feature | Detail |
|---|---|---|
| 2A.1 | Madgwick AHRS | Gyro+accel orientation filter, ported from MadgwickAHRS.c; outputs quaternion → gravity vector |
| 2A.2 | Yaw Extraction | Quaternion → Euler yaw angle (degrees) — angular axis of wrist rotation |
| 2A.3 | Lowpass Filter | 10 Hz on yaw angle before phase segmentation |
| 2A.4 | Phase FSM | IDLE → ECCENTRIC → TRANSITION → CONCENTRIC → IDLE (soft TRANSITION state) |
| 2A.5 | Phase Segmentation | Accumulated angle delta: `dirThrDeg=2.0`, `moveThrDeg=0.15`, `minAccumDeg=10.0` |
| 2A.6 | Phase Merging | `mergeAdjacentPhases` — extends same-direction adjacent phases |
| 2A.7 | Phase Pairing | `pairIntoReps` — pairs ECC+CONC phases with max 5s gap into one rep |
| 2A.8 | Calibration Mode | Auto-calibration at startSet: Madgwick warm-up + bias collection |
| 2A.9 | Noise Estimation | `_finalizeNoiseEstimate` — diagnostic RMS for movement quality |
| 2A.10 | Heuristic MCV | `heuristicMcv(ampDeg) = ampDeg × 0.0006` — approximate velocity from yaw amplitude |
| 2A.11 | Dry-Phase Feed | `debugFeedDy` / `debugFeedPhase` for offline testing |
| 2A.12 | Open-Rep Tracking | Accumulates E+C angle during an open rep, discards on timeout |

### 2B. VbtPipeline (PCA-based, legacy alternative)

| # | Feature | Detail |
|---|---|---|
| 2B.1 | Madgwick Orientation | Shared Madgwick filter with PhaseDetector |
| 2B.2 | Iterative Bias Removal | `pushCalib`: accumulates raw−g×G, computes bias from mean, with warm-up/convergence detection |
| 2B.3 | Body-Frame Projection | Converts acceleration to body frame via quaternion |
| 2B.4 | PCA Axis | Principal Component Analysis on body-frame acceleration (~200 samples) |
| 2B.5 | Gravity Axis Fallback | `useGravityAxis` — bypass PCA, use −g as movement axis |
| 2B.6 | Butterworth Filter | 2nd-order biquad lowpass (default 20 Hz) |
| 2B.7 | Segment Thresholding | Activity on `|acc| > peakG` (default 0.4 G), min duration, split gap handling |
| 2B.8 | Trapezoidal Integration | Per-segment velocity integration + linear detrend |
| 2B.9 | MCV Extraction | Mean Concentric Velocity per completed segment |
| 2B.10 | Segment Finalization | `finishSegment` + `_settlePending`: A/B mechanism for real reps vs unrack/rerack (threshold 0.6× median) |
| 2B.11 | Auto-Calibration Phase | Two-phase: 1) auto-calibrate (~200 samples/2s), 2) active detection |

### 2C. Legacy RepDetector (projected-axis)

| # | Feature | Detail |
|---|---|---|
| 2C.1 | Projected-Axis Detection | Gravity-projected acceleration, lowpass, velocity integration with per-rep reset |
| 2C.2 | Gyro Watchdog | Rotation guard: gyro threshold detection to interrupt bad reps |
| 2C.3 | Event Types | `firstMove`, `repStart`, `repEnd`, `gyroAlert`, `none` |
| 2C.4 | Rep Model | `Rep`: n, tStart, durationS, MCV, MPV, peak, lossPct, isBest, irregularity |

---

## 3. Rep Validation (Set-Level)

### 3A. RepValidator V1

| # | Feature | Detail |
|---|---|---|
| 3A.1 | Feature Extraction | Amplitude class (normal/asymmetric/low/veryLow), symmetry analysis, duration bounds, DTW-C shape similarity (evidence only, never veto) |
| 3A.2 | Ablation Configuration | `RepValidatorConfig` with per-family toggle (amplitude, symmetry, duration, DTW) |
| 3A.3 | Diagnostic Output | Returns `amplitudeClass`, `symmetryRatio`, `durationEvidence`, `dtwEvidence` |

### 3B. RepValidatorAdapter

| # | Feature | Detail |
|---|---|---|
| 3B.1 | Candidate Extraction | `extractCandidates` — converts PhaseDetector phases into E→C candidate pairs |
| 3B.2 | Series Validation | `validateSeries` — runs RepValidator, produces ValidatedReps (raw/validated/hard/suspicious) |
| 3B.3 | Region Evidence | Tempo-invariant rep-region evidence (first/middle/last position) |
| 3B.4 | DTW Computation | Optional DTW-C shape matching across candidates |

### 3C. Set-Level Observers (5 diagnostic modules)

| # | Module | Rule |
|---|---|---|
| 3C.1 | SETUP | First candidate + extremely low relAmp (<0.30 vs series median) |
| 3C.2 | RACK | Last candidate + extremely low relAmp (<0.30 vs series median) |
| 3C.3 | PHASEGAP | Internal E→C gap >0s AND reduced gyro RMS in the gap (<0.50 relative) |
| 3C.4 | LOW_AMP | Whole-series median amplitude <50° AND >60% candidates below threshold |
| 3C.5 | FRAG (experimental) | Inter-rep gap >2.5s (temporal discontinuity) |

### 3D. SetLevelReport

| # | Feature | Detail |
|---|---|---|
| 3D.1 | Aggregated Report | All 5 observer results + basic series facts (min/median amplitude, symmetry, duration, gaps) |
| 3D.2 | Facts vs Flags | Clear separation: `facts` (observations) vs `flags` (triggers) |
| 3D.3 | Empty-Series Safe | Produces valid report even with 0 candidates |

### 3E. LinearPath (Diagnostic)

| # | Feature | Detail |
|---|---|---|
| 3E.1 | PCA 1D → Integration | PCA axis → acceleration integration (trapezoidal) → highpass 0.5 Hz → zero-cross segmentation |
| 3E.2 | Per-Rep MPV | Linear detrend → positive mean = Mean Propulsive Velocity |
| 3E.3 | Batch Computation | Operates on entire collected set after completion (offline-style) |
| 3E.4 | Diagnostic Only | Never replaces phase-detector counts |

### 3F. VelocityQuality

| # | Feature | Detail |
|---|---|---|
| 3F.1 | Velocity Irregularity | G-Path formula: `Σ|residual| / Σ|velocity|` |
| 3F.2 | Linear Regression | OLS slope + intercept of velocity vs index |
| 3F.3 | Quality Threshold | Smooth reps ~0.02–0.06, jagged ~0.6–0.7, cutoff ~0.15 |

---

## 4. GPE (Perceived Effort) Model

| # | Feature | Detail |
|---|---|---|
| 4.1 | Classification | `classifyGpe(mcv)` → ratio = mcv / baseMcv → level |
| 4.2 | Levels | `base`, `tooEasy` (<0.25), `light` (0.25–0.9), `optimal` (0.9–1.1), `heavy` (1.1–1.75), `overkill` (>1.75) |
| 4.3 | Base MCV | Median of all collected MCVs (dynamic, resets per set) |
| 4.4 | Rep History | `_repHistory: List<GpeRec>` with per-rep MCV, ratio, level |
| 4.5 | Static Thresholds | `gpeOptimalMin=0.9`, `gpeOptimalMax=1.1`, `gpeTooEasy=0.25`, `gpeLight=0.7`, `gpeHeavy=1.3`, `gpeOverkill=1.75` |

---

## 5. Session Management & Data Services

### 5A. Session Control (TrikiController)

| # | Feature | Detail |
|---|---|---|
| 5A.1 | Start Set | Resets pipeline, starts streaming, begins auto-calibration |
| 5A.2 | Stop Set | Stops streaming, finalizes segment, collects SetResult, saves CSV, starts rest timer |
| 5A.3 | Auto-Stop (Velocity Loss) | Stops when MCV drops N% below best rep |
| 5A.4 | Auto-Stop (Capsule Flat) | Detects capsule lying flat (|g_z| > 0.7 for 150 samples) — only outside active detection |
| 5A.5 | Clear Session | Resets session set buffer |

### 5B. SetResult Model

| # | Feature | Detail |
|---|---|---|
| 5B.1 | Per-Set Data | setIndex, exerciseId, hevyTemplateId, mcvs[], irregularities[], baseMcv, velocityLossPct, autoStopFired, startedAt, durationSec, loadKg |
| 5B.2 | Validation Metadata | repCountRaw, repCountValidated, hardRejectedCount, suspiciousCount |
| 5B.3 | Set-Level Report | setLevelFacts, setLevelFlags, setLevelReps |
| 5B.4 | Region Evidence | RegionEvidence.toJson() |
| 5B.5 | Linear Path Diagnostic | LinearPathResult.toJson() |
| 5B.6 | JSON Serialization | `toJson()`, `toJsonString()` |
| 5B.7 | SessionResult | Complete workout: createdAt, odrHz, appVersion, List<SetResult>, full `toJson()` |

### 5C. HistoryStore

| # | Feature | Detail |
|---|---|---|
| 5C.1 | Persistent Append | Append-only JSON file in Documents (`triki_history.json`) |
| 5C.2 | Atomic Writes | Temp+rename pattern for crash safety |
| 5C.3 | Load All | `load()` → `List<HistoryEntry>` with timestamps |

### 5D. WorkoutStats

| # | Feature | Detail |
|---|---|---|
| 5D.1 | Per-Exercise Stats | Series count, rep count, weighted MCV mean, min, max, best MCV, avg velocity loss |
| 5D.2 | Time Range Filter | `StatsRange`: day, week, month, all |
| 5D.3 | Pure Computation | No state mutation, no persistence writes |

### 5E. LoadVelocity (M5 — 1RM Estimation)

| # | Feature | Detail |
|---|---|---|
| 5E.1 | Linear Regression | MCV = a×load + b from history pairs |
| 5E.2 | 1RM Estimation | `estimate1Rm(minVelocity=0.3)` — predicts load at minimal velocity |
| 5E.3 | Fit Quality | r², n points, slope + intercept |

### 5F. Export

| # | Format | Detail |
|---|---|---|
| 5F.1 | CSV (raw IMU) | Full sample recording with metadata header |
| 5F.2 | CSV (per-rep) | Per-rep MCV + irregularity |
| 5F.3 | JSON (session) | Full SessionResult — all sets, all reps, all metadata |

---

## 6. Screens (10)

| # | Screen | Description |
|---|---|---|
| 6.1 | **HomeShell** | Bottom navigation (M3) with IndexedStack: Trening, Plan, Historia, Postępy, Ustawienia |
| 6.2 | **TrainingScreen** | Live workout: connection status, exercise selector, live MCV, Start/Stop, rep counter, velocity gauge, ROM gauge, rep velocity chart |
| 6.3 | **HistoryScreen** | Past workouts from HistoryStore, grouped by day |
| 6.4 | **StatsScreen** | Per-exercise MCV trend, load-velocity regression, 1RM; swipable PageView per exercise; time range toggle |
| 6.5 | **PlanScreen** | Training plan — exercise group selection, exercise list filtered by group, persisted in AppPrefs |
| 6.6 | **SettingsScreen** | Settings hub: device info, navigation to Device / Debug / Stats |
| 6.7 | **DeviceScreen** | BLE scan & pairing — scan list, Triki-only filter, connect/disconnect |
| 6.8 | **DebugScreen** | Debug recording: exercise name, target reps, behavior, raw CSV recording/export, app log viewer |
| 6.9 | **OnboardingScreen** | First-run wizard: BLE permissions, capsule connection |
| 6.10 | **SetSummarySheet** | Modal bottom sheet after each set: rep list with GPE levels, performance summary |

---

## 7. Widgets (5)

| # | Widget | Description |
|---|---|---|
| 7.1 | **VelocityGauge** | Live velocity bar in GPE-relative scale; color-coded by effort level |
| 7.2 | **RepVelocityChart** | GPE bar chart: each bar = one rep, height = GPE ratio, color = effort; optimal zone shading [0.9, 1.1] |
| 7.3 | **RomGauge** | Arc ROM indicator for rotational exercises: position dot, E/C labels |
| 7.4 | **LinearRangeBar** | Vertical ROM bar for translational exercises: position dot + progress coloring |
| 7.5 | **RestTimer** | Rest period countdown: progress fraction, mm:ss display |

---

## 8. Infrastructure Services

| # | Service | Detail |
|---|---|---|
| 8.1 | **AppLogger** | Global in-memory ring buffer (500 entries) + file append (`triki_log.txt`) |
| 8.2 | **AppPrefs** | JSON preferences (`triki_prefs.json`): onboarding flag, plan selection |
| 8.3 | **MadgwickFilter** | Pure Madgwick AHRS: `updateIMU(gx,gy,gz, ax,ay,az)` → quaternion (q0–q3); ported from MadgwickAHRS.c |
| 8.4 | **TrikiProtocol** | BLE wire protocol: packet encoding/decoding (14B v1 + 19B v2), command building, scale factors, PacketReassembler |
| 8.5 | **TrikiController** | Central `ChangeNotifier` (1221 lines) orchestrating all services |

---

## 9. Supported Sensors

| Sensor | Connection | ODR | Notes |
|---|---|---|---|
| **GForce VBT Kapsel** (nRF52810 + LSM6DSL) | BLE NUS | 104 Hz (fixed) | GForce protocol, wire v2, `20 15 00` for stop |
| **GForce VBT Pro** (nRF52840 + BHI385) | BLE NUS | 400 Hz | QUAT 14B, MTU 247 |

---

## 10. Known Limitations (v0.1.0+11)

- FRAG detection is experimental (not production): takes candidates from full recording, not per-set
- LOW_AMP threshold scaled for rotational movement — false-positives on `pickupFromFloor`
- Some workspace files may be missing from version control (`.bak`, debug fixtures)

---

## Downloads

| File | Checksum (SHA256) |
|---|---|
| APK: `app-release/triki-g-v0.1.0.apk` | `16c9c747aa073053b8e39d86cbaa01bbe6e2887530ba89f85deb41f8ef922b81` |