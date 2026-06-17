# Romi WPILib + PhotonVision Integration Research Notes

## Overview
Goal: Get Romi software running on 2026 versions of WPILib with PhotonVision, on recent Raspberry Pi and Orange Pi operating systems. Debug boot on simulator rather than real hardware. Port Romi web server to easier-to-maintain webserver. Simplify integration of WPILib parts with bootable image burning process.

## Key Resources Researched

### 1. WPILib Documentation - Imaging Your Romi
- **URL**: https://docs.wpilib.org/en/stable/docs/romi-robot/imaging-romi.html
- Romi has two boards: Raspberry Pi (high-level communication) and Romi 32U4 Control Board (low-level motor/sensor)
- Raspberry Pi firmware based on WPILibPi (formerly FRCVision)
- Download from: https://github.com/wpilibsuite/WPILibPi/releases (look for `-Romi` suffix)
- Web dashboard at http://10.0.0.2/ or http://wpilibpi.local/
- Default SSID: `WPILibPi-<number>`, passphrase: `WPILib2021!`
- 32U4 firmware updates via web dashboard

### 2. WPILibPi Repository (v2023.2.1)
- **URL**: https://github.com/wpilibsuite/WPILibPi
- Fork of RPi-Distro/pi-gen
- Uses pi-gen build system with stages (stage0-stage5)
- **Stage 4** = Romi-specific customizations:
  - `01-sys-tweaks/`: System tweaks including camera services, configServer
  - `02-net-tweaks/`: Network configuration (wpa_supplicant, dhcpcd)
  - Contains EXPORT_IMAGE flag
- **Key files in stage4/01-sys-tweaks/files/**:
  - `configServer_run` - launches `/usr/local/sbin/configServer`
  - `camera_run`, `camera_log_run`, `runCamera`, `runService`, `runInteractive`
  - `frc.json` - configuration
  - `picamera.conf`
- **configServer** located in `deps/tools/configServer/` - this is the Romi web dashboard

### 3. PhotonVision
- **URL**: https://photonvision.org / https://github.com/PhotonVision/photonvision
- Free, fast, easy-to-use computer vision for FRC
- First-class AprilTag support, camera calibration, ML inference, driver mode, multi-camera, multi-tag pose estimation
- Latest release: v2026.3.4 (Apr 2026)
- Built with Gradle (Java/C++), pnpm for web UI (Vue/TypeScript)
- Uses Javalin for web server

### 4. PhotonVision Image Modifier
- **URL**: https://github.com/PhotonVision/photon-image-modifier
- Scripts to install PhotonVision on various aarch64 images
- **install_pi.sh**: Mounts boot partition, runs install.sh, configures hostname, disables wifi, enables ssh
- **install.sh**: Main installer - downloads PhotonVision JAR, creates systemd service, installs NetworkManager, configures cpu governor
- Supports: Raspberry Pi, Orange Pi 5, Limelight, Luma P1, Rubik Pi 3, etc.
- Latest release: v2027.0.0 (Apr 2026)

### 5. WPILib Versions (allwpilib)
- **Latest**: v2027.0.0-alpha-6 (May 2026)
- **Stable 2026**: v2026.2.2 (Feb 2026), v2026.2.1 (Jan 2026), v2026.1.1 (Jan 2026)
- **URL**: https://github.com/wpilibsuite/allwpilib/tags

## Architecture Comparison

### WPILibPi (Current Romi)
- Build system: pi-gen (Debian-based, stage-based)
- Web dashboard: Custom C++ configServer (in deps/tools/configServer/)
- Services: camera_run, multiCameraServer, configServer
- Network: wpa_supplicant + dhcpcd, AP mode by default
- Read-only
- 32U4 firmware flashing via avrdude through web UI

### PhotonVision
- Installer: Bash scripts modifying existing OS images
- Web dashboard: Javalin-based (Java/Kotlin) with Vue/TypeScript frontend
- Services: photonvision systemd service (Java JAR)
- Network: NetworkManager (when --control-networking=yes)
- Supports multiple hardware platforms via image modifier scripts

## Key Technical Findings

1. **configServer** is the Romi-specific web dashboard (C++), separate from multiCameraServer
2. **WPILibPi hasn't been updated since 2023** (v2023.2.1) - uses older WPILib
3. **PhotonVision is actively maintained** with 2026/2027 releases
4. **photon-image-modifier** approach is more flexible - modifies existing OS images rather than building from scratch
5. **NetworkManager vs wpa_supplicant/dhcpcd** - PhotonVision uses NM, WPILibPi uses older stack
6. **Romi-specific features needed**:
   - 32U4 firmware flashing via web UI
   - WebSocket interface to desktop robot code
   - AP mode with specific SSID/password defaults
   - Romi dashboard pages (network config, firmware update)

## Simulator/Debugging Approach
- pi-gen supports QEMU mode (`USE_QEMU=1`) for emulated builds
- Can build qcow2 images for faster iteration
- Docker build available (`build-docker.sh`) for non-Debian hosts
- PhotonVision install.sh has `--test` flag for dry-run

## Web Server Migration Options
- **Current**: configServer (C++, custom)
- **PhotonVision**: Javalin (Java) + Vue/TypeScript
- **Options**:
  1. Port configServer to Javalin (align with PhotonVision stack)
  2. Use existing PhotonVision web UI and add Romi pages
  3. Use lighter framework (Python Flask/FastAPI, Go, Node.js)
  4. Keep C++ but modernize build/deployment

## Image Building Simplification
- **Current**: pi-gen (complex, Debian-host required, slow)
- **PhotonVision approach**: Take base OS image → modify with scripts → output image
- **Advantage**: Faster iteration, works on any Linux host, easier to customize
- Could use photon-image-modifier as base and add Romi-specific steps

## Questions for Clarification

1. **Target hardware**: Raspberry Pi 4/5 only, or also Orange Pi 5, Rubik Pi 3?
2. **Target OS base**: Raspberry Pi OS (Bookworm/Trixie), Ubuntu, Debian?
3. **Must-have Romi features**: 32U4 flashing, WebSocket bridge, AP mode, dashboard?
4. **Web server preference**: Align with PhotonVision (Javalin), or independent choice?
5. **Simulator priority**: QEMU emulation, or just VM-based testing?
6. **Release timeline**: 2026 season (v2026.x) or 2027 season (v2027.x alpha)?
7. **Backward compatibility**: Support existing Romi images or clean break?

## Proposed Plan Structure

1. **Research & Analysis** (current phase)
2. **Architecture Decision** - Choose base OS, web framework, build approach
3. **Prototype** - Minimal Romi + PhotonVision integration on simulator
4. **Feature Parity** - Implement Romi-specific features (32U4 flash, WebSocket, AP mode)
5. **Image Builder** - Create simplified image building pipeline
6. **Testing** - Simulator + hardware validation
7. **Documentation & Release**

---
*Research completed: 2026-06-17*