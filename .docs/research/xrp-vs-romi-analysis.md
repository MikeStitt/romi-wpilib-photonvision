# XRP vs Romi Communication Protocol Analysis

**Date**: 2026-06-18 (Updated after verifying WPILib docs)  
**Purpose**: Evaluate pros/cons of switching Romi from WebSocket (NT4) to XRP protocol (UDP) for Pi-to-PC communication

---

## Executive Summary

**Recommendation: Keep current WebSocket/NT4 approach for Romi** — but for **different reasons** than initially analyzed.

**Critical Correction**: Both Romi and XRP use the **same fundamental architecture**: **robot code runs on the PC**, board acts as remote I/O. My initial analysis incorrectly claimed Romi runs robot code on the Pi. The official WPILib docs confirm robot code runs on the development computer (PC) for Romi.

---

## Architecture Comparison (Corrected)

### Romi Architecture (WebSocket/NT4 + HALSimWS)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DESKTOP PC (Development Computer)                │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Robot Code (Java/C++/Python) — runs in "Simulation" mode   │  │
│  │  • TimedRobot / CommandRobot                                 │  │
│  │  • HALSim WebSocket Client (connects to Pi)                 │  │
│  │  • NT4 Client (for SmartDashboard/Shuffleboard)             │  │
│  └──────────────────────────┬──────────────────────────────────┘  │
│                             │ HAL WebSocket (port 3300)           │
│                             │ + NT4 WebSocket (port 5810)         │
└─────────────────────────────┼─────────────────────────────────────┘
                              │
                ┌─────────────▼─────────────┐
                │    RASPBERRY PI           │
                │  ┌─────────────────────┐  │
                │  │  wpilibws-romi      │  │  ← Node.js WebSocket server
                │  │  (HALSimWS Server)  │  │
                │  └──────────┬──────────┘  │
                │           │ I2C           │
                │  ┌────────▼────────┐     │
                │  │ 32U4 Control    │     │
                │  │ Board           │     │
                │  │ (motors, encod- │     │
                │  │  ers, IMU)      │     │
                │  └────────────────┘     │
                └─────────────────────────┘
                              │
                              ▼
                ┌─────────────────────────┐
                │   PhotonVision (separate)│
                │   NT4 Publisher (5800)   │
                └─────────────────────────┘
```

**Key characteristics (Corrected)**:
- Robot code runs **on the PC** (development computer) — NOT on Pi
- Pi runs `wpilibws-romi` — a **WebSocket server** that bridges HAL simulation to 32U4 via I2C
- PhotonVision runs **locally on Pi** (camera → AprilTag → NT4 publisher on port 5800)
- Romi Dashboard (our addition) runs on Pi (port 5801) — provides network config, 32U4 flash, etc.
- NT4 bridge connects Pi to desktop for dashboard/simulation
- 32U4 communication via **I2C** (local, not networked)

### XRP Architecture (UDP/HALSim)

```
┌─────────────────────────────────────────────────────────────────────┐
│                        DESKTOP PC (Development Computer)            │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Robot Code (Simulation)                                    │  │
│  │  • HALSim WebSocket Client (for sim GUI)                   │  │
│  │  • XRP UDP Client (HALSimXRP)                               │  │
│  │  • Sends motor/servo/DIO commands                           │  │
│  │  • Receives encoder/gyro/analog feedback                    │  │
│  └──────────────────────────┬──────────────────────────────────┘  │
│                             │ UDP Port 3540 (Binary)              │
└─────────────────────────────┼─────────────────────────────────────┘
                              │
                ┌─────────────▼─────────────┐
                │      XRP BOARD            │
                │  ┌─────────────────────┐  │
                │  │ Microcontroller     │  │  (RP2040/ESP32)
                │  │ • Motor drivers (4x)│  │
                │  │ • Servos (2x)       │  │
                │  │ • DIO, Analog,      │  │
                │  │   Encoder, Gyro     │  │
                │  │ • WiFi / USB        │  │
                │  └─────────────────────┘  │
                └───────────────────────────┘
```

**Key characteristics**:
- Robot code runs **on PC** (simulation mode)
- XRP board is **dumb I/O** — no local intelligence
- Communication: **UDP binary protocol** (port 3540)
- HALSim on PC translates HAL calls ↔ UDP packets
- Designed for **simulation/education**, not competition deployment
- **No PhotonVision equivalent** — vision runs on PC if needed

---

## Architecture Similarity: Romi ≈ XRP

| Aspect | Romi | XRP |
|--------|------|-----|
| **Robot code runs on** | **PC (desktop)** | **PC (desktop)** |
| **Board role** | **Remote I/O bridge** (Pi + 32U4) | **Remote I/O** (XRP board) |
| **HAL communication** | **HALSim WebSocket** (port 3300) | **HALSim UDP** (port 3540) |
| **Protocol** | JSON WebSocket | Binary UDP |
| **Vision** | PhotonVision on Pi (local) | On PC (if needed) |
| **Dashboard** | Romi Dashboard on Pi (our addition) | WPILib Sim GUI on PC |
| **Competition deploy** | **No** (simulation only) | **No** (simulation only) |

**Both are simulation-only architectures**. Neither supports deployed autonomous robot code on the embedded board.

---

## Google AI Claims Evaluation (Re-evaluated)

### Claim 1: "XRP uses UDP packets (port 3540) rather than continuous WS-socket"

**VERDICT**: ✅ **TRUE**  
**Evidence**: HALSimXRP README confirms UDP port 3540, binary protocol. HALSimWS uses WebSocket port 3300.

### Claim 2: "XRP code executes directly on your computer (as simulation) and pushes real-time instructions to the physical robot"

**VERDICT**: ✅ **TRUE** — **AND THIS ALSO APPLIES TO ROMI**  
**Evidence**: HALSimXRPClient runs on PC, robot code runs in simulation. For Romi, `wpilibws-romi` on Pi receives HAL WebSocket from PC and drives 32U4 via I2C.

### Claim 3: "XRP uses WPILib's HAL to route commands... No code deployment to XRP board"

**VERDICT**: ✅ **TRUE** — **AND THIS ALSO APPLIES TO ROMI**  
**Evidence**: Romi robot code is NOT deployed to Pi. It runs on PC in simulation mode. Pi runs `wpilibws-romi` bridge.

### Claim 4: "You can point HALSIMXRP_HOST to Pi's IP to mimic XRP architecture"

**VERDICT**: ⚠️ **MISLEADING**  
**Analysis**: You *can* set `HALSIMXRP_HOST=192.168.x.x` to point to a Pi running an XRP-protocol daemon. But this requires:
- Writing a custom XRP-protocol daemon on Pi (doesn't exist)
- Implementing all XRP device types (Motor, Servo, DIO, Analog, Gyro, Encoder)
- Mapping Pi's actual hardware (I2C 32U4, GPIO, CSI camera) to XRP device model
- Robot code still runs on PC, not Pi

This is **not** "mimicking XRP architecture" for Romi — it's building a new XRP-compatible hardware abstraction on Pi while keeping robot code on PC.

---

## Protocol Comparison

### XRP Protocol (UDP Binary)

| Aspect | Detail |
|--------|--------|
| **Transport** | UDP (unreliable, low latency) |
| **Port** | 3540 (default) |
| **Format** | Binary (sequence + control + tagged data) |
| **Direction** | Bidirectional (PC → XRP: outputs; XRP → PC: sensors) |
| **Rate** | ~200-500 Hz (simulation periodic) |
| **Devices** | Motor, Servo, DIO, Analog, Gyro, Encoder |
| **Discovery** | None (hardcoded host/port via env vars) |
| **Security** | None (trusted network) |
| **Reliability** | Application-level (sequence numbers) |

### Romi HALSimWS Protocol (WebSocket + JSON)

| Aspect | Detail |
|--------|--------|
| **Transport** | WebSocket over TCP (reliable, ordered) |
| **Port** | 3300 (HALSimWS), 5810 (NT4) |
| **Format** | JSON text frames (human readable) |
| **Direction** | Bidirectional pub/sub |
| **Rate** | Event-driven + periodic (HAL: 50-200Hz typical) |
| **Devices** | Full HAL (PWM, DIO, AI, Encoder, Gyro, etc.) |
| **Discovery** | mDNS + manual IP config |
| **Security** | None (trusted network) |
| **Reliability** | TCP + HALSimWS acks |

### Romi NT4 Protocol (Separate WebSocket)

| Aspect | Detail |
|--------|--------|
| **Transport** | WebSocket over TCP |
| **Port** | 5810 (NT4) |
| **Format** | MessagePack (binary) over WebSocket |
| **Purpose** | NetworkTables 4.x pub/sub for SmartDashboard/Shuffleboard |

---

## Corrected Pros/Cons of Switching Romi to XRP Protocol

### PROS of XRP Protocol for Romi

| Pro | Details |
|-----|---------|
| **Lower latency** | UDP binary protocol has less overhead than WebSocket/JSON |
| **Simpler wire protocol** | Binary tagged format is compact |
| **HAL-native** | Direct HALSimXRP integration on PC side |
| **Deterministic timing** | Sequence numbers + periodic simulation loop |
| **No JSON parsing** | Less CPU on PC side (negligible) |

### CONS of XRP Protocol for Romi

| Con | Details | Severity |
|-----|---------|----------|
| **No NT4/SmartDashboard integration** | XRP protocol doesn't speak NT4. Would lose SmartDashboard/Shuffleboard/Glass. | 🔴 **Blocker** |
| **No PhotonVision integration** | XRP has no camera/ML pipeline. PhotonVision publishes NT4; would need custom bridge. | 🔴 **Blocker** |
| **No Romi Dashboard** | Our custom dashboard (network config, 32U4 flash, system monitoring) runs on Pi. XRP model has no equivalent. | 🔴 **Blocker** |
| **32U4 I2C not modeled** | XRP protocol has no I2C device type. 32U4 communication would need custom tags. | 🟠 **Major** |
| **UDP unreliability on WiFi** | Packet loss = lost motor commands/encoder data. TCP WebSocket handles this. | 🟡 **Moderate** |
| **No discovery** | Hardcoded host/port vs mDNS/NT4 announcement | 🟡 **Moderate** |
| **Custom daemon required** | Must implement XRP protocol daemon on Pi from scratch | 🟡 **Moderate** |
| **No WPILib tooling support** | VS Code "Simulate Robot Code" expects XRP hardware, not Pi | 🟡 **Moderate** |
| **Different HAL provider set** | XRP only implements subset (Motor, Servo, DIO, Analog, Gyro, Encoder). Romi needs PWM, etc. | 🟡 **Moderate** |

---

## What Would Actually Change If We Switch to XRP Protocol

1. **PC side**: Use `HALSIMXRP_HOST=pi-ip` instead of `HALSIMWS_HOST=pi-ip`
2. **Pi side**: Replace `wpilibws-romi` with custom XRP-protocol UDP daemon
3. **Lose**: SmartDashboard/Shuffleboard/Glass, PhotonVision NT4, Romi Dashboard
4. **Gain**: Marginally lower latency UDP transport (no practical benefit on WiFi)

---

## Why Keep Current Architecture

### 1. NT4 Ecosystem is Critical
- SmartDashboard, Shuffleboard, Glass, AdvantageScope all use NT4
- PhotonVision publishes AprilTag poses to NT4
- Romi Dashboard consumes NT4 for real-time variable view
- XRP protocol has **zero NT4 support**

### 2. PhotonVision Runs Locally on Pi
- Camera → AprilTag → NT4 publisher on port 5800
- This is a **core value prop** of the Romi image
- XRP model expects vision on PC (via USB camera)

### 3. Romi Dashboard is Unique Value
- Network config (AP mode, WiFi client, static IP)
- 32U4 firmware flash via web UI
- System monitoring (CPU, disk, services)
- No equivalent in XRP ecosystem

### 4. HALSimWS Already Works
- `wpilibws-romi` is mature, maintained by WPILib
- Handles full HAL device set (PWM, DIO, AI, Encoder, Gyro, etc.)
- XRP protocol only implements subset

### 5. No Practical Benefit
- WiFi latency dominates; UDP vs WebSocket difference is negligible
- 32U4 I2C is local, not networked
- Current architecture is proven (used by thousands of teams)

---

## When XRP Protocol Would Make Sense

- Building a **new** robot platform that intentionally uses PC-simulated architecture
- Ultra-low-latency motor control over **reliable wired link** (not WiFi)
- Educational simulation where PC runs physics engine + vision
- When you don't need NT4/SmartDashboard/PhotonVision

---

## For This Project

**Stick with current plan** (unchanged recommendation, but corrected reasoning):

- HALSimWS (port 3300) for PC ↔ Pi HAL communication
- NT4 WebSocket (port 5810) for Pi ↔ PC NetworkTables (SmartDashboard, PhotonVision, Romi Dashboard)
- PhotonVision publishes to NT4 locally on Pi
- Romi Dashboard (Javalin on Pi, port 5801) consumes NT4 + provides REST/WS for UI
- 32U4 via I2C (local, bridged by wpilibws-romi)
- Competition deployment: **not supported by either architecture** (both are simulation-only)

---

## References

1. **XRP Protocol**: https://github.com/wpilibsuite/allwpilib/tree/main/simulation/halsim_xrp
2. **HALSim WebSocket API**: https://github.com/wpilibsuite/allwpilib/blob/main/simulation/halsim_ws_core/doc/hardware_ws_api.md
3. **Romi Programming**: https://docs.wpilib.org/en/stable/docs/romi-robot/programming-romi.html
4. **Romi Imaging**: https://docs.wpilib.org/en/stable/docs/romi-robot/imaging-romi.html
5. **NetworkTables 4**: https://docs.wpilib.org/en/stable/docs/software/networktables/networktables-4.html
6. **HALSimXRP README**: https://raw.githubusercontent.com/wpilibsuite/allwpilib/main/simulation/halsim_xrp/README.md
7. **wpilibws-romi**: https://github.com/wpilibsuite/WPILibPi/tree/main/stage5/01-sys-tweaks/files

---

*Analysis updated 2026-06-18 after verifying WPILib documentation. Initial analysis had fundamental architecture error (claimed Romi runs robot code on Pi; actually runs on PC like XRP).*