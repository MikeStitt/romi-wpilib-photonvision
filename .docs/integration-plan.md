# Romi WPILib + PhotonVision Integration Plan

## Goals

1. **2026 Season (Primary)**: Produce bootable Romi images (RPi 4/5 + Orange Pi 5) running WPILib 2026.2.x + PhotonVision 2026.x with full Romi feature parity
2. **2027 Season (Secondary)**: Validate/update for WPILib 2027 alpha/beta when stable
3. **Simulator-First Development**: Debug boot and integration on QEMU before hardware
4. **Simplified Image Building**: Replace pi-gen with photon-image-modifier-style script pipeline

---

## Requirements Summary

| Aspect | Decision |
|--------|----------|
| **Target Hardware** | Raspberry Pi 4/5 + Orange Pi 5 |
| **Base OS** | Raspberry Pi OS Trixie (64-bit) for RPi; Armbian/Ubuntu for Orange Pi 5 |
| **WPILib Version** | 2026.2.x (stable) → 2027.x (alpha/beta later) |
| **PhotonVision Version** | 2026.3.x (current stable) |
| **Romi Features (v1)** | 32U4 firmware flash, WebSocket bridge, AP mode, full dashboard |
| **Web Framework** | Javalin + Vue/TypeScript (aligned with PhotonVision) |
| **Build Approach** | photon-image-modifier style: base OS image → modification scripts → output image |
| **Simulator** | QEMU (aarch64) for RPi; potentially Orange Pi 5 emulation |

---

## Architecture Overview

```
Base OS Images (RPi OS Trixie 64-bit, Orange Pi 5 Ubuntu/Armbian)
        │
        ▼
┌─────────────────────────────────────────────────────────────┐
│  Image Modifier Pipeline (bash/python scripts)               │
│  ├─ 01-base-setup: users, ssh, hostname, packages           │
│  ├─ 02-network: NetworkManager, AP mode, wpa_supplicant     │
│  ├─ 03-photonvision: Install PhotonVision JAR + systemd     │
│  ├─ 04-romi-core: 32U4 flash tool, avrdude, I2C tools       │
│  ├─ 05-romi-dashboard: Javalin+Vue web server (Romi pages)  │
│  ├─ 06-romi-integration: WebSocket bridge, configServer API │
│  └─ 07-finalize: cleanup, image shrink, checksums           │
└─────────────────────────────────────────────────────────────┘
        │
        ▼
Bootable .img/.xz files for each platform
```

---

## Phase 1: Foundation & Prototype (Weeks 1-3)

### 1.1 Environment Setup
- [ ] Set up build VM/container with QEMU aarch64 support
- [ ] Fork/clone photon-image-modifier as reference
- [ ] Download base images: RPi OS Trixie 64-bit lite, Orange Pi 5 Ubuntu 24.04
- [ ] Verify QEMU boot for both base images
- [ ] **Create `docs/architecture/` folder + initial ADR template**

### 1.2 Minimal PhotonVision Install on Simulator
- [ ] Adapt `photon-image-modifier/install.sh` for RPi OS Trixie base
- [ ] Test PhotonVision systemd service starts in QEMU
- [ ] Verify web UI accessible at http://localhost:5800 (forwarded)
- [ ] Document any Trixie-specific package differences
- [ ] **Write ADR-001: Base OS selection (RPi OS Trixie + Armbian for Orange Pi 5)**

### 1.3 Romi Core Package Installation
- [ ] Identify required packages: `avrdude`, `i2c-tools`, `python3-smbus`, `gpiod`
- [ ] Add 32U4 flash utility (from WPILibPi `configServer` deps or standalone)
- [ ] Test avrdude can detect 32U4 in QEMU (simulated or passthrough)
- [ ] **Write ADR-002: 32U4 flashing approach (avrdude vs custom)**
- [ ] **Document RPi ↔ 32U4 I2C communication path + pinout**

---

## Phase 2: Romi Dashboard & WebSocket Bridge (Weeks 4-6)

### 2.1 Javalin + Vue Project Setup
- [ ] Create `romi-dashboard/` Gradle project (mirroring PhotonVision structure)
- [ ] Configure Javalin server with Vue/TypeScript frontend (Vite)
- [ ] Set up shared Gradle build with PhotonVision version alignment
- [ ] Create systemd service template for romi-dashboard

### 2.2 Romi Dashboard Pages (Vue)
- [ ] **Network Page**: AP mode config, WiFi client config, hostname
- [ ] **Firmware Page**: 32U4 flash upload + progress (reuse PhotonVision file upload pattern)
- [ ] **System Page**: CPU temp, disk usage, service status, logs
- [ ] **Camera Page**: Link/embed PhotonVision camera stream
- [ ] **Settings Page**: Timezone, keyboard, overscan, SSH toggle

### 2.3 WebSocket Bridge to Desktop Robot Code
- [ ] **Leverage WPILib NT4 WebSocket support**: Use `ntcore` (Java) built-in WebSocket client/server instead of custom protocol
- [ ] Implement romi-dashboard as **NT4 WebSocket client** connecting to robot's NT4 server (desktop/sim)
- [ ] Expose NetworkTables topics via Javalin REST/WebSocket for dashboard UI consumption
- [ ] Reuse PhotonVision's NT4 integration patterns (they already bridge camera data to NT)
- [ ] Test with WPILib simulator (RobotSim) and real robot code
- [ ] Document: NT4 connection flow, topic naming conventions, reconnection logic

### 2.4 configServer API Compatibility
- [ ] Map existing configServer REST endpoints to new Javalin routes
- [ ] Ensure backward compatibility for any existing tooling
- [ ] Deprecate old C++ configServer

---

## Phase 3: Image Builder Pipeline (Weeks 7-9)

### 3.1 Script-Based Image Modifier
- [ ] Create `build-romi-image.sh` orchestrator
- [ ] Modular stage scripts (01-base, 02-network, 03-photonvision, 04-romi-core, 05-dashboard, 06-integration, 07-finalize)
- [ ] Each stage: idempotent, testable independently, logs to build log
- [ ] Support `--platform rpi|orangepi5` and `--wpilib-version 2026|2027`

### 3.2 Platform-Specific Handling
- [ ] **RPi**: RPi OS Trixie 64-bit, Pi-specific firmware/boot config
- [ ] **Orange Pi 5**: Armbian/Ubuntu 24.04, rockchip boot, different GPIO/I2C paths
- [ ] Common abstraction layer for platform differences

### 3.3 Network Configuration
- [ ] NetworkManager for both platforms (replace wpa_supplicant/dhcpcd)
- [ ] AP mode: SSID `WPILibPi-<serial>`, passphrase `WPILib2026!`
- [ ] Fallback to client mode when configured
- [ ] mDNS: `wpilibpi.local` and `romi-<serial>.local`

### 3.4 Image Output & Validation
- [ ] Output: `.img.xz` + `.img.xz.sha256` + `manifest.json` (versions, git SHA)
- [ ] QEMU boot test in CI pipeline
- [ ] Basic smoke test: SSH, PhotonVision UI, Romi dashboard, 32U4 detect

---

## Phase 4: Feature Parity & Polish (Weeks 10-12)

### 4.1 32U4 Firmware Flashing
- [ ] Integrate avrdude with proper Romi 32U4 fuse/settings
- [ ] Web UI: drag-drop .hex → flash → verify → reboot 32U4
- [ ] Bundle latest 32U4 firmware in image
- [ ] Support manual firmware upload for custom builds

### 4.2 Advanced Dashboard Features
- [ ] Real-time NetworkTables variable view/edit
- [ ] Camera calibration helper (link to PhotonVision calibration)
- [ ] System update checker (GitHub releases)
- [ ] Factory reset / safe mode

### 4.3 Documentation & Migration Guide
- [ ] **Architecture Decision Records (ADRs)** for each major choice (base OS, web framework, NT4 bridge, image builder)
- [ ] **System Architecture Doc**: Component diagram, communication paths (RPi ↔ 32U4 via I2C, Romi dashboard ↔ Robot via NT4 WebSocket, PhotonVision ↔ NT4, dashboard ↔ PhotonVision via HTTP)
- [ ] **WPILib Components Used**: `ntcore` (NT4), `wpilibj` (sim), `cameraserver` (multiCameraServer), `networktables` — versions pinned
- [ ] **Added Libraries/Tools**: Javalin, Vue/Vite, avrdude, photonvision JAR, NetworkManager, custom flash utility
- [ ] **Build Process Doc**: `build-romi-image.sh` usage, stage scripts, platform matrix, CI/CD pipeline, QEMU test matrix
- [ ] **Maintenance Guide**: How to update PhotonVision version, WPILib version, base OS, add new platform, debug boot failures
- [ ] **User-Facing Imaging Guide**: Flashing, first boot, AP mode, dashboard access, firmware update, WiFi client config
- [ ] **Developer Guide**: Building images locally, adding stages, testing in QEMU, contributing
- [ ] **Migration Guide**: From WPILibPi v2023 images (what changes, what's preserved, migration steps)

---

## Phase 5: 2027 Season Preparation (Ongoing)

### 5.1 WPILib 2027 Alpha/Beta Tracking
- [ ] Monitor allwpilib releases for 2027 alpha
- [ ] Update Gradle dependencies when 2027 MTA available
- [ ] Test NetworkTables 4.x compatibility
- [ ] Validate PhotonVision 2027 compatibility

### 5.2 CI/CD Pipeline
- [ ] GitHub Actions: build on schedule + on tag
- [ ] Multi-platform builds (RPi + Orange Pi 5)
- [ ] Release artifacts to GitHub Releases
- [ ] Automated QEMU smoke tests

---

## Technical Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| PhotonVision JAR conflicts with WPILib version | Medium | High | Pin PhotonVision version; test NT4 compatibility early |
| Javalin + Vue build complexity | Low | Medium | Copy PhotonVision's proven Gradle/Vite setup |
| Orange Pi 5 hardware differences (I2C, GPIO) | Medium | Medium | Abstract platform layer; test on hardware early |
| QEMU 32U4 passthrough unreliable | High | Medium | Develop flash logic on hardware; simulate in QEMU |
| NetworkManager vs wpa_supplicant migration issues | Low | High | Follow PhotonVision's proven NM config |
| pi-gen users expect exact image layout | Low | Medium | Document differences; provide migration path |

---

## Success Criteria

### 2026 Season Release (v2026.0.0)
- [ ] RPi 4/5 image boots in QEMU and on hardware
- [ ] Orange Pi 5 image boots in QEMU and on hardware
- [ ] PhotonVision UI accessible, camera streaming works
- [ ] Romi dashboard: network config, 32U4 flash, system info all functional
- [ ] WebSocket bridge passes NetworkTables data to/from simulator
- [ ] AP mode works out of box with default credentials
- [ ] Images pass `ninja check` equivalent (lint, test, build)
- [ ] Build time < 30 min per platform on CI

### 2027 Season Readiness
- [ ] Build script parameterized for WPILib version
- [ ] PhotonVision 2027.x compatible
- [ ] NetworkTables 4.x validated
- [ ] Documentation updated for 2027 changes

---

## Next Steps

1. **Immediate**: Set up build environment + QEMU + base images
2. **Week 1**: Get PhotonVision running on RPi OS Trixie in QEMU
3. **Week 2**: Add Romi core packages + 32U4 flash capability
4. **Week 3**: Start Javalin+Vue dashboard project

---

*Plan created: 2026-06-17 | Target 2026 release: ~12 weeks*